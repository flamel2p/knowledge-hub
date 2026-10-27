---
title: JWT - Refresh Tokens, Revocation & Storage
aliases: [JWT revocation, refresh token storage, logout JWT, jti denylist, token storage localStorage vs cookie, Supabase sessions, logout everywhere]
type: deep-dive
domain: security
tags: [domain/security, type/deep-dive, topic/jwt, topic/sessions, topic/auth]
status: draft
created: 2026-10-19
updated: 2026-10-27
version_checked: "Supabase Auth sessions (rotation on, reuse interval, time-box/inactivity, signOut scopes) — 2026-10"
parent: "[[JWT]]"
related: ["[[OAuth 2.0 & OIDC - Flows, PKCE & Token Lifecycle]]", "[[JWT - Signing Algorithms, Keys & Rotation]]", "[[Supabase]]", "[[Redis]]", "[[Next.js]]", "[[Flutter]]"]
---

# JWT - Refresh Tokens, Revocation & Storage

> [!info] Deep dive of [[JWT]]

> [!abstract] TL;DR
> - **Tokens**: short-lived JWT access tokens (5–15 min) plus **rotating, server-tracked refresh tokens**.
> - **Logout**: revoke the refresh token / session, and accept that the current access token lives until `exp`. For instant revocation on sensitive actions, add a **session check** (`session_id` exists) or a **`jti`/token-version denylist** in [[Redis]].
> - **Web storage**: HttpOnly `__Host-` cookies via a BFF/SSR layer, **not `localStorage`**.
> - **Mobile storage**: Keychain/Keystore via secure storage.
> - **Supabase**: rotation is on by default, `signOut()` is **global by default**, and revoked sessions' access tokens stay valid until expiry.

## Concept

### Where to store tokens
| Client | Access token | Refresh token | Main threat | Notes |
|---|---|---|---|---|
| **Web with BFF/SSR** ([[Next.js]]) | Server-side (session store) or HttpOnly cookie | Server-side only | CSRF (mitigated by SameSite + origin checks) | **Best**: JS never sees tokens |
| SPA without backend | Memory only | HttpOnly cookie scoped to refresh endpoint, or none (re-auth) | XSS: script runs as the user regardless; can't exfiltrate HttpOnly cookie | `localStorage` = full exfiltration on any XSS |
| Mobile ([[Flutter]]) | Memory | Keychain (iOS) / Keystore-backed EncryptedSharedPreferences (Android) | Device compromise, backups | Exclude from cloud backups. Biometric gate for high-value apps |
| CLI / desktop | Memory | OS keychain (libsecret, macOS Keychain, Windows Credential Manager) | Malware/infostealers | Never plaintext `~/.config/app/token.json` |
| Server-to-server | Memory cache until `exp` | — (client credentials re-issue) | Log leakage | Redact `Authorization` in logs/traces |

### Cookie attributes that matter
```http
Set-Cookie: __Host-sid=…; Path=/; Secure; HttpOnly; SameSite=Lax; Max-Age=2592000
```
- **`__Host-` prefix**: requires `Secure`, `Path=/` and no `Domain`, so subdomains can't overwrite it (cookie tossing).
- **`SameSite=Lax`**: blocks cross-site POST CSRF while allowing top-level OAuth/redirect returns. `Strict` breaks IdP returns.
- **`HttpOnly`**: no JS access. Combine with a CSP to reduce XSS impact.
- **CSRF**: also verify `Origin`/`Sec-Fetch-Site` on state-changing requests, or use a CSRF token for forms.

### Revocation strategies
| Strategy | Revocation latency | Cost | Use for |
|---|---|---|---|
| Short `exp` + refresh rotation | ≤ access TTL | Refresh traffic | Baseline for everything |
| Session check (`session_id` claim → session row exists) | Immediate | DB/cache lookup per request | Admin actions, payments, account settings |
| `jti` denylist (Redis, TTL = remaining `exp`) | Immediate | Redis lookup per request | Logout-now, stolen-token response |
| Per-user `token_version` (claim must equal current version) | Immediate after bump | Lookup (cacheable) | "Log out all devices", password change, role removal |
| Rotate signing key | Immediate for all | Mass logout | Key compromise only ([[JWT - Signing Algorithms, Keys & Rotation]]) |

