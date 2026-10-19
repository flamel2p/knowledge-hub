---
title: Valkey - Redis to Valkey Migration Runbook
aliases: [Redis to Valkey migration, migrate Redis to Valkey, Valkey cutover, REPLICAOF migration, BullMQ Valkey migration, n8n Valkey]
type: deep-dive
domain: databases
tags: [domain/databases, type/deep-dive, topic/valkey, topic/redis, topic/migration]
status: draft
created: 2026-10-19
updated: 2026-10-19
version_checked: "Valkey 9.1.x · official migration guide (Redis OSS ≤ 7.2 file-compatible; 7.4+ not) — 2026-10"
parent: "[[Valkey]]"
related: ["[[Redis]]", "[[Redis - Persistence & High Availability]]", "[[Message Queues - BullMQ in Production]]", "[[n8n - Queue Mode & Scaling]]", "[[Coolify]]", "[[Docker]]"]
---

# Valkey - Redis to Valkey Migration Runbook

> [!info] Deep dive of [[Valkey]]

> [!abstract] TL;DR
> - **Source is Redis OSS ≤ 7.2**: migrate with **replication** (`REPLICAOF` from Valkey to Redis, then promote) for near-zero downtime, or with an **RDB file copy** during a short write freeze.
> - **Source is Redis 7.4+ / 8.x**: the data files are **not readable** by Valkey, so use **drain-and-cutover** (queues), **cold start** (caches) or a **logical copy**.
> - **Before you start**: check which commands and modules you actually use.
> - **Rollback**: keep the old Redis untouched until rollback is no longer needed. Valkey 8+ data files can't go back to Redis 7.2.

## Concept

### Pick the migration path
| Source | Data you hold | Path | Downtime |
|---|---|---|---|
| Redis OSS 2.x–7.2.x | Anything | **A. Replication** (`REPLICAOF`) → promote | Seconds (client switch) |
| Redis OSS ≤ 7.2.x | Anything, can pause writes | **B. RDB/AOF file copy** | Minutes (stop → copy → start) |
| Any version (incl. 7.4/8.x) | BullMQ / n8n queues | **C. Drain-and-cutover**: stop producers, finish jobs, switch | Minutes, in a maintenance window |
| Any version | Pure cache (rebuildable) | **D. Cold cache**: point apps at empty Valkey | None, but expect a cache-miss spike |
| Redis 7.4+/8.x | Durable data (sessions, counters, leaderboards) | **E. Logical copy**: `SCAN` + type-aware read/write, or dual-write | Depends on volume |

### Pre-flight inventory
| Check | Command / source | Why |
|---|---|---|
| Source version | `redis-cli INFO server \| grep redis_version` | Decides file compatibility |
| Commands in use | `INFO commandstats` | Find Redis-7.4+/8-only commands (hash-field TTL syntax differs, vector sets, JSON) |
| Modules | `MODULE LIST` | RedisJSON/Search/Bloom → `valkey-bundle` modules or app changes |
| Persistence | `CONFIG GET appendonly save` | Replicate the same durability on Valkey |
| Memory | `INFO memory` (`used_memory_peak`) | Size `maxmemory`. Replication needs headroom |
| Eviction policy | `CONFIG GET maxmemory-policy` | Queues must be `noeviction` |
| ACL / auth | `ACL LIST` | Recreate users. `requirepass` vs ACL users |
| Topology | Standalone / Sentinel / Cluster | Cluster needs per-shard migration |
| Clients | ioredis, node-redis, redis-py, BullMQ, n8n versions | Confirm no `redis_version` gating in libraries |

## How It Works
```mermaid
flowchart TD
  START[Inventory source] --> V{Redis OSS ≤ 7.2?}
  V -- yes --> R{Can pause writes for minutes?}
  R -- no --> A[A. Valkey REPLICAOF redis → sync → promote → switch clients]
  R -- yes --> B[B. BGSAVE → copy dump.rdb → start Valkey]
  V -- no (7.4+/8) --> T{What data?}
  T -- queues --> C[C. Drain BullMQ/n8n → switch → resume]
  T -- cache --> D[D. Cold start on empty Valkey]
  T -- durable keys --> E[E. Logical copy / dual-write]
  A & B & C & D & E --> VERIFY[Verify: key counts, samples, job flow, error rates]
  VERIFY --> KEEP[Keep old Redis read-only 7–14 days for rollback]
```
- **Replication**: Valkey accepts the Redis 7.2 replication protocol, so a Valkey replica performs a full sync from the Redis primary (RDB transfer + replication stream). `REPLICAOF NO ONE` then makes Valkey a writable primary.
- **Why 7.4+ fails**: Redis 7.4+ bumped its RDB format after the fork. Valkey reads and writes only formats compatible with Redis OSS 7.2, and can't load newer files or replication streams.

## Practical Usage

