---
title: PostgreSQL - Bulk Imports, COPY & Legacy Data Migration
aliases: [Postgres COPY, \copy, bulk insert, CSV import, Excel import, staging tables, legacy data migration, data cleansing, upsert ON CONFLICT, MERGE, pgloader, migrate from MySQL, migrate from spreadsheets, ETL to Supabase]
type: deep-dive
domain: databases
tags: [domain/databases, type/deep-dive, topic/postgresql, topic/data-migration, topic/etl, lang/sql]
status: draft
created: 2026-11-23
updated: 2026-11-23
version_checked: "PostgreSQL 18.7 (COPY … ON_ERROR ignore since 17, LOG_VERBOSITY, MERGE … RETURNING) · 19 GA planned 2026-11-19 · pgloader 3.6.x — 2026-11"
parent: "[[PostgreSQL]]"
related: ["[[PostgreSQL - Major Version Upgrades & Migrations]]", "[[PostgreSQL - Locking, Deadlocks & Concurrency Control]]", "[[PostgreSQL - JSONB Patterns & Full-Text Search]]", "[[Supabase - Migrations, Branching & Local Development]]", "[[n8n - Google Workspace & Sheets Integrations]]", "[[MySQL]]"]
---

# PostgreSQL - Bulk Imports, COPY & Legacy Data Migration

> [!info] Deep dive of [[PostgreSQL]]

> [!abstract] TL;DR
> Most SME projects start by importing years of messy data: Excel price lists, old POS exports, MySQL/Access databases, accounting CSVs.
> - **The reliable pattern**: raw file → **staging table (all `text`)** via `COPY`/`\copy` → **clean and validate in SQL** (trim, normalize phones/dates, dedupe) → **upsert** into real tables (`INSERT … ON CONFLICT` / `MERGE`) → **reject report** for rows that failed → reconcile counts and totals with the client.
> - **`COPY` is 10–100× faster than row-by-row INSERTs**. Use `\copy` (client-side) when you can't access the server filesystem (Supabase).
> - **For database-to-database moves** (MySQL → Postgres), `pgloader` handles schema + data in one pass. Still validate afterwards.
> - **Run imports as repeatable scripts** (idempotent), not one-off manual clicks. You'll rerun them at least three times before cutover.

## Concept
| Step | Purpose | Tooling |
|---|---|---|
| Extract | Get raw data out of legacy system | CSV export, `mysqldump`, Excel → CSV (UTF-8) |
| Stage | Load as-is, no constraints | `create table staging.x (… text)`, `COPY` |
| Clean | Normalize formats, trim, map codes | SQL (`regexp_replace`, `to_date`, lookups) |
| Validate | Required fields, ranges, FK lookups, duplicates | SQL checks → `staging.rejects` |
| Load | Insert/upsert into final tables | `INSERT … ON CONFLICT`, `MERGE` |
| Reconcile | Prove completeness | Row counts, sum of amounts, spot checks with client |
| Cutover | Final delta import + switch | Freeze legacy writes, rerun scripts |

**Tool choice**:

| Source | Tool | Notes |
|---|---|---|
| CSV / Excel | `\copy` / `COPY`, or `psql` from a script | Save Excel as CSV UTF-8. Watch date formats |
| Google Sheets | n8n/script → CSV → staging | Validate headers first ([[n8n - Google Workspace & Sheets Integrations]]) |
| MySQL/MariaDB | `pgloader`, or dump CSV per table | Type mapping (zero dates, tinyint(1), unsigned) ([[MySQL]]) |
| Another Postgres | `pg_dump`/`pg_restore`, logical replication | Major versions differ → see upgrades note |
| SaaS APIs | Script/n8n paginated export → JSONB staging | Rate limits, incremental cursors |

## How It Works
```mermaid
flowchart LR
  SRC["Legacy: Excel / POS CSV / MySQL"] --> CSV["UTF-8 CSV files (versioned in a private bucket)"]
  CSV -->|"\\copy"| STG[("staging.* (text columns)")]
  STG --> CLEAN["clean.sql: normalize, map, dedupe"]
  CLEAN --> VAL{"validate"}
  VAL -->|"ok"| LOAD["upsert into public.* (ON CONFLICT / MERGE)"]
  VAL -->|"bad"| REJ[("staging.rejects + reason")]
  LOAD --> REC["reconcile: counts, totals, samples → client sign-off"]
```