## How It Works
```mermaid
sequenceDiagram
  participant C as Client (BFF / app)
  participant A as Auth server (Supabase Auth / IdP)
  participant API as API
  participant R as Redis (denylist / versions)
  C->>A: refresh_token (rotating)
  A-->>C: access JWT (exp 10 min, sid, ver) + NEW refresh token
  C->>API: Bearer access JWT
  API->>API: verify signature, iss, aud, exp (local)
  API->>R: sensitive route? check sid not revoked / ver current
  R-->>API: ok
  Note over C,A: Logout everywhere → A revokes all refresh tokens/sessions;<br/>API bumps user ver in R → old access JWTs rejected on sensitive routes immediately
```
- **Rotation**: each refresh returns a new refresh token and invalidates the old one. Reuse of an old token signals theft, so revoke the token family ([[OAuth 2.0 & OIDC - Flows, PKCE & Token Lifecycle]] has the reuse-detection schema).
- **Concurrency**: tabs or background tasks refreshing simultaneously can trip reuse detection. Supabase offers a short **reuse interval** for this. Clients should also serialise refreshes (single-flight).

## Practical Usage

### Supabase sessions
- **Rotation**: on by default. Refresh tokens never expire on their own but are single-use. `REFRESH_TOKEN_REUSE_INTERVAL` allows a brief reuse window for concurrent refreshes.
- **Session limits**: a time-box (max lifetime) and an inactivity timeout can be configured. Expired sessions are cleaned up progressively, about 24 h after reaching that state.
- **Sign-out scopes**: `supabase.auth.signOut()` defaults to **`global`** (all sessions). Use `{ scope: 'local' }` for this device only, or `'others'` for all other devices.
- **Revocation lag**: access tokens of revoked sessions stay valid until `exp`. For immediate enforcement, check the `session_id` claim against `auth.sessions`:
```sql
-- RLS helper for sensitive tables: require a live session, not just a valid JWT
create or replace function private.session_is_active() returns boolean
language sql stable security definer set search_path = '' as $$
  select exists (select 1 from auth.sessions s where s.id = (auth.jwt() ->> 'session_id')::uuid)
$$;
revoke all on function private.session_is_active() from public;
grant usage on schema private to authenticated;
grant execute on function private.session_is_active() to authenticated;
create policy payouts_live_session on payouts for all to authenticated
  using (tenant_id = (auth.jwt() -> 'app_metadata' ->> 'tenant_id')::uuid and private.session_is_active());
```
- **SSR**: use `@supabase/ssr` in Next.js so the session lives in cookies managed server-side (middleware refreshes them). Avoid the default browser `localStorage` persistence for sensitive apps.

### Denylist + token version (custom APIs)
```ts
import { jwtVerify } from "jose";

export async function authenticate(token: string, { sensitive = false } = {}) {
  const { payload } = await jwtVerify(token, JWKS, { algorithms: ["ES256"], issuer: ISS, audience: AUD });
  if (sensitive) {
    const [revoked, ver] = await redis.mGet([`revoked:jti:${payload.jti}`, `tokver:${payload.sub}`]);
    if (revoked) throw unauthorized("token revoked");
    if (ver && Number(ver) !== payload.ver) throw unauthorized("session invalidated");
  }
  return payload;
}

export async function logoutNow(payload: { jti: string; exp: number }) {
  const ttl = payload.exp - Math.floor(Date.now() / 1000);
  if (ttl > 0) await redis.set(`revoked:jti:${payload.jti}`, "1", { EX: ttl });   // auto-expires with the token
}

export async function logoutEverywhere(userId: string) {
  await redis.incr(`tokver:${userId}`);          // new tokens get the new ver claim at next refresh/login
  await revokeAllRefreshTokens(userId);          // auth server side
}
```

### Single-flight refresh on the client
```ts
let inflight: Promise<string> | null = null;
export function getFreshAccessToken() {
  inflight ??= refresh().finally(() => { inflight = null; });   // all callers share one refresh request
  return inflight;
}
```

### Flutter secure storage
```dart
const storage = FlutterSecureStorage(
  aOptions: AndroidOptions(encryptedSharedPreferences: true),
  iOptions: IOSOptions(accessibility: KeychainAccessibility.first_unlock_this_device), // not synced to other devices
);
await storage.write(key: 'refresh_token', value: rt);
```

## Patterns & Anti-patterns
| Pattern | When | Anti-pattern to avoid |
|---|---|---|
| BFF/SSR with HttpOnly `__Host-` cookies | Web apps | Tokens in `localStorage`/`sessionStorage` "because the tutorial did it" |
| Short access TTL + rotating refresh | Always | 7-day access JWTs to "avoid refresh complexity" |
| Session/denylist checks only on sensitive routes | High-value actions | Per-request DB check everywhere (then just use sessions) or nowhere |
| Logout-everywhere on password change / staff removal | Always | Password change that leaves old devices logged in |
| Single-flight refresh, reuse grace window | Multi-tab, mobile background | Parallel refreshes triggering family revocation and random logouts |
| Redact tokens in logs, APM, LLM traces | Always | Full `Authorization` headers in Sentry breadcrumbs |

