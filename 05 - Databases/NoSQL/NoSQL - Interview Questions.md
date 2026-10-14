---
title: NoSQL - Interview Questions
aliases: [SQL vs NoSQL, relational vs non-relational, NoSQL interview, 数据库面试题]
type: reference
domain: databases
tags: [domain/databases, type/reference, topic/nosql, topic/postgresql, topic/interview]
status: draft
created: 2026-10-13
updated: 2026-10-14
version_checked: "n/a (concepts) — MongoDB 8.x, PostgreSQL 18 — 2026-10"
parent: "[[NoSQL]]"
related: ["[[PostgreSQL]]", "[[MySQL]]", "[[Redis - Interview Questions]]", "[[Backend Interview Questions]]"]
---

# NoSQL - Interview Questions

> [!info] Reference for [[NoSQL]] and [[PostgreSQL]] · index: [[Backend Interview Questions]]

> [!abstract] TL;DR
> Q1: **SQL vs NoSQL**. An 8-axis comparison (model, schema, transactions, consistency, scaling…), why the line blurred by 2026 (JSONB, distributed SQL, MongoDB transactions), the default stance (Postgres as system of record, NoSQL per access pattern), and the traps.

## Q1. What is the difference between SQL and NoSQL databases?

**Grading**: do you reason from **access patterns and consistency needs**, or repeat "NoSQL scales, SQL doesn't"?

| Axis | SQL (relational) | NoSQL |
|---|---|---|
| Data model | Tables, rows, foreign keys. Normalized | Document, key-value, wide-column, graph ([[NoSQL]] families) |
| Schema | Schema-on-write, enforced (`NOT NULL`, `CHECK`, FK) | Schema-on-read. App enforces (or optional validators) |
| Query | Declarative SQL, ad-hoc joins, aggregations | Access-pattern-specific APIs. Joins limited or absent. Model **query-first** |
| Transactions | ACID multi-row/multi-table by default | Varies: single-document atomic everywhere; MongoDB multi-doc transactions since 4.0 (with cost); Cassandra LWT only |
| Consistency | Strong on the primary. Replicas async by default | Often tunable (Cassandra `QUORUM`, Dynamo eventual/strong reads). PACELC trade-offs |
| Scaling | Vertical + read replicas. Sharding is manual (Citus) or via distributed SQL | Horizontal partitioning built in (partition key design is everything) |
| Strength | Integrity, complex queries, reporting, money | Huge write throughput, flexible/denormalized aggregates, predictable key lookups |
| Examples | [[PostgreSQL]], [[MySQL]] | MongoDB, Cassandra/ScyllaDB, DynamoDB, [[Redis]], Neo4j |

- **The blur in 2026**: Postgres `JSONB` + GIN indexes cover most "flexible schema" needs. Distributed SQL (CockroachDB, YugabyteDB, Spanner) gives horizontal scale with SQL and ACID. MongoDB has transactions. So the real question is **access pattern + consistency + ops cost**, not the label.
- **My default**: Postgres as the system of record (ledgers, users, orders). Add NoSQL for a *specific* access pattern: Redis for cache/queues/rate limits, a document or wide-column store for high-volume event/telemetry writes, a vector index (pgvector first) for RAG.
- **Trap**: "NoSQL has no schema". The schema still exists, it just moves into application code and migrations become your problem. Also: "MongoDB isn't ACID" (it has been, per document always and multi-document since 4.0).
- Deep dive: [[PostgreSQL - Transactions & MVCC]] · [[NoSQL]] · [[Vector Databases]]

## Related
- [[NoSQL]] — families, CAP/PACELC, query-first modelling
- [[PostgreSQL]] · [[MySQL]] — relational side
- [[Redis - Interview Questions]] — the key-value store most interviews follow up on
- [[Backend Interview Questions]] — interview index

## References
- MongoDB transactions: https://www.mongodb.com/docs/manual/core/transactions/
- PostgreSQL JSON types: https://www.postgresql.org/docs/current/datatype-json.html
