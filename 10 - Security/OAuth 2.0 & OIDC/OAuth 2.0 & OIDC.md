---
title: OAuth 2.0 & OIDC
aliases: [OAuth, OAuth2, OAuth 2.1, OpenID Connect, OIDC, PKCE]
type: overview
domain: security
tags: [domain/security, type/overview, topic/oauth, topic/oidc, topic/auth]
status: draft
created: 2026-10-02
updated: 2026-10-02
version_checked: "OAuth 2.1 draft-15 (2026-03) · RFC 9700 BCP · OIDC Core 1.0 — 2026-10"
parent: "[[Security MOC]]"
related: ["[[JWT]]", "[[Authentication & Authorization]]", "[[Supabase]]", "[[Model Context Protocol]]", "[[OWASP Top 10]]"]
---

# OAuth 2.0 & OIDC

> [!abstract] TL;DR
> **OAuth 2.0** is *delegated authorization*: a user lets a client app call an API on their behalf without sharing a password, using scoped **access tokens**. **OpenID Connect (OIDC)** is a thin identity layer on top that adds an **`id_token`** ([[JWT]]) plus `/userinfo` so the client learns *who* logged in. In 2026 there's one correct default for almost every client: **Authorization Code + PKCE**, exact redirect URIs, short-lived access tokens, rotating refresh tokens. OAuth 2.1 (still a draft) codifies exactly that.

## Introduction
- **RFC 6749** (Oct 2012) defines the OAuth 2.0 framework. RFC 6750 covers bearer token usage. It replaced OAuth 1.0a's per-request signatures with TLS + bearer tokens.
- **OpenID Connect Core 1.0** (Feb 2014, OpenID Foundation) adds authentication. The OIDC specs were also published as ISO/IEC standards in 2024.
- **RFC 9700** (Jan 2025) is the *OAuth 2.0 Security Best Current Practice*. **OAuth 2.1** (`draft-ietf-oauth-v2-1-15`, Mar 2026) consolidates 6749 + 6750 + PKCE + the BCP and removes implicit and password grants. It isn't an RFC yet, but Okta, Entra ID and Auth0 already implement it.
- Where it sits: "Login with Google/Apple/Microsoft", SaaS-to-SaaS integrations (CRM, Gmail, Meta), API gateways, machine-to-machine auth, and authorization for [[Model Context Protocol|MCP]] servers.

## Core Concepts

### Roles
| Role | Example |
|---|---|
| **Resource Owner** | The end user |
| **Client** | Your Next.js app, Flutter app, n8n credential |
| **Authorization Server (AS)** | Google, Entra ID, Keycloak, Auth0, Supabase Auth, Zitadel |
| **Resource Server (RS)** | The API that accepts access tokens (Gmail API, your backend) |

### Client types
- **Confidential**: can keep a secret (server-side Next.js route, backend worker). Authenticates with `client_secret`, `private_key_jwt` or mTLS.
- **Public**: can't keep a secret (SPA, Flutter/mobile, CLI). It must use **PKCE** and has no secret.

### Tokens
| Token | Format | Audience | Lifetime | Purpose |
|---|---|---|---|---|
| Access token | Opaque or [[JWT]] (RFC 9068 `at+jwt`) | Resource server | 5–60 min | Call APIs. **The client must not parse it** |
| Refresh token | Opaque | AS only | Days–months, rotating | Get new access tokens without the user |
| `id_token` (OIDC) | Always a JWT | **The client** (`aud` = `client_id`) | Minutes | Prove authentication: `sub`, `iss`, `nonce`, `auth_time`, `acr`, `amr` |

### Grants (flows) in 2026
| Grant | Status | Use for |
|---|---|---|
| Authorization Code + **PKCE** | ✅ Default for everything interactive | Web, SPA (via BFF), mobile, desktop, CLI with loopback |
| Client Credentials | ✅ | Machine-to-machine, no user |
| Device Authorization (RFC 8628) | ✅ | TVs, CLIs, IoT without a browser |
| Refresh Token | ✅ | Session continuation. Rotate for public clients |
| Token Exchange (RFC 8693) | ✅ | Service-to-service delegation, impersonation |
| Implicit (`response_type=token`) | ❌ Removed in 2.1 | Token in URL fragment leaks via history/referrer |
| Resource Owner Password | ❌ Removed in 2.1 | Teaches users to type passwords into third-party apps, breaks MFA |

