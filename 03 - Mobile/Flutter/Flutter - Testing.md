---
title: Flutter - Testing
aliases: [Flutter unit test, widget test, integration_test, golden tests, Patrol, mocktail, bloc_test, Flutter test pyramid]
type: deep-dive
domain: mobile
tags: [domain/mobile, type/deep-dive, topic/flutter, topic/testing, lang/dart]
status: draft
created: 2026-10-08
updated: 2026-10-08
version_checked: "Flutter 3.47 — 2026-10"
parent: "[[Flutter]]"
related: ["[[Flutter - State Management]]", "[[Flutter - Build, Release & Store Compliance]]", "[[Flutter - Performance & DevTools]]", "[[CI-CD]]"]
---

# Flutter - Testing

> [!info] Deep dive of [[Flutter]]

> [!abstract] TL;DR
> Flutter has three built-in test levels, all run with `flutter test`:
> - **Unit tests** (pure Dart: repositories, notifiers, blocs). Milliseconds each.
> - **Widget tests** (`testWidgets` + `WidgetTester`: a headless widget tree with fake async). Fast, and the sweet spot for UI logic.
> - **Integration tests** (`integration_test` on a real device/emulator, or **Patrol** for native dialogs and permissions). Slow but real.
>
> Golden tests catch visual regressions. For solo/SME projects: many unit tests for business logic, widget tests for every screen state (loading/error/empty/data), a handful of integration smoke tests on CI, and **mock repositories, not HTTP**.

## Concept

### Test levels
| Level | API | Runs on | Speed | Confidence | Use for |
|---|---|---|---|---|---|
| Unit | `package:test` / `flutter_test` `test()` | Dart VM | ~ms | Logic only | Models, parsers, notifiers, blocs, repositories (with fakes) |
| Widget | `testWidgets`, `WidgetTester`, finders | Headless engine (no GPU), fake clock | ~10–100 ms | UI logic + layout | Screens per state, forms, navigation flows with fake router deps |
| Golden | `matchesGoldenFile` / alchemist | Headless render to PNG | Fast-ish | Pixel output | Design-system components, critical screens |
| Integration | `integration_test` + `flutter test integration_test/` | Real device/emulator/desktop | Minutes | End-to-end incl. plugins | Login → core flow smoke tests, performance traces |
| Native E2E | **Patrol** | Device, with native automation (UIAutomator/XCUITest) | Minutes | Permission dialogs, notifications, webviews | Flows crossing the native boundary |

### Test doubles
| Tool | Style | Note |
|---|---|---|
| **mocktail** | `class MockRepo extends Mock implements OrderRepository` | No codegen. Null-safety friendly. `registerFallbackValue` for custom arg types |
| mockito | `@GenerateNiceMocks` + build_runner | Codegen. Strict types |
| Hand-written fakes | `class FakeOrderRepo implements OrderRepository` | Best for stateful behaviour (in-memory list). Often clearer than mocks |
| Riverpod overrides | `ProviderScope(overrides: [repoProvider.overrideWithValue(fake)])` | Swap dependencies without DI frameworks |

## How It Works
- `testWidgets` runs inside a **`FakeAsync` zone** with `AutomatedTestWidgetsFlutterBinding`. Time only advances when you call `pump(duration)` / `pumpAndSettle()`. Timers left pending at test end fail the test ("A Timer is still pending").
- `pump()` triggers one frame (build/layout/paint). `pumpAndSettle()` pumps until no frames are scheduled, so it **never settles** with infinite animations (`CircularProgressIndicator`). Use `pump(const Duration(...))` in those cases.
- Real I/O (HTTP, `File`, platform channels) doesn't work in the fake zone. Wrap it in `tester.runAsync(() async { … })` or, better, fake the dependency.
- **HTTP** in widget tests returns 400 by default (`HttpOverrides`), which is a deliberate push to inject fakes.
- The default test surface is **800×600 logical pixels at DPR 3.0**. Set `tester.view.physicalSize` / `devicePixelRatio` (and reset with `addTearDown(tester.view.reset)`) to test phone layouts.

