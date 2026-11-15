---
title: Flutter - Supabase Integration, Auth & Realtime
aliases: [supabase_flutter, Flutter Supabase, Supabase Dart client, Flutter Supabase auth, deep link login callback, Sign in with Apple Supabase, Flutter Supabase realtime, Flutter Supabase storage, supabase-flutter v3]
type: deep-dive
domain: mobile
tags: [domain/mobile, type/deep-dive, topic/flutter, topic/supabase, topic/auth, lang/dart]
status: draft
created: 2026-11-16
updated: 2026-11-16
version_checked: "supabase_flutter 2.16.x (v2) · v3 announced 2026-11-06, stable planned 2026-11-11 (breaking, type-safe API) · Flutter 3.47 — 2026-11"
parent: "[[Flutter]]"
related: ["[[Supabase - Auth Flows, MFA & Custom Claims]]", "[[Supabase - Realtime, Presence & Broadcast]]", "[[Flutter - State Management]]", "[[Flutter - Navigation & Routing]]", "[[Flutter - Offline Storage & Sync]]", "[[Supabase - Row Level Security]]"]
---

# Flutter - Supabase Integration, Auth & Realtime

> [!info] Deep dive of [[Flutter]]

> [!abstract] TL;DR
> `supabase_flutter` wraps the Dart clients for Auth, PostgREST, Realtime, Storage and Functions, with session persistence and deep-link handling. Production rules:
> - Only the **publishable/anon key** ships in the app. **RLS** is the real security boundary ([[Supabase - Row Level Security]]).
> - Drive navigation from **`onAuthStateChange`**, not from login-button callbacks.
> - Use **native Sign in with Apple/Google** (ID token → `signInWithIdToken`) over browser OAuth where possible.
> - **Unsubscribe Realtime channels** on dispose, and re-fetch on resume.
> - Plan the **v3 upgrade**: a breaking, type-safe API announced 2026-11. Check third-party packages built on v2 before bumping.

## Concept
| Area | API | Notes |
|---|---|---|
| Init | `Supabase.initialize(url:, anonKey:)` | Once in `main()`. Persists session (SharedPreferences by default) |
| Auth state | `auth.onAuthStateChange` stream | Events: `initialSession`, `signedIn`, `signedOut`, `tokenRefreshed`, `passwordRecovery` |
| Email OTP / magic link | `signInWithOtp(email:, emailRedirectTo:)` | Deep link back to the app |
| Phone OTP | `signInWithOtp(phone:)` + `verifyOTP` | SMS cost and abuse. Rate-limit and add CAPTCHA |
| Native social | `signInWithIdToken(provider:, idToken:, nonce:)` | Apple (required on iOS if other social logins exist), Google |
| Data | `from('t').select().eq()…` | RLS applies with the user's JWT |
| RPC | `rpc('fn', params: {...})` | Business logic in Postgres |
| Realtime | `channel().onPostgresChanges(...).subscribe()` | Respects RLS. Also Broadcast/Presence |
| Storage | `storage.from('bucket').uploadBinary / createSignedUrl` | Private buckets + signed URLs |
| Functions | `functions.invoke('name', body:)` | Edge Functions with the user JWT |

## How It Works
```mermaid
sequenceDiagram
  participant App as Flutter app
  participant Auth as Supabase Auth
  participant API as PostgREST / Realtime
  App->>Auth: signInWithIdToken(apple idToken, nonce)
  Auth-->>App: session (access JWT ~1h + refresh token)
  App->>App: persist session, onAuthStateChange(signedIn) → router redirect
  App->>API: queries / channel subscribe (Authorization: Bearer JWT)
  API-->>App: rows filtered by RLS / change events
  Note over App,Auth: client auto-refreshes before expiry; on resume, refresh + refetch
```

