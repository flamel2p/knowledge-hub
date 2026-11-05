---
title: OAuth 2.0 & OIDC - Social Login with Google, Apple & Meta
aliases: [social login, Sign in with Apple, Google Sign-In, Credential Manager, Facebook Login, Limited Login, signInWithIdToken, ID token verification, nonce, account linking, App Store guideline 4.8]
type: deep-dive
domain: security
tags: [domain/security, type/deep-dive, topic/oauth, topic/oidc, topic/social-login, lang/dart]
status: draft
created: 2026-11-05
updated: 2026-11-05
version_checked: "Android: legacy Google Sign-In removed from Play Services Auth (2025) → Credential Manager · Apple guideline 4.8 (privacy-focused equivalent login) · Supabase signInWithIdToken — 2026-11"
parent: "[[OAuth 2.0 & OIDC]]"
related: ["[[OAuth 2.0 & OIDC - Flows, PKCE & Token Lifecycle]]", "[[OAuth 2.0 & OIDC - Attacks & Hardening]]", "[[Supabase - Auth Flows, MFA & Custom Claims]]", "[[Flutter - App Security, Obfuscation & Secure Storage]]", "[[Flutter - Build, Release & Store Compliance]]", "[[JWT - Signing Algorithms, Keys & Rotation]]"]
---

# OAuth 2.0 & OIDC - Social Login with Google, Apple & Meta

> [!info] Deep dive of [[OAuth 2.0 & OIDC]]

> [!abstract] TL;DR
> - **Mobile**: use native SDKs to get an **OIDC ID token** (Google via Credential Manager on Android, Sign in with Apple, Facebook Limited Login on iOS), then exchange it with your auth server: `supabase.auth.signInWithIdToken` with a **nonce**.
> - **Web**: use the redirect flow (authorization code + PKCE).
> - **Verify everything server-side**: signature (JWKS), `iss`, `aud`, `exp`, nonce and `email_verified`.
> - **Store rules**: an iOS app offering Google/Facebook login must offer a privacy-focused equivalent (Sign in with Apple qualifies), and in-app account deletion must also revoke Apple tokens.
> - **Linking**: link identities by email only when the email is verified, or you invite account takeover.

## Concept
| Provider | Mobile mechanism | Web | Tokens you get | Quirks |
|---|---|---|---|---|
| Google | Android: **Credential Manager** (legacy Google Sign-In removed). iOS: Google Sign-In SDK | GIS button / OAuth redirect | ID token (`aud` = your **web** client ID when using `serverClientId`) | Separate client IDs per platform. SHA-1/SHA-256 fingerprints for Android |
| Apple | `ASAuthorizationAppleIDProvider` (`sign_in_with_apple`) | Sign in with Apple JS / redirect (Services ID) | ID token, authorization code | Name only on **first** sign-in. Private relay emails. Client secret JWT expires ≤ 6 months |
| Meta (Facebook) | Facebook SDK. iOS **Limited Login** returns an OIDC token when tracking isn't allowed | OAuth redirect | Access token (classic) or OIDC token (limited) | App Review for extra permissions. Email may be missing or unverified |

**Store/platform obligations**:

| Rule | What it means |
|---|---|
| App Store guideline 4.8 | If you offer third-party/social login, also offer an option that limits data to name + email, allows private email, and doesn't track. Sign in with Apple satisfies it |
| App Store 5.1.1(v) | Apps with account creation must offer in-app account deletion, and Sign in with Apple users' tokens should be revoked via Apple's REST API |
| Google Play / Android | Use Credential Manager. Declare data collection in the Data safety form |
| Meta Platform Terms | App Review + business verification for permissions beyond `public_profile`/`email`. Data deletion callback/URL |

## How It Works
```mermaid
sequenceDiagram
  participant App as Flutter app
  participant P as Provider SDK (Apple/Google/Meta)
  participant SB as Supabase Auth (or your backend)
  participant J as Provider JWKS
  App->>App: rawNonce = random(32); hashed = sha256(rawNonce)
  App->>P: sign in (nonce = hashed)
  P-->>App: ID token (nonce claim = hashed)
  App->>SB: signInWithIdToken(provider, idToken, nonce = rawNonce)
  SB->>J: verify signature, iss, aud, exp; sha256(rawNonce) == token.nonce
  SB->>SB: find/create user, link identity if verified email matches
  SB-->>App: session (access + refresh token)
```