### Scopes vs claims
- **Scopes** = what the client may do (`openid profile email`, `https://www.googleapis.com/auth/gmail.send`). The user consents to them.
- **Claims** = facts about the user in the `id_token`/userinfo. OIDC scopes `profile`/`email`/`address`/`phone` map to claim bundles.
- Fine-grained, transactional authorization (e.g. "pay RM 120 to X") → **RAR** (RFC 9396 `authorization_details`).

## Architecture / How It Works

```mermaid
sequenceDiagram
  participant U as User/Browser
  participant C as Client (BFF / app)
  participant AS as Authorization Server
  participant RS as Resource API
  C->>C: code_verifier = random(43–128); code_challenge = BASE64URL(SHA256(verifier)); state, nonce
  C->>U: 302 → AS /authorize?response_type=code&client_id&redirect_uri&scope=openid…&state&nonce&code_challenge&code_challenge_method=S256
  U->>AS: login + MFA + consent
  AS->>U: 302 → redirect_uri?code=…&state=…&iss=…
  U->>C: code + state
  C->>C: check state (CSRF) and iss (mix-up, RFC 9207)
  C->>AS: POST /token grant_type=authorization_code&code&redirect_uri&code_verifier (+ client auth)
  AS->>AS: SHA256(verifier) == challenge? code unused? redirect_uri exact?
  AS-->>C: access_token, refresh_token, id_token
  C->>C: validate id_token (sig via JWKS, iss, aud=client_id, exp, nonce)
  C->>RS: Authorization: Bearer access_token
```

- **PKCE** (RFC 7636) binds the code to the client instance that started the flow, so a stolen authorization code is useless without the `code_verifier`. Always use `S256`, never `plain`. OAuth 2.1 requires PKCE for **all** clients, confidential included.
- **`state`** protects against CSRF on the redirect. **`nonce`** binds the `id_token` to the session (replay). **`iss` in the response** (RFC 9207) defends against mix-up attacks when you support several IdPs.
- **Discovery**: `/.well-known/openid-configuration` (OIDC) or `/.well-known/oauth-authorization-server` (RFC 8414) lists the endpoints, JWKS URI, supported algorithms and PKCE methods. Configure libraries with the issuer URL, not hand-typed endpoints.
- **Validation split**: the **client** validates the `id_token`. The **resource server** validates the access token (as a JWT via JWKS, or via introspection RFC 7662). Never use the `id_token` as an API credential.

## Project Structure
N/A — a protocol, not a framework. The typical integration footprint:

```text
app/
├── api/auth/[...]          # BFF routes: /login (build authorize URL), /callback (code → tokens), /logout
├── lib/oidc.ts             # discovery + client config (openid-client / Auth.js / supabase-js)
├── middleware.ts           # session cookie check; refresh access token server-side
└── .env                    # OIDC_ISSUER, OIDC_CLIENT_ID, OIDC_CLIENT_SECRET, OIDC_REDIRECT_URI
```
```ts
// Next.js BFF using openid-client v6 (panva)
import * as client from "openid-client";

const config = await client.discovery(new URL(process.env.OIDC_ISSUER!), process.env.OIDC_CLIENT_ID!, process.env.OIDC_CLIENT_SECRET!);

export async function loginUrl(session: Session) {
  session.verifier = client.randomPKCECodeVerifier();
  session.state = client.randomState();
  session.nonce = client.randomNonce();
  return client.buildAuthorizationUrl(config, {
    redirect_uri: process.env.OIDC_REDIRECT_URI!, scope: "openid email profile offline_access",
    code_challenge: await client.calculatePKCECodeChallenge(session.verifier), code_challenge_method: "S256",
    state: session.state, nonce: session.nonce,
  });
}

export async function callback(currentUrl: URL, session: Session) {
  const tokens = await client.authorizationCodeGrant(config, currentUrl, {
    pkceCodeVerifier: session.verifier, expectedState: session.state, expectedNonce: session.nonce,
  });
  return { claims: tokens.claims(), tokens };   // store tokens server-side; set HttpOnly session cookie
}
```

## Use Cases
| Use case | Why it fits |
|---|---|
| Social / enterprise SSO ("Login with Google / Microsoft") | OIDC: standardized identity + MFA handled by the IdP |
| SaaS integrations (Gmail, Google Sheets, HubSpot, Meta Graph) | User-delegated, scoped, revocable access without passwords |
| Machine-to-machine APIs | Client Credentials, short-lived tokens with `aud` per API |
| B2B multi-tenant SSO | Per-tenant OIDC/SAML connections behind one AS (Keycloak, Zitadel, WorkOS) |
| MCP server authorization | MCP spec mandates OAuth 2.1 + Protected Resource Metadata (RFC 9728) |
| CLI / TV login | Device flow: user approves on a phone |
| CI/CD → cloud without static keys | GitHub Actions OIDC token → AWS/GCP workload identity federation |

