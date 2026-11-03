---
title: Supabase - Database Functions, Triggers & RPC
aliases: [Supabase RPC, supabase.rpc, Postgres functions, plpgsql, security definer, security invoker, search_path, triggers, handle_new_user, pg_net, database webhooks, PostgREST rpc]
type: deep-dive
domain: backend
tags: [domain/backend, type/deep-dive, topic/supabase, topic/postgres-functions, lang/sql]
status: draft
created: 2026-11-04
updated: 2026-11-04
version_checked: "n/a — PostgreSQL 17/18 functions + PostgREST /rpc on Supabase platform; Supabase advisors (function_search_path_mutable) — 2026-11"
parent: "[[Supabase]]"
related: ["[[Supabase - Row Level Security]]", "[[Supabase - Edge Functions, Cron & Queues]]", "[[Supabase - Migrations, Branching & Local Development]]", "[[PostgreSQL - Transactions & MVCC]]", "[[System Design - Event-Driven Architecture & Outbox Pattern]]", "[[TypeScript - Error Handling & Result Types]]"]
---

# Supabase - Database Functions, Triggers & RPC

> [!info] Deep dive of [[Supabase]]

> [!abstract] TL;DR
> Put **atomic, data-local business operations** (place order + decrement stock + write outbox) in Postgres functions called via **`supabase.rpc()`**: one round trip, one transaction, RLS still applies with **`security invoker`** (the default).
> - Use **`security definer`** only for narrowly scoped privileged helpers. Always set **`search_path = ''`**, keep them in a non-exposed schema, and `revoke execute` from `anon`/`public`.
> - Use **triggers** for invariants (`updated_at`, audit rows, denormalized counters), but keep them fast and failure-safe. A broken `auth.users` trigger blocks every signup.
> - Anything slow or external (HTTP, LLM, email) belongs in queues/Edge Functions, not in the transaction.

## Concept
| Tool | Runs | Good for | Avoid for |
|---|---|---|---|
| SQL function (`language sql`) | In the DB, inlinable | Read helpers, simple writes, RLS helpers | Complex control flow |
| PL/pgSQL function | In the DB | Multi-step transactional logic, validation, `raise exception` | Long-running work, HTTP |
| Trigger (`before` / `after`, row / statement) | Automatically on DML | `updated_at`, audit, invariants, outbox rows | Business workflows, external calls |
| RPC (`POST /rest/v1/rpc/fn`) | PostgREST → function | Atomic operations from Next.js/Flutter | Exposing internal helpers |
| Database webhook / `pg_net` | Async HTTP after commit (pg_net queue) | Notify n8n/Edge Function | Guaranteed delivery (use an outbox) |
| Edge Function | Deno, outside the DB | Third-party APIs, secrets, heavy compute | Multi-row atomic writes |

**Security modes**:

| Mode | Executes as | RLS | Use |
|---|---|---|---|
| `security invoker` (default) | The calling role (`authenticated`, `anon`) | ✅ applies | Almost everything |
| `security definer` | The function owner (often `postgres`) | ❌ bypassed (owner privileges) | Narrow privileged lookups (e.g. "is member of tenant" for RLS helpers) |

## How It Works
```mermaid
sequenceDiagram
  participant C as Next.js / Flutter
  participant PR as PostgREST
  participant F as place_order() (security invoker)
  participant T as Triggers
  C->>PR: POST /rpc/place_order {items, idempotency_key} + user JWT
  PR->>F: set role authenticated, request.jwt.claims; BEGIN
  F->>F: validate, lock stock rows (FOR UPDATE)
  F->>T: insert orders / order_items → before trigger sets totals, after trigger writes audit + outbox
  F-->>PR: return order row (RLS-filtered)
  PR->>PR: COMMIT (or rollback on raise)
  PR-->>C: 200 JSON / 400 with error code
```