## Practical Usage
**Sign in with Apple → Supabase** (Flutter):
```dart
final rawNonce = supabase.auth.generateRawNonce();
final hashedNonce = sha256.convert(utf8.encode(rawNonce)).toString();

final cred = await SignInWithApple.getAppleIDCredential(
  scopes: [AppleIDAuthorizationScopes.email, AppleIDAuthorizationScopes.fullName],
  nonce: hashedNonce,
);
final res = await supabase.auth.signInWithIdToken(
  provider: OAuthProvider.apple, idToken: cred.identityToken!, nonce: rawNonce);

// Apple returns the name only on the first authorization: persist it now
if (cred.givenName != null) {
  await supabase.from('profiles').update({'display_name': '${cred.givenName} ${cred.familyName ?? ''}'.trim()})
      .eq('id', res.user!.id);
}
```
**Google → Supabase** (Flutter; `serverClientId` = the web client ID Supabase is configured with):
```dart
// google_sign_in 7.x API (uses Credential Manager on Android); check the package README for your version
await GoogleSignIn.instance.initialize(serverClientId: webClientId, clientId: iosClientId);
final account = await GoogleSignIn.instance.authenticate();
final idToken = account.authentication.idToken!;
await supabase.auth.signInWithIdToken(provider: OAuthProvider.google, idToken: idToken);
```
**Web redirect flow** (Next.js; PKCE handled by Supabase):
```ts
await supabase.auth.signInWithOAuth({
  provider: 'google',
  options: { redirectTo: `${origin}/auth/callback`, queryParams: { prompt: 'select_account' } },
});
// /auth/callback route: exchangeCodeForSession(code), then redirect
```
**Verifying an ID token in your own backend** (no Supabase):
```ts
import { createRemoteJWKSet, jwtVerify } from 'jose';
const appleJwks = createRemoteJWKSet(new URL('https://appleid.apple.com/auth/keys'));

const { payload } = await jwtVerify(idToken, appleJwks, {
  issuer: 'https://appleid.apple.com',
  audience: 'my.rarticle.app',                          // bundle ID (native) or Services ID (web)
});
if (payload.nonce !== sha256Hex(rawNonce)) throw new Error('nonce mismatch');
const email = payload.email_verified ? String(payload.email) : null;   // never link on unverified email
```
**Wrong way**:
```text
❌ Trusting a decoded (unverified) ID token from the client      → anyone forges {"email":"boss@client.my"}
❌ Accepting tokens for any audience                            → tokens minted for another app log in to yours
❌ Auto-linking accounts on unverified provider email           → attacker creates a Facebook account with victim's email → takeover
❌ Asking Apple users for their email again after private relay → breaks the privacy promise; App Review rejection risk
```

## Patterns & Anti-patterns
| Pattern | When | Anti-pattern to avoid |
|---|---|---|
| Native SDK → ID token → server exchange with nonce | Mobile apps | Embedded WebView OAuth (blocked by Google, poor UX, phishing-like) |
| Code + PKCE redirect flow | Web | Implicit flow |
| Verify `iss`, `aud`, `exp`, signature, nonce | Every ID token | Decoding without verification |
| Link identities only on verified emails, or require re-auth to link | Multi-provider accounts | Silent merge on any matching email |
| Minimal scopes (`openid email profile`) | Login | Requesting contacts/friends "for later" |
| Account deletion flow that revokes provider tokens | iOS apps, PDPA | Soft-deleting rows but keeping Apple tokens active |
| Keep an email/WhatsApp OTP fallback | Provider outages, lost accounts | Social-only login for SME staff apps |

## Performance & Trade-offs
- **Conversion**: one-tap Google/Apple login measurably beats email signups on mobile. Malaysian consumer apps commonly offer Google + Apple + phone/WhatsApp OTP, with Facebook less critical than it once was.
- **Provider dependency**: each provider adds config (client IDs, secrets, fingerprints, redirect URIs) per environment and flavour. More providers mean more breakage points during key/secret rotations.
- **Apple client secret** (JWT signed with your key) must be regenerated before it expires (≤ 6 months). Calendar it, or automate it.
- **Data quality**: Apple private relay emails and missing Facebook emails mean email isn't a reliable identifier. Key users by provider subject (`sub`) + internal user ID.

