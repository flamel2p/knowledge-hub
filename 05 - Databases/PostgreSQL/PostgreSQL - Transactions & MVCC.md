---
title: PostgreSQL - Transactions & MVCC
aliases: [MVCC, Isolation levels, VACUUM, Autovacuum, Bloat, XID wraparound, Row locking, Serializable]
type: deep-dive
domain: databases
tags: [domain/databases, type/deep-dive, topic/postgresql, topic/transactions]
status: draft
created: 2026-10-05
updated: 2026-10-05
version_checked: "PostgreSQL 18 — 2026-10"
parent: "[[PostgreSQL]]"
related: ["[[PostgreSQL - Indexing]]", "[[PostgreSQL - Query Planning & EXPLAIN]]", "[[Supabase]]", "[[Redis]]"]
---

# PostgreSQL - Transactions & MVCC

> [!info] Deep dive of [[PostgreSQL]]

> [!abstract] TL;DR
> PostgreSQL implements isolation with **Multi-Version Concurrency Control**: every UPDATE/DELETE creates or expires a **row version** tagged with transaction IDs, and each transaction reads from a **snapshot**, so readers never block writers and vice versa. The cost: dead row versions pile up until **VACUUM** reclaims them, and 32-bit transaction IDs must be **frozen** before wraparound. Remember: **long-running or idle-in-transaction sessions are the root of most bloat, lock and wraparound incidents.** Keep transactions short.

## Concept
- **ACID**: Atomicity (all or nothing, via WAL), Consistency (constraints), Isolation (MVCC + locks), Durability (WAL fsync at commit).
- **Tuple header fields**: `xmin` (inserting XID), `xmax` (deleting/locking XID), `ctid` (physical location), infomask bits (committed/aborted hints, frozen).
- **Snapshot**: `(xmin, xmax, xip[])`, i.e. which transactions were in progress. A tuple is visible if its inserter committed before the snapshot and its deleter didn't.
- **Isolation levels**:

| Level | Snapshot taken | Prevents | Still possible | Notes |
|---|---|---|---|---|
| READ COMMITTED (default) | Per **statement** | Dirty reads | Non-repeatable reads, phantoms, lost updates (read-modify-write) | UPDATE re-checks the row's latest version (EvalPlanQual) |
| REPEATABLE READ | Per **transaction** (first query) | + non-repeatable reads, phantoms | Write skew | Serialization failure `40001` on concurrent update of the same row |
| SERIALIZABLE | Per transaction + **SSI** predicate tracking | All anomalies incl. write skew | — | Must retry on `40001`. Some false positives |
| READ UNCOMMITTED | Same as READ COMMITTED in PG | — | — | Postgres never shows dirty reads |

## How It Works

```mermaid
sequenceDiagram
  participant T1 as Tx 101 (UPDATE)
  participant H as Heap page
  participant T2 as Tx 102 (SELECT, snapshot before 101 commits)
  T1->>H: old tuple xmax=101; new tuple xmin=101
  T2->>H: read → old tuple visible (101 in progress), new invisible
  T1->>H: COMMIT
  T2->>H: (READ COMMITTED next statement) → new tuple visible
  Note over H: old tuple is now dead once no snapshot needs it
  H->>H: VACUUM: remove dead tuples, update visibility map + FSM, freeze old xmins
```

- **Dead tuples** stay until VACUUM proves no running snapshot can see them. That horizon is held back by the **oldest running transaction**, replication slots (`catalog_xmin`/`xmin`), prepared transactions and `hot_standby_feedback` replicas.
- **Autovacuum** triggers per table when dead tuples > `autovacuum_vacuum_threshold + scale_factor × reltuples` (default 50 + 20%). For big tables 20% is far too late, so tune per table.
- **Freezing**: rows older than `vacuum_freeze_min_age` get marked frozen (visible to all). `autovacuum_freeze_max_age` (200M default) forces an anti-wraparound vacuum. At ~2^31 transactions of age, Postgres stops accepting writes.
- **Locks**: row locks (`FOR UPDATE`, `FOR NO KEY UPDATE`, `FOR SHARE`, `FOR KEY SHARE`) are stored in tuple headers (+ MultiXacts for shared locks). Table locks range from `ACCESS SHARE` (SELECT) to `ACCESS EXCLUSIVE` (most `ALTER TABLE`, `DROP`). DDL waits behind long transactions **and blocks everyone queued behind it**.
- **Deadlocks** are detected after `deadlock_timeout` (1 s), and one transaction is aborted (`40P01`).

