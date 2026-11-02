---
title: Supabase - Migrations, Branching & Local Development
aliases: [Supabase CLI, supabase start, supabase db diff, supabase db push, declarative schemas, Supabase branching, preview branches, supabase gen types, migration repair, seed.sql]
type: deep-dive
domain: backend
tags: [domain/backend, type/deep-dive, topic/supabase, topic/migrations, topic/local-dev]
status: draft
created: 2026-11-02
updated: 2026-11-02
version_checked: "Supabase CLI v2.106 (2026-06: declarative schema sync, branches --git-branch) · branching without Git default (2026-05) · pg-delta diff (alpha) — 2026-11"
parent: "[[Supabase]]"
related: ["[[Supabase - Row Level Security]]", "[[Testing Strategy - Integration Testing with Testcontainers & Supabase]]", "[[CI-CD - Deploy Pipelines for Coolify & Flutter Releases]]", "[[Coolify - Preview Deployments, Environments & Secrets]]", "[[PostgreSQL - Partitioning & Large Table Maintenance]]", "[[Git - Branching, Releases & Recovery]]"]
---

# Supabase - Migrations, Branching & Local Development

> [!info] Deep dive of [[Supabase]]

> [!abstract] TL;DR
> **Git is the source of truth for the schema.** Develop against the local stack (`supabase start`), and define tables, functions and RLS in **declarative schema files** or hand-written migrations.
> - Generate and review timestamped migrations, and reset locally (`supabase db reset`) until they apply cleanly with `seed.sql`.
> - Test RLS with pgTAP, then let CI apply them: `db push` to staging, then production.
> - **Preview branches** (Git-linked or, since 2026, dashboard-created) give each PR an isolated database with schema and seed but no production data.
> - **Never edit production in Studio.** That drift is what breaks every later migration.

## Concept
| Approach | How | Pros | Cons |
|---|---|---|---|
| Migration-first | `supabase migration new x` → write SQL | Full control (concurrent indexes, data backfills) | You maintain the full history by hand |
| Studio-first + diff | Change locally in Studio → `supabase db diff -f x` | Fast for prototyping | Diffs can miss or mis-order objects. Review every file |
| **Declarative schemas** | Desired state in `supabase/schemas/*.sql` → diff generates migrations | One readable file per table/policy set, reviewable | Data migrations and some objects still need manual migrations |
| Remote pull | `supabase db pull` | Adopt an existing project | Captures drift. Only for onboarding |

**Local stack defaults**:

| Service | Local URL / port |
|---|---|
| API (PostgREST, Auth, Storage, Realtime via Kong/Envoy) | `http://127.0.0.1:54321` |
| Postgres | `postgresql://postgres:postgres@127.0.0.1:54322/postgres` |
| Studio | `http://127.0.0.1:54323` |
| Email testing (Inbucket/Mailpit) | `http://127.0.0.1:54324` |

**Branching**:

| Mode | Trigger | Gets |
|---|---|---|
| Git-linked preview branch | PR opened in a connected repo, or `supabase branches create --git-branch <b>` | Migrations from the branch + `seed.sql`, own API keys/URL |
| Dashboard branch (default since 2026-05) | Created in the dashboard/CLI without Git | Copy of schema, changes merged back via a generated diff (pg-delta) |
| Persistent branch | Long-lived (staging) | Same, kept between PRs |

## How It Works
```mermaid
flowchart LR
  DEV["Local: supabase start"] --> EDIT["Edit schemas/*.sql or migrations"]
  EDIT --> DIFF["supabase db diff -f change (review!)"]
  DIFF --> RESET["supabase db reset (migrations + seed)"]
  RESET --> TEST["supabase test db (pgTAP) + gen types"]
  TEST --> PR["Pull request"]
  PR --> BR["Preview branch DB (+ Coolify preview app)"]
  PR --> CI["CI: lint SQL, reset, tests"]
  CI -->|"merge"| STG["db push → staging"]
  STG -->|"approval"| PROD["db push → production"]
```

