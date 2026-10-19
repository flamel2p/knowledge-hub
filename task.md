# Task Queue

Last run: 2026-10-19 (T161)

> Loop rules live in CLAUDE.md §8. Each run takes the **first** unchecked `- [ ] T###` line (skip lines containing ⚠️), completes it, ticks it with ` | ✅ YYYY-MM-DD`, commits, pushes to main, and stops. No wikilinks in this file.
> Line format: `- [ ] T### | type | Topic | path` · types: overview, deep-dive, moc, maint, replenish
> ZP can reorder, insert or delete lines anytime; new tasks take the next free T### number.

## Queue

### Priority — Requested 2026-09-29 (LangChain before LCAE exam 2026-10-05, then auth)

- [x] T056 | overview | LangChain & LangGraph | 13 - AI Engineering/LangChain & LangGraph/LangChain & LangGraph.md | ✅ 2026-10-01
- [x] T105 | deep-dive | LangChain & LangGraph - Graph API, State & Control Flow | 13 - AI Engineering/LangChain & LangGraph/LangChain & LangGraph - Graph API, State & Control Flow.md | ✅ 2026-10-02
- [x] T106 | deep-dive | LangChain & LangGraph - Agents, Tools & Middleware | 13 - AI Engineering/LangChain & LangGraph/LangChain & LangGraph - Agents, Tools & Middleware.md | ✅ 2026-10-02
- [x] T107 | deep-dive | LangChain & LangGraph - Persistence, Memory & Human-in-the-Loop | 13 - AI Engineering/LangChain & LangGraph/LangChain & LangGraph - Persistence, Memory & Human-in-the-Loop.md | ✅ 2026-10-02
- [x] T108 | deep-dive | LangChain & LangGraph - Multi-Agent Systems & LangSmith | 13 - AI Engineering/LangChain & LangGraph/LangChain & LangGraph - Multi-Agent Systems & LangSmith.md | ✅ 2026-10-02
- [x] T109 | overview | JWT | 10 - Security/JWT/JWT.md | ✅ 2026-10-02
- [x] T110 | overview | OAuth 2.0 & OIDC | 10 - Security/OAuth 2.0 & OIDC/OAuth 2.0 & OIDC.md | ✅ 2026-10-02

### Priority — Requested 2026-10-07 (Flutter / Dart, from interview question bank)

- [x] T111 | deep-dive | Dart - Async, Streams & Isolates | 01 - Languages/Dart/Dart - Async, Streams & Isolates.md | ✅ 2026-10-07
- [x] T112 | reference | Flutter - Interview Questions | 03 - Mobile/Flutter/Flutter - Interview Questions.md | ✅ 2026-10-07
- [x] T094 | deep-dive | Flutter - Rendering & Widget Lifecycle | 03 - Mobile/Flutter/Flutter - Rendering & Widget Lifecycle.md | ✅ 2026-10-08
- [x] T093 | deep-dive | Flutter - State Management | 03 - Mobile/Flutter/Flutter - State Management.md | ✅ 2026-10-08
- [x] T113 | deep-dive | Dart - Type System, Null Safety & Mixins | 01 - Languages/Dart/Dart - Type System, Null Safety & Mixins.md | ✅ 2026-10-08
- [x] T114 | deep-dive | Dart - VM, Compilation & Garbage Collection | 01 - Languages/Dart/Dart - VM, Compilation & Garbage Collection.md | ✅ 2026-10-08
- [x] T115 | deep-dive | Flutter - Performance & DevTools | 03 - Mobile/Flutter/Flutter - Performance & DevTools.md | ✅ 2026-10-08
- [x] T116 | deep-dive | Flutter - Build, Release & Store Compliance | 03 - Mobile/Flutter/Flutter - Build, Release & Store Compliance.md | ✅ 2026-10-08
- [x] T117 | deep-dive | Flutter - Platform Channels & FFI | 03 - Mobile/Flutter/Flutter - Platform Channels & FFI.md | ✅ 2026-10-08
- [x] T118 | deep-dive | Flutter - Testing | 03 - Mobile/Flutter/Flutter - Testing.md | ✅ 2026-10-08
- [x] T119 | deep-dive | Flutter - Navigation & Routing | 03 - Mobile/Flutter/Flutter - Navigation & Routing.md | ✅ 2026-10-08
- [x] T120 | deep-dive | Flutter - Offline Storage & Sync | 03 - Mobile/Flutter/Flutter - Offline Storage & Sync.md | ✅ 2026-10-08

