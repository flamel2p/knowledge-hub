---
title: OAuth 2.0 & OIDC - Flows, PKCE & Token Lifecycle
aliases: [OAuth flows, PKCE, authorization code flow, refresh token rotation, BFF pattern, client credentials, device flow, token exchange]
type: deep-dive
domain: security
tags: [domain/security, type/deep-dive, topic/oauth, topic/oidc, topic/pkce, topic/tokens]
status: draft
created: 2026-10-18
updated: 2026-10-18
version_checked: "RFC 9700 BCP · RFC 10017 browser apps BCP (2026-08) · OAuth 2.1 draft — 2026-10"
parent: "[[OAuth 2.0 & OIDC]]"
related: ["[[JWT]]", "[[JWT - Signing Algorithms, Keys & Rotation]]", "[[Authentication & Authorization]]", "[[Next.js]]", "[[Flutter]]", "[[Model Context Protocol - Building MCP Servers]]"]
---

# OAuth 2.0 & OIDC - Flows, PKCE & Token Lifecycle

> [!info] Deep dive of [[OAuth 2.0 & OIDC]]

> [!abstract] TL;DR
> - **Users**: always use **Authorization Code + PKCE (S256)**.
>   - Web apps: run it server-side in a **BFF** (tokens never reach browser JS).
>   - Mobile/desktop: use the system browser with claimed HTTPS redirects.
> - **Machines**: client credentials, with **token exchange** for downstream calls.
> - **Access tokens**: short (5–15 min).
> - **Refresh tokens**: **rotate on every use, with reuse detection**, stored server-side or in secure OS storage.
> - **Logout**: means revoking refresh tokens, not just deleting a cookie.

## Concept

### Which flow for which client
| Client | Flow | Token storage | Redirect URI |
|---|---|---|---|
| Server-rendered web / Next.js | Auth code + PKCE (confidential, client secret or `private_key_jwt`) | Server session (encrypted cookie → session store) | `https://app.example.my/auth/callback` (exact) |
| SPA (React/Vite) | **BFF**: same as above via a backend-for-frontend | HttpOnly `__Host-` cookie only. No tokens in JS | BFF callback |
| Mobile / Flutter | Auth code + PKCE (public) via system browser (AppAuth) | Keychain / Android Keystore (secure storage) | Claimed HTTPS (Universal/App Links) |
| CLI / desktop | Auth code + PKCE with loopback `http://127.0.0.1:<port>`, or **device flow** | OS keychain | Loopback (any port allowed by RFC 8252) |
| TV / IoT / no browser | Device authorization grant (RFC 8628) | Device secure storage | — |
| Service → service | Client credentials (`private_key_jwt` or mTLS auth) | Memory cache until `exp` | — |
| API calling another API on behalf of a user | **Token exchange** (RFC 8693) | Memory | — |

### PKCE, `state` and `nonce`: different jobs
| Value | Created by | Checked by | Protects against |
|---|---|---|---|
| `code_verifier` → `code_challenge = BASE64URL(SHA256(verifier))` | Client per attempt (43–128 chars, high entropy) | AS at token endpoint | Stolen/injected authorization codes |
| `state` | Client per attempt | Client on callback | CSRF on the redirect endpoint (also carries return URL) |
| `nonce` | Client per attempt (OIDC) | Client in `id_token` | `id_token` replay/injection |
| `iss` response param (RFC 9207) | AS | Client | Mix-up attacks with multiple IdPs |

### Token lifetimes (sane defaults)
| Token | Lifetime | Notes |
|---|---|---|
| Authorization code | ≤ 60 s, single use | AS must reject reuse and revoke tokens issued from a replayed code |
| Access token | 5–15 min | JWT (`at+jwt`) or opaque + introspection |
| Refresh token | Idle 7–30 days, absolute 30–90 days | **Rotating**. Bound to client (+ DPoP/mTLS where possible) |
| `id_token` | Minutes | For the client only. Never send it to APIs |
| App session (BFF cookie) | Idle 30 min–8 h, absolute ≤ 30 days | Independent of IdP session |

