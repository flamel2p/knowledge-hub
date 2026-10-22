---
title: Supabase - Edge Functions, Cron & Queues
aliases: [Supabase Edge Functions, Supabase Cron, Supabase Queues, pg_cron, pgmq, pg_net, EdgeRuntime.waitUntil, database webhooks]
type: deep-dive
domain: backend
tags: [domain/backend, type/deep-dive, topic/supabase, topic/background-jobs, topic/queues, lang/typescript]
status: draft
created: 2026-10-22
updated: 2026-10-22
version_checked: "Edge Functions limits 150 s/400 s wall, 2 s CPU · Supabase Queues (pgmq) · pg_cron — 2026-10"
parent: "[[Supabase]]"
related: ["[[PostgreSQL]]", "[[Message Queues]]", "[[Message Queues - BullMQ in Production]]", "[[n8n]]", "[[System Design - Reliability Patterns]]", "[[Supabase - Row Level Security]]"]
---

# Supabase - Edge Functions, Cron & Queues

> [!info] Deep dive of [[Supabase]]

> [!abstract] TL;DR
> Supabase's background-job stack has four parts:
> - **Queues**, built on `pgmq` in Postgres: durable messages, enqueued in the same transaction as your data;
> - **Cron** (`pg_cron`): schedules;
> - **`pg_net`**: async HTTP from SQL;
> - **Edge Functions** (Deno): the workers.
>
> Edge Functions have hard limits: about **150 s wall clock on Free / 400 s on paid, 2 s CPU per request, 256 MB**. Design for short batches with idempotent handlers, not long jobs.

## Concept
| Primitive | What it is | Good for | Not for |
|---|---|---|---|
| **Edge Functions** | Deno isolates behind the API gateway; `EdgeRuntime.waitUntil()` for post-response work | Webhooks, third-party API calls, secret-holding logic, short workers | CPU-heavy work (>2 s CPU), long jobs, persistent connections |
| **Cron** (`pg_cron`) | Job scheduler inside Postgres; jobs run SQL or call functions | Periodic triggers, cleanups, rollups | Sub-second precision; heavy logic in SQL strings |
| **`pg_net`** | Async, non-blocking HTTP client in SQL (`net.http_post`) | Calling Edge Functions from triggers or cron | Guaranteed delivery: it is fire-and-forget, with no retries |
| **Queues** (`pgmq`) | Durable queue tables with visibility timeouts; `pgmq_public` RPC wrappers for the Data API | Outbound WhatsApp/email, AI jobs, fan-out with retries | Very high throughput (>1–5k msg/s sustained); cross-system streaming ([[Kafka]]) |
| **Database Webhooks** | Triggers calling `pg_net` on insert/update/delete | Simple "row changed → call URL" | Anything needing retries or ordering |

- **The key advantage**: `pgmq.send()` runs inside your transaction. "Insert order + enqueue notification" is atomic, which gives you the **transactional outbox** pattern for free ([[System Design - Reliability Patterns]]).
- **Semantics**: a read hides the message for `vt` seconds. If it isn't deleted or archived before `vt` expires, it reappears (`read_ct` increments). That makes delivery **at-least-once**, so handlers must be idempotent despite the "exactly once within the visibility window" marketing.

## How It Works
```mermaid
sequenceDiagram
  participant App as App / trigger
  participant PG as Postgres (pgmq)
  participant Cron as pg_cron
  participant EF as Edge Function worker
  participant Meta as Meta Cloud API
  App->>PG: BEGIN; insert order; pgmq.send('wa_outbound'); COMMIT
  Cron->>EF: every 10 s: net.http_post /worker
  EF->>PG: pgmq_public.read(queue, vt=60, n=20)
  EF->>Meta: send template messages
  EF->>PG: pgmq_public.delete(msg_id) on success
  Note over EF,PG: failures stay invisible until vt expires, then are retried
```