## Practical Usage
**Staging table and client-side load** (works with Supabase via the direct/pooler connection):
```sql
create schema if not exists staging;
create table staging.customers_raw (
  legacy_id text, name text, phone text, email text, ic_no text, joined text, outlet text, points text
);
```
```bash
psql "$DATABASE_URL" -c "\copy staging.customers_raw from 'customers.csv' with (format csv, header true, encoding 'UTF8')"
```
**Tolerate bad rows during exploration** (PostgreSQL 17+):
```sql
copy staging.customers_raw from '/data/customers.csv'
  with (format csv, header true, on_error ignore, log_verbosity verbose);   -- server-side file; logs skipped rows
```
**Clean and normalize** (Malaysian phones, dates, emails):
```sql
create table staging.customers_clean as
select
  trim(legacy_id) as legacy_id,
  initcap(trim(name)) as name,
  case
    when regexp_replace(phone, '\D', '', 'g') ~ '^0\d{8,10}$'  then '6' || regexp_replace(phone, '\D', '', 'g')
    when regexp_replace(phone, '\D', '', 'g') ~ '^60\d{8,10}$' then regexp_replace(phone, '\D', '', 'g')
  end as phone_e164,                                         -- 0123456789 → 60123456789
  nullif(lower(trim(email)), '') as email,
  case when joined ~ '^\d{1,2}/\d{1,2}/\d{4}$' then to_date(joined, 'DD/MM/YYYY') end as joined_on,
  nullif(regexp_replace(points, '[^\d]', '', 'g'), '')::int as points,
  trim(outlet) as outlet_name
from staging.customers_raw;
```
**Reject report**:
```sql
create table if not exists staging.rejects (src text, legacy_id text, reason text, row jsonb);
insert into staging.rejects
select 'customers', legacy_id, 'invalid_phone', to_jsonb(c) from staging.customers_clean c where phone_e164 is null
union all
select 'customers', legacy_id, 'unknown_outlet', to_jsonb(c) from staging.customers_clean c
  where not exists (select 1 from public.outlets o where o.name = c.outlet_name);
```
**Dedupe and upsert** (idempotent, rerunnable):
```sql
insert into public.customers (tenant_id, legacy_id, name, phone, email, joined_on, points, outlet_id)
select distinct on (c.phone_e164)
  $1, c.legacy_id, c.name, c.phone_e164, c.email, c.joined_on, coalesce(c.points, 0), o.id
from staging.customers_clean c
join public.outlets o on o.name = c.outlet_name
where c.phone_e164 is not null
order by c.phone_e164, c.joined_on desc nulls last
on conflict (tenant_id, phone) do update
  set name = excluded.name, email = coalesce(excluded.email, customers.email),
      points = greatest(customers.points, excluded.points);
```
**MySQL → Postgres with pgloader**:
```text
LOAD DATABASE
  FROM mysql://user:pass@legacy-host/possys
  INTO postgresql://app:pass@localhost/app
WITH include drop, create tables, create indexes, reset sequences, workers = 4
CAST type datetime to timestamptz drop default drop not null using zero-dates-to-null,
     type tinyint when (= precision 1) to boolean using tinyint-to-boolean
ALTER SCHEMA 'possys' RENAME TO 'legacy';
```
**Reconcile**:
```sql
select (select count(*) from staging.customers_raw) as raw,
       (select count(*) from public.customers where tenant_id = $1) as loaded,
       (select count(*) from staging.rejects where src = 'customers') as rejected;
select sum(total_cents)/100.0 from public.orders where tenant_id = $1 and legacy_id is not null;   -- vs legacy report total
```
**Wrong way**:
```text
❌ Importing Excel directly into final tables with constraints   → first bad row aborts everything, or bad data slips in
❌ Row-by-row INSERTs from a script for 500k rows                 → hours instead of seconds
❌ One-off manual imports with no script                           → can't reproduce for the final cutover
❌ Silently dropping invalid rows                                  → client discovers missing customers later
```

## Patterns & Anti-patterns
| Pattern | When | Anti-pattern to avoid |
|---|---|---|
| Text-only staging tables | Any messy source | Typed columns rejecting rows at load |
| Keep `legacy_id` on migrated rows | Traceability | Losing link to source records |
| Idempotent upsert scripts in git | Repeated dry runs | Manual SQL in the dashboard |
| Reject table with reasons → client review | Data quality | Silent drops |
| Disable non-essential triggers/indexes during huge loads, rebuild after | Millions of rows | Audit triggers firing per row unknowingly |
| Load in batches/transactions per table | Large imports | One giant transaction for everything |
| Reconciliation report signed by client | Cutover | "Looks fine" sign-off |