## How It Works
```mermaid
sequenceDiagram
  participant B as Browser
  participant BFF as BFF (Next.js server)
  participant AS as Authorization Server
  participant API as Resource API
  B->>BFF: GET /login
  BFF->>BFF: create state, nonce, code_verifier (store in temp cookie/session)
  BFF-->>B: 302 /authorize?response_type=code&client_id&redirect_uri&scope=openid profile orders:read&state&nonce&code_challenge&code_challenge_method=S256
  B->>AS: authenticate (passkey / SSO)
  AS-->>B: 302 redirect_uri?code&state&iss
  B->>BFF: GET /auth/callback?code&state&iss
  BFF->>AS: POST /token (code, code_verifier, client auth)
  AS-->>BFF: access_token (10 min), refresh_token (rotating), id_token
  BFF->>BFF: validate id_token (sig, iss, aud, nonce, exp) → session; tokens server-side
  BFF-->>B: Set-Cookie __Host-sid (HttpOnly, Secure, SameSite=Lax)
  B->>BFF: GET /api/orders (cookie)
  BFF->>API: Authorization: Bearer access_token
  Note over BFF,AS: on expiry: POST /token grant_type=refresh_token → NEW refresh token, old one invalidated
```

## Practical Usage

### Token endpoint calls (raw HTTP, for debugging)
```bash
# Code exchange (confidential client)
curl -s https://auth.example.my/oauth/token -u "$CLIENT_ID:$CLIENT_SECRET" \
  -d grant_type=authorization_code -d code="$CODE" -d redirect_uri=https://app.example.my/auth/callback \
  -d code_verifier="$VERIFIER"

# Refresh (rotation: response contains a NEW refresh_token; store it atomically)
curl -s https://auth.example.my/oauth/token -u "$CLIENT_ID:$CLIENT_SECRET" \
  -d grant_type=refresh_token -d refresh_token="$RT"

# Client credentials with a narrow scope + audience
curl -s https://auth.example.my/oauth/token -u "$SVC_ID:$SVC_SECRET" \
  -d grant_type=client_credentials -d scope="invoices:write" -d resource=https://billing.example.my

# Revoke on logout (RFC 7009)
curl -s https://auth.example.my/oauth/revoke -u "$CLIENT_ID:$CLIENT_SECRET" -d token="$RT" -d token_type_hint=refresh_token
```

### Refresh-token rotation with reuse detection (if you run the AS)
```sql
create table refresh_tokens (
  id          uuid primary key,
  family_id   uuid not null,              -- one family per login
  user_id     uuid not null,
  client_id   text not null,
  token_hash  bytea not null unique,      -- store SHA-256, never the raw token
  expires_at  timestamptz not null,
  used_at     timestamptz,
  revoked_at  timestamptz
);
```
```ts
async function rotate(raw: string) {
  return db.tx(async (tx) => {
    const rt = await tx.one("select * from refresh_tokens where token_hash = $1 for update", [sha256(raw)]);
    if (rt.revoked_at || rt.expires_at < new Date()) throw invalidGrant();
    if (rt.used_at) {                                          // REUSE → likely theft: kill the whole family
      await tx.none("update refresh_tokens set revoked_at = now() where family_id = $1", [rt.family_id]);
      alertSecurity("refresh token reuse", rt);
      throw invalidGrant();
    }
    await tx.none("update refresh_tokens set used_at = now() where id = $1", [rt.id]);
    const next = randomToken();
    await tx.none("insert into refresh_tokens (id, family_id, user_id, client_id, token_hash, expires_at) values ($1,$2,$3,$4,$5,$6)",
      [uuid(), rt.family_id, rt.user_id, rt.client_id, sha256(next), absoluteExpiry(rt)]);
    return { refreshToken: next, accessToken: await issueAccessToken(rt.user_id) };
  });
}
```
- Allow a short grace window (e.g. 10–30 s) for concurrent refreshes from the same client, or serialise refreshes client-side. Otherwise legitimate tab/race conditions trigger family revocation.

