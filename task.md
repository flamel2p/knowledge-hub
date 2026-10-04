# Task Queue

Last run: 2026-10-04 (T011–T015)

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
- [ ] T016 | overview | REST API | 09 - Networking & APIs/REST API/REST API.md
- [ ] T017 | overview | WebSocket | 09 - Networking & APIs/WebSocket/WebSocket.md
- [ ] T018 | overview | Big O Notation | 11 - CS Fundamentals/Big O Notation/Big O Notation.md
- [ ] T019 | maint | Link audit + MOC sync (all domains touched so far) | -

### Wave 2 — Core stack deep dives

- [ ] T020 | deep-dive | JavaScript - Event Loop & Async | 01 - Languages/JavaScript/JavaScript - Event Loop & Async.md
- [ ] T021 | deep-dive | TypeScript - Advanced Types | 01 - Languages/TypeScript/TypeScript - Advanced Types.md
- [ ] T022 | deep-dive | React - Hooks | 02 - Frontend/React/React - Hooks.md
- [ ] T023 | deep-dive | React - Server Components | 02 - Frontend/React/React - Server Components.md
- [ ] T024 | deep-dive | Next.js - App Router & Rendering Strategies | 02 - Frontend/Next.js/Next.js - App Router & Rendering Strategies.md
- [ ] T025 | deep-dive | Next.js - Caching | 02 - Frontend/Next.js/Next.js - Caching.md
- [ ] T026 | deep-dive | PostgreSQL - Indexing | 05 - Databases/PostgreSQL/PostgreSQL - Indexing.md
- [ ] T027 | deep-dive | PostgreSQL - Transactions & MVCC | 05 - Databases/PostgreSQL/PostgreSQL - Transactions & MVCC.md
- [ ] T028 | deep-dive | Redis - Data Structures & Patterns | 05 - Databases/Redis/Redis - Data Structures & Patterns.md
- [ ] T029 | deep-dive | Redis - Persistence & High Availability | 05 - Databases/Redis/Redis - Persistence & High Availability.md
- [ ] T030 | deep-dive | Docker - Dockerfile & Image Best Practices | 07 - DevOps & Infrastructure/Docker/Docker - Dockerfile & Image Best Practices.md
- [ ] T031 | deep-dive | Docker - Networking & Volumes | 07 - DevOps & Infrastructure/Docker/Docker - Networking & Volumes.md
- [ ] T032 | deep-dive | n8n - Queue Mode & Scaling | 08 - Automation/n8n/n8n - Queue Mode & Scaling.md
- [ ] T033 | deep-dive | n8n - Error Handling & Workflow Patterns | 08 - Automation/n8n/n8n - Error Handling & Workflow Patterns.md
- [ ] T034 | deep-dive | LLM Fundamentals - Tokens, Context & Sampling | 13 - AI Engineering/LLM Fundamentals/LLM Fundamentals - Tokens, Context & Sampling.md
- [ ] T035 | deep-dive | RAG - Chunking & Retrieval Strategies | 13 - AI Engineering/RAG/RAG - Chunking & Retrieval Strategies.md
- [ ] T036 | deep-dive | AI Agents - Tool Calling & Agent Loops | 13 - AI Engineering/AI Agents/AI Agents - Tool Calling & Agent Loops.md
- [ ] T037 | deep-dive | HTTP & HTTPS - TLS Handshake & Certificates | 09 - Networking & APIs/HTTP & HTTPS/HTTP & HTTPS - TLS Handshake & Certificates.md
- [ ] T038 | maint | Link audit + refresh version_checked on 3 oldest notes | -

### Wave 3 — Stack expansion overviews

- [ ] T039 | overview | Python | 01 - Languages/Python/Python.md
- [ ] T040 | overview | Go | 01 - Languages/Go/Go.md
- [ ] T041 | overview | Kafka | 06 - Messaging & Streaming/Kafka/Kafka.md
- [ ] T042 | overview | Supabase | 04 - Backend/Supabase/Supabase.md
- [ ] T043 | overview | Flutter | 03 - Mobile/Flutter/Flutter.md
- [ ] T044 | overview | Dart | 01 - Languages/Dart/Dart.md
- [ ] T045 | overview | React Native | 03 - Mobile/React Native/React Native.md
- [ ] T046 | overview | Node.js | 04 - Backend/Node.js/Node.js.md
- [ ] T047 | overview | Kubernetes | 07 - DevOps & Infrastructure/Kubernetes/Kubernetes.md
- [ ] T048 | overview | Network Security | 10 - Security/Network Security/Network Security.md
- [ ] T049 | overview | Blockchain Fundamentals | 12 - Web3/Blockchain Fundamentals/Blockchain Fundamentals.md
- [ ] T050 | overview | Ethereum & EVM | 12 - Web3/Ethereum & EVM/Ethereum & EVM.md
- [ ] T051 | overview | Solidity | 12 - Web3/Solidity/Solidity.md
- [ ] T052 | overview | Web3 Wallets | 12 - Web3/Web3 Wallets/Web3 Wallets.md
- [ ] T053 | overview | MySQL | 05 - Databases/MySQL/MySQL.md
- [ ] T054 | overview | NoSQL | 05 - Databases/NoSQL/NoSQL.md
- [ ] T055 | overview | Vector Databases | 05 - Databases/Vector Databases/Vector Databases.md
- [ ] T057 | overview | Traefik | 07 - DevOps & Infrastructure/Traefik/Traefik.md
- [ ] T058 | overview | Cloudflare | 07 - DevOps & Infrastructure/Cloudflare/Cloudflare.md
- [ ] T059 | overview | Windmill | 08 - Automation/Windmill/Windmill.md
- [ ] T060 | maint | Link audit + MOC sync + orphan check | -

