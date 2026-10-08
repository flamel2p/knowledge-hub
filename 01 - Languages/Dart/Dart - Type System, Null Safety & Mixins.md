---
title: Dart - Type System, Null Safety & Mixins
aliases: [Dart null safety, Dart mixins, Dart generics, const vs final, sealed classes, extension types, class modifiers]
type: deep-dive
domain: languages
tags: [domain/languages, type/deep-dive, topic/dart, topic/type-system, lang/dart]
status: draft
created: 2026-10-08
updated: 2026-10-08
version_checked: "Dart 3.13 — 2026-10"
parent: "[[Dart]]"
related: ["[[Flutter - Interview Questions]]", "[[Dart - Async, Streams & Isolates]]", "[[TypeScript - Advanced Types]]", "[[Flutter - Rendering & Widget Lifecycle]]"]
---

# Dart - Type System, Null Safety & Mixins

> [!info] Deep dive of [[Dart]]

> [!abstract] TL;DR
> Dart has a **sound, nominal, statically checked** type system with reified generics, plus a few runtime checks at explicit escape hatches.
> - **Sound null safety** makes `T` and `T?` different types. Flow analysis promotes nullable values after checks, and because every program is fully null-safe (Dart 3), a non-nullable type is a **guarantee**, not a hint.
> - **Code reuse**: single inheritance (`extends`), interfaces (`implements`, where every class is an implicit interface), and **mixins** (`with`, linearized composition), all governed by Dart 3 **class modifiers**.
> - **`const` vs `final`**: `const` is compile-time and canonicalized, and that's what lets Flutter skip identical widget subtrees.

## Concept

### Soundness and nominal typing
- **Sound**: if an expression has static type `T`, its runtime value is always a `T` (unlike TypeScript, whose types are erased and unsound). The compiler and runtime enforce this, and casts (`as`) are checked at runtime.
- **Nominal**: compatibility is by declared names and subtype relationships, not shape. Two classes with identical fields aren't interchangeable.
- **Reified generics**: type arguments exist at runtime. `List<int>` is checked (`list is List<int>` works), unlike Java's erasure.
- **Top/bottom types**: `Object?` (top, holds anything incl. null), `Object` (any non-null), `dynamic` (opts out of static checking), `Never` (bottom, for functions that never return).

### Null safety
```dart
String? maybeName;          // nullable
String name = 'Ali';        // non-nullable: can never hold null

int len(String? s) {
  if (s == null) return 0;
  return s.length;          // promoted to String by flow analysis
}

final display = maybeName ?? 'Guest';          // ?? default
final upper = maybeName?.toUpperCase();        // ?. null-aware call → String?
maybeName ??= 'Guest';                         // assign if null
final forced = maybeName!;                     // runtime check: throws if null
late final Database db;                        // non-nullable, assigned later; runtime check on read
final items = [?maybeName, 'b'];               // null-aware element (3.8): skipped if null
```
- **Promotion** works for local variables and private final fields (since 3.2) when the compiler can prove no intervening change. Public fields don't promote (a getter could return something different), so copy them to a local.
- **Pattern-based null checks**: `if (json case {'id': final String id?})`, `switch (x) { case final v?: … }`.
- **Escape hatches** (where runtime checks remain): `!`, `as`, `late`, `dynamic`, and data entering from outside (`jsonDecode` returns `dynamic`).

### Classes, interfaces, mixins
| Mechanism | Syntax | Gets implementation | Count |
|---|---|---|---|
| Inheritance | `class B extends A` | Yes | One superclass |
| Interface | `class B implements A, C` | No: must implement all members | Many (every class defines an implicit interface) |
| Mixin | `class B extends A with M1, M2` | Yes | Many, applied in order |
| Constrained mixin | `mixin M on A { … }` | Yes, may call `super` on A's members | Only on subtypes of A |
| Extension methods | `extension on String { … }` | Adds static-dispatch methods | Many, no state |
| Extension types (3.3) | `extension type OrderId(String v) {}` | Zero-cost wrapper with its own API | Compile-time only |

