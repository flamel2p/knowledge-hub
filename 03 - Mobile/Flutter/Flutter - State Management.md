---
title: Flutter - State Management
aliases: [Riverpod, BLoC, Cubit, Provider, ValueNotifier, ChangeNotifier, Flutter state]
type: deep-dive
domain: mobile
tags: [domain/mobile, type/deep-dive, topic/flutter, topic/state-management, lang/dart]
status: draft
created: 2026-10-08
updated: 2026-10-08
version_checked: "Riverpod 3.4 · flutter_bloc 9.1 — 2026-10"
parent: "[[Flutter]]"
related: ["[[Flutter - Rendering & Widget Lifecycle]]", "[[Dart - Async, Streams & Isolates]]", "[[Supabase]]", "[[Flutter - Interview Questions]]"]
---

# Flutter - State Management

> [!info] Deep dive of [[Flutter]]

> [!abstract] TL;DR
> State management is about **where state lives, who can change it, and which widgets rebuild when it changes**.
> - **Ephemeral UI state** (a checkbox, a text field, an animation) belongs in `StatefulWidget`/`ValueNotifier`.
> - **App/server state** (auth, cart, orders from [[Supabase]]) belongs in a dedicated layer. **Riverpod 3** (compile-safe DI + async caching + auto-dispose) is the best default for SME apps. **BLoC/Cubit** fits teams that want explicit events → states and strict auditability.
>
> Remember: **pick one approach per app, keep business logic out of widgets, and scope rebuilds as narrowly as possible.**

## Concept

### Kinds of state
| Kind | Examples | Where it lives |
|---|---|---|
| **Ephemeral / UI** | Selected tab, form field focus, expanded tile, animation | `State` + `setState`, `ValueNotifier`, controllers |
| **App state** | Auth session, theme, locale, cart, feature flags | Riverpod providers / BLoC / Provider |
| **Server state (cache)** | Orders, products, chat messages | Async providers / repositories with caching, invalidation, refresh |
| **Navigation state** | Current route, deep link params | Router (go_router) + URL |
| **Persisted state** | Drafts, offline queue, settings | Local DB (Drift/Isar/SharedPreferences) surfaced through providers |

### Options compared
| Approach | Model | Strengths | Weaknesses |
|---|---|---|---|
| `setState` / `ValueNotifier` / `ListenableBuilder` | Local mutable state | Zero deps, simplest | Doesn't scale across screens |
| `InheritedWidget` / `InheritedNotifier` | Built-in DI + rebuild on change | What everything else builds on | Boilerplate |
| **Provider** (6.x) | `ChangeNotifier` + `InheritedWidget` | Simple, familiar | Runtime errors (`ProviderNotFoundException`), context-bound. Superseded by Riverpod |
| **Riverpod 3** | Providers outside the widget tree, `Notifier`/`AsyncNotifier`, `ref.watch/read/listen` | Compile-safe, testable, async built in (`AsyncValue`), auto-dispose, family params, automatic retry, `Ref.mounted` | Learning curve. Codegen optional but common |
| **BLoC / Cubit** (9.x) | Events → `Bloc` → States (or Cubit methods → states), streams underneath | Explicit, traceable transitions, great for large teams, `bloc_test`, hydrated_bloc | More boilerplate. Async/caching patterns are manual |
| Signals / MobX | Fine-grained reactive values | Minimal rebuilds | Smaller ecosystems |
| GetX | All-in-one | Fast to start | Global magic, hard to test/maintain. Avoid for client work |

## How It Works

```mermaid
flowchart LR
  UI["Widgets (ConsumerWidget / BlocBuilder)"] -->|ref.watch / context.watch| ST[State holder: Notifier / Bloc]
  UI -->|user intent: ref.read(p.notifier).add / bloc.add(Event)| ST
  ST --> REPO[Repository]
  REPO --> SB[(Supabase / REST)]
  REPO --> LDB[(Local DB / cache)]
  ST -->|new immutable state| UI
```

