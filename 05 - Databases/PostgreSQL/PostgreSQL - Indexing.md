---
title: PostgreSQL - Indexing
aliases: [Postgres indexes, B-tree index, GIN index, BRIN, partial index, covering index, HNSW]
type: deep-dive
domain: databases
tags: [domain/databases, type/deep-dive, topic/postgresql, topic/indexing]
status: draft
created: 2026-10-05
updated: 2026-10-05
version_checked: "PostgreSQL 18 — 2026-10"
parent: "[[PostgreSQL]]"
related: ["[[PostgreSQL - Query Planning & EXPLAIN]]", "[[PostgreSQL - Transactions & MVCC]]", "[[Supabase]]", "[[Big O Notation]]"]
---

# PostgreSQL - Indexing

> [!info] Deep dive of [[PostgreSQL]]

> [!abstract] TL;DR
> An index is a separate data structure that lets the planner find rows in O(log n) (B-tree) or via specialised lookups (GIN, GiST, BRIN, HNSW) instead of scanning the whole table. Every index speeds up some reads and **slows down every write** (plus vacuum, storage and cache pressure). Design indexes from **actual query patterns** (`pg_stat_statements` + `EXPLAIN`): equality columns first, then range/sort columns, partial indexes for hot subsets, `INCLUDE` for index-only scans, and `CREATE INDEX CONCURRENTLY` on live tables. Remember: **an index the planner never uses is pure cost.**

## Concept
| Type | Structure | Operators / use | Notes |
|---|---|---|---|
| **B-tree** (default) | Balanced tree, sorted | `= < <= > >= BETWEEN IN`, `IS NULL`, `ORDER BY`, prefix `LIKE 'abc%'` (C collation / `text_pattern_ops`) | 90% of indexes. Unique constraints. 18 adds **skip scan** for leading columns not in the WHERE |
| **Hash** | Hash buckets | `=` only | Rarely better than B-tree. WAL-logged since 10 |
| **GIN** | Inverted index (value → row list) | `jsonb @> ? ?& ?\|`, arrays `@> &&`, full-text `@@`, `pg_trgm` `ILIKE '%x%'`/similarity | Slow writes (`fastupdate` pending list). Great reads |
| **GiST** | Generalised search tree | Ranges `&& @>`, geometry (PostGIS), exclusion constraints, nearest-neighbour `<->` | Lossy, rechecks rows |
| **SP-GiST** | Space-partitioned trees | Points, IP ranges (`inet`), text prefixes | Niche |
| **BRIN** | Min/max per block range | Huge append-only tables correlated with physical order (time series) | Tiny (KBs for GBs of table) |
| **HNSW / IVFFlat** (pgvector) | ANN graph / clusters | `<=>` cosine, `<->` L2, `<#>` inner product | Approximate. Tune `ef_search` / `probes` |

## How It Works
- **B-tree lookup**: root → internal pages → leaf (sorted keys + TIDs pointing to heap tuples) → heap fetch to check visibility (MVCC). Height is typically 3–4 for millions of rows.
- **Index-only scan**: if all needed columns are in the index **and** the heap page is marked all-visible in the **visibility map**, the heap fetch is skipped. That needs regular VACUUM.
- **Bitmap scan**: the index builds a bitmap of matching pages, then the heap is read in physical order. It combines multiple indexes (`BitmapAnd/Or`).
- **Multicolumn order matters**: `(tenant_id, status, created_at)` serves `tenant_id = ?`, `tenant_id = ? AND status = ?`, and `… ORDER BY created_at`. Rule: **equality columns first, then the range/sort column**.
- **Planner choice** is cost-based: low selectivity (e.g. `status = 'active'` matching 80% of rows) → seq scan wins. Stats come from `ANALYZE` (`default_statistics_target` 100). Extended statistics (`CREATE STATISTICS`) help with correlated columns.
- **HOT updates**: if an UPDATE changes no indexed column and the page has room, no index entries are written. **Every extra index reduces HOT updates** and increases write amplification.