### Wave 4 — Breadth overviews

- [ ] T061 | overview | Astro | 02 - Frontend/Astro/Astro.md
- [ ] T062 | overview | WebGL | 02 - Frontend/WebGL/WebGL.md
- [ ] T063 | overview | Rust | 01 - Languages/Rust/Rust.md
- [ ] T064 | overview | Java | 01 - Languages/Java/Java.md
- [ ] T065 | overview | Spring Boot | 04 - Backend/Spring Boot/Spring Boot.md
- [ ] T066 | overview | Kotlin | 01 - Languages/Kotlin/Kotlin.md
- [ ] T067 | overview | Swift | 01 - Languages/Swift/Swift.md
- [ ] T068 | overview | PHP | 01 - Languages/PHP/PHP.md
- [ ] T069 | overview | Git | 07 - DevOps & Infrastructure/Git/Git.md
- [ ] T070 | overview | Linux Essentials | 07 - DevOps & Infrastructure/Linux Essentials/Linux Essentials.md
- [ ] T071 | overview | CI-CD | 07 - DevOps & Infrastructure/CI-CD/CI-CD.md
- [ ] T072 | overview | Nginx | 07 - DevOps & Infrastructure/Nginx/Nginx.md
- [ ] T073 | overview | GraphQL | 09 - Networking & APIs/GraphQL/GraphQL.md
- [ ] T074 | overview | gRPC | 09 - Networking & APIs/gRPC/gRPC.md
- [ ] T075 | overview | Authentication & Authorization | 10 - Security/Authentication & Authorization/Authentication & Authorization.md
- [ ] T076 | overview | OWASP Top 10 | 10 - Security/OWASP Top 10/OWASP Top 10.md
- [ ] T077 | overview | Data Structures | 11 - CS Fundamentals/Data Structures/Data Structures.md
- [ ] T078 | overview | Algorithms | 11 - CS Fundamentals/Algorithms/Algorithms.md
- [ ] T079 | overview | System Design | 11 - CS Fundamentals/System Design/System Design.md
- [ ] T080 | overview | Design Patterns | 11 - CS Fundamentals/Design Patterns/Design Patterns.md
- [ ] T081 | maint | Link audit + refresh version_checked on 3 oldest notes | -
- [ ] T082 | overview | Tailwind CSS | 02 - Frontend/Tailwind CSS/Tailwind CSS.md
- [ ] T083 | overview | Browser & Web Fundamentals | 02 - Frontend/Browser & Web Fundamentals/Browser & Web Fundamentals.md
- [ ] T084 | overview | Testing Strategy | 04 - Backend/Testing Strategy/Testing Strategy.md
- [ ] T085 | overview | AI Evals & Observability | 13 - AI Engineering/AI Evals & Observability/AI Evals & Observability.md
- [ ] T086 | overview | Fine-tuning LLMs | 13 - AI Engineering/Fine-tuning LLMs/Fine-tuning LLMs.md
- [ ] T087 | overview | LLM APIs & SDKs | 13 - AI Engineering/LLM APIs & SDKs/LLM APIs & SDKs.md
- [ ] T088 | overview | Message Queues | 06 - Messaging & Streaming/Message Queues/Message Queues.md

### Wave 5 — Expansion deep dives

- [ ] T089 | deep-dive | Kafka - Partitions, Consumer Groups & Delivery Semantics | 06 - Messaging & Streaming/Kafka/Kafka - Partitions, Consumer Groups & Delivery Semantics.md
- [ ] T090 | deep-dive | Go - Concurrency | 01 - Languages/Go/Go - Concurrency.md
- [ ] T091 | deep-dive | Rust - Ownership & Borrowing | 01 - Languages/Rust/Rust - Ownership & Borrowing.md
- [ ] T092 | deep-dive | Python - Async & Concurrency | 01 - Languages/Python/Python - Async & Concurrency.md
- [ ] T093 | deep-dive | Flutter - State Management | 03 - Mobile/Flutter/Flutter - State Management.md
- [ ] T094 | deep-dive | Flutter - Rendering & Widget Lifecycle | 03 - Mobile/Flutter/Flutter - Rendering & Widget Lifecycle.md
- [ ] T095 | deep-dive | React Native - New Architecture | 03 - Mobile/React Native/React Native - New Architecture.md
- [ ] T096 | deep-dive | Kubernetes - Core Objects | 07 - DevOps & Infrastructure/Kubernetes/Kubernetes - Core Objects.md
- [ ] T097 | deep-dive | Network Security - Common Web Attacks | 10 - Security/Network Security/Network Security - Common Web Attacks.md
- [ ] T098 | deep-dive | Solidity - Smart Contract Security | 12 - Web3/Solidity/Solidity - Smart Contract Security.md
- [ ] T099 | deep-dive | Web3 Wallets - Key Management & Account Abstraction | 12 - Web3/Web3 Wallets/Web3 Wallets - Key Management & Account Abstraction.md
- [ ] T100 | deep-dive | WebSocket - Scaling & Reliability | 09 - Networking & APIs/WebSocket/WebSocket - Scaling & Reliability.md
- [ ] T101 | deep-dive | Supabase - Row Level Security | 04 - Backend/Supabase/Supabase - Row Level Security.md
- [ ] T102 | deep-dive | PostgreSQL - Query Planning & EXPLAIN | 05 - Databases/PostgreSQL/PostgreSQL - Query Planning & EXPLAIN.md
- [ ] T103 | maint | Link audit + MOC sync + orphan check | -
- [ ] T104 | replenish | Next 12 tasks per CLAUDE.md §9 | -