### A. Replication cutover (Redis ≤ 7.2 → Valkey 9.1)
```bash
# 1. Start Valkey (same Docker network as Redis), same maxmemory/policy/AOF settings, ACL users created
docker exec -it valkey valkey-cli CONFIG SET masterauth "$REDIS_PASSWORD"   # if source uses requirepass
docker exec -it valkey valkey-cli REPLICAOF redis 6379

# 2. Wait for full sync
docker exec -it valkey valkey-cli INFO replication | grep -E "master_link_status|master_sync_in_progress"
# master_link_status:up and master_sync_in_progress:0

# 3. Compare
redis-cli -h redis DBSIZE; valkey-cli -h valkey DBSIZE
redis-cli -h redis --scan --pattern 'bull:*' | head -1000 | sort > a.txt   # sample keyspaces match
valkey-cli -h valkey --scan --pattern 'bull:*' | head -1000 | sort > b.txt; diff a.txt b.txt | head

# 4. Cutover (minimise the write gap)
#    a) stop or pause writers (scale API/workers to 0 or enable maintenance mode)
#    b) confirm replication offset caught up (master_repl_offset == slave_repl_offset)
docker exec -it valkey valkey-cli REPLICAOF NO ONE
#    c) switch REDIS_URL → redis://app:***@valkey:6379 in Coolify env, redeploy API, workers, n8n
#    d) keep the old Redis untouched (no clients, volume preserved) for the rollback window
```

### B. File copy (short freeze)
```bash
redis-cli -h redis BGSAVE && redis-cli -h redis LASTSAVE        # wait until LASTSAVE timestamp changes
docker stop app-workers app-api n8n-worker n8n-main             # freeze writers
redis-cli -h redis SAVE                                         # final consistent snapshot
docker cp redis:/data/dump.rdb ./dump.rdb
docker cp ./dump.rdb valkey:/data/dump.rdb && docker restart valkey
valkey-cli -h valkey DBSIZE                                     # compare with source
```
- When AOF is enabled on the target, start from the RDB first, then enable `appendonly yes` with `CONFIG SET` (it rewrites the AOF from memory). Otherwise an empty AOF can take precedence over the copied RDB.

### C. Drain-and-cutover for BullMQ / n8n (works from any Redis version)
1. Announce a 10–20 min window. For n8n, pause production triggers or set webhooks to return 503 temporarily, so providers retry later ([[REST API - Webhooks & Idempotency]]).
2. Stop producers (API enqueue paths, n8n main) and keep workers running until `waiting + active + delayed` reaches 0 for every queue.
   ```ts
   const counts = await queue.getJobCounts("waiting", "active", "delayed", "prioritized", "waiting-children");
   ```
3. Note the delayed/repeatable schedules. Job schedulers are re-created at boot if you use `upsertJobScheduler` idempotently ([[Message Queues - BullMQ in Production]]).
4. Switch `REDIS_URL` / n8n `QUEUE_BULL_REDIS_HOST` to Valkey and start workers, then producers ([[n8n - Queue Mode & Scaling]]).
5. Failed-job history stays in the old Redis. Export it first if you need it for audits.

### E. Logical copy (Redis 7.4+/8 durable keys)
```ts
// Type-aware copy of a key pattern; safe for strings, hashes, sets, zsets, lists with TTL preserved
for await (const key of src.scanIterator({ MATCH: "sess:*", COUNT: 1000 })) {
  const [type, ttl] = await Promise.all([src.type(key), src.pTTL(key)]);
  if (type === "hash") await dst.hSet(key, await src.hGetAll(key));
  else if (type === "string") await dst.set(key, (await src.get(key))!);
  else if (type === "zset") { const m = await src.zRangeWithScores(key, 0, -1); if (m.length) await dst.zAdd(key, m); }
  else if (type === "set") { const m = await src.sMembers(key); if (m.length) await dst.sAdd(key, m); }
  else if (type === "list") { const m = await src.lRange(key, 0, -1); if (m.length) await dst.rPush(key, m); }
  if (ttl > 0) await dst.pExpire(key, ttl);
}
// Not point-in-time: keys written during the copy need dual-write or a final freeze + delta pass
```
- **Don't rely on `DUMP`/`RESTORE`** across 7.4+/8 → Valkey. The serialized payload carries an RDB version that Valkey may reject.

### Verification checklist (after any path)
- `DBSIZE` and a sampled key diff per prefix (`sess:*`, `bull:*`, `rl:*`).
- App error rates, BullMQ throughput (completed/min), n8n executions succeeding, and `evicted_keys = 0`.
- `INFO persistence`: AOF on, last rewrite OK. Then reboot-test once, since the data must survive a container restart.
- Monitoring/GUI tools connect (some check `redis_version`; Valkey reports a compatibility version field).

