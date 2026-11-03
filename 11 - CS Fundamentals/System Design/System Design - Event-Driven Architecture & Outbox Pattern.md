---
title: System Design - Event-Driven Architecture & Outbox Pattern
aliases: [event-driven architecture, transactional outbox, inbox pattern, idempotent consumer, dual write problem, domain events, CDC, Debezium, saga, choreography, orchestration, SKIP LOCKED relay]
type: deep-dive
domain: cs-fundamentals
tags: [domain/cs-fundamentals, type/deep-dive, topic/system-design, topic/event-driven, topic/outbox, lang/sql]
status: draft
created: 2026-11-03
updated: 2026-11-03
version_checked: "n/a (pattern) — PostgreSQL 18 (SKIP LOCKED, logical replication), pgmq, Debezium referenced — 2026-11"
parent: "[[System Design]]"
related: ["[[System Design - Reliability Patterns]]", "[[PostgreSQL - Transactions & MVCC]]", "[[Supabase - Edge Functions, Cron & Queues]]", "[[Message Queues - Choosing BullMQ, pgmq, RabbitMQ or NATS]]", "[[Design Patterns - State Machines for Business Workflows]]", "[[REST API - Webhooks & Idempotency]]"]
---

# System Design - Event-Driven Architecture & Outbox Pattern

> [!info] Deep dive of [[System Design]]

> [!abstract] TL;DR
> Writing to the DB and then publishing to a queue/webhook/n8n is a **dual write**: a crash in between loses the event (or publishes one for a rolled-back change).
> - **Fix**: insert the event into an **outbox table in the same transaction** as the business change. A **relay** (polling with `FOR UPDATE SKIP LOCKED`, or CDC) publishes it afterwards, **at least once**.
> - **Consumers**: make them **idempotent** with an **inbox** table keyed by event ID.
> - **Ordering**: per aggregate (order, conversation), not globally.
> - **For SME systems**: a Postgres outbox + pgmq/BullMQ beats Kafka and event sourcing in cost, ops and debuggability.

## Concept
| Term | Meaning |
|---|---|
| Domain event | Something that happened in the business (`order.paid`), past tense, immutable |
| Command | A request to do something (`send_receipt`), may fail |
| Event notification | Small event, consumers fetch details (thin) |
| Event-carried state transfer | Event includes the data consumers need (fat) |
| Event sourcing | State *is* the event log (rebuild by replay). Powerful, heavy |
| Outbox | Events written in the producer's DB transaction, relayed later |
| Inbox | Consumer records processed event IDs to dedupe |
| Saga | Multi-step business process across services with compensations |

**Relay options**:

| Relay | Latency | Ops | Ordering | Pick it when… |
|---|---|---|---|---|
| Polling worker (`SKIP LOCKED`, every 0.5–2 s) | ~1 s | Lowest | Per aggregate with care | Default for Postgres/Supabase apps |
| Polling + `LISTEN/NOTIFY` wake-up | ~10–100 ms | Low | Same | Lower latency, NOTIFY only as a hint |
| pgmq as the outbox (`pgmq.send` in the txn) | ~poll interval | Low (extension) | Queue semantics | Supabase Queues already in use |
| CDC (Debezium / logical replication) | ~100 ms | High (Kafka/Connect or slots) | WAL order | Many consumers, analytics, existing Kafka |

## How It Works
```mermaid
sequenceDiagram
  participant API as API / Server Action
  participant DB as Postgres
  participant R as Relay worker
  participant Q as Queue (BullMQ / pgmq / n8n webhook)
  participant C as Consumer
  API->>DB: BEGIN; update orders set status='paid'; insert outbox(order.paid); COMMIT
  R->>DB: select … from outbox where published_at is null for update skip locked limit 100
  R->>Q: publish(event_id, payload)
  R->>DB: update outbox set published_at = now()
  Q->>C: deliver (at least once)
  C->>DB: insert into inbox(event_id, consumer) on conflict do nothing
  alt new event
    C->>C: handle (send receipt, update stats)
  else duplicate
    C->>C: skip
  end
```

