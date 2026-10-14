---
title: Redis - Interview Questions
aliases: [Redis interview, cache avalanche, cache breakdown, cache penetration, Redis 面试题]
type: reference
domain: databases
tags: [domain/databases, type/reference, topic/redis, topic/interview]
status: draft
created: 2026-10-13
updated: 2026-10-14
version_checked: "Redis 8.10 — 2026-10"
parent: "[[Redis]]"
related: ["[[Redis - Persistence & High Availability]]", "[[Redis - Data Structures & Patterns]]", "[[System Design - Interview Questions]]", "[[Backend Interview Questions]]"]
---

# Redis - Interview Questions

> [!info] Reference for [[Redis]] · index: [[Backend Interview Questions]]

> [!abstract] TL;DR
> Q1: **deploying Redis and keeping it fast and durable when things go wrong**. Classify the workload first (cache vs queue/state), then deploy, choose HA by scale (single + backups → Sentinel → Cluster), and handle abnormal cases: crash, failover, OOM, Redis down, cache avalanche/breakdown/penetration, lost jobs.

## Q1. How do you deploy a Redis instance, and what patterns ensure performance and durability in abnormal situations?

**Grading**: do you **classify the workload first** (cache vs queue/state), know the persistence and HA knobs, and design the *application* to survive Redis failing?

### Step 1: decide the role, because config differs
| Role | Persistence | `maxmemory-policy` | Losing data means |
|---|---|---|---|
| Cache | Optional (RDB for warm restarts) | `allkeys-lru` / `allkeys-lfu` | Cache miss → DB load spike |
| Queue / job state / sessions / rate limits | **AOF `everysec` + RDB** | **`noeviction`** | Lost jobs, logged-out users |

Run them as **separate instances** when both exist. One eviction policy can't serve both.

### Step 2: deploy
- Docker/[[Coolify]] with a config file, persistent volume, **no published port** (internal Docker network only), ACL users or `requirepass`, `protected-mode yes`, healthcheck, `maxmemory` at ~70–75% of the container limit (leave room for fork copy-on-write during `BGSAVE`/AOF rewrite). Host: `vm.overcommit_memory=1`, disable Transparent Huge Pages.
- Full compose and config: [[Redis - Persistence & High Availability]].

```conf
appendonly yes
appendfsync everysec
aof-use-rdb-preamble yes
save 900 1 300 100
maxmemory 2gb
maxmemory-policy noeviction
min-replicas-to-write 1
min-replicas-max-lag 10
```

### Step 3: HA by scale
| Setup | When | Abnormal-case behaviour |
|---|---|---|
| Single node + AOF + off-host backups (RDB → R2 hourly) | Single KVM, small team | Crash: ≤ ~1 s of writes lost. Host loss: restore from backup (RPO = backup interval) |
| Primary + replica(s) + **3 Sentinels** (quorum 2, on separate hosts) | Need automatic failover | Failover in seconds. Clients must be Sentinel-aware (ioredis `sentinels`, Lettuce) |
| **Redis Cluster** (≥ 3 primaries + replicas) | Dataset or throughput beyond one node | Hash-slot sharding. Multi-key ops need `{hashtag}`s |
| Managed (Upstash, ElastiCache, Redis Cloud) | Don't want to run it | Vendor lock-in and per-request/GB pricing |

### Step 4: patterns for abnormal situations
| Situation | Pattern |
|---|---|
| Process crash / restart | AOF `everysec` (≤ ~1 s loss) or `always` for critical writes. `WAITAOF` for per-command durability |
| Primary dies | Sentinel failover. `min-replicas-to-write` stops an isolated primary accepting writes (limits split-brain loss). Asynchronous replication can still lose the last acknowledged writes, so say so |
| Memory full | `maxmemory` + right policy. Alert at 80%. Find big keys (`redis-cli --bigkeys`, `MEMORY USAGE`) |
| Redis down (cache role) | **Circuit breaker + fallback to DB**, with a **bulkhead** (rate-limited DB path) so the database isn't stampeded. Short client timeouts (50–200 ms) |
| **Cache avalanche** (many keys expire at once) | TTL **jitter** (`ttl + random(0, 10%)`), staggered warm-up |
| **Cache breakdown** (hot key expires, thundering herd) | Single-flight mutex (`SET lock NX PX`) or logical expiry with background refresh |
| **Cache penetration** (queries for keys that don't exist) | Cache nulls with short TTL, Bloom filter in front, input validation |
| Lost queue jobs | Redis is **not** the system of record. Write the business fact to Postgres first (Transactional Outbox), make consumers idempotent, reconcile on restart |
| Latency spikes | Pipelining/`MULTI`, no `KEYS *` (use `SCAN`), avoid big keys/O(N) commands, `SLOWLOG`, connection pooling |

```typescript
export async function getCached<T>(key: string, ttlSec: number, load: () => Promise<T>): Promise<T> {
  try {
    const hit = await redis.get(key);
    if (hit !== null) return JSON.parse(hit) as T;

    const lock = await redis.set(`lock:${key}`, '1', 'PX', 5_000, 'NX');
    if (!lock) {
      await sleep(50 + Math.random() * 50);
      const retry = await redis.get(key);
      if (retry !== null) return JSON.parse(retry) as T;
    }

    const value = await load();
    const jitter = Math.floor(ttlSec * 0.1 * Math.random());
    await redis.set(key, JSON.stringify(value), 'EX', ttlSec + jitter);
    if (lock) await redis.del(`lock:${key}`);
    return value;
  } catch (err) {
    if (!isRedisUnavailable(err)) throw err;
    return dbBulkhead.run(load);
  }
}
```

- **Close with**: "I monitor `used_memory`, `evicted_keys`, `connected_clients`, replication lag and `aof_last_bgrewrite_status`, and I test restores, not just backups."
- **Licensing aside** (if asked about alternatives): Redis 8 is tri-licensed including AGPLv3. **Valkey** (Linux Foundation fork) is the BSD drop-in many clouds moved to. See [[Redis]].
- **Trap**: "Redis is in-memory so it isn't durable" (AOF exists), or the opposite: treating Redis with async replication as a durable system of record for money.
- Deep dive: [[Redis - Persistence & High Availability]] · [[Redis - Data Structures & Patterns]] · [[n8n - Queue Mode & Scaling]]

## Related
- [[Redis]] — overview
- [[Redis - Persistence & High Availability]] — full config, compose, backups
- [[Redis - Data Structures & Patterns]] — locks, rate limits, Lua
- [[n8n - Queue Mode & Scaling]] — the queue-role Redis in ZP's stack
- [[System Design - Interview Questions]] — Redis seat holds in the ticketing design
- [[Backend Interview Questions]] — interview index

## References
- Redis persistence: https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/
- Redis Sentinel: https://redis.io/docs/latest/operate/oss_and_stack/management/sentinel/
- Redis eviction policies: https://redis.io/docs/latest/develop/reference/eviction/
