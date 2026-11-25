---
title: Cloudflare - Hyperdrive & Connecting Workers to Postgres
aliases: [Cloudflare Hyperdrive, Workers Postgres, connection pooling at the edge, Hyperdrive query caching, Workers + Supabase, postgres.js Workers, node-postgres Workers, Hyperdrive with Cloudflare Tunnel, edge database latency]
type: deep-dive
domain: devops
tags: [domain/devops, type/deep-dive, topic/cloudflare, topic/postgresql, topic/edge]
status: draft
created: 2026-11-25
updated: 2026-11-25
version_checked: "Cloudflare Hyperdrive (Postgres + MySQL, connection pooling, query caching, private DBs via Tunnel) · Workers nodejs_compat · Supabase Supavisor — 2026-11"
parent: "[[Cloudflare]]"
related: ["[[Cloudflare - Workers for Webhooks & Edge APIs]]", "[[Cloudflare - KV, D1 & Durable Objects for Edge State]]", "[[PostgreSQL - Connection Pooling with Supavisor & PgBouncer]]", "[[Cloudflare - Zero Trust Tunnel & Access]]", "[[Supabase - Read Replicas, High Availability & Multi-Region]]", "[[Cloudflare - Pages & Workers Static Assets for Frontends]]"]
---

# Cloudflare - Hyperdrive & Connecting Workers to Postgres

> [!info] Deep dive of [[Cloudflare]]

> [!abstract] TL;DR
> Workers are short-lived isolates in hundreds of locations. Opening a fresh TCP + TLS + auth connection to Postgres per request costs several round trips and exhausts database connections.
> **Hyperdrive** sits between Workers and your database:
> - keeps **warm connection pools** near the database;
> - speeds up connection setup from the edge;
> - optionally **caches read query results**.
>
> Your Worker uses a normal driver (`postgres`/`pg`) with the Hyperdrive connection string.
> - **Use it when** Workers must query Postgres/Supabase directly (edge APIs, webhook handlers, OpenNext apps).
> - **Don't expect** it to remove the round trip to a single-region database. Writes and uncached reads still travel to Singapore (or wherever the DB lives).
> - **Private databases** (self-hosted Postgres on a VPS) can be reached through **Cloudflare Tunnel** without exposing port 5432.

## Concept
| Component | Role |
|---|---|
| Hyperdrive config | Stores DB origin (host, port, user, password, db) and caching settings |
| Worker binding | `env.HYPERDRIVE.connectionString` used by your driver |
| Connection pool | Maintained by Cloudflare near the origin, reused across requests |
| Query cache | Caches results of eligible read queries (TTL, stale-while-revalidate). Disable per config |
| Tunnel origin | Private DB reachable via `cloudflared` with Access service token |

**Connection options from Workers**:

| Option | Latency/limits | Pick it when… |
|---|---|---|
| Hyperdrive + Postgres driver | Pooled, cached reads | Workers need SQL against Postgres/Supabase |
| Supabase Data API (PostgREST over HTTPS) | Simple fetch, RLS applies | Simple CRUD with user JWTs |
| Direct TCP from Worker (no Hyperdrive) | New connection per request | Prototypes only |
| D1 / Durable Objects | Edge-local data | Data that can live on Cloudflare ([[Cloudflare - KV, D1 & Durable Objects for Edge State]]) |

## How It Works
```mermaid
flowchart LR
  U["User (Malaysia)"] --> W["Worker (KUL/SIN PoP)"]
  W -->|"env.HYPERDRIVE.connectionString"| HD["Hyperdrive (edge + pool near DB)"]
  HD -->|"cached read?"| C[("Query cache")]
  HD -->|"pooled connections"| PG[("Postgres / Supabase (Singapore)")]
  HD -.->|"private origin"| TUN["Cloudflare Tunnel → self-hosted Postgres on Coolify VPS"]
```

## Practical Usage
**Create a Hyperdrive config**:
```bash
npx wrangler hyperdrive create app-db \
  --connection-string="postgres://hyperdrive_user:****@db.<ref>.supabase.co:5432/postgres"
# options: --caching-disabled, --max-age=60, --swr=15  (verify flag names in current wrangler)
```
**wrangler.jsonc**:
```jsonc
{
  "name": "edge-api",
  "main": "src/index.ts",
  "compatibility_date": "2026-10-01",
  "compatibility_flags": ["nodejs_compat"],
  "hyperdrive": [{ "binding": "HYPERDRIVE", "id": "<hyperdrive-id>" }]
}
```
**Query from a Worker with postgres.js**:
```ts
import postgres from "postgres";

export default {
  async fetch(req: Request, env: Env, ctx: ExecutionContext) {
    const sql = postgres(env.HYPERDRIVE.connectionString, { max: 5, fetch_types: false });
    try {
      const url = new URL(req.url);
      const tenant = url.searchParams.get("tenant")!;
      const rows = await sql`select id, name, price_cents from products where tenant_id = ${tenant} and active order by name limit 50`;
      return Response.json(rows);
    } finally {
      ctx.waitUntil(sql.end());            // release connection back to Hyperdrive pool
    }
  },
};
```
**Least-privilege DB role for Workers**:
```sql
create role hyperdrive_user login password '…';
grant usage on schema public to hyperdrive_user;
grant select on public.products, public.outlets to hyperdrive_user;     -- read-only edge API
-- or use RLS-aware RPCs via Data API instead for user-scoped access
```
**Private self-hosted Postgres via Tunnel** (outline):
```text
cloudflared tunnel on the VPS → route TCP to localhost:5432 → Access application + service token
wrangler hyperdrive create … --origin-host=db.internal.example --access-client-id=… --access-client-secret=…
```
**Wrong way**:
```text
❌ Connecting Workers straight to Postgres without pooling    → connection storms, "too many clients"
❌ Using the Supabase service-role/postgres superuser in Workers → full DB access from every edge isolate
❌ Caching reads that must be fresh (stock, balances)          → stale values from the query cache
❌ Expecting edge SSR + Hyperdrive to beat a server next to the DB → writes/uncached reads still cross to the origin region
```