## Practical Usage

### Lost update vs atomic update
```sql
-- ❌ Read-modify-write in app code (two requests both read stock=5, both write 4)
SELECT stock FROM products WHERE id = $1;      -- app computes stock - 1
UPDATE products SET stock = $2 WHERE id = $1;

-- ✅ Atomic statement with guard
UPDATE products SET stock = stock - 1 WHERE id = $1 AND stock > 0 RETURNING stock;

-- ✅ Or explicit row lock when logic is complex
BEGIN;
SELECT * FROM wallets WHERE id = $1 FOR UPDATE;   -- serialize concurrent spenders on this row
-- check balance, insert ledger row, update balance
COMMIT;
```

### Job queue with SKIP LOCKED
```sql
WITH job AS (
  SELECT id FROM jobs WHERE status = 'queued' AND run_at <= now()
  ORDER BY run_at FOR UPDATE SKIP LOCKED LIMIT 10
)
UPDATE jobs SET status = 'running', locked_at = now() FROM job WHERE jobs.id = job.id RETURNING jobs.*;
```

### Serializable with retry
```ts
for (let attempt = 0; attempt < 5; attempt++) {
  try {
    return await db.tx({ isolationLevel: "serializable" }, t => transfer(t, from, to, amount));
  } catch (e: any) {
    if (e.code === "40001" || e.code === "40P01") { await sleep(2 ** attempt * 20 + Math.random() * 20); continue; }
    throw e;
  }
}
```

### Safe migrations (lock-aware)
```sql
SET lock_timeout = '3s';            -- fail fast instead of blocking the app behind a queued ACCESS EXCLUSIVE
SET statement_timeout = '15min';
ALTER TABLE orders ADD COLUMN channel text;                       -- fast (no rewrite)
ALTER TABLE orders ADD CONSTRAINT orders_channel_chk CHECK (channel IN ('wa','web')) NOT VALID;
ALTER TABLE orders VALIDATE CONSTRAINT orders_channel_chk;        -- scans with weaker lock
```

### Monitoring queries
```sql
-- Long/idle transactions holding back vacuum
SELECT pid, state, now() - xact_start AS age, left(query, 80) FROM pg_stat_activity
WHERE xact_start IS NOT NULL ORDER BY xact_start LIMIT 10;

-- Wraparound risk
SELECT datname, age(datfrozenxid) FROM pg_database ORDER BY 2 DESC;

-- Bloat / dead tuples per table
SELECT relname, n_dead_tup, n_live_tup, last_autovacuum FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 20;

-- Who blocks whom
SELECT pid, pg_blocking_pids(pid) AS blocked_by, left(query, 60) FROM pg_stat_activity WHERE cardinality(pg_blocking_pids(pid)) > 0;
```

### Per-table autovacuum tuning for hot tables
```sql
ALTER TABLE messages SET (autovacuum_vacuum_scale_factor = 0.02, autovacuum_vacuum_cost_limit = 2000);
```

## Patterns & Anti-patterns
| Pattern | When | Anti-pattern to avoid |
|---|---|---|
| Short transactions, no network calls inside | Always | `BEGIN` → call Stripe/WhatsApp API → `COMMIT` (locks + held snapshot) |
| Atomic `UPDATE … WHERE guard RETURNING` | Counters, stock, balances | Read-modify-write in application code |
| `SELECT … FOR UPDATE SKIP LOCKED` | DB-backed job queues | Polling `UPDATE … WHERE status='queued' LIMIT 1` without locks (double processing) |
| Serializable + retry loop | Complex invariants across rows | Serializable without retry handling |
| `idle_in_transaction_session_timeout` set | All app roles | ORMs/pools leaking open transactions |
| Outbox table in the same transaction | Emit events reliably with DB writes | Publishing to [[Redis]]/queue before commit (ghost events on rollback) |

