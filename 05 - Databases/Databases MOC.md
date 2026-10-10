---
title: "Databases MOC"
aliases: ["Databases"]
type: moc
domain: databases
tags: [domain/databases, type/moc]
status: draft
created: 2026-09-28
updated: 2026-10-10
parent: "[[00 - Home]]"
---

# Databases MOC

> [!abstract] Scope
> Relational, key-value, document and vector data stores.

## Notes
- [[Redis]] — in-memory data structure server: cache, queues, rate limiting, pub/sub
  - [[Redis - Data Structures & Patterns]]
  - [[Redis - Persistence & High Availability]]
- [[PostgreSQL]] — relational system of record: MVCC, indexes, RLS, pgvector, 18 async I/O
  - [[PostgreSQL - Indexing]]
  - [[PostgreSQL - Transactions & MVCC]]
  - (planned) [[PostgreSQL - Query Planning & EXPLAIN]]
- [[MySQL]] — InnoDB OLTP workhorse; 8.4 & 9.7 LTS, 8.0 EOL 2026-04, Oracle stewardship risk, Vitess sharding
- [[NoSQL]] — key-value/document/wide-column/graph families, CAP/PACELC, query-first modeling, MongoBleed, relicensing
- [[Vector Databases]] — embeddings + ANN (HNSW/IVF/DiskANN), quantization, hybrid RRF, pgvector vs Qdrant/Milvus/Pinecone, tenant leakage

## Planned

## Cross-domain Links
- [[00 - Home]]