## Practical Usage
**Queue + enqueue in the same transaction**:
```sql
select pgmq.create('wa_outbound');

create or replace function public.confirm_order(_id uuid) returns void
language plpgsql security invoker as $$
begin
  update public.orders set status = 'confirmed' where id = _id;
  perform pgmq.send('wa_outbound', jsonb_build_object('order_id', _id, 'template', 'order_confirmed'));
end $$;
```
**Cron → worker**, with secrets in Vault (never inline in the cron command):
```sql
select vault.create_secret('https://<ref>.supabase.co/functions/v1/wa-worker', 'wa_worker_url');
select vault.create_secret('<secret key>', 'worker_secret');

select cron.schedule('wa-worker', '10 seconds', $$
  select net.http_post(
    url     := (select decrypted_secret from vault.decrypted_secrets where name = 'wa_worker_url'),
    headers := jsonb_build_object('Authorization', 'Bearer ' ||
               (select decrypted_secret from vault.decrypted_secrets where name = 'worker_secret')),
    body    := '{}'::jsonb,
    timeout_milliseconds := 5000)
$$);
```
**Worker Edge Function**: a bounded batch, idempotent, with a dead-letter step.
```ts
// supabase/functions/wa-worker/index.ts
import { createClient } from 'npm:@supabase/supabase-js@2';

const db = createClient(Deno.env.get('SUPABASE_URL')!, Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!);
const q = db.schema('pgmq_public');

Deno.serve(async (req) => {
  if (req.headers.get('Authorization') !== `Bearer ${Deno.env.get('WORKER_SECRET')}`) return new Response('no', { status: 401 });
  const { data: msgs, error } = await q.rpc('read', { queue_name: 'wa_outbound', sleep_seconds: 60, n: 20 });
  if (error) throw error;

  for (const m of msgs ?? []) {
    if (m.read_ct > 5) {                                         // poison message → park it
      await db.from('dead_letters').insert({ queue: 'wa_outbound', payload: m.message, read_ct: m.read_ct });
      await q.rpc('archive', { queue_name: 'wa_outbound', message_id: m.msg_id });
      continue;
    }
    try {
      await sendTemplate(m.message, { idempotencyKey: `wa:${m.message.order_id}:${m.message.template}` });
      await q.rpc('delete', { queue_name: 'wa_outbound', message_id: m.msg_id });
    } catch (e) {
      console.error('send failed', m.msg_id, e);                  // left invisible → retried after vt
    }
  }
  return Response.json({ processed: msgs?.length ?? 0 });
});
```
**Webhook receiver**: acknowledge fast, process after the response.
```ts
Deno.serve(async (req) => {
  const body = await req.json();
  if (!verifyMetaSignature(req.headers.get('x-hub-signature-256'), body)) return new Response('bad sig', { status: 401 });
  EdgeRuntime.waitUntil(db.schema('pgmq_public').rpc('send', { queue_name: 'wa_inbound', message: body }));
  return new Response('ok');                                    // Meta retries if no quick 200
});
```
**Wrong way**:
```ts
// ❌ heavy work inline: LLM call + 3 API calls in the webhook. Hits 2 s CPU / wall limits,
//    Meta times out and redelivers, and the same message gets processed twice with no idempotency
Deno.serve(async (req) => { const b = await req.json(); await runAgent(b); await reply(b); return new Response('ok'); });
```

## Patterns & Anti-patterns
| Pattern | When | Anti-pattern to avoid |
|---|---|---|
| Enqueue inside the business transaction | Any side effect after a write | `pg_net` call or Database Webhook as the only delivery path (no retry) |
| Cron-driven batch worker (n = 10–50) | Steady background load | One Edge Function invocation per message via trigger (thundering herd, rate limits) |
| `read_ct` threshold + `dead_letters` table | Every queue | Infinite redelivery of a poison message |
| Idempotency key per message (`order_id + action`) | External side effects | Assuming visibility timeout = exactly-once |
| Secrets in Vault, read in the cron SQL | Cron → HTTP | Bearer tokens pasted into `cron.schedule` (stored in `cron.job` in plain text) |
| `vt` > p99 processing time × 2 | Worker tuning | `vt=5` with 10 s API calls, so messages are processed twice in parallel |

## Performance & Trade-offs
- **Edge Function limits** (re-checked 2026-10):
  - wall clock 150 s on Free and 400 s on paid; `waitUntil` doesn't extend it;
  - **2 s CPU** per request (async I/O excluded);
  - 256 MB memory.
  - Cold starts are tens to a few hundred ms. Keep imports lean and avoid huge npm trees.
- **Throughput**: pgmq handles thousands of msgs/s on decent hardware, but each message is a row with MVCC churn. High-volume queues need `pgmq.archive` rotation or partitioned queues (`pgmq.create_partitioned`, needs `pg_partman`), plus autovacuum tuning.
- **Cron**: `pg_cron` supports second-level schedules (`'10 seconds'`) on recent versions. Every run writes to `cron.job_run_details`, so purge it with a cleanup job or it grows unbounded. `pg_net` responses land in `net._http_response` (short TTL), useful for debugging failed calls.
- **When to leave Supabase for jobs**:

| Need | Use instead |
|---|---|
| Jobs > 400 s, CPU-heavy (PDF, video, embeddings in bulk) | Worker container on Coolify ([[Message Queues - BullMQ in Production|BullMQ]] or a Postgres-polling worker) |
| Visual flows, many SaaS connectors, non-dev editing | [[n8n]] (queue mode) |
| Event streaming, replay, multi-consumer | [[Kafka]] |