## Performance & Trade-offs
- MVCC means **UPDATE = INSERT + mark old dead**. Update-heavy tables need fillfactor < 100 (e.g. 80–90) to enable HOT updates and aggressive autovacuum.
- SERIALIZABLE adds predicate-lock tracking overhead and retries. Use it on specific transactions, not globally.
- Huge `DELETE`s create massive dead tuples. Batch them (`DELETE … WHERE id IN (SELECT … LIMIT 5000)`), or partition and `DROP PARTITION`.
- `VACUUM FULL`/`CLUSTER` rewrite the table under `ACCESS EXCLUSIVE`. Use `pg_repack` (or 19's `REPACK CONCURRENTLY`) online.

## Tips & Reminders
> [!tip]
> - Set role-level guardrails: `ALTER ROLE app SET idle_in_transaction_session_timeout = '60s'; ALTER ROLE app SET statement_timeout = '30s';`
> - Alert on: `age(datfrozenxid) > 1e9`, the oldest transaction > 15 min, replication slots inactive with growing lag, `n_dead_tup` spikes.
> - Use `SAVEPOINT`s to recover from expected errors inside a transaction without aborting it.
> - **In ZP's stack**: [[Supabase]] transaction pooler (6543) can't keep session state (prepared statements, `SET`, advisory session locks). Use transaction-scoped settings (`SET LOCAL`) and `pg_advisory_xact_lock`. Long-running n8n workflows must never hold DB transactions across nodes.

## Version Notes
| Version | Change |
|---|---|
| 9.1 | SSI-based true SERIALIZABLE |
| 9.6 | Freeze map (vacuum skips all-frozen pages) |
| 13 | Parallel index vacuum. Insert-triggered autovacuum |
| 14 | `idle_session_timeout`, vacuum emergency failsafe for wraparound |
| 17 | Vacuum memory rewrite (TID store), much lower memory use |
| 18 (2025-09) | Async I/O speeds vacuum. Eager freezing improvements |
| 19 (beta) | Parallel autovacuum of indexes, autovacuum priority scoring, `REPACK [CONCURRENTLY]` |

## Critical Issues & Gotchas
> [!danger] Transaction ID wraparound outages
> Sentry (2015) and Mailchimp/Mandrill (2019) were forced read-only for hours when autovacuum couldn't freeze fast enough, usually because a long transaction or stale replication slot held back the horizon. Monitor `age(datfrozenxid)`, drop abandoned slots, and never leave transactions open.

> [!danger] DDL lock queues
> An `ALTER TABLE` waiting for an `ACCESS EXCLUSIVE` lock behind one long SELECT blocks **all** subsequent queries on that table, which looks like an instant outage. Always set `lock_timeout` in migrations and retry.

> [!warning] Gotchas
> - `READ COMMITTED` statements in one transaction can see different data. Don't assume consistency across statements.
> - `SELECT … FOR UPDATE` on a join locks rows in **all** joined tables unless you use `FOR UPDATE OF t`.
> - Sequences aren't transactional: rolled-back inserts leave gaps. Never use sequence gaps to detect missing data (invoice numbering needs a separate gapless counter with row locking).
> - `now()` is the transaction start time. Use `clock_timestamp()` for wall time inside long transactions.
> - Hint-bit writes make the **first** read after bulk load write to disk (surprising I/O on replicas/read-only queries).

## Related
- [[PostgreSQL]]
- [[PostgreSQL - Indexing]] — HOT updates, visibility map, index bloat
- [[PostgreSQL - Query Planning & EXPLAIN]]
- [[Supabase]] — pooler modes and transaction scope
- [[Redis]] — complement for locks/queues where Postgres contention is too high

## References
- Concurrency control: https://www.postgresql.org/docs/current/mvcc.html
- Routine vacuuming: https://www.postgresql.org/docs/current/routine-vacuuming.html
- Explicit locking: https://www.postgresql.org/docs/current/explicit-locking.html
- Mailchimp/Mandrill wraparound post-mortem (2019): https://mailchimp.com/what-we-learned-from-the-recent-mandrill-outage/