### Priority — Requested 2026-10-13 (backend interview question bank)

- [x] T121 | reference | Backend Interview Questions | 04 - Backend/Backend Interview Questions/Backend Interview Questions.md | ✅ 2026-10-13
- [x] T122 | reference | Spring Boot - Interview Questions | 04 - Backend/Spring Boot/Spring Boot - Interview Questions.md | ✅ 2026-10-14
- [x] T123 | reference | Redis - Interview Questions | 05 - Databases/Redis/Redis - Interview Questions.md | ✅ 2026-10-14
- [x] T124 | reference | NoSQL - Interview Questions | 05 - Databases/NoSQL/NoSQL - Interview Questions.md | ✅ 2026-10-14
- [x] T125 | reference | Authentication & Authorization - Interview Questions | 10 - Security/Authentication & Authorization/Authentication & Authorization - Interview Questions.md | ✅ 2026-10-14
- [x] T126 | reference | Design Patterns - Interview Questions | 11 - CS Fundamentals/Design Patterns/Design Patterns - Interview Questions.md | ✅ 2026-10-14
- [x] T127 | reference | System Design - Interview Questions | 11 - CS Fundamentals/System Design/System Design - Interview Questions.md | ✅ 2026-10-14

### Wave 1 — Core stack overviews

- [x] T001 | overview | Redis | 05 - Databases/Redis/Redis.md | ✅ 2026-09-28
- [x] T002 | overview | TypeScript | 01 - Languages/TypeScript/TypeScript.md | ✅ 2026-10-02
- [x] T003 | overview | JavaScript | 01 - Languages/JavaScript/JavaScript.md | ✅ 2026-10-02
- [x] T004 | overview | React | 02 - Frontend/React/React.md | ✅ 2026-10-02
- [x] T005 | overview | Next.js | 02 - Frontend/Next.js/Next.js.md | ✅ 2026-10-02
- [x] T006 | overview | PostgreSQL | 05 - Databases/PostgreSQL/PostgreSQL.md | ✅ 2026-10-03
- [x] T007 | overview | Docker | 07 - DevOps & Infrastructure/Docker/Docker.md | ✅ 2026-10-03
- [x] T008 | overview | n8n | 08 - Automation/n8n/n8n.md | ✅ 2026-10-03
- [x] T009 | overview | Coolify | 07 - DevOps & Infrastructure/Coolify/Coolify.md | ✅ 2026-10-03
- [x] T010 | overview | LLM Fundamentals | 13 - AI Engineering/LLM Fundamentals/LLM Fundamentals.md | ✅ 2026-10-03
- [x] T011 | overview | Prompt Engineering | 13 - AI Engineering/Prompt Engineering/Prompt Engineering.md | ✅ 2026-10-04
- [x] T012 | overview | RAG | 13 - AI Engineering/RAG/RAG.md | ✅ 2026-10-04
- [x] T013 | overview | AI Agents | 13 - AI Engineering/AI Agents/AI Agents.md | ✅ 2026-10-04
- [x] T014 | overview | Model Context Protocol | 13 - AI Engineering/Model Context Protocol/Model Context Protocol.md | ✅ 2026-10-04
- [x] T015 | overview | HTTP & HTTPS | 09 - Networking & APIs/HTTP & HTTPS/HTTP & HTTPS.md | ✅ 2026-10-04
- [x] T016 | overview | REST API | 09 - Networking & APIs/REST API/REST API.md | ✅ 2026-10-05
- [x] T017 | overview | WebSocket | 09 - Networking & APIs/WebSocket/WebSocket.md | ✅ 2026-10-05
- [x] T018 | overview | Big O Notation | 11 - CS Fundamentals/Big O Notation/Big O Notation.md | ✅ 2026-10-05
- [x] T019 | maint | Link audit + MOC sync (all domains touched so far) | - | ✅ 2026-10-05