### Mobile (Flutter) with AppAuth
```dart
final result = await const FlutterAppAuth().authorizeAndExchangeCode(
  AuthorizationTokenRequest(
    'shop-mobile', 'https://app.example.my/oauth/callback',      // claimed HTTPS (App Links / Universal Links)
    issuer: 'https://auth.example.my',
    scopes: ['openid', 'profile', 'offline_access', 'orders:read'],
  ),
);   // PKCE S256 + state handled by AppAuth
await secureStorage.write(key: 'rt', value: result.refreshToken);  // flutter_secure_storage → Keychain/Keystore
```
- Avoid custom schemes (`myapp://callback`), since another app can register the same scheme. Use claimed HTTPS redirects, and never embedded WebViews for login (RFC 8252).

### Token exchange for downstream calls (RFC 8693)
```bash
# API A received the user's token (aud=api-a); it needs to call API B as that user with fewer rights
curl -s https://auth.example.my/oauth/token -u "$API_A_ID:$API_A_SECRET" \
  -d grant_type=urn:ietf:params:oauth:grant-type:token-exchange \
  -d subject_token="$USER_ACCESS_TOKEN" -d subject_token_type=urn:ietf:params:oauth:token-type:access_token \
  -d audience=https://api-b.example.my -d scope=orders:read
```
- This is the correct replacement for "forward the user's token" (token passthrough), which MCP and RFC 9700 forbid ([[Model Context Protocol - Building MCP Servers]]).

## Patterns & Anti-patterns
| Pattern | When | Anti-pattern to avoid |
|---|---|---|
| BFF with HttpOnly session cookie | SPAs, Next.js | Access/refresh tokens in `localStorage` (one XSS = account takeover) |
| PKCE S256 for every client, incl. confidential | Always | `plain` method, or PKCE "only for mobile" |
| Exact redirect URI registration | Always | Wildcards/prefix matching, `localhost` URIs left in production |
| Refresh rotation + reuse detection + absolute lifetime | Long sessions | Non-expiring, non-rotating refresh tokens |
| Narrow scopes + `resource`/`audience` per API | Multi-API platforms | One "god" token accepted by every service |
| `private_key_jwt` / mTLS client auth | Confidential clients | Client secrets in mobile apps or front-end bundles |
| DPoP-bound tokens for public clients (where AS supports it) | High-value mobile/desktop | Bearer refresh tokens copied off a device and reused anywhere |
| RP-initiated + back-channel logout | SSO across apps | "Logout" that only deletes the local cookie |

## Performance & Trade-offs
- **JWT access tokens vs opaque + introspection**:
  - **JWT**: local verification, so no AS round-trip, but revocation lags by the TTL.
  - **Introspection** (RFC 7662): instant revocation, but adds a network call per request. Cache results for ≤ 30 s.
  - **Pick**: short JWTs for most APIs, introspection for high-risk operations.
- **Short access TTL** means more refresh traffic: a 10-min TTL with 1,000 active users is ~100 refreshes/min. That's trivial for an AS, but make the refresh path idempotent and race-safe.
- **BFF vs tokens in the browser**: a BFF adds a server hop and session store ([[Redis]]/Postgres), but removes token exfiltration via XSS. RFC 10017 (browser-based apps BCP) recommends the BFF as the most secure architecture.
- **Device flow** is phishable ("enter this code"), so use it only when no browser exists. Keep short code lifetimes and show the client name prominently.

## Tips & Reminders
> [!tip]
> - **Configure clients from discovery** (`/.well-known/openid-configuration`) and validate `iss` on callbacks when you support multiple IdPs.
> - **Request `offline_access`** (OIDC) only when you really need refresh tokens. Some IdPs (Google) also need `prompt=consent` + `access_type=offline` to return one.
> - **Store refresh tokens encrypted at rest** (envelope encryption) for integrations. They're the crown jewels in Drift/Klue-style breaches ([[OAuth 2.0 & OIDC]]).
> - **Log** grant type, client, scopes, `sub` and outcome for every token issuance. Alert on reuse detection and on refresh volume spikes per client.
> - **In ZP's stack**:
>   - **Next.js apps**: BFF pattern with server-side sessions, using Supabase Auth or Keycloak as the IdP.
>   - **Flutter apps**: AppAuth with App Links, plus secure storage.
>   - **[[n8n]] integrations**: use client credentials or a dedicated service account with narrow scopes, never a staff member's personal OAuth grant.
>   - **MCP servers**: validate audience-bound tokens, and use token exchange for downstream APIs.

