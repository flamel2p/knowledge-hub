---
title: Redis - Data Structures & Patterns
aliases: [Redis data types, Redis patterns, Redis rate limiting, Redis distributed lock, Redis Streams, Redis caching patterns]
type: deep-dive
domain: databases
tags: [domain/databases, type/deep-dive, topic/redis, topic/caching]
status: draft
created: 2026-10-05
updated: 2026-10-05
version_checked: "Redis 8.10 — 2026-10"
parent: "[[Redis]]"
related: ["[[Redis - Persistence & High Availability]]", "[[n8n]]", "[[WebSocket]]", "[[Big O Notation]]"]
---

# Redis - Data Structures & Patterns

> [!info] Deep dive of [[Redis]]

> [!abstract] TL;DR
> Redis is a server of **data structures**, not just a key-value cache: strings, hashes, lists, sets, sorted sets, streams, bitmaps, HyperLogLog, geo, and (since 8.0) JSON, vector sets and query engine indexes built in. Pick the structure whose **O(1)/O(log n) commands** match your access pattern, keep each operation **atomic** (single command, `MULTI`, or Lua/Functions), and always set **TTLs and size bounds**. Remember: **one slow O(n) command on a big key blocks every client**, because command execution is single-threaded.

## Concept
| Type | Commands (complexity) | Typical patterns |
|---|---|---|
| **String** | `GET/SET` O(1), `INCR` O(1), `SET NX PX` | Cache values, counters, locks, idempotency keys |
| **Hash** | `HGET/HSET` O(1), `HINCRBY`, `HEXPIRE` (7.4+, per-field TTL) | Objects/sessions, per-field counters |
| **List** | `LPUSH/RPOP` O(1), `BLMOVE`, `LRANGE` O(S+N) | Simple queues, recent items (capped with `LTRIM`) |
| **Set** | `SADD/SISMEMBER` O(1), `SINTER` O(N·M) | Tags, unique members, dedupe |
| **Sorted set** | `ZADD/ZRANK` O(log n), `ZRANGE BYSCORE` O(log n + M) | Leaderboards, sliding windows, delayed jobs, priority queues |
| **Stream** | `XADD` O(1), `XREADGROUP`, `XACK`, `XAUTOCLAIM` | Event log, durable queues with consumer groups, chat history |
| **Bitmap / Bitfield** | `SETBIT/BITCOUNT` | Daily active flags, feature flags per user ID |
| **HyperLogLog** | `PFADD/PFCOUNT` (~0.81% error, 12 KB) | Unique visitor counts |
| **Geo** | `GEOADD/GEOSEARCH` | Nearby outlets / riders |
| **JSON** (8.0 built-in) | `JSON.SET/GET` with paths | Nested documents with partial updates |
| **Vector set** (8.x) / Query Engine | `VADD/VSIM`, `FT.SEARCH` | Semantic cache, similarity search |
| **Pub/Sub** | `PUBLISH/SUBSCRIBE`, sharded `SPUBLISH` | Fire-and-forget fan-out (no persistence) |

## How It Works
- **Single-threaded command execution** means each command is atomic and isolated. I/O threads only parallelise network reads/writes and parsing.
- **Encodings**: small hashes/sets/zsets use compact `listpack`/`intset`, then convert to hashtable/skiplist past thresholds (`hash-max-listpack-entries` 128 by default). Many small keys inside hashes can save lots of memory.
- **Expiration**: lazy (on access) + active sampling. Expired keys don't vanish exactly on time. Under memory pressure, `maxmemory-policy` (e.g. `allkeys-lru`, `volatile-lfu`, `noeviction`) decides what gets evicted.
- **Atomic multi-step logic**: `MULTI/EXEC` (queued, no conditional logic), `WATCH` (optimistic), or **Lua scripts / Functions** (run atomically on the server).
- **Cluster**: keys hash to 16,384 slots. Multi-key ops need keys in the same slot, so use hash tags `{tenant:42}:cart`.

## Practical Usage