### Wave 2 — Core stack deep dives

- [x] T020 | deep-dive | JavaScript - Event Loop & Async | 01 - Languages/JavaScript/JavaScript - Event Loop & Async.md | ✅ 2026-10-05
- [x] T021 | deep-dive | TypeScript - Advanced Types | 01 - Languages/TypeScript/TypeScript - Advanced Types.md | ✅ 2026-10-05
- [x] T022 | deep-dive | React - Hooks | 02 - Frontend/React/React - Hooks.md | ✅ 2026-10-05
- [x] T023 | deep-dive | React - Server Components | 02 - Frontend/React/React - Server Components.md | ✅ 2026-10-05
- [x] T024 | deep-dive | Next.js - App Router & Rendering Strategies | 02 - Frontend/Next.js/Next.js - App Router & Rendering Strategies.md | ✅ 2026-10-05
- [x] T025 | deep-dive | Next.js - Caching | 02 - Frontend/Next.js/Next.js - Caching.md | ✅ 2026-10-05
- [x] T026 | deep-dive | PostgreSQL - Indexing | 05 - Databases/PostgreSQL/PostgreSQL - Indexing.md | ✅ 2026-10-05
- [x] T027 | deep-dive | PostgreSQL - Transactions & MVCC | 05 - Databases/PostgreSQL/PostgreSQL - Transactions & MVCC.md | ✅ 2026-10-05
- [x] T028 | deep-dive | Redis - Data Structures & Patterns | 05 - Databases/Redis/Redis - Data Structures & Patterns.md | ✅ 2026-10-05
- [x] T029 | deep-dive | Redis - Persistence & High Availability | 05 - Databases/Redis/Redis - Persistence & High Availability.md | ✅ 2026-10-05
- [x] T030 | deep-dive | Docker - Dockerfile & Image Best Practices | 07 - DevOps & Infrastructure/Docker/Docker - Dockerfile & Image Best Practices.md | ✅ 2026-10-05
- [x] T031 | deep-dive | Docker - Networking & Volumes | 07 - DevOps & Infrastructure/Docker/Docker - Networking & Volumes.md | ✅ 2026-10-05
- [x] T032 | deep-dive | n8n - Queue Mode & Scaling | 08 - Automation/n8n/n8n - Queue Mode & Scaling.md | ✅ 2026-10-05
- [x] T033 | deep-dive | n8n - Error Handling & Workflow Patterns | 08 - Automation/n8n/n8n - Error Handling & Workflow Patterns.md | ✅ 2026-10-05
- [x] T034 | deep-dive | LLM Fundamentals - Tokens, Context & Sampling | 13 - AI Engineering/LLM Fundamentals/LLM Fundamentals - Tokens, Context & Sampling.md | ✅ 2026-10-05
- [x] T035 | deep-dive | RAG - Chunking & Retrieval Strategies | 13 - AI Engineering/RAG/RAG - Chunking & Retrieval Strategies.md | ✅ 2026-10-05
- [x] T036 | deep-dive | AI Agents - Tool Calling & Agent Loops | 13 - AI Engineering/AI Agents/AI Agents - Tool Calling & Agent Loops.md | ✅ 2026-10-05
- [x] T037 | deep-dive | HTTP & HTTPS - TLS Handshake & Certificates | 09 - Networking & APIs/HTTP & HTTPS/HTTP & HTTPS - TLS Handshake & Certificates.md | ✅ 2026-10-05
- [x] T038 | maint | Link audit + refresh version_checked on 3 oldest notes | - | ✅ 2026-10-05

### Wave 3 — Stack expansion overviews