## Pros & Cons
| Pros | Cons |
|---|---|
| Users never hand passwords to third-party apps | Huge spec surface (30+ RFCs and drafts). Easy to implement a subtly broken subset |
| Scoped, revocable, short-lived credentials | Misconfiguration = account takeover (open redirect, missing PKCE/state) |
| IdP centralizes MFA, passkeys, lockout, audit | IdP is a single point of failure and lock-in (Auth0/Okta pricing) |
| Ecosystem: certified libraries, discovery, JWKS | Refresh tokens held by integrations become a high-value breach target |
| OIDC gives standardized identity (`sub` + `iss`) | Consent screens and app verification (Google, Meta) add weeks of review |

## Alternatives & Peers
| Alternative | Strength vs OAuth 2.0 & OIDC | Weakness vs OAuth 2.0 & OIDC | Pick it when… |
|---|---|---|---|
| SAML 2.0 | Entrenched in enterprise/government SSO | XML signature wrapping bugs, no API delegation, poor mobile | The customer's IdP only speaks SAML |
| First-party session auth (cookie + DB) | Simplest, instant revocation | No delegation, no SSO across apps | Your own single app with its own users |
| API keys | Trivial for server-to-server | Long-lived, no user context, no scoping standard | Internal or low-risk M2M with rotation |
| Passkeys / WebAuthn | Phishing-resistant authentication | Authn only. Pairs with OIDC rather than replacing it | Primary login factor at your IdP |
| mTLS / SPIFFE | Strong workload identity, no bearer tokens | PKI operational overhead | Service mesh, zero-trust internals |

## Tips & Reminders
> [!tip] Non-negotiables
> - Auth Code + PKCE (`S256`) for every interactive client. Validate `state`, `nonce` and `iss`.
> - **Exact-match** redirect URIs. No wildcards, no open redirectors on the redirect host.
> - Identify users by **`iss` + `sub`**, never by `email` (emails are mutable and sometimes unverified, as in the 2023 "nOAuth" Entra ID takeover).
> - Request minimal scopes and add more incrementally.
> - Rotate refresh tokens with reuse detection. Store them encrypted, server-side only.

> [!tip] Browser apps
> Use a **BFF**: the server does the OAuth dance and holds tokens, and the browser gets an `HttpOnly` session cookie. Tokens in `localStorage` are one XSS away from exfiltration. This is the IETF "OAuth for Browser-Based Apps" recommendation.

> [!tip] In ZP's stack
> - **[[Supabase]] Auth** is an OIDC-style AS for your own users (Google/Apple/Azure providers). It uses PKCE by default in `@supabase/ssr`. For server-side Next.js, use the `exchangeCodeForSession` callback route.
> - **[[Flutter]]**: use the platform browser (ASWebAuthenticationSession / Custom Tabs) via `flutter_appauth` or Supabase's deep-link flow, never an embedded WebView (Google blocks it with `disallowed_useragent`).
> - **[[n8n]]** OAuth2 credentials store refresh tokens in its DB, encrypted with `N8N_ENCRYPTION_KEY`. Back the key up separately, and treat the n8n DB as holding live client SaaS tokens.
> - **Meta / WhatsApp Cloud API**: system-user tokens are OAuth access tokens with business scopes. Prefer system users over personal long-lived tokens, and rotate them.

## Versions & Breaking Changes
| Version | Released | Key changes | Breaking / migration notes |
|---|---|---|---|
| RFC 6749 / 6750 (OAuth 2.0) | 2012-10 | Framework + bearer tokens | Replaced OAuth 1.0a signatures |
| OIDC Core 1.0 | 2014-02 | `id_token`, discovery, userinfo | Errata set 2 in 2023 |
| RFC 7636 (PKCE) | 2015-09 | Code challenge/verifier | Now mandatory in 2.1 |
| RFC 8252 (native apps) | 2017-10 | System browser, loopback/claimed-HTTPS redirects | Embedded WebViews discouraged |
| RFC 8628 (device grant) | 2019-08 | Input-constrained devices | — |
| RFC 9126 PAR / 9101 JAR | 2021-09 / 2021-08 | Pushed + signed authorization requests | Required by FAPI 2.0 |
| RFC 9207 (`iss` response param) | 2022-03 | Mix-up attack defense | Clients should validate `iss` |
| RFC 9449 (DPoP) | 2023-09 | Sender-constrained tokens | Opt-in |
| **RFC 9700 (Security BCP)** | 2025-01 | Deprecates implicit + password, PKCE for all, refresh rotation | Treat it as the baseline audit checklist |
| RFC 9728 (Protected Resource Metadata) | 2025-04 | RS advertises its AS, used by MCP | — |
| **OAuth 2.1** (`-15` draft) | 2026-03 | Consolidated spec: PKCE everywhere, exact redirects, no implicit/password, no bearer tokens in query strings | Not yet an RFC. Implement now regardless |