## Practical Usage

### Unit: Riverpod notifier with a fake repo
```dart
test('markPaid updates the order list', () async {
  final repo = FakeOrderRepo([Order(id: 'o1', status: OrderStatus.pending)]);
  final container = ProviderContainer(overrides: [orderRepoProvider.overrideWithValue(repo)]);
  addTearDown(container.dispose);

  await container.read(ordersProvider(tenantId: 't1').future);
  await container.read(ordersProvider(tenantId: 't1').notifier).markPaid('o1');

  final orders = await container.read(ordersProvider(tenantId: 't1').future);
  expect(orders.single.status, OrderStatus.paid);
});
```

### Unit: bloc_test
```dart
blocTest<CheckoutCubit, CheckoutState>(
  'emits [Paying, Failed] when the charge throws',
  build: () { when(() => payments.charge(any())).thenThrow(PaymentDeclined()); return CheckoutCubit(payments); },
  act: (c) => c.pay(cart),
  expect: () => [isA<CheckoutPaying>(), isA<CheckoutFailed>()],
);
```

### Widget: every state of a screen
```dart
Future<void> pumpOrders(WidgetTester t, OrderRepository repo) => t.pumpWidget(ProviderScope(
  overrides: [orderRepoProvider.overrideWithValue(repo)],
  child: const MaterialApp(home: OrdersPage(tenantId: 't1')),
));

testWidgets('shows empty state', (t) async {
  await pumpOrders(t, FakeOrderRepo([]));
  await t.pump();                                   // resolve the Future
  expect(find.text('No orders yet'), findsOneWidget);
});

testWidgets('tapping Pay marks the order paid', (t) async {
  await pumpOrders(t, FakeOrderRepo([Order(id: 'o1', status: OrderStatus.pending)]));
  await t.pump();
  await t.tap(find.byKey(const ValueKey('pay-o1')));
  await t.pumpAndSettle();
  expect(find.text('Paid'), findsOneWidget);
});
```
Finders: `find.text`, `find.byKey`, `find.byType`, `find.bySemanticsLabel` (also checks accessibility), `find.descendant(of:, matching:)`.

### Golden test
```dart
testWidgets('OrderTile golden', (t) async {
  await t.pumpWidget(MaterialApp(theme: appTheme, home: OrderTile(order: sampleOrder)));
  await expectLater(find.byType(OrderTile), matchesGoldenFile('goldens/order_tile.png'));
});
// update: flutter test --update-goldens
```

### Integration test
```dart
// integration_test/login_flow_test.dart
void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();
  testWidgets('login → orders', (t) async {
    app.main(env: 'staging');
    await t.pumpAndSettle();
    await t.enterText(find.byKey(const ValueKey('email')), 'qa@example.my');
    await t.enterText(find.byKey(const ValueKey('password')), const String.fromEnvironment('QA_PASSWORD'));
    await t.tap(find.text('Sign in'));
    await t.pumpAndSettle(const Duration(seconds: 5));
    expect(find.byType(OrdersPage), findsOneWidget);
  });
}
```
```bash
flutter test integration_test/ -d emulator-5554 --dart-define=QA_PASSWORD=$QA_PASSWORD
flutter test --coverage && genhtml coverage/lcov.info -o coverage/html
```

## Patterns & Anti-patterns
| Pattern | When | Anti-pattern to avoid |
|---|---|---|
| Fake/mock at the **repository** boundary | All UI and state tests | Mocking `http.Client`/Supabase client internals |
| Test each async state (loading, error, empty, data) | Every data-driven screen | Only the happy path |
| `ValueKey`s on interactive widgets in critical flows | Widget/integration tests | Finding by brittle text that changes with copy/i18n |
| Dedicated staging backend + seeded test accounts | Integration tests | Running E2E against production |
| Goldens rendered on one fixed CI platform (Linux) | Golden tests | Generating goldens on macOS and comparing on Linux (font/anti-alias diffs) |
| `addTearDown` for containers, controllers, view resets | All tests | Leaking state between tests |

