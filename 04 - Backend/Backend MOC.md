---
title: "Backend MOC"
aliases: ["Backend"]
type: moc
domain: backend
tags: [domain/backend, type/moc]
status: draft
created: 2026-09-28
updated: 2026-10-23
parent: "[[00 - Home]]"
---

# Backend MOC

> [!abstract] Scope
> Server runtimes, backend frameworks, BaaS, testing.

## Notes
- [[Supabase]] — Postgres platform: Auth, PostgREST, Realtime, Storage, Edge Functions; new API keys + JWT signing keys
  - [[Supabase - Row Level Security]]
  - [[Supabase - Edge Functions, Cron & Queues]]
- [[Node.js]] — V8 + libuv runtime; 24 LTS, 26 → LTS Oct 2026, yearly majors from 27; npm supply-chain risk
  - [[Node.js - Performance & Diagnostics]]
  - [[Node.js - Streams & Backpressure]]
- [[Spring Boot]] — JVM framework; 4.1 on Spring Framework 7, virtual threads, JSpecify, Jakarta EE 11; actuator exposure risk
  - [[Spring Boot - Interview Questions]] — interview prep reference (DI, singleton vs prototype)
  - [[Spring Boot - Spring Security Resource Server]]
- [[Testing Strategy]] — testing trophy, real-DB integration tests (Testcontainers/Supabase), authZ matrices, Playwright smoke, flaky-test hygiene
  - [[Testing Strategy - Integration Testing with Testcontainers & Supabase]]
- [[Backend Interview Questions]] — index of every interview-question note, plus general questions (tech stack, debugging)

## Planned
_None — all planned overviews are written; queued deep dives are listed under Notes as (planned)._

## Cross-domain Links
- [[00 - Home]]