## Practical Usage
**Outbox table and the write path**:
```sql
create table public.outbox (
  id uuid primary key default gen_random_uuid(),       -- prefer UUIDv7 (time-ordered) where available
  tenant_id uuid not null,
  aggregate_type text not null,                         -- 'order'
  aggregate_id uuid not null,
  type text not null,                                   -- 'order.paid'
  version int not null default 1,                       -- payload schema version
  payload jsonb not null,
  occurred_at timestamptz not null default now(),
  published_at timestamptz,
  attempts int not null default 0
);
create index outbox_unpublished on public.outbox (occurred_at) where published_at is null;

-- in the same transaction as the business change
begin;
update orders set status = 'paid', paid_at = now() where id = $1 and tenant_id = $2 and status = 'pending';
insert into outbox (tenant_id, aggregate_type, aggregate_id, type, payload)
values ($2, 'order', $1, 'order.paid', jsonb_build_object('orderId', $1, 'amount', $3, 'currency', 'MYR'));
commit;
```
**Relay worker** (TypeScript; safe to run several replicas):
```ts
export async function relayOnce() {
  return db.begin(async (tx) => {
    const rows = await tx`
      select id, tenant_id, type, version, payload, aggregate_id
      from outbox where published_at is null and attempts < 10
      order by occurred_at limit 100
      for update skip locked`;
    for (const e of rows) {
      try {
        await queue.add(e.type, e, { jobId: e.id });                  // jobId = event id → broker-side dedupe
        await tx`update outbox set published_at = now() where id = ${e.id}`;
      } catch {
        await tx`update outbox set attempts = attempts + 1 where id = ${e.id}`;
      }
    }
    return rows.length;
  });
}
// loop: relayOnce(); if 0 rows, sleep 1 s (or wait for NOTIFY 'outbox' as a wake-up hint)
```
**Idempotent consumer with an inbox**:
```ts
export async function onOrderPaid(e: OutboxEvent) {
  await db.begin(async (tx) => {
    const [fresh] = await tx`
      insert into inbox (event_id, consumer) values (${e.id}, 'receipt-sender')
      on conflict do nothing returning event_id`;
    if (!fresh) return;                                               // already handled
    await tx`insert into receipts (order_id, tenant_id) values (${e.payload.orderId}, ${e.tenant_id})`;
  });
  await whatsapp.sendTemplate(/* … */, { idempotencyKey: e.id });     // external side effect keyed by event id
}
```
**Wrong way**:
```ts
await db.orders.markPaid(id);
await queue.add('order.paid', { id });          // ❌ crash here → event lost; customer never gets a receipt
// or
await queue.add('order.paid', { id });
await db.orders.markPaid(id);                   // ❌ txn fails → consumers act on a payment that didn't happen
pg.query("NOTIFY order_paid, '…'")              // ❌ NOTIFY is not durable: no listener connected = event gone
```

## Patterns & Anti-patterns
| Pattern | When | Anti-pattern to avoid |
|---|---|---|
| Outbox in the same transaction | Any DB change that triggers async work | Publish-after-commit dual writes |
| Inbox / idempotent consumers keyed by event ID | Every consumer | Assuming exactly-once delivery |
| Versioned event envelope (`type`, `version`, `tenant_id`, `occurred_at`) | Integration events | Unversioned ad-hoc JSON |
| Per-aggregate ordering (partition/group by `aggregate_id`) | Order lifecycles, conversations | Expecting global ordering |
| Orchestrated saga (state machine) for multi-step money flows | Payments, bookings | Long choreography chains nobody can trace |
| Retention: delete or partition published outbox rows | Always | Outbox growing forever |
| Correlation IDs from request → event → consumer logs | Debugging | Events you can't trace back to a request |