## Practical Usage
**Initialization** (keys via `--dart-define`, never hard-coded in git):
```dart
Future<void> main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Supabase.initialize(
    url: const String.fromEnvironment('SUPABASE_URL'),
    anonKey: const String.fromEnvironment('SUPABASE_PUBLISHABLE_KEY'),
    authOptions: const FlutterAuthClientOptions(authFlowType: AuthFlowType.pkce),
  );
  runApp(const ProviderScope(child: App()));
}
final supabase = Supabase.instance.client;
```
**Auth-driven routing** (go_router + Riverpod):
```dart
final authStateProvider = StreamProvider<AuthState>((ref) => supabase.auth.onAuthStateChange);

final routerProvider = Provider<GoRouter>((ref) {
  final refresh = ValueNotifier(0);
  ref.listen(authStateProvider, (_, __) => refresh.value++);
  return GoRouter(
    refreshListenable: refresh,
    redirect: (context, state) {
      final loggedIn = supabase.auth.currentSession != null;
      final onLogin = state.matchedLocation == '/login';
      if (!loggedIn && !onLogin) return '/login';
      if (loggedIn && onLogin) return '/';
      return null;
    },
    routes: [/* ... */],
  );
});
```
**Native Sign in with Apple** (nonce flow):
```dart
final rawNonce = supabase.auth.generateRawNonce();
final hashed = sha256.convert(utf8.encode(rawNonce)).toString();
final cred = await SignInWithApple.getAppleIDCredential(
  scopes: [AppleIDAuthorizationScopes.email, AppleIDAuthorizationScopes.fullName],
  nonce: hashed,
);
await supabase.auth.signInWithIdToken(
  provider: OAuthProvider.apple, idToken: cred.identityToken!, nonce: rawNonce);
```
**Magic link / OAuth deep link**: register the redirect in Supabase Auth → URL Configuration, plus platform config:
```xml
<!-- AndroidManifest.xml, inside the main activity -->
<intent-filter>
  <action android:name="android.intent.action.VIEW" />
  <category android:name="android.intent.category.DEFAULT" />
  <category android:name="android.intent.category.BROWSABLE" />
  <data android:scheme="my.rarticle.app" android:host="login-callback" />
</intent-filter>
```
**Typed queries through a repository** (keeps widgets free of Supabase calls):
```dart
class OrderRepo {
  Future<List<Order>> recent(String tenantId) async {
    final rows = await supabase.from('orders')
        .select('id, status, total_cents, created_at, items:order_items(count)')
        .eq('tenant_id', tenantId)                 // RLS still enforces; filter is for the index
        .order('created_at', ascending: false).limit(50);
    return rows.map(Order.fromJson).toList();
  }
}
```
**Realtime with cleanup**:
```dart
late final RealtimeChannel _ch;
@override void initState() {
  super.initState();
  _ch = supabase.channel('orders:${widget.tenantId}')
    .onPostgresChanges(
      event: PostgresChangeEvent.update, schema: 'public', table: 'orders',
      filter: PostgresChangeFilter(type: PostgresChangeFilterType.eq, column: 'tenant_id', value: widget.tenantId),
      callback: (p) => ref.invalidate(ordersProvider))
    .subscribe();
}
@override void dispose() { supabase.removeChannel(_ch); super.dispose(); }
```
**Wrong way**:
```text
❌ service_role key in --dart-define        → extractable from the APK/IPA; full DB access
❌ Navigation after await signIn() only      → misses refresh/expiry/sign-out events from other paths
❌ Realtime channels never removed           → duplicate callbacks, socket leaks, quota usage
❌ Trusting client-side tenant/role checks   → bypassable; enforce in RLS / RPC
```

## Patterns & Anti-patterns
| Pattern | When | Anti-pattern to avoid |
|---|---|---|
| Repository layer over `supabase` client | All apps | Queries inside widgets |
| `onAuthStateChange` → router `refreshListenable` | Auth flows | Manual `Navigator.push` after login |
| PKCE flow + deep links | Magic link, OAuth | Implicit flow tokens in URLs |
| Native Apple/Google sign-in via ID token | Social login | Browser OAuth when native is available |
| RPC for multi-step writes | Checkout, stock updates | Several client writes with no transaction |
| Refetch on `AppLifecycleState.resumed` | Realtime screens | Assuming the socket survived background |
| Custom claims (tenant, role) in the JWT via auth hook | Multi-tenant | Extra query per screen for role lookups |