**Mixin linearization**: `class C extends A with M1, M2` is semantically `A → (A+M1) → (A+M1+M2) → C`. Each mixin application creates an anonymous superclass. Lookup goes from `C` upward, so **M2 overrides M1, which overrides A**, and `super.m()` inside M2 calls M1's version. That deterministic order is why Dart avoids the diamond problem.

### Dart 3 class modifiers
| Modifier | Outside its library you can… |
|---|---|
| (none) | Construct, extend, implement |
| `abstract` | Not construct. Extend/implement OK |
| `base` | Extend only (no `implements`), so invariants in the base implementation hold |
| `interface` | Implement only (no `extends`) |
| `final` | Neither extend nor implement (closed hierarchy) |
| `sealed` | Neither. **Exhaustive switching** over direct subtypes. Implicitly abstract |
| `mixin class` | Use as a class **and** as a mixin |

### const vs final
| | `final` | `const` |
|---|---|---|
| When fixed | Runtime, assigned once | Compile time |
| Object mutability | Object may be mutable (`final list = [];` then `list.add`) | Deeply immutable |
| Instances | New instance each evaluation | **Canonicalized**: identical const expressions share one instance |
| Requirements | None | Const constructor, all fields `final`, const arguments |

## How It Works
- **Static analysis** (CFE, the common front end) runs type inference, flow analysis (definite assignment, promotion, reachability) and null-safety checks. Errors block compilation.
- **Soundness payoff**: since a non-nullable static type can't hold null at runtime, AOT compilers drop null checks and use unboxed representations. Null-safe code is smaller and faster.
- **Runtime checks** are inserted only where unsoundness could enter: casts (`as`), `!`, `late` reads, covariant parameter checks (`covariant` keyword, generic class parameters), and dynamic calls.
- **const canonicalization**: the compiler evaluates const expressions and stores one instance per distinct value. At runtime, `identical(const Text('a'), const Text('a'))` is `true`. Flutter's `Element.updateChild` short-circuits when `identical(oldWidget, newWidget)`, so const subtrees skip rebuild work.

## Practical Usage

### Sealed result types + exhaustive switch
```dart
sealed class ApiResult<T> { const ApiResult(); }
final class Ok<T> extends ApiResult<T> { const Ok(this.value); final T value; }
final class Err<T> extends ApiResult<T> { const Err(this.error, [this.stack]); final Object error; final StackTrace? stack; }

String describe(ApiResult<Order> r) => switch (r) {
  Ok(:final value) => 'Order ${value.id}',
  Err(:final error) => 'Failed: $error',
};   // add a third subclass → compile error here until handled
```

### Mixins in Flutter
```dart
class _ChatPageState extends State<ChatPage>
    with SingleTickerProviderStateMixin, WidgetsBindingObserver {    // provides vsync + lifecycle callbacks
  late final _anim = AnimationController(vsync: this, duration: const Duration(milliseconds: 250));

  @override
  void initState() { super.initState(); WidgetsBinding.instance.addObserver(this); }

  @override
  void didChangeAppLifecycleState(AppLifecycleState s) { if (s == AppLifecycleState.resumed) _refresh(); }

  @override
  void dispose() { WidgetsBinding.instance.removeObserver(this); _anim.dispose(); super.dispose(); }
}

mixin Loggable on ChangeNotifier {           // constrained mixin: only for ChangeNotifier subclasses
  @override
  void notifyListeners() { debugPrint('$runtimeType changed'); super.notifyListeners(); }
}
```

### Extension types for IDs
```dart
extension type TenantId(String value) {}
extension type OrderId(String value) {}
Future<Order> getOrder(TenantId t, OrderId id) async { … }
// getOrder(OrderId('A1'), TenantId('T1'));  → compile error (swapped)
```

### Safe JSON → typed model
```dart
Order orderFromJson(Map<String, Object?> json) => switch (json) {
  {'id': final String id, 'total_sen': final int total, 'status': final String status} =>
      Order(id: id, totalSen: total, status: OrderStatus.values.byName(status)),
  _ => throw FormatException('Invalid order JSON: $json'),
};
```

