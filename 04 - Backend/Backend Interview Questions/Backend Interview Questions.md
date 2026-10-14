---
title: Backend Interview Questions
aliases: [backend interview, full-stack interview questions, interview question index, 后端面试题]
type: reference
domain: backend
tags: [domain/backend, type/reference, topic/interview]
status: draft
created: 2026-10-13
updated: 2026-10-14
version_checked: "n/a — index + behavioural questions — 2026-10"
parent: "[[Backend MOC]]"
related: ["[[Spring Boot - Interview Questions]]", "[[Redis - Interview Questions]]", "[[NoSQL - Interview Questions]]", "[[Authentication & Authorization - Interview Questions]]", "[[Design Patterns - Interview Questions]]", "[[System Design - Interview Questions]]", "[[Flutter - Interview Questions]]"]
---

# Backend Interview Questions

> [!info] Index of all interview-question notes in the vault, plus general questions that have no topic home

> [!abstract] TL;DR
> Topic questions live in their own `<Topic> - Interview Questions` notes (see the index). This note keeps the two **general/behavioural** ones: Q1 your tech stack (pre-filled from ZP's stack, with a `[!question]` box for project facts) and Q2 how you debug a complex issue (6-step method, tools per layer, story template).

> [!tip] How to use these notes
> Answer with a **causal chain**: *decision → why (mechanism or constraint) → how it ran in production → trade-off or what you'd change*. Interviewers score trade-offs and real incidents, not definitions. Keep each answer to **~2 minutes spoken** and let them pull the details.

## Question Index
| Question | Note |
|---|---|
| What tech stacks have you used? | Q1 below |
| How do you debug a complex issue? | Q2 below |
| Explain OOP | [[Design Patterns - Interview Questions]] Q1 |
| Design patterns in your recent projects | [[Design Patterns - Interview Questions]] Q2 |
| SQL vs NoSQL | [[NoSQL - Interview Questions]] Q1 |
| Dependency injection in Spring Boot | [[Spring Boot - Interview Questions]] Q1 |
| Singleton vs prototype bean | [[Spring Boot - Interview Questions]] Q2 |
| Deploy Redis; performance and durability in abnormal situations | [[Redis - Interview Questions]] Q1 |
| Authentication for a finance app | [[Authentication & Authorization - Interview Questions]] Q1 |
| Design a payment method and its flow | [[System Design - Interview Questions]] Q1 |
| Concert ticketing under huge traffic | [[System Design - Interview Questions]] Q2 |
| Dart / Flutter (12 questions) | [[Flutter - Interview Questions]] |

> [!info] Adding questions
> New questions go into the `<Topic> - Interview Questions` note inside that topic's folder, then get a row here. Only general or behavioural questions with no topic home stay in this note.

## Q1. What tech stacks have you used in your previous projects?

**Grading**: breadth with *depth in one stack*, and **why** you chose each piece. A list of logos fails.

**Answer frame**: group by project type, give one reason per choice, and name one thing you'd change.

| Layer | Stack | Why (the line to say) |
|---|---|---|
| Web / SaaS | [[Next.js]] + [[React]] + [[TypeScript]], Tailwind | SSR/RSC for SEO and dashboards, one language end to end |
| Mobile | [[Flutter]] / [[Dart]] | One codebase for iOS + Android, near-native performance, good offline story |
| Backend | [[Supabase]] (Auth, RLS, Edge Functions), [[Node.js]]; [[Spring Boot]]/[[Java]] for heavier domain services | Supabase for speed on CRUD + auth. JVM where strong typing and transactions dominate |
| Data | [[PostgreSQL]] (system of record, pgvector), [[Redis]] (cache, queues, rate limits) | Postgres covers relational + JSON + vectors. Redis for sub-ms hot paths |
| AI | LLM APIs with provider routing, [[RAG]], [[AI Agents]], [[Model Context Protocol\|MCP]], [[LangChain & LangGraph]] | Agents built per client workflow, or integrate an off-the-shelf product when it fits better |
| Automation / messaging | [[n8n]] self-hosted in queue mode, WhatsApp Cloud API | Fast client-specific workflows. WhatsApp is the primary channel for Malaysian SMEs |
| Infra | [[Docker]], [[Coolify]], [[Traefik]], [[Cloudflare]] on Hostinger KVM | Self-hosted PaaS keeps cost predictable for a small team. Cloudflare for DNS, WAF, R2 backups |

- **Depth signal**: pick one project and walk the request path end to end: Cloudflare → Traefik → Next.js → Supabase/Postgres → Redis queue → worker → WhatsApp.
- **Honest trade-off to offer**: "Single KVM host plus Coolify is a single point of failure. I mitigate it with off-host backups to R2 and a tested restore runbook. At the next scale step I'd split the DB onto managed Postgres or a second node."
- **Trap**: claiming expertise in everything. Mark levels: *production* vs *built projects with* vs *familiar*.

> [!question] Fill in before the interview
> For 2–3 projects: one-line problem, your role, users/traffic, and one measurable outcome. Keep client names generic if under NDA.

## Q2. How do you approach debugging a complex issue in your code?

**Grading**: a **systematic method** (not "add console.log until it works"), use of observability, and closing the loop with a regression test.

1. **Define and contain**: exact symptom, scope (who, since when, which version), severity. Mitigate first in prod: roll back, toggle the feature flag, scale up.
2. **Reproduce**: smallest reliable reproduction (same data, versions, env flags). A bug you can't reproduce, you instrument instead.
3. **Observe, don't guess**: logs with correlation IDs, traces (OpenTelemetry), metrics around the start time, recent deploys and config changes.
4. **Hypothesize → test one variable at a time**. Bisect: `git bisect` over commits, binary search over inputs/config, disable half the middleware.
5. **Find the root cause** (5 whys). The fix goes where the invariant broke, not where the symptom appeared.
6. **Lock it in**: failing test first, then fix. Add the alert or log that would have caught it sooner. Write a short postmortem for prod incidents.

| Layer | Tools |
|---|---|
| Node.js / Next.js | `node --inspect` + Chrome DevTools, `--cpu-prof`, heap snapshots, Sentry, OTel traces |
| JVM / Spring | Actuator metrics, `jcmd <pid> Thread.print`, JFR + JMC, async-profiler |
| Flutter | DevTools (performance, memory, network), `flutter run --profile` |
| PostgreSQL | `EXPLAIN (ANALYZE, BUFFERS)`, `pg_stat_statements`, `pg_locks` / `pg_stat_activity` for lock waits |
| Redis | `SLOWLOG GET`, `INFO memory`/`INFO stats` (evicted_keys), `LATENCY DOCTOR`, `--bigkeys` |
| Network / infra | `curl -v`, `dig`, `docker logs`, Traefik access logs, `tcpdump` as last resort |
| Git | `git bisect run ./repro.sh`, `git log -S 'symbol'` |

> [!example] Story template (adapt to a real incident)
> *Symptom*: n8n executions randomly stuck "queued" under load. *Observe*: workers healthy, Redis `INFO stats` showed `evicted_keys` rising. *Root cause*: `maxmemory-policy allkeys-lru` was evicting Bull job keys. *Fix*: `noeviction` plus memory alert at 80% and a separate Redis for cache. *Lock-in*: config checked into the compose file and a dashboard panel on evictions. See [[Redis - Persistence & High Availability]].

- **Trap**: jumping to the fix without reproducing, or "it was a race condition" without saying how you proved it.

## Related
- [[Design Patterns - Interview Questions]] · [[System Design - Interview Questions]] — CS fundamentals
- [[Spring Boot - Interview Questions]] — backend framework
- [[NoSQL - Interview Questions]] · [[Redis - Interview Questions]] — databases
- [[Authentication & Authorization - Interview Questions]] — security
- [[Flutter - Interview Questions]] — mobile

## References
- OpenTelemetry docs: https://opentelemetry.io/docs/
- `git bisect`: https://git-scm.com/docs/git-bisect
- PostgreSQL `EXPLAIN`: https://www.postgresql.org/docs/current/using-explain.html