## Practical Usage

### Composite + partial + covering
```sql
-- Hot query: open orders for a tenant, newest first, page by keyset
SELECT id, total, created_at FROM orders
WHERE tenant_id = $1 AND status = 'open' AND (created_at, id) < ($2, $3)
ORDER BY created_at DESC, id DESC LIMIT 50;

CREATE INDEX CONCURRENTLY idx_orders_open_tenant_created
  ON orders (tenant_id, created_at DESC, id DESC)
  INCLUDE (total)                       -- index-only scan for the selected columns
  WHERE status = 'open';                -- partial: only the hot subset (smaller, faster)
```

### Expression index
```sql
CREATE INDEX CONCURRENTLY idx_customers_email_lower ON customers (lower(email));
-- query must use the same expression:
SELECT * FROM customers WHERE lower(email) = lower($1);
```

### JSONB
```sql
CREATE INDEX idx_events_payload ON events USING gin (payload jsonb_path_ops);   -- smaller, supports @> only
SELECT * FROM events WHERE payload @> '{"type":"order.paid"}';
-- For one frequently-filtered key, a B-tree expression index is better:
CREATE INDEX idx_events_type ON events ((payload->>'type'));
```

### Fuzzy search (trigram)
```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX idx_products_name_trgm ON products USING gin (name gin_trgm_ops);
SELECT * FROM products WHERE name ILIKE '%kopi%' ORDER BY similarity(name, 'kopi') DESC LIMIT 20;
```

### Time series with BRIN
```sql
CREATE INDEX idx_logs_created_brin ON logs USING brin (created_at) WITH (pages_per_range = 64);
```

### pgvector HNSW with filters
```sql
CREATE INDEX CONCURRENTLY idx_chunks_embedding ON chunks USING hnsw (embedding vector_cosine_ops) WITH (m = 16, ef_construction = 64);
SET hnsw.ef_search = 100;                         -- recall vs latency
SET hnsw.iterative_scan = relaxed_order;          -- pgvector 0.8+: keep scanning when filters drop results
SELECT id FROM chunks WHERE tenant_id = $1 ORDER BY embedding <=> $2 LIMIT 10;
```

### Find unused / missing indexes
```sql
-- Unused (since stats reset): candidates to drop (check replicas too!)
SELECT relname, indexrelname, idx_scan, pg_size_pretty(pg_relation_size(indexrelid))
FROM pg_stat_user_indexes WHERE idx_scan = 0 ORDER BY pg_relation_size(indexrelid) DESC;

-- Tables with heavy seq scans: candidates for new indexes
SELECT relname, seq_scan, seq_tup_read, idx_scan FROM pg_stat_user_tables ORDER BY seq_tup_read DESC LIMIT 20;
```

## Patterns & Anti-patterns
| Pattern | When | Anti-pattern to avoid |
|---|---|---|
| Index every FK used in joins/`ON DELETE` | Always | Missing FK index → seq scan on child table for every parent delete |
| Composite index matching WHERE + ORDER BY | Hot list endpoints | Separate single-column indexes hoping for bitmap AND |
| Partial index on hot subset | `status='open'`, `deleted_at IS NULL` | Indexing a boolean with 50/50 distribution |
| `INCLUDE` columns for index-only scans | Read-heavy, narrow selects | Indexing every column "just in case" |
| `CONCURRENTLY` for create/drop/reindex in prod | Live tables | Plain `CREATE INDEX` → write lock for minutes |
| Expression index matching query expression | `lower(email)`, `date(created_at)` | Wrapping indexed column in a function in WHERE (index unused) |
| Tenant-leading indexes | Multi-tenant SaaS / RLS | Indexes without `tenant_id` when every query filters by it |