## Practical Usage
**Atomic RPC with validation and idempotency**:
```sql
create or replace function public.place_order(p_items jsonb, p_idempotency_key text)
returns public.orders
language plpgsql
security invoker
set search_path = ''
as $$
declare
  v_tenant uuid := (auth.jwt() ->> 'tenant_id')::uuid;
  v_order public.orders;
  v_item jsonb;
begin
  select * into v_order from public.orders
   where tenant_id = v_tenant and idempotency_key = p_idempotency_key;
  if found then return v_order; end if;                          -- replay-safe

  insert into public.orders (tenant_id, customer_id, idempotency_key, status)
  values (v_tenant, auth.uid(), p_idempotency_key, 'pending')
  returning * into v_order;

  for v_item in select * from jsonb_array_elements(p_items) loop
    update public.stock
       set qty = qty - (v_item ->> 'qty')::int
     where tenant_id = v_tenant and sku = v_item ->> 'sku' and qty >= (v_item ->> 'qty')::int;
    if not found then
      raise exception 'out_of_stock:%', v_item ->> 'sku' using errcode = 'P0001';
    end if;
    insert into public.order_items (order_id, sku, qty)
    values (v_order.id, v_item ->> 'sku', (v_item ->> 'qty')::int);
  end loop;

  insert into public.outbox (tenant_id, aggregate_type, aggregate_id, type, payload)
  values (v_tenant, 'order', v_order.id, 'order.placed', to_jsonb(v_order));
  return v_order;
end $$;

revoke execute on function public.place_order(jsonb, text) from anon, public;
grant execute on function public.place_order(jsonb, text) to authenticated;
```
```ts
const { data, error } = await supabase.rpc('place_order', { p_items: items, p_idempotency_key: key });
if (error?.message.startsWith('out_of_stock:')) return { ok: false, code: 'out_of_stock', sku: error.message.split(':')[1] };
```
**Privileged helper for RLS** (definer, private schema, fixed search_path):
```sql
create schema if not exists private;
create or replace function private.is_tenant_member(p_tenant uuid)
returns boolean language sql stable security definer set search_path = '' as $$
  select exists (select 1 from public.memberships m where m.tenant_id = p_tenant and m.user_id = auth.uid());
$$;
revoke all on function private.is_tenant_member(uuid) from public;
grant execute on function private.is_tenant_member(uuid) to authenticated;
-- policy: using ((select private.is_tenant_member(tenant_id)))
```
**Triggers that must never block**:
```sql
create or replace function private.set_updated_at() returns trigger language plpgsql set search_path = '' as $$
begin new.updated_at := now(); return new; end $$;
create trigger orders_updated_at before update on public.orders for each row execute function private.set_updated_at();

-- new-user profile: exception-safe, so a profile bug doesn't block every signup
create or replace function private.handle_new_user() returns trigger
language plpgsql security definer set search_path = '' as $$
begin
  insert into public.profiles (id, display_name) values (new.id, new.raw_user_meta_data ->> 'name')
  on conflict (id) do nothing;
  return new;
exception when others then
  raise warning 'handle_new_user failed for %: %', new.id, sqlerrm;
  return new;
end $$;
create trigger on_auth_user_created after insert on auth.users for each row execute function private.handle_new_user();
```
**Wrong way**:
```sql
create function public.admin_set_role(uid uuid, r text) returns void
language sql security definer as $$ update profiles set role = r where id = uid $$;
-- ❌ exposed via /rpc to anon/authenticated, bypasses RLS, mutable search_path: any user makes themselves admin

create trigger notify after insert on orders for each row execute function call_llm_via_http();
-- ❌ synchronous external call inside the write transaction: slow inserts, failures roll back orders
```

## Patterns & Anti-patterns
| Pattern | When | Anti-pattern to avoid |
|---|---|---|
| RPC for multi-table atomic operations | Checkout, booking, stock | Several client-side calls hoping none fails midway |
| `security invoker` + RLS by default | All exposed functions | Definer "to make it work" |
| Definer helpers in `private`, `search_path = ''`, explicit grants | RLS membership checks | Definer functions in `public` |
| Fully qualified names (`public.orders`) inside functions | Always (with empty search_path) | Relying on caller's search_path |
| Domain errors as stable codes (`raise … 'out_of_stock:SKU'`, errcode) | RPC errors | Free-text errors parsed by UI |
| Triggers for invariants + outbox rows | Data integrity | Business processes chained through triggers |
| Outbox + worker for side effects | Notifications, webhooks | `pg_net`/HTTP in triggers as the only delivery |