## Patterns & Anti-patterns
| Pattern | When | Anti-pattern to avoid |
|---|---|---|
| Dedicated least-privilege role per Hyperdrive config | Always | Superuser/service credentials |
| Disable caching for transactional queries (or separate configs) | Mixed workloads | One cached config for everything |
| Parameterized queries via driver | All SQL | String-built SQL |
| Tunnel for self-hosted DBs | Coolify-hosted Postgres | Opening 5432 to the internet |
| Keep heavy SSR on a server near the DB | Supabase-heavy apps | Edge for its own sake |
| Short queries, small result sets | Workers CPU limits | Large reports in Workers |

## Performance & Trade-offs
- **Gains**: the connection-setup round trips (TCP, TLS, auth) disappear for warm pools. Cached reads return from Cloudflare without touching the DB. Uncached queries still pay one round trip to the origin region.
- **Supabase**: connect Hyperdrive to the database's direct port or session pooler per Cloudflare/Supabase guidance. Stacking Hyperdrive on Supavisor transaction mode can conflict with prepared statements ([[PostgreSQL - Connection Pooling with Supavisor & PgBouncer]]).
- **Query cache semantics**: only certain read queries are cached. Mutations and queries in transactions/with volatile functions aren't. Know which endpoints tolerate staleness.
- **Cost/limits**: Hyperdrive usage limits and pricing vary by plan. Check current quotas for connections and queries.

## Tips & Reminders
> [!tip] In ZP's stack
> - **When it helps**: webhook-handling Workers (WhatsApp/Stripe) that must read tenant config or write events to Postgres, and edge APIs serving product catalogues to Flutter apps ([[Cloudflare - Workers for Webhooks & Edge APIs]]).
> - **When to skip**: full Next.js apps with Supabase. Keep them on Coolify in Singapore next to the DB, or use the Supabase Data API from Workers for simple RLS-scoped reads ([[Cloudflare - Pages & Workers Static Assets for Frontends]]).
> - **Self-hosted Postgres for clients**: Hyperdrive + Tunnel lets Workers reach a Coolify-hosted DB without public exposure ([[Cloudflare - Zero Trust Tunnel & Access]]).

- Close/release connections with `ctx.waitUntil(sql.end())` (or the driver's recommended pattern).
- Disable `fetch_types` in postgres.js to cut startup queries.
- Monitor DB connection counts (`pg_stat_activity`) after enabling Hyperdrive.
- Use separate Hyperdrive configs for read-only cached endpoints vs write endpoints.

## Version Notes
| Date | Change relevant here |
|---|---|
| 2023-09 | Hyperdrive announced (beta) |
| 2024-04 | Hyperdrive GA (Postgres) |
| 2025 | Private database access via Tunnel. MySQL support. Free-plan availability (with limits) reported |
| 2025–26 | Improved connection latency (pools placed near origins). Workers TCP sockets mature |

> [!warning] Unverified — check before relying on this
> Wrangler flag names for caching, current plan limits, MySQL support status and Cloudflare's recommended Supabase connection mode weren't verified this run. Check https://developers.cloudflare.com/hyperdrive/.

## Critical Issues & Gotchas
> [!danger] Over-privileged database credentials at the edge
> The Hyperdrive connection string grants whatever its DB role can do, from every Worker invocation worldwide. A bug or injection in a Worker with the `postgres` superuser (or Supabase service role) exposes or wipes every tenant's data. Create least-privilege roles per use case, enforce tenant filters (or RLS via the Data API), and never reuse admin credentials.

> [!danger] Stale cached reads in business flows
> Hyperdrive's query cache can serve results that are seconds old. Stock levels, payment status or booking availability read through a cached config lead to overselling or double bookings. Disable caching for transactional reads, or use a separate uncached config, and confirm critical writes with fresh reads.

- **Prepared statements and poolers**: driver settings may need adjusting (postgres.js `prepare: false` in some setups).
- **Long transactions** hold pooled connections. Keep transactions short in Workers.
- **Worker CPU/memory limits** make large result processing fail. Paginate.
- **IP allow-lists** on the database must account for Hyperdrive egress, or use Tunnel.

## Related
- [[Cloudflare]]
- [[Cloudflare - Workers for Webhooks & Edge APIs]] — Workers that need DB access
- [[Cloudflare - KV, D1 & Durable Objects for Edge State]] — edge-native data alternatives
- [[PostgreSQL - Connection Pooling with Supavisor & PgBouncer]] — pooling concepts
- [[Cloudflare - Zero Trust Tunnel & Access]] — private origins
- [[Supabase - Read Replicas, High Availability & Multi-Region]] — region and replica choices

## References
- https://developers.cloudflare.com/hyperdrive/
- https://developers.cloudflare.com/hyperdrive/configuration/query-caching/
- https://developers.cloudflare.com/hyperdrive/configuration/connect-to-private-database/
- https://developers.cloudflare.com/hyperdrive/examples/connect-to-postgres/
- https://supabase.com/docs/guides/database/connecting-to-postgres