## Tips & Reminders
> [!tip] In ZP's stack
> - **WhatsApp**: Meta webhooks go to an Edge Function that verifies the signature, enqueues with `waitUntil` and returns 200 in about 100 ms.
> - **Outbound**: a cron-driven worker drains `wa_outbound` in batches, respecting Meta's per-number throughput.
> - **Supabase vs n8n**: use Supabase Queues for app-coupled transactional jobs, and keep [[n8n]] for client-editable integrations.
> - **Same pattern in Coolify**: the queue lives in Postgres; a Node worker reads with `pgmq.read` over a direct connection, with no Edge Function limits.

- `pg_cron` runs in **UTC**. "Daily at 9am MYT" is `0 1 * * *`. Comment the local time next to every schedule.
- Cron jobs and `service_role` workers **bypass RLS**. Pass and filter `tenant_id` explicitly in every job ([[Supabase - Row Level Security]]).
- Test locally with `supabase functions serve`. Note that the local CLI terminates instances after the response, so `waitUntil` background work may not finish locally. Test it against a branch or staging project.
- Expose `pgmq_public` to clients only if needed. With the Data API enabled, RLS must be on for queue tables. Keep queues server-side by default.
- Monitor queue depth: `select queue_name, queue_length, oldest_msg_age_sec from pgmq.metrics_all();`. Alert when the oldest message age exceeds ~5 min.

## Version Notes
| Date | Change |
|---|---|
| 2024-12 | Supabase Cron and Supabase Queues (pgmq) launched as dashboard modules. Edge Functions gain background tasks (`EdgeRuntime.waitUntil`), ephemeral storage and WebSockets |
| 2025 | Edge Functions on Deno 2 with npm compatibility. New publishable/secret API keys (see [[Supabase]]) |
| 2026 | Legacy `anon`/`service_role` key removal scheduled for late 2026. Rotate worker secrets to the new secret keys |
| pgmq 1.x | FIFO / topic-routing features reported in 2026 releases |

> [!warning] Unverified — check before relying on this
> The pgmq 2026 feature and version claims (v1.9–v1.12) came from a third-party article. Check https://github.com/pgmq/pgmq releases and the version installed on your project (`select extversion from pg_extension where extname = 'pgmq'`).

## Critical Issues & Gotchas
> [!danger] `pg_net` and Database Webhooks don't retry
> If the target is down or slow (default timeout ~2–5 s), the request is lost, and only `net._http_response` shows the failure, and only briefly. Anything that must happen goes through a queue.

> [!danger] Visibility timeout shorter than processing = duplicate side effects
> Two workers (an overlapping cron tick plus a slow API) can both process a message whose `vt` expired. With WhatsApp that means a customer receives the same message twice. Set a generous `vt`, keep batches small, and dedupe with idempotency keys, e.g. a `sent_messages(idempotency_key unique)` table.

- **Overlapping cron runs**: `pg_cron` starts a new run even if the previous HTTP-triggered worker is still processing. The queue's visibility timeout protects message-level work, but non-queue cron jobs need advisory locks (`pg_try_advisory_lock`).
- **Edge Function CPU limit** is per request and easy to blow with JSON parsing of large payloads, crypto or templating loops. Move heavy work to a container worker.
- **Secrets in logs**: `console.log(req.headers)` in Edge Functions puts bearer tokens into the logs (retained per plan). Log message IDs only.
- **Platform dependency**: Edge Functions run on Supabase's Deno-based runtime. Self-hosting is possible (`supabase/edge-runtime`) but operationally heavier. Keep worker logic in plain TS modules so it can move to a Node container.

## Related
- [[Supabase]]
- [[Supabase - Row Level Security]] — jobs bypass RLS; re-check tenancy
- [[Message Queues]] · [[Message Queues - BullMQ in Production]] — Redis-based alternative and comparison
- [[System Design - Reliability Patterns]] — outbox, idempotency, retries, DLQs
- [[n8n]] — when workflows beat code workers
- [[REST API - Webhooks & Idempotency]] — enqueue-and-ack receivers, dedupe keys
- [[PostgreSQL]] — MVCC churn and vacuum for queue tables

## References
- https://supabase.com/docs/guides/functions/limits
- https://supabase.com/docs/guides/functions/background-tasks
- https://supabase.com/docs/guides/troubleshooting/edge-functions-worker-timeouts-and-websocket-drops
- https://supabase.com/docs/guides/queues
- https://supabase.com/blog/edge-functions-background-tasks-websockets
- https://www.issoh.co.jp/tech/details/4240/