## Performance & Trade-offs
- **`COPY`** streams rows with minimal overhead. Indexes and triggers dominate load time on big tables. For multi-million-row loads, create indexes after loading.
- **`\copy` through Supabase's pooler** works for moderate sizes. For very large loads, use the direct connection and run from a nearby machine (Singapore region) to avoid latency.
- **`MERGE` vs `ON CONFLICT`**: `ON CONFLICT` is simpler and concurrency-safe for single-key upserts. `MERGE` handles complex insert/update/delete logic but isn't a race-free upsert under concurrency.
- **Audit triggers** double write volume during imports. Disable deliberately with a documented reason, or mark imported rows ([[PostgreSQL - Audit Logging & Change History]]).

## Tips & Reminders
> [!tip] In ZP's stack
> - **Scope it in the SOW**: "data migration" is where fixed-price projects lose money. Specify sources, number of files, data cleaning responsibility (client fixes source data vs you transform), number of dry runs, and acceptance via reconciliation report.
> - **Malaysian data quirks**: phone formats (`012-345 6789`, `+60…`), IC numbers with/without dashes, DD/MM/YYYY dates, mixed BM/EN/ZH names, and outlet names spelled three ways. Build mapping tables.
> - **PDPA**: legacy exports often contain ICs and addresses. Keep raw files in a private bucket, delete them after migration sign-off, and migrate only needed fields ([[Supabase - Data Retention, PDPA & Account Deletion]]).

- Save Excel as **CSV UTF-8**. Default Windows CSV encoding breaks Chinese characters.
- Reset sequences after importing explicit IDs: `select setval(pg_get_serial_sequence('t','id'), max(id)) from t;`
- Freeze the legacy system (read-only) during the final delta import.
- Keep staging tables until the client signs off, then drop them.

## Version Notes
| Version | Change relevant here |
|---|---|
| 9.5 | `INSERT … ON CONFLICT` (upsert) |
| 12 | `COPY FROM … WHERE` filter |
| 15 | `MERGE` |
| 17 | `COPY … ON_ERROR ignore`, `LOG_VERBOSITY` |
| 18 | `MERGE … RETURNING` with `OLD/NEW`. Async I/O speeds large scans ([[PostgreSQL]]) |

> [!warning] Unverified — check before relying on this
> Whether Supabase's pooler/connection limits affect large `\copy` loads, and pgloader's current release and Postgres 18 compatibility, weren't verified this run. Test with a full-size dry run.

## Critical Issues & Gotchas
> [!danger] Silent data loss during migration
> Rows dropped by `ON_ERROR ignore`, date parsing that turns `13/02/2024` into null, `distinct on` dedupe choosing the wrong record, or truncated text from encoding issues all lose data without errors. Count, sum and sample at every step, keep reject tables, and get written client sign-off on reconciliation before decommissioning the legacy system.

> [!danger] Migrating personal data you shouldn't keep
> Legacy systems hoard years of IC numbers, addresses and old customers who never consented to the new system. Migrating everything multiplies PDPA exposure. Agree a data minimization scope (fields, time range) with the client, and drop or anonymize the rest.

- **Encoding**: Excel/Windows-1252 vs UTF-8 mismatches corrupt names (`Ã©`). Detect with `file -i`, convert with `iconv`.
- **Duplicates across sources** (POS and loyalty app) need explicit merge rules (latest wins, max points).
- **Time zones**: legacy timestamps are often local time without zone. Interpret as `Asia/Kuala_Lumpur` explicitly.
- **Locks**: bulk updates on live tables during business hours block users. Load into staging, then swap or batch ([[PostgreSQL - Locking, Deadlocks & Concurrency Control]]).

## Related
- [[PostgreSQL]]
- [[PostgreSQL - Major Version Upgrades & Migrations]] — Postgres-to-Postgres moves
- [[PostgreSQL - Locking, Deadlocks & Concurrency Control]] — safe bulk updates
- [[PostgreSQL - JSONB Patterns & Full-Text Search]] — staging API exports as JSONB
- [[Supabase - Migrations, Branching & Local Development]] — schema migrations around imports
- [[MySQL]] — common legacy source

## References
- https://www.postgresql.org/docs/current/sql-copy.html
- https://www.postgresql.org/docs/current/app-psql.html#APP-PSQL-META-COMMANDS-COPY
- https://www.postgresql.org/docs/current/sql-insert.html#SQL-ON-CONFLICT
- https://www.postgresql.org/docs/current/sql-merge.html
- https://pgloader.readthedocs.io/