- [x] T039 | overview | Python | 01 - Languages/Python/Python.md | ✅ 2026-10-05
- [x] T040 | overview | Go | 01 - Languages/Go/Go.md | ✅ 2026-10-05
- [x] T041 | overview | Kafka | 06 - Messaging & Streaming/Kafka/Kafka.md | ✅ 2026-10-06
- [x] T042 | overview | Supabase | 04 - Backend/Supabase/Supabase.md | ✅ 2026-10-06
- [x] T043 | overview | Flutter | 03 - Mobile/Flutter/Flutter.md | ✅ 2026-10-06
- [x] T044 | overview | Dart | 01 - Languages/Dart/Dart.md | ✅ 2026-10-06
- [x] T045 | overview | React Native | 03 - Mobile/React Native/React Native.md | ✅ 2026-10-06
- [x] T046 | overview | Node.js | 04 - Backend/Node.js/Node.js.md | ✅ 2026-10-08
- [x] T047 | overview | Kubernetes | 07 - DevOps & Infrastructure/Kubernetes/Kubernetes.md | ✅ 2026-10-08
- [x] T048 | overview | Network Security | 10 - Security/Network Security/Network Security.md | ✅ 2026-10-08
- [x] T049 | overview | Blockchain Fundamentals | 12 - Web3/Blockchain Fundamentals/Blockchain Fundamentals.md | ✅ 2026-10-08
- [x] T050 | overview | Ethereum & EVM | 12 - Web3/Ethereum & EVM/Ethereum & EVM.md | ✅ 2026-10-08
- [x] T051 | overview | Solidity | 12 - Web3/Solidity/Solidity.md | ✅ 2026-10-10
- [x] T052 | overview | Web3 Wallets | 12 - Web3/Web3 Wallets/Web3 Wallets.md | ✅ 2026-10-10
- [x] T053 | overview | MySQL | 05 - Databases/MySQL/MySQL.md | ✅ 2026-10-10
- [x] T054 | overview | NoSQL | 05 - Databases/NoSQL/NoSQL.md | ✅ 2026-10-10
- [x] T055 | overview | Vector Databases | 05 - Databases/Vector Databases/Vector Databases.md | ✅ 2026-10-10
- [x] T057 | overview | Traefik | 07 - DevOps & Infrastructure/Traefik/Traefik.md | ✅ 2026-10-11
- [x] T058 | overview | Cloudflare | 07 - DevOps & Infrastructure/Cloudflare/Cloudflare.md | ✅ 2026-10-11
- [x] T059 | overview | Windmill | 08 - Automation/Windmill/Windmill.md | ✅ 2026-10-11
- [x] T060 | maint | Link audit + MOC sync + orphan check | - | ✅ 2026-10-11

### Wave 4 — Breadth overviews

