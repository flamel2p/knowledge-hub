---
title: Testing Strategy
aliases: [test strategy, test pyramid, testing trophy, unit testing, integration testing, E2E testing, Vitest, Playwright, Testcontainers]
type: overview
domain: backend
tags: [domain/backend, type/overview, topic/testing]
status: draft
created: 2026-10-13
updated: 2026-10-30
version_checked: "Vitest 4.x · Jest 30 · Playwright 1.5x — 2026-10"
parent: "[[Backend MOC]]"
related: ["[[CI-CD]]", "[[Flutter - Testing]]", "[[PostgreSQL]]", "[[Supabase]]", "[[Node.js]]", "[[AI Evals & Observability]]"]
---

# Testing Strategy

> [!abstract] TL;DR
> A testing strategy decides **what to test, at which level, with which tools, and when it runs**, so you can ship changes to client systems confidently and quickly.
> - **Modern consensus ("testing trophy")**: static checks (types, lint) at the base, a thick layer of **integration tests against real dependencies** ([[PostgreSQL]] via Testcontainers or local Supabase), focused unit tests for pure logic, and a few **E2E smoke tests** (Playwright) of money paths.
> - **For a solo founder**: tests are leverage and liability protection. Prioritize authorization, payments, data integrity and integrations over coverage percentages.

## Introduction
- **Lineage**: xUnit (SUnit 1994, JUnit 1997), TDD (Kent Beck, early 2000s), Mike Cohn's test pyramid (2009), Kent C. Dodds' testing trophy (2018), contract testing (Pact), property-based testing (QuickCheck, 1999).
- **Problem**: regressions in code nobody remembers, fear of refactoring, and silent breakage when dependencies or third-party APIs change.
- **Where it sits**: in the inner dev loop (watch mode), pre-commit hooks, [[CI-CD]] gates on every PR, and post-deploy smoke checks and monitoring (testing in production).

## Core Concepts

### Test levels
| Level | Scope | Speed | Doubles | Tools (TS / Python / Dart) |
|---|---|---|---|---|
| Static | Types, lint, formatting | Instant | — | `tsc`, ESLint/Biome, Ruff/mypy, `dart analyze` |
| **Unit** | Pure functions, domain logic | ms | Fakes for ports | Vitest/Jest/`node:test`, pytest, `package:test` |
| **Integration** | Module + real DB/queue/HTTP stub | 100 ms–s | Real Postgres/Redis (containers), HTTP mocked at the boundary (MSW, nock) | Vitest + Testcontainers, pytest + testcontainers, Supabase local |
| Contract | API producer ↔ consumer agreement | Fast | Generated stubs | Pact, OpenAPI schema validation (Schemathesis) |
| **E2E** | Full system through UI/API | Seconds–minutes | Staging services | Playwright, `integration_test`/Patrol ([[Flutter - Testing]]) |
| Non-functional | Load, security, accessibility | Varies | — | k6, OWASP ZAP, axe-core |

### Test doubles
| Double | What | Use |
|---|---|---|
| Fake | Working lightweight implementation (in-memory repo) | Preferred for ports you own |
| Stub | Returns canned answers | Third-party API responses |
| Mock | Verifies interactions | Side effects that must happen (email sent once) |
| Spy | Records calls on real objects | Observability of calls |

Rule: mock what you **don't own** at the boundary (HTTP to Stripe/Meta), and use real instances of what you **do own** (your DB schema, your queries).

### Integration test with a real Postgres
```ts
// orders.int.test.ts (Vitest + Testcontainers)
import { PostgreSqlContainer, type StartedPostgreSqlContainer } from '@testcontainers/postgresql';
import { beforeAll, afterAll, test, expect } from 'vitest';

let pg: StartedPostgreSqlContainer;
beforeAll(async () => {
  pg = await new PostgreSqlContainer('postgres:17').start();
  await migrate(pg.getConnectionUri());                    // run real migrations
}, 60_000);
afterAll(() => pg.stop());

test('markPaid is idempotent per idempotency key', async () => {
  const db = connect(pg.getConnectionUri());
  const order = await createOrder(db, { tenantId: 't1', totalSen: 5000 });
  await markPaid(db, order.id, 'idem-123');
  await markPaid(db, order.id, 'idem-123');                // retry
  expect(await countPayments(db, order.id)).toBe(1);
});
```