### Cache-aside with stampede protection
```ts
async function getProduct(id: string) {
  const key = `product:v3:${id}`;
  const hit = await redis.get(key);
  if (hit) return JSON.parse(hit);
  const lock = await redis.set(`${key}:lock`, "1", { NX: true, PX: 5000 });   // single rebuilder
  if (!lock) { await sleep(50); return getProduct(id); }                     // or serve stale copy
  const fresh = await db.product.find(id);
  await redis.set(key, JSON.stringify(fresh), { EX: 600 + Math.floor(Math.random() * 60) });  // TTL jitter
  await redis.del(`${key}:lock`);
  return fresh;
}
```

### Rate limiting: sliding window with sorted set (atomic via Lua)
```lua
-- KEYS[1]=rl:{user}  ARGV: now_ms, window_ms, limit, member
redis.call('ZREMRANGEBYSCORE', KEYS[1], 0, ARGV[1] - ARGV[2])
local n = redis.call('ZCARD', KEYS[1])
if n >= tonumber(ARGV[3]) then return 0 end
redis.call('ZADD', KEYS[1], ARGV[1], ARGV[4])
redis.call('PEXPIRE', KEYS[1], ARGV[2])
return 1
```
- A cheaper alternative is a fixed window with `INCR` + `EXPIRE` (bursty at window edges), or GCRA (token bucket) in one Lua script.

### Distributed lock (single instance) done right
```ts
const token = crypto.randomUUID();
const ok = await redis.set("lock:invoice:2026-10", token, { NX: true, PX: 30_000 });
if (ok) {
  try { await generateInvoices(); }
  finally {   // release only if we still own it
    await redis.eval(`if redis.call('get',KEYS[1])==ARGV[1] then return redis.call('del',KEYS[1]) else return 0 end`,
                     { keys: ["lock:invoice:2026-10"], arguments: [token] });
  }
}
```
- Locks with TTL aren't safe against GC pauses or long work: use **fencing tokens** (incrementing number checked by the resource), or a DB row lock for correctness-critical sections.

### Idempotency for webhooks (WhatsApp/Stripe)
```ts
const first = await redis.set(`idem:wa:${msg.id}`, "1", { NX: true, EX: 86_400 });
if (!first) return res.status(200).end();     // duplicate delivery → ack and skip
```

### Streams as a durable work queue
```bash
XADD events MAXLEN ~ 100000 * type order.paid order_id 9812
XGROUP CREATE events billing $ MKSTREAM
XREADGROUP GROUP billing worker-1 COUNT 10 BLOCK 5000 STREAMS events >
XACK events billing 1727000000000-0
XAUTOCLAIM events billing worker-2 60000 0-0 COUNT 10     # reclaim stuck messages after 60s
```

### Leaderboard / delayed jobs with sorted sets
```bash
ZADD delayed {run_at_epoch_ms} job:123          # schedule
ZRANGE delayed -inf {now} BYSCORE LIMIT 0 100   # due jobs (move atomically via Lua or ZPOPMIN-with-check)
```

## Patterns & Anti-patterns
| Pattern | When | Anti-pattern to avoid |
|---|---|---|
| Namespaced, versioned keys `app:entity:v3:{id}` | Always | Unprefixed keys → collisions, no safe bulk invalidation |
| TTL on every cache key + jitter | Caching | Keys without TTL growing until OOM/eviction chaos |
| `SCAN`/`HSCAN` cursors | Iterating keys/fields | `KEYS *` / `SMEMBERS` on huge sets in production |
| Streams + consumer groups | Durable queues needing acks/replay | Pub/Sub for jobs (messages lost when no subscriber) |
| Lua/Functions for read-modify-write | Rate limits, inventory reservations | GET → compute in app → SET (race conditions) |
| Bounded collections (`LTRIM`, `XADD MAXLEN ~`) | Logs, histories, feeds | Unbounded lists/streams |
| Hash tags for related keys in Cluster | Multi-key ops | Cross-slot `MGET`/transactions (CROSSSLOT error) |