## Critical Issues & Gotchas
> [!danger] Integration OAuth tokens are the new breach path
> - **Salesloft Drift (Aug 2025)**: attackers stole Drift's stored OAuth tokens and bulk-exported Salesforce data from hundreds of tenants.
> - **Vercel via Context.ai (Apr 2026)**: an infostealer on a third-party employee's machine leaked OAuth/session tokens and led to platform access.
> - **Klue (Jun 2026)**: an intruder harvested customer CRM OAuth tokens, and every customer credential was revoked.
>
> **Mitigation**: least-privilege scopes, encrypt refresh tokens at rest, alert on anomalous API volume per token, keep a token-revocation runbook per integration, and audit third-party OAuth apps granted in your Google Workspace / M365.

> [!danger] OIDC signature not validated
> **CVE-2026-48558** (SimpleHelp, in CISA KEV since Jun 2026) accepted `id_token`s **without verifying the signature**, so anyone could forge an admin login. **CVE-2026-65583** (Apache CXF) skipped `sub`/`aud`/`exp` checks on self-issued tokens. Use a certified library and never hand-roll `id_token` validation.

> [!danger] Open redirect + loose `redirect_uri` = token theft
> Prefix or wildcard redirect matching, combined with any open redirect on the client domain, lets an attacker receive the authorization code. PKCE limits the damage, implicit flow does not. Use exact string matching.

> [!danger] Account takeover via mutable claims ("nOAuth", 2023)
> Apps keyed users by the `email` claim from multi-tenant Entra ID, where tenants could set arbitrary unverified emails. Key by `iss`+`sub`. Link accounts only on `email_verified: true` **and** a trusted issuer.

> [!warning] Platform / vendor risks
> - Google OAuth apps requesting **restricted scopes** (Gmail, Drive) need verification plus an annual third-party security assessment (CASA), which costs real money and weeks. Design around sensitive scopes where you can.
> - "Testing" mode Google OAuth apps issue refresh tokens that **expire after 7 days**, a classic "n8n integration broke after a week" cause.
> - Meta app review/business verification gates WhatsApp/Instagram scopes. Plan lead time.
> - Auth0/Okta MAU pricing climbs steeply for B2C. Self-hosted Keycloak/Zitadel/Authentik on [[Coolify]] are viable, but you then own patching and HA.

## Deep Dives
- (planned, unscheduled) OAuth 2.0 & OIDC - Flows, PKCE & Token Lifecycle
- (planned, unscheduled) OAuth 2.0 & OIDC - Attacks & Hardening

## Related
- [[JWT]] — `id_token` format and access-token validation rules
- [[Authentication & Authorization]]
- [[Supabase]] — Auth providers, PKCE in `@supabase/ssr`
- [[Model Context Protocol]] — OAuth 2.1-based authorization for remote MCP servers
- [[OWASP Top 10]] — A01 Broken Access Control, A07 Identification & Authentication Failures
- [[HTTP & HTTPS]] — redirects, cookies, TLS

## References
- OAuth 2.1 draft: https://datatracker.ietf.org/doc/html/draft-ietf-oauth-v2-1-15
- RFC 9700 (Security BCP): https://www.rfc-editor.org/rfc/rfc9700
- RFC 7636 (PKCE): https://www.rfc-editor.org/rfc/rfc7636
- OpenID Connect Core: https://openid.net/specs/openid-connect-core-1_0.html
- Spec index: https://oauth.net/specs/
- openid-client (Node): https://github.com/panva/openid-client
- Vercel / Context.ai breach analysis: https://trendmicro.com/en_us/research/26/d/vercel-breach-oauth-supply-chain.html
- CVE-2026-48558 SimpleHelp: https://socradar.io/blog/cve-2026-48558-simplehelp-oidc-infostealer/
