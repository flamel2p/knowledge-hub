---
title: Next.js - Middleware, Auth & Multi-Tenant Routing
aliases: [proxy.ts, Next.js middleware, multi-tenant Next.js, subdomain routing, custom domains, tenant rewrite, session refresh]
type: deep-dive
domain: frontend
tags: [domain/frontend, type/deep-dive, topic/nextjs, topic/multi-tenancy, topic/auth, lang/typescript]
status: draft
created: 2026-10-28
updated: 2026-10-28
version_checked: "Next.js 16 (proxy.ts on Node runtime; middleware.ts deprecated for Edge) · 16.2 LTS / 16.3 — 2026-10"
parent: "[[Next.js]]"
related: ["[[Next.js - App Router & Rendering Strategies]]", "[[Next.js - Caching]]", "[[System Design - Multi-Tenant SaaS Architecture]]", "[[Authentication & Authorization - Multi-Tenant Authorization with RLS]]", "[[Supabase]]", "[[Traefik]]"]
---

# Next.js - Middleware, Auth & Multi-Tenant Routing

> [!info] Deep dive of [[Next.js]]

> [!abstract] TL;DR
> In Next.js 16, request interception lives in **`proxy.ts`** (formerly `middleware.ts`). It runs on the **Node.js runtime** before routing. Use it for three jobs:
> 1. resolve the tenant from the `Host` (subdomain or custom domain) and **rewrite** to `/[tenant]/…`;
> 2. refresh the Supabase session cookie;
> 3. redirect anonymous users away from app routes.
>
> It is **not** the authorization boundary (CVE-2025-29927 proved that). Re-check user and tenant in every Server Action, route handler and query (RLS). Strip any client-supplied `x-tenant-*` headers before setting your own.

## Concept
| Concern | Where it belongs | Why |
|---|---|---|
| Host → tenant resolution | `proxy.ts` | One place, before routing, cheap if cached |
| Session refresh (Supabase `@supabase/ssr`) | `proxy.ts` | Server Components can't write cookies |
| Coarse gating (anonymous → `/login`) | `proxy.ts` | UX: avoid rendering protected shells |
| **Authorization** (may this user act on this tenant/object?) | Server Actions, route handlers, data layer + **RLS** | The proxy can be bypassed or misconfigured. Data access must re-check |
| Locale / A-B / geo variants | `proxy.ts` rewrites or headers | No client flicker |

**Tenant addressing options**:

| Style | Example | TLS / routing | Notes |
|---|---|---|---|
| Path | `app.example.my/t/kedai-ali` | One cert | Simplest; weakest branding |
| Subdomain | `kedai-ali.example.my` | **Wildcard cert** (DNS-01) | Common default for SaaS |
| Custom domain | `order.kedaiali.my` | Per-domain cert (HTTP-01/ACME or Cloudflare for SaaS) | Highest effort: domain verification, cert automation, cookie scope |

## How It Works
```mermaid
sequenceDiagram
  participant B as Browser
  participant CF as Cloudflare / Traefik
  participant PX as proxy.ts (Node)
  participant APP as App route /[tenant]/...
  participant DB as Postgres (RLS)
  B->>CF: GET https://kedai-ali.example.my/orders
  CF->>PX: Host: kedai-ali.example.my
  PX->>PX: strip x-tenant-*; resolve host → tenant (cache)
  PX->>PX: refresh Supabase session cookie
  PX->>APP: rewrite /kedai-ali/orders + header x-tenant-id
  APP->>DB: query as user (RLS checks membership in tenant)
  DB-->>APP: rows for that tenant only
```