## Performance & Trade-offs
- Unit and widget tests run in parallel (`--concurrency`, default ≈ CPU cores). Keep them hermetic so parallelism is safe.
- Integration tests need emulators: slow to boot on CI and flaky with animations. Keep them to a small smoke suite; use Firebase Test Lab or a device farm for broader coverage.
- Goldens give high value for design systems, but churn when fonts, Flutter versions or themes change. Pin fonts (load them in `flutter_test_config.dart`) and regenerate in one PR per Flutter upgrade.
- Coverage % is a weak metric. Prioritize money paths (checkout, payment status, invoice totals) and auth.

## Tips & Reminders
> [!tip]
> - `flutter_test_config.dart` in `test/` runs before all tests in that folder. Use it for font loading, `registerFallbackValue`, leak tracking settings.
> - Enable leak tracking in widget tests (`LeakTesting.enable()`, see [[Dart - VM, Compilation & Garbage Collection]]) to catch undisposed controllers early.
> - Use `tester.binding.setSurfaceSize` or `tester.view` to test small phones (360×640), where overflows actually happen.
> - **In ZP's stack**: test Supabase-backed features through repository fakes. For RLS rules, test at the database level (pgTAP / `supabase test db`), not in Flutter. Run `flutter analyze && flutter test` on every PR, and integration smoke tests on the release branch before store builds ([[Flutter - Build, Release & Store Compliance]]).

## Version Notes
| Version | Change |
|---|---|
| Flutter 2.0 | `integration_test` replaces `flutter_driver` as the recommended E2E package |
| Flutter 3.10+ | `tester.view` / `TestFlutterView` replace `window` test APIs |
| Flutter 3.18+ | leak_tracker integrated into `flutter_test` (opt-in) |
| bloc_test 10 / mocktail 1.x | Current majors, null-safe, Dart 3 compatible |
| Patrol 3.x–4.x | Native automation, test bundling (`patrol test`), hot restart for tests |

> [!warning] Unverified — check before relying on this
> Package majors (bloc_test, mocktail, Patrol) and the leak_tracker introduction version weren't re-checked this run. Confirm on pub.dev before pinning.

## Critical Issues & Gotchas
> [!danger] Green tests, broken release build
> Tests run in debug/JIT with asserts on. They won't catch release-only issues: R8/ProGuard stripping reflection-based plugin classes, obfuscation breaking `runtimeType.toString()`-based logic, missing iOS permissions strings (`NSCameraUsageDescription`) that crash on first use. Smoke-test the actual release/TestFlight build before rollout.

> [!warning] Gotchas
> - `pumpAndSettle` times out with infinite animations (spinners, shimmer). Use a fixed `pump(Duration)`.
> - `DateTime.now()` in widgets makes tests time-dependent. Inject a clock (`package:clock`, `withClock`).
> - Platform channels return `MissingPluginException` in widget tests. Mock with `setMockMethodCallHandler` or wrap the plugin behind an interface.
> - `SharedPreferences` needs `SharedPreferences.setMockInitialValues({})`. Supabase/Firebase need their own init or fakes.
> - Golden tests show blank text unless fonts are loaded (the default test font is the "Ahem" box font).

## Related
- [[Flutter]]
- [[Flutter - State Management]] — testable state holders, overrides
- [[Flutter - Performance & DevTools]] — perf traces in integration tests
- [[Flutter - Build, Release & Store Compliance]] — CI test gate before store uploads
- [[CI-CD]] — pipelines

## References
- Testing Flutter apps: https://docs.flutter.dev/testing/overview
- Widget testing: https://docs.flutter.dev/cookbook/testing/widget/introduction
- Integration testing: https://docs.flutter.dev/testing/integration-tests
- mocktail: https://pub.dev/packages/mocktail
- bloc_test: https://pub.dev/packages/bloc_test
- Patrol: https://patrol.leancode.co/