## Tips & Reminders
> [!tip] In ZP's stack
> - **Flutter consumer apps**: Apple + Google via `signInWithIdToken`, plus WhatsApp OTP for users without either. Staff/admin apps use email OTP or Google Workspace SSO ([[Supabase - Auth Flows, MFA & Custom Claims]]).
> - **Config per flavour** (dev/staging/prod): separate Google client IDs, Apple Services IDs and redirect URLs, documented in the client handover file ([[Flutter - Build, Release & Store Compliance]]).
> - **Client ownership**: Google Cloud project, Apple Developer account and Meta app should belong to the client. App Review and business verification are tied to that entity.

- If you send email to Apple private relay addresses, register your sending domains in Apple's developer portal with SPF/DKIM, or mail bounces ([[Cloudflare - DNS, Email Authentication & Domain Ops]]).
- Store refresh/session tokens securely on device ([[Flutter - App Security, Obfuscation & Secure Storage]]).
- Show which providers are linked in the profile screen, and allow unlinking as long as one sign-in method remains.
- Test the "Hide My Email" path, first-login name capture, and account deletion + re-signup on real devices before App Review.

## Version Notes
| Date | Change |
|---|---|
| 2019–20 | Sign in with Apple required when offering third-party login on iOS (original 4.8) |
| 2022-06 | App Store requires in-app account deletion (5.1.1(v)) |
| 2024-01 | Guideline 4.8 relaxed to "equivalent privacy-focused login" (Sign in with Apple one option) |
| 2024 | Facebook Limited Login (iOS) returns OIDC tokens. Classic tokens only with tracking permission |
| 2025 | Legacy Google Sign-In for Android removed from Play Services Auth. Credential Manager + AuthorizationClient |

> [!warning] Unverified — check before relying on this
> Current wording of App Store guideline 4.8, the `google_sign_in` 7.x API surface, and Meta's 2026 App Review requirements weren't verified this run. Check Apple's App Review Guidelines, pub.dev, and Meta's developer docs.

## Critical Issues & Gotchas
> [!danger] Account takeover via unverified email linking
> If your backend links a new social identity to an existing account because the email matches, an attacker can register a provider account (or a misconfigured tenant in a multi-tenant IdP) with the victim's email and log in as them. Link only on `email_verified=true` from trusted providers, require the user to authenticate to the existing account before linking, and audit links ([[OAuth 2.0 & OIDC - Attacks & Hardening]]).

> [!danger] Audience confusion and replay
> Accepting ID tokens without checking `aud` lets tokens issued to another app (including a malicious one using the same provider) authenticate to yours. Missing nonce checks allow replay of intercepted tokens. Always validate `aud` against your exact client IDs (per platform) and use nonces for native flows.

- **Android SHA fingerprints**: Play App Signing uses Google's signing key, so register the Play signing SHA-1/256, not only your upload key, or Google sign-in fails in production builds only.
- **Apple Services ID vs bundle ID**: native tokens have `aud` = bundle ID, and web tokens `aud` = Services ID. Configure both in your auth server.
- **Name data loss**: Apple sends name fields only once. If your first sign-in request fails after the credential call, that data is gone.
- **Provider outages / account bans**: a user who loses their Google/Apple account loses access. Offer recovery via a verified email or phone.

## Related
- [[OAuth 2.0 & OIDC]]
- [[OAuth 2.0 & OIDC - Flows, PKCE & Token Lifecycle]] — code + PKCE, token lifetimes
- [[OAuth 2.0 & OIDC - Attacks & Hardening]] — takeover, mix-up, redirect attacks
- [[Supabase - Auth Flows, MFA & Custom Claims]] — `signInWithIdToken`, identity linking
- [[Flutter - App Security, Obfuscation & Secure Storage]] — session storage on device
- [[JWT - Signing Algorithms, Keys & Rotation]] — JWKS verification and key rotation

## References
- https://developer.apple.com/app-store/review/guidelines/#login-services
- https://developer.apple.com/documentation/sign_in_with_apple/revoke_tokens
- https://developer.android.com/identity/sign-in/legacy-gsi-migration
- https://supabase.com/docs/guides/auth/social-login/auth-apple
- https://developers.facebook.com/docs/facebook-login/limited-login