- [x] T061 | overview | Astro | 02 - Frontend/Astro/Astro.md | ✅ 2026-10-11
- [x] T062 | overview | WebGL | 02 - Frontend/WebGL/WebGL.md | ✅ 2026-10-11
- [x] T063 | overview | Rust | 01 - Languages/Rust/Rust.md | ✅ 2026-10-11
- [x] T064 | overview | Java | 01 - Languages/Java/Java.md | ✅ 2026-10-11
- [x] T065 | overview | Spring Boot | 04 - Backend/Spring Boot/Spring Boot.md | ✅ 2026-10-11
- [x] T066 | overview | Kotlin | 01 - Languages/Kotlin/Kotlin.md | ✅ 2026-10-11
- [x] T067 | overview | Swift | 01 - Languages/Swift/Swift.md | ✅ 2026-10-11
- [x] T068 | overview | PHP | 01 - Languages/PHP/PHP.md | ✅ 2026-10-11
- [x] T069 | overview | Git | 07 - DevOps & Infrastructure/Git/Git.md | ✅ 2026-10-11
- [x] T070 | overview | Linux Essentials | 07 - DevOps & Infrastructure/Linux Essentials/Linux Essentials.md | ✅ 2026-10-11
- [x] T071 | overview | CI-CD | 07 - DevOps & Infrastructure/CI-CD/CI-CD.md | ✅ 2026-10-11
- [x] T072 | overview | Nginx | 07 - DevOps & Infrastructure/Nginx/Nginx.md | ✅ 2026-10-12
- [x] T073 | overview | GraphQL | 09 - Networking & APIs/GraphQL/GraphQL.md | ✅ 2026-10-12
- [x] T074 | overview | gRPC | 09 - Networking & APIs/gRPC/gRPC.md | ✅ 2026-10-12
- [x] T075 | overview | Authentication & Authorization | 10 - Security/Authentication & Authorization/Authentication & Authorization.md | ✅ 2026-10-12
- [x] T076 | overview | OWASP Top 10 | 10 - Security/OWASP Top 10/OWASP Top 10.md | ✅ 2026-10-12
- [x] T077 | overview | Data Structures | 11 - CS Fundamentals/Data Structures/Data Structures.md | ✅ 2026-10-13
- [x] T078 | overview | Algorithms | 11 - CS Fundamentals/Algorithms/Algorithms.md | ✅ 2026-10-13
- [x] T079 | overview | System Design | 11 - CS Fundamentals/System Design/System Design.md | ✅ 2026-10-13
- [x] T080 | overview | Design Patterns | 11 - CS Fundamentals/Design Patterns/Design Patterns.md | ✅ 2026-10-13
- [x] T081 | maint | Link audit + refresh version_checked on 3 oldest notes | - | ✅ 2026-10-13
- [x] T082 | overview | Tailwind CSS | 02 - Frontend/Tailwind CSS/Tailwind CSS.md | ✅ 2026-10-13
- [x] T083 | overview | Browser & Web Fundamentals | 02 - Frontend/Browser & Web Fundamentals/Browser & Web Fundamentals.md | ✅ 2026-10-13
- [x] T084 | overview | Testing Strategy | 04 - Backend/Testing Strategy/Testing Strategy.md | ✅ 2026-10-13
- [x] T085 | overview | AI Evals & Observability | 13 - AI Engineering/AI Evals & Observability/AI Evals & Observability.md | ✅ 2026-10-13
- [x] T086 | overview | Fine-tuning LLMs | 13 - AI Engineering/Fine-tuning LLMs/Fine-tuning LLMs.md | ✅ 2026-10-13
- [x] T087 | overview | LLM APIs & SDKs | 13 - AI Engineering/LLM APIs & SDKs/LLM APIs & SDKs.md | ✅ 2026-10-14
- [x] T088 | overview | Message Queues | 06 - Messaging & Streaming/Message Queues/Message Queues.md | ✅ 2026-10-14

### Wave 5 — Expansion deep dives

- [x] T089 | deep-dive | Kafka - Partitions, Consumer Groups & Delivery Semantics | 06 - Messaging & Streaming/Kafka/Kafka - Partitions, Consumer Groups & Delivery Semantics.md | ✅ 2026-10-14
- [x] T090 | deep-dive | Go - Concurrency | 01 - Languages/Go/Go - Concurrency.md | ✅ 2026-10-14
- [x] T091 | deep-dive | Rust - Ownership & Borrowing | 01 - Languages/Rust/Rust - Ownership & Borrowing.md | ✅ 2026-10-14
- [x] T092 | deep-dive | Python - Async & Concurrency | 01 - Languages/Python/Python - Async & Concurrency.md | ✅ 2026-10-14
- [x] T095 | deep-dive | React Native - New Architecture | 03 - Mobile/React Native/React Native - New Architecture.md | ✅ 2026-10-14
- [x] T096 | deep-dive | Kubernetes - Core Objects | 07 - DevOps & Infrastructure/Kubernetes/Kubernetes - Core Objects.md | ✅ 2026-10-14
- [x] T097 | deep-dive | Network Security - Common Web Attacks | 10 - Security/Network Security/Network Security - Common Web Attacks.md | ✅ 2026-10-14
- [x] T098 | deep-dive | Solidity - Smart Contract Security | 12 - Web3/Solidity/Solidity - Smart Contract Security.md | ✅ 2026-10-14
- [x] T099 | deep-dive | Web3 Wallets - Key Management & Account Abstraction | 12 - Web3/Web3 Wallets/Web3 Wallets - Key Management & Account Abstraction.md | ✅ 2026-10-15
- [x] T100 | deep-dive | WebSocket - Scaling & Reliability | 09 - Networking & APIs/WebSocket/WebSocket - Scaling & Reliability.md | ✅ 2026-10-15
- [x] T101 | deep-dive | Supabase - Row Level Security | 04 - Backend/Supabase/Supabase - Row Level Security.md | ✅ 2026-10-15
- [x] T102 | deep-dive | PostgreSQL - Query Planning & EXPLAIN | 05 - Databases/PostgreSQL/PostgreSQL - Query Planning & EXPLAIN.md | ✅ 2026-10-15
- [x] T103 | maint | Link audit + MOC sync + orphan check | - | ✅ 2026-10-15
- [x] T104 | replenish | Next 12 tasks per CLAUDE.md §9 | - | ✅ 2026-10-16