## Patterns & Anti-patterns
| Pattern | When | Anti-pattern to avoid |
|---|---|---|
| Inventory commands/modules first | Every migration | Discovering a RedisJSON dependency after cutover |
| Replication + short write pause | Redis ≤ 7.2 with live traffic | Copying `dump.rdb` while writes continue (data loss window) |
| Drain queues instead of copying them | BullMQ / n8n | Copying in-flight `active` jobs (stalls/duplicates after switch) |
| Keep the old instance for 7–14 days | Rollback safety | Deleting the Redis volume right after cutover |
| Same `maxmemory-policy noeviction` + AOF on the queue instance | Queues | Default container config (no AOF) on the new Valkey |
| Change one variable at a time | Production | Migrating Redis → Valkey and upgrading BullMQ/n8n in the same window |

## Performance & Trade-offs
- **Replication full sync** needs extra memory on the source (fork + copy-on-write) and bandwidth for the RDB. Run it off-peak on hosts with < 2× headroom.
- **Drain-and-cutover** is the simplest and safest for queues but needs a maintenance window. Webhook providers retry, so short 503 windows are tolerable when receivers are idempotent.
- **Logical copy** is slow for millions of keys (round trips). Pipeline/batch it, or limit it to keys that matter, and let caches rebuild.
- **Performance after cutover**: enable `io-threads` (2–4 on small VPSs) to benefit from Valkey's threading. Benchmark with `valkey-benchmark` against your real command mix.

## Tips & Reminders
> [!tip]
> - **Rehearse on a staging copy first**: restore last night's RDB into a scratch Valkey, then run the app's smoke tests.
> - **Rename env vars** only after the switch is stable. Keep `REDIS_URL` as the variable name to avoid config churn.
> - **Update runbooks**: `valkey-cli` instead of `redis-cli` (symlinks exist in most images), `valkey.conf`, and the new image name in Coolify compose.
> - **Clients**: ioredis/node-redis/BullMQ work as-is. Pin the BullMQ version during the migration window.
> - **In ZP's stack**:
>   - **n8n + BullMQ on Redis 7.2**: use path A, with a ~1-minute pause during promote and redeploy.
>   - **Redis 8 projects**: use path C for queues plus path D for caches, then keep Redis 8 stopped for two weeks before removal.

## Version Notes
| Item | Note |
|---|---|
| Redis OSS ≤ 7.2.x → Valkey 7.2+ | Officially supported via replication or RDB/AOF files |
| Redis CE 7.4+ / 8.x → Valkey | **No file-based path** (format diverged). Use logical approaches |
| Valkey 8.0+ | RDB format bump. Rollback to Redis 7.2 by file isn't possible, so roll back by switching clients to the preserved Redis |
| Valkey 9.x | Hash-field TTL commands exist in both Redis 7.4+ and Valkey 9, but test syntax/behaviour parity before migrating code that uses them |

> [!warning] Unverified — check before relying on this
> Some third-party guides claim Redis 7.4+ → Valkey file migration works, but the official Valkey migration guide says it doesn't. Follow https://valkey.io/topics/migration/ and test on a copy.

## Critical Issues & Gotchas
> [!danger] Losing queued jobs during cutover
> Switching `REDIS_URL` while jobs are `waiting`/`delayed` in the old Redis silently strands them: no error, just orders never processed or WhatsApp replies never sent. Either replicate (path A, with writers paused before promote) or drain completely (path C). Verify counts reach zero before switching.

> [!danger] New instance without durability settings
> A fresh `valkey/valkey` container runs with no password, no AOF and no `maxmemory` (it grows until the OOM killer hits). Cutting over to an ephemeral Valkey means the next container restart wipes the queue. Mount a config with ACL, AOF, `maxmemory-policy noeviction` and a persistent volume, and restart-test before cutover ([[Valkey]]).

> [!warning] Gotchas
> - `REPLICAOF` from a password-protected source needs `masterauth` (and `masteruser` for ACL users) on Valkey.
> - Sentinel setups: migrate replicas and Sentinels deliberately. Mixed Redis/Valkey Sentinel groups work only within the 7.2-compatible protocol, so don't run them long-term.
> - Cluster mode: migrate shard by shard, or rebuild the cluster and use path C/D. Slot ownership must match.
> - Keyspace notifications (`notify-keyspace-events`) and `CONFIG` values aren't replicated. Copy them explicitly.
> - Clients with hard-coded `redis` hostnames in n8n credentials or env files get missed. Grep configs before the window.

## Related
- [[Valkey]]
- [[Redis]] — source system, licensing context
- [[Redis - Persistence & High Availability]] — RDB/AOF, replication mechanics
- [[Message Queues - BullMQ in Production]] — draining, schedulers, `noeviction`
- [[n8n - Queue Mode & Scaling]] — queue settings and env vars
- [[Coolify]] — env switch and redeploy

## References
- Valkey migration guide: https://valkey.io/topics/migration/
- Percona Valkey migration guide: https://docs.percona.com/valkey/migration-guide/migration.html
- Valkey replication: https://valkey.io/topics/replication/
- BullMQ going to production: https://docs.bullmq.io/guide/going-to-production
- n8n queue mode: https://docs.n8n.io/hosting/scaling/queue-mode/