## Performance & Trade-offs
- Write cost: each index adds ~1 index tuple write per INSERT/non-HOT UPDATE plus WAL. 10 indexes ≈ 10× index write amplification.
- Size: check `pg_relation_size`. Bloated indexes (after mass updates/deletes) → `REINDEX CONCURRENTLY` (or 19's `REPACK CONCURRENTLY` for tables).
- GIN `fastupdate` makes inserts cheap but searches scan the pending list. Tune `gin_pending_list_limit` or vacuum more.
- HNSW build is slow and memory hungry. Raise `maintenance_work_mem` (e.g. 2–8 GB) and use parallel builds (`max_parallel_maintenance_workers`).
- RLS policies add predicates. Make sure they're indexable (`tenant_id = (select auth.jwt()->>'tenant_id')::uuid`). Wrapping functions in `(select …)` lets Postgres evaluate them once per query.

## Tips & Reminders
> [!tip]
> - Workflow: `pg_stat_statements` top queries → `EXPLAIN (ANALYZE, BUFFERS)` → add or adjust one index → re-measure.
> - Name indexes descriptively (`idx_<table>_<cols>_<where>`). They're schema migrations, so version them.
> - `CREATE INDEX CONCURRENTLY` can't run inside a transaction block. Many migration tools wrap migrations in transactions, so disable that per migration.
> - A failed concurrent build leaves an **INVALID** index. Check `pg_index.indisvalid` and drop/recreate it.
> - **In ZP's stack**: the [[Supabase]] dashboard has an Index Advisor (`index_advisor` extension) and Performance Advisor. Use them, but verify with `EXPLAIN`. For [[n8n]]'s own DB, the big tables are `execution_entity` and `execution_data`, so prune instead of indexing.

## Version Notes
| Version | Change |
|---|---|
| 11 | Covering indexes (`INCLUDE`), parallel B-tree builds |
| 12 | `REINDEX CONCURRENTLY` |
| 13 | B-tree deduplication (smaller indexes for duplicate keys) |
| 14–16 | Bottom-up index deletion, BRIN multi-minmax/bloom, incremental sort improvements |
| 17 | Faster B-tree `IN` list scans, BRIN parallel builds |
| 18 (2025-09) | **B-tree skip scan** (use multicolumn index without the leading column when it has few distinct values), async I/O helps bitmap heap scans, GIN parallel builds |
| pgvector 0.5 / 0.8 | HNSW (2023) / iterative index scans for filtered ANN (2024) |

## Critical Issues & Gotchas
> [!danger] Locking during index creation
> Plain `CREATE INDEX` takes a `SHARE` lock that blocks INSERT/UPDATE/DELETE for the whole build, which can take minutes to hours on big tables and causes an outage. Always use `CONCURRENTLY` on live tables, and set `lock_timeout` in migrations so they fail fast instead of queueing behind long transactions.

> [!warning] Gotchas
> - Collation changes (glibc upgrades in OS/Docker images, e.g. Debian 12 → 13) can silently corrupt text B-tree indexes. Reindex after major OS/libc upgrades, and check `pg_collation` version warnings.
> - Implicit casts (`WHERE id = '123'` on a bigint) or mismatched types in joins → index not used.
> - `OR` across different columns often defeats indexes. Use `UNION ALL` or a bitmap-friendly design.
> - `LIKE '%term'` (leading wildcard) can't use B-tree. Use trigram GIN.
> - Unused-index stats are per node. An index unused on the primary may serve read replicas.

## Related
- [[PostgreSQL]]
- [[PostgreSQL - Query Planning & EXPLAIN]] — reading plans to validate indexes
- [[PostgreSQL - Transactions & MVCC]] — HOT updates, visibility map, bloat
- [[Supabase]] — Index Advisor, RLS-friendly indexes
- [[Big O Notation]] — O(log n) vs O(n) in practice
- [[RAG]] — pgvector HNSW for retrieval

## References
- Index types: https://www.postgresql.org/docs/current/indexes-types.html
- Multicolumn indexes: https://www.postgresql.org/docs/current/indexes-multicolumn.html
- Index-only scans: https://www.postgresql.org/docs/current/indexes-index-only-scans.html
- pgvector: https://github.com/pgvector/pgvector
- Use The Index, Luke: https://use-the-index-luke.com/