### Authorization matrix test (highest-value tests in SaaS)
```ts
test.each([
  ['owner',   't1', 200],
  ['cashier', 't1', 403],
  ['owner',   't2', 404],      // other tenant: must not leak existence
])('PATCH /orders/:id as %s of %s → %d', async (role, tenant, status) => {
  const res = await api.as({ role, tenant }).patch(`/orders/${t1Order.id}`, { status: 'paid' });
  expect(res.status).toBe(status);
});
```

### Property-based testing
```ts
import fc from 'fast-check';
test('money split always sums to the total', () => {
  fc.assert(fc.property(fc.integer({ min: 0, max: 10_000_000 }), fc.integer({ min: 1, max: 20 }), (sen, parts) => {
    const shares = splitSen(sen, parts);
    return shares.length === parts && shares.reduce((a, b) => a + b, 0) === sen;
  }));
});
```

## Architecture / How It Works
```mermaid
flowchart LR
  DEV[Dev: watch mode, unit + integration] --> PC[Pre-commit: lint, typecheck, fast tests]
  PC --> PR[PR CI: full unit + integration with containers, contract checks, coverage on changed files]
  PR --> STG[Staging: E2E smoke (Playwright), migrations dry-run]
  STG --> PROD[Prod: synthetic checks, error monitoring, canary]
  PROD -. bugs become regression tests .-> DEV
```
- **Fast feedback first**: keep the PR pipeline under ~10 min. Parallelize (Vitest threads, Playwright shards), and reuse containers per worker.
- **Determinism**: control time (fake timers / injected clock), randomness (seeds), network (no real third-party calls) and data (factories per test, transactions rolled back, or a fresh schema).
- **Database testing**: run migrations in tests. Test RLS policies at the DB level (pgTAP / `supabase test db`), since app-level tests can bypass them with service keys.

## Project Structure
```text
api/
├── src/
├── test/
│   ├── unit/                 # *.test.ts: pure logic, no I/O
│   ├── integration/          # *.int.test.ts: Testcontainers Postgres/Redis, MSW for external HTTP
│   ├── contract/             # OpenAPI/Pact checks
│   ├── factories/            # test data builders (fishery, faker with fixed seed)
│   └── setup.ts
├── e2e/                      # Playwright: login → create order → pay (staging)
├── supabase/tests/*.sql      # pgTAP RLS tests
└── vitest.config.ts          # projects: unit (fast), integration (pool: forks, longer timeout)
```
```bash
vitest --project unit --watch
vitest run --project integration
npx playwright test --project=chromium --shard=1/4
supabase test db
```

## Use Cases
| Risk area | Test approach |
|---|---|
| Multi-tenant data access | Authorization matrix + RLS pgTAP tests |
| Payments, invoices, money math | Integration tests with idempotency + property tests on rounding/splitting |
| Webhooks (Meta, payment gateways) | Contract fixtures from real payloads, signature verification tests, replay/duplicate tests |
| n8n workflows / integrations | Pin test webhooks, staging workflows, assert outputs. Export workflows to Git |
| Migrations | Up/down on a copy of production schema, backward-compatibility for rolling deploys |
| Critical user journeys | Playwright E2E smoke on staging after each deploy |
| LLM features | Eval suites with datasets and graders (see [[AI Evals & Observability]]) |

## Pros & Cons
| Approach | Pros | Cons |
|---|---|---|
| Heavy unit testing with mocks | Fast, precise failures | Tests implementation details. False confidence |
| Integration with real dependencies | High confidence, catches SQL/config bugs | Slower, needs Docker in CI |
| E2E | Tests what users do | Slow, flaky, expensive to maintain |
| TDD | Better design, regression safety | Slower start. Hard for exploratory work |
| Coverage targets | Visibility | Gaming. 100% coverage ≠ correctness |

## Alternatives & Peers
| Tool | Strength | Pick it when… |
|---|---|---|
| **Vitest** | Fast, Vite-native, Jest-compatible API, browser mode | TS/JS projects (default) |
| Jest 30 | Mature ecosystem | Existing Jest suites, React Native |
| `node:test` | Zero deps, built in | Small Node libraries |
| **Playwright** | Cross-browser E2E, auto-wait, tracing, API testing | Web E2E (default) |
| Cypress | Great DX for frontend devs | Existing Cypress suites |
| pytest / Pest / JUnit 5 | Language standards | Python / PHP / JVM services |
| k6 / Artillery | Scriptable load tests | Capacity planning, SLO validation |