## Performance & Trade-offs
- **Round trips**: one RPC replaces 3–6 PostgREST calls from a phone on 4G. That's a big latency win for Flutter apps.
- **Inlining**: simple `language sql` `stable` functions can be inlined into queries. PL/pgSQL can't, so keep RLS helpers in SQL and wrap calls in `(select …)` so they're evaluated once per statement ([[Supabase - Row Level Security]]).
- **Lock scope**: `FOR UPDATE` / conditional updates in RPCs serialize hot rows (stock of a popular item). Keep transactions short ([[PostgreSQL - Transactions & MVCC]]).
- **Triggers cost every write**: row-level triggers on bulk imports multiply work. Use statement-level triggers with transition tables for batch audit.
- **Testability**: DB logic needs pgTAP tests and migrations discipline ([[Supabase - Migrations, Branching & Local Development]]). Without them, functions become untested hidden code.

## Tips & Reminders
> [!tip] In ZP's stack
> - **Flutter + Supabase SME apps**: checkout, stock reservation and loyalty-point redemption as RPCs. Call them with typed wrappers generated from `supabase gen types`.
> - **n8n integration**: after-commit outbox rows relayed to n8n, rather than database webhooks firing directly, so n8n downtime doesn't lose events ([[System Design - Event-Driven Architecture & Outbox Pattern]]).
> - **Supabase advisors**: run the security/performance advisors after each migration. They flag mutable `search_path`, definer functions in exposed schemas, and missing RLS.

- Name parameters `p_*` to avoid ambiguity with column names inside PL/pgSQL.
- Return rows/JSON shaped for the client, and keep them small. PostgREST serializes the whole result.
- Use `stable`/`immutable` correctly. Wrong volatility can cache results within a statement or block index usage.
- Version breaking function changes (`place_order_v2`) while old Flutter app versions are still in the field.

## Version Notes
N/A — stable PostgreSQL/PostgREST features. Relevant: PostgREST exposes functions in exposed schemas (default `public`) as `/rpc/*`, and Supabase's database advisors flag `function_search_path_mutable` and security-definer exposure. Supabase's API key changes (publishable/secret) don't change RPC semantics ([[Supabase]]).

## Critical Issues & Gotchas
> [!danger] Exposed SECURITY DEFINER functions are privilege escalation
> Any function in an exposed schema is callable via `/rpc` by roles with `EXECUTE`, and Postgres grants `EXECUTE` to `PUBLIC` by default. A definer function that updates roles, reads other tenants or returns secrets becomes an API for attackers. Revoke from `public`/`anon`, move privileged helpers to `private`, set `search_path = ''`, and review every definer function in PRs.

> [!danger] auth.users triggers that fail block authentication
> An `after insert on auth.users` trigger that errors (constraint violation, missing column after a migration) rolls back the signup, so every new user fails with a generic error. Make such triggers exception-safe, test them in CI, and monitor signup error rates.

- **`raise exception` messages** reach clients. Don't include internal details or other tenants' data in them.
- **`pg_net` is asynchronous and best-effort**: requests are queued after commit, and failures aren't retried by your app logic. Don't use it for anything that must be delivered.
- **Overloaded functions** (same name, different args) confuse PostgREST resolution. Avoid overloading RPCs.
- **Long-running RPCs** hit statement timeouts (`authenticated` role timeouts on Supabase) and hold connections in the pooler. Move batch work to workers ([[PostgreSQL - Connection Pooling with Supavisor & PgBouncer]]).

## Related
- [[Supabase]]
- [[Supabase - Row Level Security]] — invoker functions inherit policies; definer helpers for membership checks
- [[Supabase - Edge Functions, Cron & Queues]] — where external and slow work goes
- [[Supabase - Migrations, Branching & Local Development]] — functions as versioned, tested schema
- [[PostgreSQL - Transactions & MVCC]] — locks and isolation inside RPCs
- [[System Design - Event-Driven Architecture & Outbox Pattern]] — outbox rows from triggers/RPCs

## References
- https://supabase.com/docs/guides/database/functions
- https://supabase.com/docs/guides/database/postgres/triggers
- https://supabase.com/docs/guides/database/database-advisors
- https://docs.postgrest.org/en/stable/references/api/functions.html
- https://www.postgresql.org/docs/current/sql-createfunction.html#SQL-CREATEFUNCTION-SECURITY