## Practical Usage
**Project layout**:
```text
supabase/
├── config.toml           # local stack config, auth redirect URLs, seed paths
├── schemas/              # declarative desired state (one file per area)
│   ├── 10_orders.sql
│   └── 20_orders_rls.sql
├── migrations/           # generated/handwritten, timestamped, immutable once merged
│   └── 20261102031500_orders.sql
├── seed.sql              # fake data only (no real ICs/phones)
└── tests/
    └── orders_rls.test.sql
```
**Daily loop**:
```bash
supabase start                                     # Docker stack
supabase db diff -f orders_status_enum             # schema files/Studio → new migration file
supabase db reset                                  # re-apply all migrations + seed from scratch
supabase test db                                   # pgTAP tests (RLS, functions)
supabase gen types typescript --local > packages/db/types.ts
```
**RLS test** (pgTAP; runs in `supabase test db`):
```sql
begin;
select plan(2);
select set_config('request.jwt.claims', '{"sub":"u1","tenant_id":"t1","role":"authenticated"}', true);
set local role authenticated;
select is((select count(*) from orders where tenant_id = 't2')::int, 0, 'cannot see other tenant');
select lives_ok($$insert into orders (tenant_id, total) values ('t1', 10)$$, 'can insert own tenant');
select * from finish();
rollback;
```
**CI deploy** (GitHub Actions; staging then production with approval):
```yaml
jobs:
  migrate-staging:
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4
      - uses: supabase/setup-cli@v1
        with: { version: latest }        # pin a version for reproducible deploys
      - run: supabase link --project-ref "$STAGING_REF"
        env: { SUPABASE_ACCESS_TOKEN: "${{ secrets.SUPABASE_ACCESS_TOKEN }}", SUPABASE_DB_PASSWORD: "${{ secrets.STAGING_DB_PASSWORD }}" }
      - run: supabase db push --dry-run && supabase db push
        env: { SUPABASE_ACCESS_TOKEN: "${{ secrets.SUPABASE_ACCESS_TOKEN }}", SUPABASE_DB_PASSWORD: "${{ secrets.STAGING_DB_PASSWORD }}" }
```
**Fix migration history mismatch** (remote has a version your repo doesn't, or vice versa):
```bash
supabase migration list                             # compare local vs remote
supabase migration repair --status reverted 20261020101500
supabase db pull                                    # only to capture known, intended remote changes
```
**Wrong way**:
```text
❌ Adding a column in production Studio "quickly" → next db push fails or silently diverges
❌ Editing an already-applied migration file → environments disagree about the schema forever
❌ seed.sql exported from production → customer PII in every preview branch and laptop
❌ Big table change in one migration (add NOT NULL + backfill + index) → long locks during deploy
```

## Patterns & Anti-patterns
| Pattern | When | Anti-pattern to avoid |
|---|---|---|
| Schema in Git, CI applies it | Every project | Manual SQL in the dashboard |
| Declarative schemas for tables/RLS, migrations for data changes | Ongoing development | Hand-diffing in your head |
| `db reset` must pass before PR merge | Always | Migrations that only work on an existing DB |
| Expand → migrate → contract (add nullable, backfill, enforce, drop later) | Live production tables | Rename/drop columns used by deployed clients (Flutter apps lag) |
| Preview branch + Coolify preview app per PR | Feature work | Testing schema changes on the shared staging DB |
| Generated types committed (or generated in CI) | TS/Flutter clients | Hand-maintained types drifting from the schema |
| Seed with fake Malaysian-format data | Local, previews | Real customer data outside production |

## Performance & Trade-offs
- **Branch cost**: each preview branch is a running compute instance billed per hour. Auto-pause/delete on PR close, and don't leave dozens open.
- **Reset speed**: `db reset` replays every migration. Hundreds of files slow it down, so periodically **squash** old migrations into a baseline (`supabase migration squash`) in a coordinated change.
- **Declarative diffing** depends on the diff engine. pg-delta is newer (alpha) and the classic diff (migra-based) has known gaps (some grants, ownership, comments). Always read the generated SQL.
- **Locks**: DDL in migrations takes locks. Long-running ones (`ALTER TABLE … SET NOT NULL` on big tables, non-concurrent indexes) need the safe patterns from [[PostgreSQL - Partitioning & Large Table Maintenance]].

## Tips & Reminders
> [!tip] In ZP's stack
> - **One repo per product**: `supabase/` next to `apps/web` (Next.js) and `apps/mobile` (Flutter). CI runs `db reset` + pgTAP + `gen types` on every PR ([[Testing Strategy - Integration Testing with Testcontainers & Supabase]]).
> - **Environments**: local → preview branch (per PR) → staging project → production project, each with its own keys in Coolify environments ([[Coolify - Preview Deployments, Environments & Secrets]]).
> - **Flutter clients** in the stores lag behind deploys by weeks, so keep columns and RPC signatures backward compatible for at least one app release.

- Put `auth` redirect URLs for previews (`https://*.preview.client.my/**`) in `config.toml` and the branch settings.
- Never `alter` Supabase-managed schemas (`auth`, `storage`, `realtime`) beyond documented hooks. Platform upgrades may overwrite or conflict.
- Keep extensions in the `extensions` schema and enable them in migrations (`create extension if not exists pg_trgm with schema extensions;`).
- Self-hosted Supabase has no branching. Apply the same migrations with `supabase db push --db-url "$URL"`.

## Version Notes
| Date / version | Change |
|---|---|
| 2023–24 | Branching (Git-based preview DBs) launched, then GA |
| 2025 | Declarative schemas (`supabase/schemas`) in the CLI |
| 2026-05 | Branching without Git becomes the default. Merges use pg-delta diffs (alpha) |
| CLI v2.106 (2026-06-11) | `db schema declarative sync --apply` records the platform baseline. `branches create --git-branch`. Faster `db pull` with pg-delta |

> [!warning] Unverified — check before relying on this
> Exact `db diff` behaviour under the new declarative flow, branch pricing and auto-pause defaults weren't confirmed. Check https://supabase.com/docs/guides/deployment/branching and the CLI reference for your CLI version.

## Critical Issues & Gotchas
> [!danger] Schema drift from dashboard edits
> Changes made directly in production Studio aren't in Git. The next `db push` either fails on conflicts or applies migrations against a schema nobody can reproduce, and restores/preview branches won't match production. Restrict production dashboard access, and if drift happens, capture it with `db pull` into a reviewed migration immediately.

> [!danger] Destructive migrations against production
> A `drop column` or a wrong `update` without `where` in a migration runs with full privileges on deploy. Require PR review for every migration, run `db push --dry-run` in CI, take a backup or confirm PITR before risky deploys, and keep destructive steps in separate "contract" migrations after clients stop using the column ([[PostgreSQL - Backups, PITR & Restore Drills]]).

- **Migration timestamp collisions** in parallel branches: two PRs adding migrations apply in timestamp order, not merge order. Rebase and re-check `db reset` before merging.
- **RLS forgotten on new tables**: diff-generated migrations don't add policies. Enable RLS in the same migration, and have a pgTAP or SQL lint check that every `public` table has RLS on ([[Supabase - Row Level Security]]).
- **Realtime publications and storage policies** are part of the schema too. Include them in migrations, or previews behave differently.
- **Docker resource limits** on laptops: the full local stack needs ~2–4 GB RAM. Disable unused services in `config.toml`.

## Related
- [[Supabase]]
- [[Supabase - Row Level Security]] — policies that must ship with every table
- [[Testing Strategy - Integration Testing with Testcontainers & Supabase]] — pgTAP and real-DB tests
- [[CI-CD - Deploy Pipelines for Coolify & Flutter Releases]] — pipeline stages around `db push`
- [[Coolify - Preview Deployments, Environments & Secrets]] — preview apps paired with preview branches
- [[Git - Branching, Releases & Recovery]] — branch and release flow the migrations follow

## References
- https://supabase.com/docs/guides/local-development
- https://supabase.com/docs/guides/local-development/declarative-database-schemas
- https://supabase.com/docs/guides/deployment/branching
- https://supabase.com/blog/branching-without-git-is-now-the-default
- https://github.com/supabase/cli/releases/tag/v2.106.0