## Tips & Reminders
> [!tip]
> - Write tests for **behaviour at boundaries** (HTTP API responses, DB state, emitted events), not private functions.
> - Every production bug gets a failing regression test first.
> - Quarantine flaky tests quickly with an owner and deadline, and fix root causes (timing, shared state, real network). Never retry-until-green silently.
> - Use test data factories with explicit tenant/user IDs. Avoid shared global fixtures.
> - Measure coverage on changed lines in PRs rather than chasing a global percentage.
> - **In ZP's stack**:
>   - Vitest + Testcontainers (or `supabase start`) for Node/Next.js APIs, and pgTAP for RLS.
>   - Playwright smoke tests on staging via GitHub Actions ([[CI-CD]]), `flutter test` for apps ([[Flutter - Testing]]).
>   - A staging n8n instance with test credentials.
>   - Put "test coverage of critical flows" in SOWs as a deliverable.

## Versions & Breaking Changes
| Tool / version | Released | Key changes | Migration notes |
|---|---|---|---|
| Testcontainers (Node/Java/Python/Go) | ongoing | Language modules, reusable containers | Needs Docker in CI (GitHub-hosted runners have it) |
| Jest 30 | 2025-06 | Faster, ESM improvements, dropped old Node versions | Check deprecated matchers/config |
| Vitest 3 / **4** | 2025-01 / 2025-10 | Projects (workspace) config, **browser mode stable** (4), improved coverage (v8 AST-based) | `workspace` → `projects`. Review reporter/coverage options |
| Playwright 1.4x–1.5x | 2024–2026 | UI mode, component tests, clock API, aria snapshots | Pin browsers via `npx playwright install` in CI |
| Node `node:test` | Node 20–24 | Stable runner, mocking, snapshots, coverage | — |

> [!warning] Unverified — check before relying on this
> Tool release dates weren't re-checked this run. Check https://vitest.dev/blog and https://playwright.dev/docs/release-notes.

## Critical Issues & Gotchas
> [!danger] Tests that bypass the real security boundary
> Integration tests using the Supabase **service-role key** or a DB superuser skip RLS, so "all tests pass" while tenant isolation is broken in production. Run authorization tests with real user JWTs/anon keys and dedicated pgTAP RLS tests.

> [!danger] Mocked third parties drifting from reality
> Hand-written mocks of Meta WhatsApp, payment gateways or LLM APIs drift as providers change payloads (new fields, changed error codes, signature schemes). Record real sandbox payloads as fixtures, validate against provider schemas, and run a scheduled smoke test against sandbox environments to catch drift.

> [!warning] Gotchas
> - Tests that hit production APIs or send real WhatsApp messages/emails. Isolate credentials per environment.
> - Time zones and DST: run CI with `TZ=UTC` and also test `Asia/Kuala_Lumpur` formatting explicitly.
> - Shared DB state between parallel tests causes flakiness. Use a schema/database per worker or transactions.
> - Snapshot tests of large outputs get rubber-stamped on update. Keep them small and intentional.
> - E2E tests depending on fixed sleeps (`waitForTimeout`) are flaky. Use web-first assertions.

## Deep Dives
- [[Testing Strategy - Integration Testing with Testcontainers & Supabase]]
- [[Testing Strategy - E2E Testing with Playwright]] — storageState auth, API-seeded fixtures, route mocking, sharded CI, deploy smoke tests
- [[Testing Strategy - Unit Testing with Vitest & Mocking Boundaries]] — fakes at ports, MSW for HTTP, fake timers in MYT, property-based money tests, Vitest 4 projects + browser mode

## Related
- [[CI-CD]] — where tests gate merges and deploys
- [[Flutter - Testing]] — mobile-specific testing
- [[PostgreSQL]] · [[Supabase]] — DB-level and RLS tests
- [[Node.js]] — runtime for most of ZP's backend tests
- [[AI Evals & Observability]] — testing non-deterministic LLM features
- [[Authentication & Authorization]] — authorization test matrices

## References
- Testing Trophy (Kent C. Dodds): https://kentcdodds.com/blog/the-testing-trophy-and-testing-classifications
- Vitest: https://vitest.dev/
- Playwright: https://playwright.dev/
- Testcontainers: https://testcontainers.com/
- Supabase database testing (pgTAP): https://supabase.com/docs/guides/database/testing
- fast-check: https://fast-check.dev/