## Patterns & Anti-patterns
| Pattern | When | Anti-pattern to avoid |
|---|---|---|
| Model optionality explicitly (`T?`) and handle it at boundaries | Data from APIs/DB | `!` sprinkled to silence errors |
| `sealed` hierarchies for states/results | UI state, API results, events | Enums + nullable fields that can contradict |
| `final` fields + `const` constructors | Widgets, value objects | Mutable widget fields |
| Mixins for cross-cutting behaviour | Lifecycle observers, logging, animation vsync | Deep inheritance chains for code reuse |
| `interface`/`final` modifiers on public library types | Packages/shared modules | Accidental subclassing breaking invariants |
| Extension types for IDs/units | Many string/int IDs | Plain `String` for every ID |

## Performance & Trade-offs
- Null-safe AOT code drops null checks, giving smaller and faster binaries.
- `const` widgets save allocation and rebuild work. The benefit is largest in frequently rebuilt subtrees (lists, animations).
- `dynamic` disables static optimization and adds runtime checks. Keep it at the edges.
- Mixins add a class per application in the hierarchy. That costs negligible runtime but makes method resolution order something to reason about.

## Tips & Reminders
> [!tip]
> - Analyzer options: `strict-casts`, `strict-inference`, `strict-raw-types`, and lints like `prefer_const_constructors`, `avoid_dynamic_calls`, `unnecessary_null_checks`.
> - When the analyzer won't promote a field, assign it to a local (`final v = this.value; if (v != null) …`).
> - Prefer `late final` + an initializer (`late final x = compute();`, lazy evaluation) over a `late` that is assigned "somewhere later".
> - **In ZP's stack**: model Supabase rows as immutable classes (freezed or hand-written) with `fromJson` via patterns, and API/state results as `sealed` classes. Riverpod's `AsyncValue` and BLoC states fit the same approach.

## Version Notes
| Version | Change |
|---|---|
| 2.12 (2021) | Sound null safety (opt-in), `late`, `required` |
| 2.17 | Enhanced enums (fields, methods), super-initializer parameters |
| **3.0** (2023-05) | Null safety mandatory. Records, patterns, `switch` expressions, class modifiers (`sealed`, `base`, `interface`, `final`, `mixin class`) |
| 3.2 | Private final field promotion |
| 3.3 | Extension types |
| 3.8 | Null-aware elements (`?x` in collection literals) |
| 3.10 | Dot shorthands (`.center`) |

## Critical Issues & Gotchas
> [!danger] Unsound edges still crash at runtime
> `!` on a null, a failing `as` cast, reading an unassigned `late` field, or a `dynamic` call to a missing method all throw at runtime, typically from JSON or platform-channel data. Validate external data into typed models at the boundary, and treat each `!` as a code-review flag.

> [!warning] Gotchas
> - `const` requires every nested value to be const. One non-const argument makes the whole expression non-const (and the lint may silently stop suggesting it).
> - A mixin's `super` call depends on application order. Reordering `with A, B` can change behaviour.
> - `implements` a class means re-implementing **all** its members, including private-library ones you can't see (in Dart 3 a `base` class forbids it outside its library).
> - Generic covariance: `List<Object> objs = <int>[]; objs.add('x');` compiles but throws at runtime (covariant check).
> - `==`/`hashCode` default to identity. Use records, freezed or `Equatable` for value equality in state management.

## Related
- [[Dart]]
- [[Flutter - Interview Questions]] — Q4, Q5, Q8 outlines
- [[Dart - Async, Streams & Isolates]]
- [[Flutter - Rendering & Widget Lifecycle]] — const widgets and `Element.updateChild`
- [[TypeScript - Advanced Types]] — contrast: structural, erased, unsound types

## References
- Sound null safety: https://dart.dev/null-safety
- Understanding null safety (deep dive): https://dart.dev/null-safety/understanding-null-safety
- Mixins: https://dart.dev/language/mixins
- Class modifiers: https://dart.dev/language/class-modifiers
- Extension types: https://dart.dev/language/extension-types
- Patterns: https://dart.dev/language/patterns