## Performance & Trade-offs
- **Select only needed columns** and use embedded counts. PostgREST over mobile networks punishes `select('*')` with nested relations.
- **Realtime Postgres Changes** evaluates RLS per subscriber per change. For high-fanout updates use **Broadcast** from the server ([[Supabase - Realtime, Presence & Broadcast]]).
- **Offline**: the client has no offline cache. Pair with Drift/Isar or PowerSync-style sync if the app must work offline ([[Flutter - Offline Storage & Sync]]).
- **Session storage**: the default SharedPreferences persistence isn't encrypted. Use a secure-storage `LocalStorage` implementation for sensitive apps ([[Flutter - App Security, Obfuscation & Secure Storage]]).

## Tips & Reminders
> [!tip] In ZP's stack
> - **One Supabase project, two clients**: Next.js admin (`@supabase/ssr`) and Flutter customer app share RLS policies. Test policies once in SQL, not per client.
> - **Phone OTP in Malaysia**: SMS costs add up and OTP pumping is common. Prefer WhatsApp OTP via an auth hook, or email/social, and rate-limit ([[Supabase - Auth Flows, MFA & Custom Claims]]).
> - **Push notifications**: store FCM tokens in a `devices` table keyed by `auth.uid()`, and send from Edge Functions or n8n ([[Flutter - Push Notifications & Background Work]]).

- Use separate Supabase projects (or branches) for dev/staging/prod, selected by `--dart-define-from-file`.
- Apple requires account deletion in-app: implement an RPC/Edge Function that deletes the user via the admin API.
- Generate Dart types from the schema, or hand-write `fromJson` models with tests. Schema drift breaks silently.
- Handle `AuthException` and `PostgrestException` codes centrally (e.g. `PGRST301` expired JWT).

## Version Notes
| Version | Change relevant here |
|---|---|
| 2.0 (2024) | PKCE default, `onPostgresChanges` API, `signInWithIdToken` for native social |
| 2.x (2025–26) | New publishable/secret API keys supported, JWT signing keys (asymmetric) |
| 2.16.x | Latest v2 line seen at writing |
| **3.0** (stable planned 2026-11-11) | Breaking: type safety across the API surface. Imports/deps unchanged for direct users. Third-party packages built on v2 need v3 releases |

> [!warning] Unverified — check before relying on this
> Whether supabase_flutter 3.0.0 stable shipped on schedule, and its exact API changes (snippets above use the v2 API), weren't confirmed. Read https://supabase.com/changelog/supabase-flutter-v3 before upgrading.

## Critical Issues & Gotchas
> [!danger] Shipping privileged keys in the app binary
> Anything in `--dart-define`, assets or code is extractable from the APK/IPA in minutes. A `service_role`/secret key there gives full database and storage access, bypassing RLS. Ship only the publishable key, and do privileged work in Edge Functions or RPCs with `security definer` and explicit checks.

> [!danger] Legacy anon key removal and the v3 upgrade landing together
> Supabase plans to remove legacy `anon`/`service_role` keys in late 2026. Mobile apps embed the key at build time, so users who don't update the app keep using the old key. Release an app version with the new publishable key **well before** the cutoff, and force-update older versions ([[Supabase]]).

- **Deep-link mismatch**: the redirect URL must exactly match the allow-list in Supabase Auth settings. Otherwise users land on the Site URL (often the web app).
- **iOS Sign in with Apple** returns name/email only on first authorization. Persist them immediately.
- **Realtime + RLS**: tables need `replica identity` and inclusion in the `supabase_realtime` publication, or no events arrive.
- **Clock skew** on devices can cause "JWT issued in the future" errors right after login. Recent clients tolerate small skew.

## Related
- [[Flutter]]
- [[Supabase - Auth Flows, MFA & Custom Claims]] — server side of these flows
- [[Supabase - Realtime, Presence & Broadcast]] — channel types and limits
- [[Flutter - State Management]] — Riverpod providers around the client
- [[Flutter - Navigation & Routing]] — auth redirects and deep links
- [[Supabase - Row Level Security]] — the real authorization layer

## References
- https://supabase.com/docs/reference/dart/introduction
- https://supabase.com/docs/guides/auth/native-mobile-deep-linking
- https://supabase.com/docs/guides/auth/social-login/auth-apple
- https://supabase.com/changelog/supabase-flutter-v3
- https://pub.dev/packages/supabase_flutter