### Wave 6 — Stack deep dives (ZP's production stack first)

- [x] T128 | deep-dive | Coolify - Operations, Backups & Hardening | 07 - DevOps & Infrastructure/Coolify/Coolify - Operations, Backups & Hardening.md | ✅ 2026-10-16
- [x] T129 | deep-dive | Traefik - Middlewares, TLS & Routing | 07 - DevOps & Infrastructure/Traefik/Traefik - Middlewares, TLS & Routing.md | ✅ 2026-10-16
- [x] T130 | deep-dive | Cloudflare - Zero Trust Tunnel & Access | 07 - DevOps & Infrastructure/Cloudflare/Cloudflare - Zero Trust Tunnel & Access.md | ✅ 2026-10-16
- [x] T131 | deep-dive | Node.js - Performance & Diagnostics | 04 - Backend/Node.js/Node.js - Performance & Diagnostics.md | ✅ 2026-10-16
- [x] T132 | deep-dive | REST API - Webhooks & Idempotency | 09 - Networking & APIs/REST API/REST API - Webhooks & Idempotency.md | ✅ 2026-10-16
- [x] T133 | deep-dive | LLM APIs & SDKs - Prompt Caching & Cost Optimization | 13 - AI Engineering/LLM APIs & SDKs/LLM APIs & SDKs - Prompt Caching & Cost Optimization.md | ✅ 2026-10-16
- [x] T134 | deep-dive | Model Context Protocol - Building MCP Servers | 13 - AI Engineering/Model Context Protocol/Model Context Protocol - Building MCP Servers.md | ✅ 2026-10-16
- [x] T135 | deep-dive | Vector Databases - pgvector Tuning & Hybrid Search | 05 - Databases/Vector Databases/Vector Databases - pgvector Tuning & Hybrid Search.md | ✅ 2026-10-16
- [x] T136 | deep-dive | Authentication & Authorization - Passkeys & WebAuthn | 10 - Security/Authentication & Authorization/Authentication & Authorization - Passkeys & WebAuthn.md | ✅ 2026-10-17
- [x] T137 | deep-dive | CI-CD - GitHub Actions Security Hardening | 07 - DevOps & Infrastructure/CI-CD/CI-CD - GitHub Actions Security Hardening.md | ✅ 2026-10-17
- [x] T138 | maint | Link audit + refresh version_checked on 3 oldest notes | - | ✅ 2026-10-17
- [x] T139 | replenish | Next 12 tasks per CLAUDE.md §9 | - | ✅ 2026-10-17

### Wave 7 — Ops, auth & quality deep dives