## Version Notes
| Spec | Status | Relevance |
|---|---|---|
| RFC 7636 PKCE | 2015 | Mandatory for all clients in 2.1/RFC 9700 |
| RFC 8252 Native apps BCP | 2017 | System browser, claimed HTTPS, loopback |
| RFC 8628 Device grant | 2019 | CLIs/TVs |
| RFC 8693 Token exchange | 2020 | Downstream delegation |
| RFC 9126 PAR | 2021 | Pushes authorization params server-side (FAPI 2.0, recommended for confidential clients) |
| RFC 9449 DPoP | 2023 | Sender-constrained tokens |
| RFC 9700 Security BCP | 2025-01 | Baseline checklist |
| **RFC 10017** (BCP 212) Browser-based apps | 2026-08 | BFF > token-mediating backend > browser-only. Formalises SPA guidance |
| OAuth 2.1 | Draft (`-16`, 2026-09) | Consolidation. Implement its rules now |

> [!warning] Unverified — check before relying on this
> RFC 10017's number and date come from the IETF mailing list and RFC Editor listing found via search. Confirm at rfc-editor.org. Supabase Auth's support for acting as an OAuth 2.1 authorization server (for third-party/MCP clients) wasn't verified.

## Critical Issues & Gotchas
> [!danger] Refresh-token theft without rotation = silent long-term access
> A refresh token exfiltrated by an infostealer, a log line or a compromised integration vendor keeps working for months if it doesn't rotate or expire. Rotation with reuse detection turns theft into a detectable event, and sender constraint (DPoP/mTLS) makes it useless elsewhere. Keep an absolute lifetime and a per-user "revoke all sessions" button.

> [!danger] Authorization code injection and PKCE downgrade
> If the AS accepts token requests without `code_verifier` for a code issued with a challenge, or the client accepts any `state`, attackers can inject stolen codes into a victim session. The AS must **enforce** PKCE when a challenge was sent (and by policy for all clients), and clients must bind `state`/`verifier` to the browser session that started the flow.

> [!warning] Gotchas
> - Concurrent refreshes (multiple tabs, mobile background fetch) with strict rotation log users out. Serialise refreshes, or allow a tiny grace window.
> - Clock skew makes fresh tokens "not yet valid" (`iat`/`nbf`). Allow 30–60 s leeway.
> - `SameSite=Strict` session cookies break the redirect back from the IdP. Use `Lax` for the session cookie, and `None; Secure` only if cross-site embedding is genuinely needed.
> - Google "Testing" mode refresh tokens expire after 7 days. Publish the consent screen for production integrations.
> - The `id_token` is not an API credential. APIs must accept only access tokens with their own `aud` ([[JWT]]).

## Related
- [[OAuth 2.0 & OIDC]]
- [[JWT]] — access-token format and validation
- [[JWT - Signing Algorithms, Keys & Rotation]] — AS signing keys and JWKS
- [[Authentication & Authorization]] — sessions, passkeys at the IdP
- [[Next.js]] — BFF implementation
- [[Flutter]] — AppAuth, secure storage, App Links

## References
- RFC 9700 OAuth 2.0 Security BCP: https://www.rfc-editor.org/rfc/rfc9700
- RFC 10017 OAuth 2.0 for Browser-Based Applications: https://www.rfc-editor.org/rfc/rfc10017.html
- OAuth 2.1 draft: https://datatracker.ietf.org/doc/draft-ietf-oauth-v2-1/
- RFC 7636 PKCE: https://www.rfc-editor.org/rfc/rfc7636
- RFC 8693 Token Exchange: https://www.rfc-editor.org/rfc/rfc8693
- RFC 8252 OAuth for Native Apps: https://www.rfc-editor.org/rfc/rfc8252
