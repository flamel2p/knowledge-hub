---
title: "Databases MOC"
aliases: ["Databases"]
type: moc
domain: databases
tags: [domain/databases, type/moc]
status: draft
created: 2026-09-28
updated: 2026-10-19
parent: "[[00 - Home]]"
---

# Databases MOC

> [!abstract] Scope
> Relational, key-value, document and vector data stores.

## Notes
- [[Redis]] — in-memory data structure server: cache, queues, rate limiting, pub/sub
  - [[Redis - Data Structures & Patterns]]
  - [[Redis - Persistence & High Availability]]
  - [[Redis - Interview Questions]] — interview prep reference (deploy, HA, cache failure patterns)
- [[Valkey]] — BSD-3 Linux Foundation fork of Redis 7.2: RESP-compatible, 9.x I/O threading, hash-field TTL, atomic slot migration
  - [[Valkey - Redis to Valkey Migration Runbook]]
- [[PostgreSQL]] — relational system of record: MVCC, indexes, RLS, pgvector, 18 async I/O
  - [[PostgreSQL - Indexing]]
  - [[PostgreSQL - Transactions & MVCC]]
  - [[PostgreSQL - Query Planning & EXPLAIN]]
- [[MySQL]] — InnoDB OLTP workhorse; 8.4 & 9.7 LTS, 8.0 EOL 2026-04, Oracle stewardship risk, Vitess sharding
  - [[MySQL - InnoDB Internals & Tuning]]
- [[NoSQL]] — key-value/document/wide-column/graph families, CAP/PACELC, query-first modeling, MongoBleed, relicensing
  - [[NoSQL - Interview Questions]] — interview prep reference (SQL vs NoSQL)
- [[Vector Databases]] — embeddings + ANN (HNSW/IVF/DiskANN), quantization, hybrid RRF, pgvector vs Qdrant/Milvus/Pinecone, tenant leakage
  - [[Vector Databases - pgvector Tuning & Hybrid Search]]

## Planned
_None — all planned overviews are written; queued deep dives are listed under Notes as (planned)._

## Cross-domain Links
- [[00 - Home]]