- [x] T140 | deep-dive | Message Queues - BullMQ in Production | 06 - Messaging & Streaming/Message Queues/Message Queues - BullMQ in Production.md | ✅ 2026-10-17
- [x] T141 | deep-dive | Linux Essentials - VPS Hardening Runbook | 07 - DevOps & Infrastructure/Linux Essentials/Linux Essentials - VPS Hardening Runbook.md | ✅ 2026-10-18
- [x] T142 | deep-dive | Testing Strategy - Integration Testing with Testcontainers & Supabase | 04 - Backend/Testing Strategy/Testing Strategy - Integration Testing with Testcontainers & Supabase.md | ✅ 2026-10-18
- [x] T143 | deep-dive | JWT - Signing Algorithms, Keys & Rotation | 10 - Security/JWT/JWT - Signing Algorithms, Keys & Rotation.md | ✅ 2026-10-18
- [x] T144 | deep-dive | OAuth 2.0 & OIDC - Flows, PKCE & Token Lifecycle | 10 - Security/OAuth 2.0 & OIDC/OAuth 2.0 & OIDC - Flows, PKCE & Token Lifecycle.md | ✅ 2026-10-18
- [x] T145 | deep-dive | Browser & Web Fundamentals - Core Web Vitals Optimization | 02 - Frontend/Browser & Web Fundamentals/Browser & Web Fundamentals - Core Web Vitals Optimization.md | ✅ 2026-10-18
- [x] T146 | deep-dive | AI Evals & Observability - Eval Datasets & LLM-as-Judge | 13 - AI Engineering/AI Evals & Observability/AI Evals & Observability - Eval Datasets & LLM-as-Judge.md | ✅ 2026-10-18
- [x] T147 | deep-dive | OWASP Top 10 - LLM & Agent Security | 10 - Security/OWASP Top 10/OWASP Top 10 - LLM & Agent Security.md | ✅ 2026-10-18
- [x] T148 | deep-dive | Git - Branching, Releases & Recovery | 07 - DevOps & Infrastructure/Git/Git - Branching, Releases & Recovery.md | ✅ 2026-10-18
- [x] T149 | overview | Valkey | 05 - Databases/Valkey/Valkey.md | ✅ 2026-10-18
- [x] T150 | maint | Link audit + MOC sync + refresh version_checked on 3 oldest notes | - | ✅ 2026-10-18
- [x] T151 | replenish | Next 12 tasks per CLAUDE.md §9 | - | ✅ 2026-10-19

### Wave 8 — SaaS architecture, client ops & breadth deep dives

- [x] T152 | deep-dive | System Design - Multi-Tenant SaaS Architecture | 11 - CS Fundamentals/System Design/System Design - Multi-Tenant SaaS Architecture.md | ✅ 2026-10-19
- [x] T153 | deep-dive | Valkey - Redis to Valkey Migration Runbook | 05 - Databases/Valkey/Valkey - Redis to Valkey Migration Runbook.md | ✅ 2026-10-19
- [x] T154 | deep-dive | PHP - WordPress Hardening & Maintenance SOP | 01 - Languages/PHP/PHP - WordPress Hardening & Maintenance SOP.md | ✅ 2026-10-19
- [x] T155 | deep-dive | JWT - Refresh Tokens, Revocation & Storage | 10 - Security/JWT/JWT - Refresh Tokens, Revocation & Storage.md | ✅ 2026-10-19
- [x] T156 | deep-dive | Tailwind CSS - Theming & Component Architecture | 02 - Frontend/Tailwind CSS/Tailwind CSS - Theming & Component Architecture.md | ✅ 2026-10-19
- [x] T157 | deep-dive | Windmill - Windmill vs n8n Architecture Decision | 08 - Automation/Windmill/Windmill - Windmill vs n8n Architecture Decision.md | ✅ 2026-10-19
- [x] T158 | deep-dive | Design Patterns - State Machines for Business Workflows | 11 - CS Fundamentals/Design Patterns/Design Patterns - State Machines for Business Workflows.md | ✅ 2026-10-19
- [x] T159 | deep-dive | Astro - Islands & Content Layer | 02 - Frontend/Astro/Astro - Islands & Content Layer.md | ✅ 2026-10-19
- [x] T160 | deep-dive | GraphQL - Security Hardening | 09 - Networking & APIs/GraphQL/GraphQL - Security Hardening.md | ✅ 2026-10-19
- [x] T161 | deep-dive | Ethereum & EVM - Gas Optimization & EVM Internals | 12 - Web3/Ethereum & EVM/Ethereum & EVM - Gas Optimization & EVM Internals.md | ✅ 2026-10-19
- [ ] T162 | maint | Link audit + MOC sync + refresh version_checked on 3 oldest notes | -
- [ ] T163 | replenish | Next 12 tasks per CLAUDE.md §9 | -