## Practical Usage
**`proxy.ts`**: tenant rewrite + session refresh + gating:
```ts
import { NextResponse, type NextRequest } from 'next/server';
import { createServerClient } from '@supabase/ssr';
import { resolveTenant } from '@/lib/tenants';            // host → { id, slug } with an in-memory LRU + TTL

const ROOT = process.env.ROOT_DOMAIN!;                    // example.my

export async function proxy(req: NextRequest) {
  const host = (req.headers.get('host') ?? '').toLowerCase().replace(/:\d+$/, '');
  const url = req.nextUrl.clone();

  const headers = new Headers(req.headers);
  for (const h of [...headers.keys()]) if (h.startsWith('x-tenant-')) headers.delete(h);   // never trust client values
  headers.delete('x-middleware-subrequest');                                               // CVE-2025-29927 defence in depth

  if (host === ROOT || host === `www.${ROOT}`) return NextResponse.next({ request: { headers } });  // marketing site

  const tenant = await resolveTenant(host);               // subdomain or verified custom domain
  if (!tenant) return NextResponse.rewrite(new URL('/not-found-tenant', req.url));
  headers.set('x-tenant-id', tenant.id);

  const res = NextResponse.rewrite(new URL(`/${tenant.slug}${url.pathname}${url.search}`, req.url), { request: { headers } });

  const supabase = createServerClient(process.env.NEXT_PUBLIC_SUPABASE_URL!, process.env.NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY!, {
    cookies: {
      getAll: () => req.cookies.getAll(),
      setAll: (list) => list.forEach(({ name, value, options }) => res.cookies.set(name, value, options)),
    },
  });
  const { data: { user } } = await supabase.auth.getUser();          // refreshes + validates the session

  if (!user && url.pathname.startsWith('/app')) {
    return NextResponse.redirect(new URL(`/login?next=${encodeURIComponent(url.pathname)}`, req.url));
  }
  return res;
}

export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico|robots.txt|sitemap.xml|.*\\.(?:png|jpg|svg|webp|css|js)$).*)'],
};
```
**Re-check in the data layer** (the real boundary):
```ts
'use server';
export async function updateMenuItem(input: unknown) {
  const tenantId = (await headers()).get('x-tenant-id')!;            // routing hint only
  const { user, db } = await requireUser();                           // Supabase client bound to the user's JWT
  const data = UpdateMenuItem.parse(input);
  const { error } = await db.from('menu_items').update(data).eq('id', data.id).eq('tenant_id', tenantId);  // RLS enforces membership
  if (error) throw error;
}
```
**Tenant resolution cache** (avoid a DB hit per request):
```ts
const cache = new Map<string, { t: Tenant | null; exp: number }>();
export async function resolveTenant(host: string) {
  const hit = cache.get(host);
  if (hit && hit.exp > Date.now()) return hit.t;
  const t = await db.oneOrNone('select id, slug from tenant_domains where host = $1 and verified', [host]);
  cache.set(host, { t, exp: Date.now() + 60_000 });
  return t;
}
```
**Infra**:
- Wildcard DNS `*.example.my` → Cloudflare → Traefik with a **DNS-01 wildcard cert** ([[Traefik - Middlewares, TLS & Routing]]).
- Custom domains: the client adds a CNAME, you verify ownership (TXT), then issue a cert via Traefik HTTP-01 or Cloudflare for SaaS custom hostnames.

**Wrong way**:
```ts
if (!req.cookies.get('sb-access-token')) return redirect('/login');   // ❌ presence ≠ validity; auth only here = bypassable
const tenantId = req.headers.get('x-tenant-id');                       // ❌ client-controlled unless you strip it first
```

## Patterns & Anti-patterns
| Pattern | When | Anti-pattern to avoid |
|---|---|---|
| Rewrite to `app/[tenant]/…` segments | Subdomain/custom-domain SaaS | Branching on host inside every page |
| Strip-then-set internal headers | Any header passed downstream | Trusting incoming `x-tenant-id` / `x-user-id` |
| `getUser()` (validated) in proxy, `requireUser()` in actions | Supabase auth | `getSession()` alone on the server (not revalidated) |
| Narrow `matcher` excluding static assets | Always | Proxy running for every image and chunk |
| Tenant in cache keys/tags (`tag: tenant:${id}:menu`) | Cached data | Shared cache entries leaking across tenants ([[Next.js - Caching]]) |
| Verified `tenant_domains` table (TXT check) | Custom domains | Accepting any Host header pointed at your IP |

## Performance & Trade-offs
- **Proxy runs on every matched request**. Keep it to a cache lookup + cookie work (~1 ms). A DB query per request adds latency and load. Cache host→tenant (in-memory LRU per instance, or Redis for many instances) and invalidate on domain changes.
- **Node runtime** (default in 16) allows normal DB/SDK clients, but runs in your server region, not at the edge. That's fine behind Cloudflare for an SME SaaS.
- **Subdomains vs paths**: subdomains isolate cookies per tenant if you scope them per host, and look professional. They cost wildcard certs and DNS setup. Paths are simpler but share one cookie jar.
- **Custom domains** multiply cert issuance (Let's Encrypt rate limits) and support load. Cloudflare for SaaS offloads certs for a fee per hostname.

## Tips & Reminders
> [!tip] In ZP's stack
> - **Hosting**: one Next.js app on Coolify serves all tenants. Cloudflare (wildcard proxied DNS) → Traefik wildcard cert (Cloudflare DNS-01 token) → `proxy.ts` tenant rewrite → Supabase with RLS by tenant membership ([[Authentication & Authorization - Multi-Tenant Authorization with RLS]]).
> - **Custom domains**: offer them only on higher plans, priced to cover the setup.

- After upgrading to 16, run `npx @next/codemod@canary middleware-to-proxy .`. A leftover `middleware.ts` compiles cleanly but may not do what you expect.
- Log `tenant_id` and `host` on every request (and in traces). Cross-tenant bugs are only debuggable with it.
- Return a branded "tenant not found" page for unknown hosts, not your marketing site, to avoid SEO duplicates and confusion.
- Keep auth cookies host-only (no `Domain`) per tenant subdomain unless you deliberately want SSO across tenants ([[Browser & Web Fundamentals - Storage, Cookies & Service Workers]]).

## Version Notes
| Version | Change relevant here |
|---|---|
| 12 (2021) | Middleware introduced (Edge runtime) |
| 13–15 | Middleware matured. Node runtime for middleware experimental, then stable (15.x) |
| 2025-03 | **CVE-2025-29927**: `x-middleware-subrequest` header bypassed middleware (fixed 12.3.5 / 13.5.9 / 14.2.25 / 15.2.3) |
| 16 (2025-10) | **`proxy.ts`** replaces `middleware.ts` (Node runtime, not configurable). `middleware.ts` kept only for Edge and deprecated. Codemod `middleware-to-proxy` |
| 2026 | Monthly security releases, LTS lines (16.2 Active LTS). Keep patched |

## Critical Issues & Gotchas
> [!danger] Proxy-only authorization
> CVE-2025-29927 let attackers skip middleware entirely with one header, and every app that authorized only in middleware was exposed. Treat `proxy.ts` as routing and UX. Enforce authorization in Server Actions, route handlers and Postgres RLS.

> [!danger] Host header trust
> Your server receives whatever `Host` the client sends if it's reachable directly. Without a verified domain table and an origin locked behind Cloudflare/Traefik, attackers can map arbitrary hosts to tenants, poison caches, or generate password-reset links with their domain. Build absolute URLs from configured domains, not the raw Host.

- **Rewrites and caching**: rewritten tenant paths must include the tenant in every cache key and tag, or one tenant's cached page can be served to another.
- **Static generation per tenant** (`generateStaticParams`) explodes build time with many tenants. Use on-demand rendering with caching instead.
- **Redirect loops** between proxy rules (`/login` ↔ protected routes) when the session cookie is set on a different host. Check cookie domain and `Secure` on custom domains.
- **Edge-only code** moved into `proxy.ts` on Node may behave differently (globals, crypto APIs). Test after migration.

## Related
- [[Next.js]]
- [[Next.js - App Router & Rendering Strategies]] — dynamic segments and rendering modes
- [[Next.js - Caching]] — tenant-safe cache tags
- [[System Design - Multi-Tenant SaaS Architecture]] — tenancy models and isolation
- [[Authentication & Authorization - Multi-Tenant Authorization with RLS]] — the real authorization layer
- [[Traefik]] — wildcard and per-domain certificates

## References
- https://nextjs.org/blog/next-16
- https://nextjs.org/docs/app/api-reference/file-conventions/proxy
- https://vercel.com/academy/nextjs-foundations/proxy-basics
- https://supabase.com/docs/guides/auth/server-side/nextjs
- https://dev.to/shubhradev/i-got-the-proxyts-matcher-wrong-for-three-projects-before-i-understood-why-4e5c