## Performance & Trade-offs
- Sub-millisecond ops, but **network RTT dominates**. Pipeline batches (`multi`/pipeline in clients) cut round trips by 10–100×.
- Big keys (> 1 MB values, > 10k-element collections) cause latency spikes on read, delete and migration. Use `UNLINK` instead of `DEL` for async free. Find them with `redis-cli --bigkeys` / `--memkeys`.
- Memory: every key has ~50–70 bytes of overhead. Pack small objects into hashes. Pick `maxmemory` + an eviction policy deliberately (cache: `allkeys-lfu`. Queue/state: `noeviction`).
- Lua scripts block the server while running. Keep them short (no loops over big data).

## Tips & Reminders
> [!tip]
> - Separate Redis **roles**: cache (evictable, `allkeys-lfu`) vs queue/state (`noeviction` + persistence). Mixing them lets the cache evict your BullMQ jobs.
> - Use `OBJECT ENCODING key` and `MEMORY USAGE key` to understand memory.
> - Monitor `SLOWLOG GET`, `INFO commandstats` and `latency doctor`.
> - **In ZP's stack**: [[n8n]] queue mode and BullMQ use Redis lists/zsets/streams. Give them a dedicated DB/instance with `noeviction` and AOF. For WhatsApp webhooks, use `SET NX` idempotency keys plus a sorted-set rate limiter per `wa_id`. [[WebSocket]] fan-out via Pub/Sub, with Streams for replay.

## Version Notes
| Version | Change |
|---|---|
| 5.0 | Streams |
| 6.2 | `GETDEL`, `GETEX`, `ZRANGE` unified syntax |
| 7.0 | Functions, sharded Pub/Sub, `listpack` everywhere |
| 7.4 | Hash field expiration (`HEXPIRE`…). License → RSALv2/SSPL |
| 8.0 (2025) | JSON, Query Engine, TimeSeries, probabilistic types built in. Vector sets (beta). `HGETEX/HSETEX/HGETDEL`. AGPLv3 option |
| 8.2–8.10 | Performance, stream/bitmap ops, vector set improvements |

## Critical Issues & Gotchas
> [!danger] Lua sandbox & Redis exposure CVEs
> **CVE-2025-49844 "RediShell"** (CVSS 10, Oct 2025): a use-after-free in the Lua engine let authenticated users escape the sandbox to RCE, and it affected ~13 years of versions. With ~60k Redis instances exposed without auth, that meant mass exploitation risk. Patch, require `requirepass`/ACLs, and restrict `EVAL`/`FUNCTION` via ACLs for app users. Never expose 6379 publicly.

> [!warning] Gotchas
> - `KEYS`, `FLUSHALL`, `DEBUG` should be disabled or renamed for app ACL users.
> - Expiring a key resets only on write commands that set a TTL. `SET` without `KEEPTTL` **removes** an existing TTL.
> - Pub/Sub messages are lost if subscribers disconnect. Use Streams when you need delivery guarantees.
> - Redlock across multiple masters is debated (Kleppmann vs antirez). Don't rely on it for correctness-critical mutual exclusion.
> - Client-side timeouts + retries on non-idempotent commands (`INCR`, `LPUSH`) can double-apply.

## Related
- [[Redis]]
- [[Redis - Persistence & High Availability]] — durability for queue/state roles
- [[n8n]] — queue mode on Redis
- [[WebSocket]] — Pub/Sub fan-out
- [[Big O Notation]] — command complexity
- [[PostgreSQL - Transactions & MVCC]] — when DB locks/queues are the better choice

## References
- Redis data types: https://redis.io/docs/latest/develop/data-types/
- Commands (with complexity): https://redis.io/docs/latest/commands/
- Distributed locks: https://redis.io/docs/latest/develop/use/patterns/distributed-locks/
- Kleppmann, "How to do distributed locking": https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html
- RediShell CVE-2025-49844 (Wiz): https://www.wiz.io/blog/wiz-research-redis-rce-cve-2025-49844