- **Riverpod**: a `ProviderContainer` (`ProviderScope` at the root) holds provider states.
  - `ref.watch(p)` subscribes the widget or provider and rebuilds it on change.
  - `ref.read(p)` reads once (use it in callbacks).
  - `ref.listen` runs side effects (snackbars, navigation).
  - Providers form a dependency graph. When a dependency changes, dependents recompute.
  - **Auto-dispose** (the default for generated providers) frees state when nobody listens, after an optional `keepAlive`.
- **BLoC**: a `Bloc<Event, State>` processes events (with concurrency transformers from `bloc_concurrency`: `droppable`, `restartable`, `sequential`) and emits states on a stream. `BlocBuilder` and `BlocListener` subscribe and auto-cancel. `BlocProvider` creates the bloc and closes it on dispose.
- **Rebuild scoping**: `select` (`ref.watch(p.select((s) => s.total))`, `context.select`, `BlocSelector`) rebuilds only when the selected slice changes, compared with `==`. So states should be **immutable with value equality** (freezed / records / `Equatable`).

## Practical Usage

### Riverpod 3: async server state with Supabase
```dart
@riverpod
class Orders extends _$Orders {
  @override
  Future<List<Order>> build({required String tenantId}) async {
    final rows = await ref.watch(supabaseProvider).from('orders')
        .select().eq('tenant_id', tenantId).order('created_at', ascending: false).limit(50);
    return rows.map(Order.fromJson).toList();
  }

  Future<void> markPaid(String id) async {
    await ref.read(supabaseProvider).from('orders').update({'status': 'paid'}).eq('id', id);
    ref.invalidateSelf();                       // refetch; or update state optimistically
  }
}

class OrdersPage extends ConsumerWidget {
  const OrdersPage({super.key, required this.tenantId});
  final String tenantId;
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final orders = ref.watch(ordersProvider(tenantId: tenantId));
    return switch (orders) {
      AsyncData(:final value) => OrderList(orders: value),
      AsyncError(:final error) => ErrorView(error),
      _ => const Center(child: CircularProgressIndicator()),
    };
  }
}
```

### Riverpod: realtime stream, auto-disposed
```dart
@riverpod
Stream<List<Message>> conversation(Ref ref, String conversationId) {
  return ref.watch(supabaseProvider).from('messages')
      .stream(primaryKey: ['id']).eq('conversation_id', conversationId)
      .map((rows) => rows.map(Message.fromJson).toList());
  // subscription cancelled automatically when no widget watches this provider (auto-dispose)
}
```

### Cubit for a simple, explicit flow
```dart
sealed class CheckoutState { const CheckoutState(); }
class CheckoutIdle extends CheckoutState { const CheckoutIdle(); }
class CheckoutPaying extends CheckoutState { const CheckoutPaying(); }
class CheckoutDone extends CheckoutState { const CheckoutDone(this.receiptUrl); final String receiptUrl; }
class CheckoutFailed extends CheckoutState { const CheckoutFailed(this.reason); final String reason; }

class CheckoutCubit extends Cubit<CheckoutState> {
  CheckoutCubit(this._payments) : super(const CheckoutIdle());
  final PaymentRepository _payments;

  Future<void> pay(Cart cart) async {
    if (state is CheckoutPaying) return;                    // guard double-tap
    emit(const CheckoutPaying());
    try { emit(CheckoutDone(await _payments.charge(cart))); }
    catch (e) { emit(CheckoutFailed('$e')); }
  }
}
```

### Local state stays local
```dart
final _expanded = ValueNotifier(false);   // in State; dispose() it
ValueListenableBuilder(valueListenable: _expanded, builder: (_, open, __) => open ? const Details() : const SizedBox.shrink());
```

## Patterns & Anti-patterns
| Pattern | When | Anti-pattern to avoid |
|---|---|---|
| Repository layer between state and data sources | All apps | Supabase calls directly inside widgets |
| Immutable state + value equality (freezed/records/Equatable) | Riverpod/BLoC | Mutating lists in place (no rebuild, or rebuilds everywhere) |
| `ref.watch` in build, `ref.read` in callbacks, `ref.listen` for effects | Riverpod | `ref.read` in build (misses updates) / navigation inside build |
| `select`/`BlocSelector` for narrow rebuilds | Large states | Watching a whole state object for one field |
| Sealed state classes + exhaustive switch | Async/flow states | Boolean flags (`isLoading`, `hasError`, `isEmpty`) that can conflict |
| Auto-dispose by default, `keepAlive` deliberately | Screen-scoped data | Global singletons holding every screen's data forever |
| One state approach per app | Consistency | Mixing GetX + Provider + BLoC in one codebase |