## Performance & Trade-offs
- **Stateless vs revocable**: every revocation mechanism adds state. Keep JWT verification local for most routes, and add Redis lookups (~0.2–1 ms) only where immediate revocation matters.
- **Session checks in RLS** add a subquery per statement. Use them only on sensitive tables, and keep the helper `stable` so it's evaluated once per statement, not per row.
- **Access TTL tuning**: 5 min means faster revocation but more refreshes. 15 min is the common balance. Mobile apps on flaky networks benefit from a slightly longer TTL plus robust refresh.
- **Opaque sessions** (session ID cookie + store) are simpler and instantly revocable for a single web app. JWTs pay off when multiple services or the edge need to verify identity without calling home.

## Tips & Reminders
> [!tip]
> - **Bind sessions to context** for admin panels (IP range/ASN, device fingerprint). Re-authenticate on change, plus step-up auth for payouts ([[Authentication & Authorization - Passkeys & WebAuthn]]).
> - **Show users their active sessions** (device, last seen, location) with "sign out" per session. Supabase exposes sessions in `auth.sessions`.
> - **Revoke on staff offboarding**: remove the membership, bump the token version, and revoke sessions in the same admin action.
> - **In ZP's stack**:
>   - **Next.js + Supabase**: `@supabase/ssr` cookies, 10–15-min access tokens, and `session_is_active()` on payouts/settings tables.
>   - **Flutter**: secure storage with `first_unlock_this_device`.
>   - **Admin logouts**: n8n admin flows call `signOut` with the global scope for staff removal.

## Version Notes
| Item | Note |
|---|---|
| Cookie prefixes (`__Host-`, `__Secure-`) | Supported in all modern browsers |
| `SameSite=Lax` default | Chrome default since 2020 for cookies without `SameSite` |
| Supabase Auth | Rotation default on. Reuse interval configurable. Time-box/inactivity session limits. `signOut` scopes `global`/`local`/`others` |
| RFC 9449 DPoP | Sender-constrained tokens reduce the value of stolen tokens (where IdP supports it) |

> [!warning] Unverified — check before relying on this
> Supabase plan availability of time-box/inactivity settings, and whether Supabase revokes the whole token family on refresh-token reuse, weren't confirmed in official docs. Check the Supabase sessions guide for your plan.

## Critical Issues & Gotchas
> [!danger] Logout that doesn't log out
> - **The trap**: deleting the client-side token or calling sign-out leaves the **access token valid until `exp`**, and if refresh tokens aren't revoked server-side, it leaves the session itself alive.
> - **Who exploits it**: attackers with a stolen token (infostealer, shared device, leaked log) keep access, and staff removed from a tenant keep reading data.
> - **Fix**: revoke server-side, keep access TTL short, and enforce session checks on sensitive data.

> [!danger] XSS turns `localStorage` tokens into account takeover
> - **The exposure**: any XSS (a vulnerable npm package, an injected third-party script, an unsanitised LLM output rendered as HTML) can read `localStorage` and ship refresh tokens to an attacker. The attacker then mints tokens from anywhere for weeks.
> - **HttpOnly isn't immune**: cookies don't stop XSS from acting *in* the page, but they stop silent exfiltration.
> - **Fix**: combine BFF/HttpOnly cookies with CSP and output sanitisation ([[OWASP Top 10 - LLM & Agent Security]]).

> [!warning] Gotchas
> - `SameSite=None` requires `Secure` and re-opens CSRF exposure. Use it only for genuine cross-site embeds.
> - Mobile OS backups can include app storage. Use device-only Keychain accessibility and exclude token files from backups.
> - Clock skew makes freshly issued tokens fail `nbf`/`iat` checks. Allow 30–60 s leeway.
> - JWT claims (roles, tenant) are snapshots. After a role change, users keep old claims until refresh, so force a refresh or bump the version.
> - Denylist keys without TTL grow forever. Always set TTL = remaining token lifetime.

## Related
- [[JWT]]
- [[Browser & Web Fundamentals - Storage, Cookies & Service Workers]] — browser-side storage rules for sessions and tokens
- [[OAuth 2.0 & OIDC - Flows, PKCE & Token Lifecycle]] — refresh rotation, reuse detection, BFF
- [[JWT - Signing Algorithms, Keys & Rotation]] — key compromise = mass revocation
- [[Supabase]] — sessions, SSR cookies, sign-out scopes
- [[Redis]] — denylists and token versions
- [[Flutter]] — secure storage on mobile

## References
- Supabase user sessions: https://supabase.com/docs/guides/auth/sessions
- Supabase signing out: https://supabase.com/docs/guides/auth/signout
- Supabase SSR (cookies): https://supabase.com/docs/guides/auth/server-side
- OWASP Session Management Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html
- MDN Set-Cookie (prefixes, SameSite): https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie
