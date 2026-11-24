---
title: Next.js - Content Security Policy, Nonces & Security Headers
aliases: [Next.js CSP, Content-Security-Policy Next.js, CSP nonce, strict-dynamic, proxy.ts CSP, security headers Next.js, HSTS, frame-ancestors, CSP report-only, Trusted Types, SRI Next.js, XSS mitigation]
type: deep-dive
domain: frontend
tags: [domain/frontend, type/deep-dive, topic/nextjs, topic/security, topic/csp, lang/typescript]
status: draft
created: 2026-11-24
updated: 2026-11-24
version_checked: "Next.js 16.4.x (proxy.ts, nonce-based CSP forces dynamic rendering, experimental SRI for static) · CSP Level 3 — 2026-11"
parent: "[[Next.js]]"
related: ["[[OWASP Top 10 - Web Application Top 10 2025 in Practice]]", "[[Next.js - Middleware, Auth & Multi-Tenant Routing]]", "[[Network Security - Common Web Attacks]]", "[[Next.js - Performance Profiling & Bundle Optimization]]", "[[Cloudflare - Turnstile & Bot Protection for Forms]]", "[[Next.js - Payments with Stripe & Malaysian Gateways]]"]
---

# Next.js - Content Security Policy, Nonces & Security Headers

> [!info] Deep dive of [[Next.js]]

> [!abstract] TL;DR
> - **Baseline headers for every Next.js app** (via `next.config` `headers()` or the proxy): HSTS, `X-Content-Type-Options: nosniff`, `Referrer-Policy`, `Permissions-Policy`, and **CSP `frame-ancestors`** (clickjacking).
> - **Full CSP** that actually stops XSS needs **nonces + `strict-dynamic`**, generated per request in **`proxy.ts`** (formerly middleware). Next.js applies the nonce to its own scripts automatically.
>   - Trade-off: nonce-based CSP makes pages **dynamically rendered**, so you lose static/ISR caching for those routes.
>   - Alternatives: hash-based CSP or experimental SRI for static pages, or a relaxed CSP (`'self'` + allow-listed hosts) for marketing pages.
> - **Rollout**: start with **`Content-Security-Policy-Report-Only`** + a report endpoint, fix violations (Stripe, Turnstile, analytics, Supabase, fonts), then enforce.
> - CSP is defense in depth. It doesn't replace output encoding, avoiding `dangerouslySetInnerHTML`, and sanitizing user HTML.

## Concept
| Header | Recommended value | Protects against |
|---|---|---|
| `Strict-Transport-Security` | `max-age=63072000; includeSubDomains; preload` | SSL stripping |
| `Content-Security-Policy` | Nonce + `strict-dynamic` (see below) | XSS, data injection, clickjacking (`frame-ancestors`) |
| `X-Content-Type-Options` | `nosniff` | MIME sniffing |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | URL leakage (tokens in paths) |
| `Permissions-Policy` | `camera=(), microphone=(), geolocation=()` (enable as needed) | Feature abuse by injected scripts/iframes |
| `Cross-Origin-Opener-Policy` | `same-origin` (check OAuth popups) | Cross-window attacks |
| `X-Frame-Options` | `DENY` (legacy, CSP `frame-ancestors` supersedes) | Clickjacking in old browsers |

**CSP strategies**:

| Strategy | Security | Caching | Pick it when… |
|---|---|---|---|
| Nonce + `strict-dynamic` | Strong | Dynamic rendering only | Apps with auth, payments, user content |
| Hashes / SRI (experimental for static) | Strong for static scripts | Static OK | Mostly static sites |
| Allow-list (`'self'` + hosts), no inline | Medium (bypassable via allowed hosts) | Static OK | Marketing sites, quick wins |
| `'unsafe-inline'` | Weak | Static OK | Avoid for scripts |

## How It Works
```mermaid
sequenceDiagram
  participant B as Browser
  participant P as proxy.ts
  participant R as Next.js render
  B->>P: GET /dashboard
  P->>P: nonce = random 128-bit (base64)
  P->>R: forward with header x-nonce + CSP header
  R->>R: Next.js adds nonce to its <script> tags; your <Script nonce={nonce}>
  R-->>B: HTML + Content-Security-Policy: script-src 'nonce-…' 'strict-dynamic'
  B->>B: executes only nonce'd scripts (and scripts they load); blocks injected inline JS
  B-->>B: violations → report endpoint (report-to)
```

## Practical Usage
**Static baseline headers** (`next.config.ts`):
```ts
const securityHeaders = [
  { key: "Strict-Transport-Security", value: "max-age=63072000; includeSubDomains; preload" },
  { key: "X-Content-Type-Options", value: "nosniff" },
  { key: "Referrer-Policy", value: "strict-origin-when-cross-origin" },
  { key: "Permissions-Policy", value: "camera=(), microphone=(), geolocation=(self)" },
  { key: "Cross-Origin-Opener-Policy", value: "same-origin-allow-popups" },   // OAuth popups
];
export default { async headers() { return [{ source: "/(.*)", headers: securityHeaders }]; } };
```
**Nonce-based CSP in `proxy.ts`**:
```ts
import { NextResponse, type NextRequest } from "next/server";

export function proxy(req: NextRequest) {
  const nonce = Buffer.from(crypto.randomUUID()).toString("base64");
  const isDev = process.env.NODE_ENV !== "production";
  const csp = [
    `default-src 'self'`,
    `script-src 'self' 'nonce-${nonce}' 'strict-dynamic' https://js.stripe.com https://challenges.cloudflare.com${isDev ? " 'unsafe-eval'" : ""}`,
    `style-src 'self' 'unsafe-inline'`,                           // tighten with nonces if feasible
    `img-src 'self' data: blob: https://*.supabase.co`,
    `font-src 'self'`,
    `connect-src 'self' https://*.supabase.co wss://*.supabase.co https://api.stripe.com`,
    `frame-src https://js.stripe.com https://challenges.cloudflare.com`,
    `frame-ancestors 'none'`,
    `base-uri 'self'`, `form-action 'self'`, `object-src 'none'`,
    `upgrade-insecure-requests`,
    `report-to csp`,
  ].join("; ");

  const headers = new Headers(req.headers);
  headers.set("x-nonce", nonce);
  headers.set("Content-Security-Policy", csp);
  const res = NextResponse.next({ request: { headers } });
  res.headers.set("Content-Security-Policy", csp);              // swap to ...-Report-Only during rollout
  res.headers.set("Reporting-Endpoints", `csp="/api/csp-report"`);
  return res;
}
export const config = { matcher: [{ source: "/((?!_next/static|_next/image|favicon.ico).*)" }] };
```
**Use the nonce for your own scripts**:
```tsx
import { headers } from "next/headers";
import Script from "next/script";
export default async function RootLayout({ children }: { children: React.ReactNode }) {
  const nonce = (await headers()).get("x-nonce") ?? undefined;
  return (
    <html lang="ms"><body>
      {children}
      <Script src="https://www.googletagmanager.com/gtag/js?id=G-XXXX" strategy="afterInteractive" nonce={nonce} />
    </body></html>
  );
}
```
**Violation report endpoint** (sample and rate-limit; don't store PII):
```ts
export async function POST(req: Request) {
  const body = await req.text();
  console.warn("csp-violation", body.slice(0, 2000));
  return new Response(null, { status: 204 });
}
```
**Wrong way**:
```text
❌ script-src 'self' 'unsafe-inline' 'unsafe-eval' https:   → CSP present but useless against XSS
❌ Nonce generated once at build/startup                    → same nonce for everyone = bypassable
❌ Enforcing a new CSP directly in production               → broken checkout (Stripe/Turnstile blocked)
❌ dangerouslySetInnerHTML with user content "because CSP"   → stored XSS still possible via allowed vectors
```

## Patterns & Anti-patterns
| Pattern | When | Anti-pattern to avoid |
|---|---|---|
| Report-Only → fix → enforce | Every rollout | Big-bang enforcement |
| Nonce CSP on authenticated routes, relaxed on static marketing | Mixed apps | Forcing dynamic rendering everywhere |
| `frame-ancestors 'none'` (or specific partners) | All apps | Missing clickjacking protection |
| Per-third-party allow-lists documented | Stripe, Turnstile, analytics | Wildcards like `https:` |
| Sanitize user HTML (DOMPurify) + CSP | Rich text/CMS | Relying on CSP alone |
| Header checks in CI (securityheaders scan) | Releases | Headers silently dropped by proxy changes |

## Performance & Trade-offs
- **Nonce CSP = dynamic rendering**: pages read the request header, so no static/ISR caching or CDN HTML caching. Higher server load on Coolify. Scope the proxy matcher to app routes ([[Next.js - Performance Profiling & Bundle Optimization]]).
- **`style-src 'unsafe-inline'`** is common because CSS-in-JS/Tailwind runtime styles and some libraries inject styles. Lower risk than script injection, but tighten where possible.
- **Third-party scripts** (chat widgets, tag managers) often load more scripts dynamically. `strict-dynamic` helps, but tag managers can effectively reintroduce arbitrary script execution. Limit them.
- **Report volume** can be high (browser extensions cause noise). Sample and filter.

## Tips & Reminders
> [!tip] In ZP's stack
> - **Typical allow-list**: Supabase (`*.supabase.co` https + wss), Stripe (`js.stripe.com`, `api.stripe.com`), Cloudflare Turnstile (`challenges.cloudflare.com`), local gateways' redirect domains (form-action/navigation), and analytics if used ([[Cloudflare - Turnstile & Bot Protection for Forms]], [[Next.js - Payments with Stripe & Malaysian Gateways]]).
> - **Headers at which layer**: set them in Next.js (versioned with code) rather than only in Traefik labels, so they survive infrastructure changes. Avoid duplicating conflicting values at both layers.
> - **Deliverable**: an A-grade securityheaders.com report is a quick, visible win for client security reviews ([[OWASP Top 10 - Web Application Top 10 2025 in Practice]]).

- Test OAuth/social login popups with your COOP setting.
- `upgrade-insecure-requests` plus HSTS. Submit to the preload list only when all subdomains are HTTPS.
- Include CSP in E2E tests (Playwright can listen for `securitypolicyviolation` events).
- Revisit the CSP when adding any new third-party script.

## Version Notes
| Version | Change relevant here |
|---|---|
| Next.js 13.4+ | App Router. Nonce support documented for CSP |
| Next.js 15 | Experimental SRI (`experimental.sri`) for static builds |
| Next.js 16 (2025-10) | `middleware.ts` renamed `proxy.ts` (function `proxy`). Nonce flow unchanged |
| CSP Level 3 | `strict-dynamic`, `report-to`, nonces/hashes for scripts |

> [!warning] Unverified — check before relying on this
> Whether Next.js 16.4 still requires dynamic rendering for nonce CSP (vs. newer PPR/Cache Components options), the exact SRI config key, and current Stripe/Turnstile CSP host lists weren't verified this run. Check the Next.js CSP guide and vendor CSP docs.

## Critical Issues & Gotchas
> [!danger] A permissive CSP gives false confidence
> Policies with `'unsafe-inline'`, `'unsafe-eval'` or broad `https:` sources pass audits but don't stop XSS. Attackers inject inline handlers or load scripts from any HTTPS host. Use nonces with `strict-dynamic` for app routes, keep allow-lists tight, and verify with CSP evaluators.

> [!danger] Breaking payments and login with an untested CSP
> Enforcing CSP without Report-Only testing commonly blocks Stripe Elements iframes, Turnstile challenges, Supabase Realtime websockets or OAuth redirects. Checkout and login fail silently for users while monitoring shows "200 OK". Roll out with Report-Only, test payment/auth flows end to end, then enforce.

- **Static pages** can't receive per-request nonces. Exclude them from the nonce matcher or use hashes.
- **Inline event handlers** (`onclick="…"`) in third-party HTML fail under strict CSP. Refactor or sandbox in iframes.
- **Duplicate CSP headers** (Next.js + Traefik + Cloudflare) are combined by browsers as **all must pass**, giving unexpectedly strict results.
- **Browser extensions** inject scripts and generate violation noise. Filter reports by source.

## Related
- [[Next.js]]
- [[OWASP Top 10 - Web Application Top 10 2025 in Practice]] — headers in the security baseline
- [[Next.js - Middleware, Auth & Multi-Tenant Routing]] — proxy.ts mechanics
- [[Network Security - Common Web Attacks]] — XSS and clickjacking
- [[Next.js - Performance Profiling & Bundle Optimization]] — dynamic rendering cost
- [[Cloudflare - Turnstile & Bot Protection for Forms]] — Turnstile CSP requirements

## References
- https://nextjs.org/docs/app/guides/content-security-policy
- https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP
- https://csp-evaluator.withgoogle.com/
- https://cheatsheetseries.owasp.org/cheatsheets/Content_Security_Policy_Cheat_Sheet.html
- https://securityheaders.com/