## Performance & Trade-offs
- Rebuild scope is the main performance lever. Widgets watching a provider rebuild on every change. Split providers or use `select`.
- `AsyncValue` / sealed states avoid flicker. Riverpod keeps previous data during refresh (`isRefreshing`) for smooth pull-to-refresh.
- BLoC event transformers (`restartable` for search, `droppable` for submit) solve race conditions that hand-written code often gets wrong.
- Codegen (`riverpod_generator`, `freezed`) adds build_runner time but removes whole classes of bugs. Run `build_runner watch` during development.

## Tips & Reminders
> [!tip]
> - **Decision rule**: solo or small team, data-heavy SME app → **Riverpod 3**. Larger team, audit-heavy flows (payments, KYC), or an existing BLoC codebase → **BLoC/Cubit**. Tiny app → `setState` + `ValueNotifier` + a repository.
> - Test state holders without widgets: `ProviderContainer` overrides (Riverpod) or `blocTest` (BLoC). Mock repositories, not HTTP.
> - Keep auth state in one provider/bloc that drives router redirects (go_router `refreshListenable` / `redirect`).
> - **In ZP's stack**: [[Supabase]] auth `onAuthStateChange` → auth provider → router redirect. Realtime tables → auto-disposed `StreamProvider`s per screen. Offline drafts → local DB repository exposed through providers.

## Version Notes
| Version | Change |
|---|---|
| Provider 4–6 | Popular `ChangeNotifier` DI. Now maintenance mode in practice |
| Riverpod 2.x (2022–24) | Notifier/AsyncNotifier, codegen (`@riverpod`) |
| **Riverpod 3.0** (2025-09-10) | Unified Notifier interfaces (AutoDispose merged), automatic retry with backoff, `Ref.mounted`, experimental **offline persistence** and **mutations**. Latest 3.4.x |
| flutter_bloc 9.x / bloc 9.x | Current major. Stable API, `bloc_concurrency`, hydrated_bloc |

> [!warning] Unverified — check before relying on this
> Riverpod 3's experimental features (offline persistence, mutations) change between minors, and sources disagree on codegen-only usage. Check https://riverpod.dev/docs/whats_new before adopting them in client apps.

## Critical Issues & Gotchas
> [!danger] State leaks and stale data across users
> Global or keep-alive providers/blocs that cache user-specific data (orders, profile) survive logout unless explicitly reset, so the next user on the same device sees the previous user's data. On sign-out, invalidate/reset user-scoped providers (or recreate the `ProviderScope`/blocs). This is a privacy (PDPA) issue, not just a bug.

> [!warning] Gotchas
> - `ref.watch` inside callbacks or `initState` → errors or missed updates. Use `ref.read`/`ref.listen`.
> - Emitting the same (equal) state does nothing in BLoC. Mutating a list then emitting it "doesn't update" because it's equal.
> - Forgetting `BlocProvider` disposal semantics: `BlocProvider.value` does **not** close the bloc, and `BlocProvider(create:)` does.
> - Async gaps: after `await`, the provider/widget may be disposed. Check `ref.mounted` / `context.mounted`.
> - Overusing `keepAlive` turns auto-dispose into a memory leak.

## Related
- [[Flutter]]
- [[Flutter - Rendering & Widget Lifecycle]] — what rebuilds cost
- [[Dart - Async, Streams & Isolates]] — streams under BLoC/StreamProvider
- [[Supabase]] — data source in ZP's stack
- [[Flutter - Interview Questions]] — Q7 (framework-managed subscriptions)

## References
- Flutter state management options: https://docs.flutter.dev/data-and-backend/state-mgmt/options
- What's new in Riverpod 3.0: https://riverpod.dev/docs/whats_new
- Riverpod changelog: https://pub.dev/packages/riverpod/changelog
- flutter_bloc: https://pub.dev/packages/flutter_bloc
- BLoC library docs: https://bloclibrary.dev/