## Performance & Trade-offs
- **Throughput**: a polling relay with `SKIP LOCKED` and batches of 100 handles thousands of events/s on modest Postgres. SME workloads are orders of magnitude below that.
- **Latency**: polling adds ~0.5–2 s. Add `LISTEN/NOTIFY` as a wake-up when users wait on the result (e.g. a WhatsApp receipt).
- **Ordering vs parallelism**: concurrent relays with `SKIP LOCKED` can publish events of the same aggregate out of order. Either group by aggregate (one in-flight event per aggregate, BullMQ groups/keys), or make consumers tolerate reordering (version checks).
- **Thin vs fat events**: thin events need consumer callbacks into the producer (coupling, load), while fat events carry personal data around (retention, PDPA). Include only what consumers need.
- **Event sourcing** gives a full audit history and temporal queries at the cost of projections, upcasting and replay tooling. An outbox + audit table gives most of the benefit for SMEs.

## Tips & Reminders
> [!tip] In ZP's stack
> - **Supabase/Next.js**: write the outbox in the same RPC or Server Action transaction. A small worker on Coolify (or a pg_cron-triggered Edge Function) relays to BullMQ/pgmq ([[Supabase - Edge Functions, Cron & Queues]]).
> - **n8n as a consumer**: the relay POSTs events to n8n webhooks with `event_id`, and n8n workflows dedupe on it, so you don't depend on n8n being up at commit time ([[n8n - Error Handling & Workflow Patterns]]).
> - **WhatsApp/payment webhooks** are the inbound mirror: ingest → inbox/dedupe → domain change + outbox in one transaction ([[REST API - Webhooks & Idempotency]]).

- Generate event IDs as UUIDv7 or ULIDs for natural time ordering and index locality.
- Expose an admin view of unpublished/failed outbox rows with "retry" and "dead-letter" actions.
- Keep the outbox per database. Cross-database "global outboxes" reintroduce dual writes.
- Partition or purge published rows after N days ([[PostgreSQL - Partitioning & Large Table Maintenance]]).

## Version Notes
N/A — pattern. Relevant platform features: `FOR UPDATE SKIP LOCKED` (PostgreSQL 9.5+), logical replication (10+), `pgmq` on Supabase (Supabase Queues), and Kafka-based CDC via Debezium ([[Kafka - KRaft, Redpanda & Small-Infra Trade-offs]]).

## Critical Issues & Gotchas
> [!danger] At-least-once delivery means duplicates will happen
> Relays crash after publishing but before marking `published_at`, and brokers redeliver. Consumers without inbox/idempotency keys send two receipts, double-count revenue or, worst, issue two refunds. Every consumer and every external side effect must be keyed by the event ID.

> [!danger] Poison events block or loop forever
> An event a consumer always fails on (bad payload, deleted entity) retries endlessly, consuming resources and sometimes blocking ordered processing for that aggregate. Cap attempts, move failures to a dead-letter table with the error, alert, and provide a manual replay path.

- **Logical replication slots** used for CDC retain WAL while consumers are down, which can fill the disk. Monitor slot lag, and drop abandoned slots ([[PostgreSQL - Backups, PITR & Restore Drills]]).
- **Outbox bloat**: high-churn updates on `published_at` create dead tuples. Tune autovacuum on the table, or use insert-only + delete-in-batches.
- **Schema evolution**: consumers deployed later than producers must handle older and newer versions. Add fields, don't rename them, and bump `version` for breaking changes.
- **PDPA**: events with names, phones or ICs get copied into queues, logs and n8n executions. Minimize payloads, reference IDs, and set retention on every store.

## Related
- [[System Design]]
- [[System Design - Reliability Patterns]] — retries, idempotency, DLQs
- [[PostgreSQL - Transactions & MVCC]] — transactional guarantees and `SKIP LOCKED`
- [[Supabase - Edge Functions, Cron & Queues]] — pgmq and cron-driven relays
- [[Message Queues - Choosing BullMQ, pgmq, RabbitMQ or NATS]] — where the relay publishes
- [[Design Patterns - State Machines for Business Workflows]] — orchestrated sagas

## References
- https://microservices.io/patterns/data/transactional-outbox.html
- https://microservices.io/patterns/communication-style/idempotent-consumer.html
- https://www.postgresql.org/docs/current/sql-select.html#SQL-FOR-UPDATE-SHARE
- https://debezium.io/documentation/reference/stable/transformations/outbox-event-router.html
- https://github.com/tembo-io/pgmq
