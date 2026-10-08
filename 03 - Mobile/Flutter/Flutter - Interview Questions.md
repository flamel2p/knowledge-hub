---
title: Flutter - Interview Questions
aliases: [Flutter interview, Dart interview questions, Flutter 面试题, Flutter面经]
type: reference
domain: mobile
tags: [domain/mobile, type/reference, topic/flutter, topic/interview, lang/dart]
status: draft
created: 2026-10-07
updated: 2026-10-07
version_checked: "Dart 3.13 / Flutter 3.47 — 2026-10"
parent: "[[Flutter]]"
related: ["[[Dart - Async, Streams & Isolates]]", "[[Flutter - Rendering & Widget Lifecycle]]", "[[Flutter - State Management]]", "[[Dart]]"]
---

# Flutter - Interview Questions

> [!info] Reference for [[Flutter]] and [[Dart]]

> [!abstract] TL;DR
> 12 high-frequency Flutter/Dart interview questions on **Dart core & async** (Q1–Q8) and the **rendering pipeline** (Q9–Q12), each with a verified answer outline, the trap interviewers look for, and links to the deep dives. Q7 (StreamSubscription leaks + broadcast streams) is fully worked in the "interviewer's view → framework → strong vs weak answer" format. Source: a Xiaohongshu interview question bank compiled from candidates who received offers, fact-checked and corrected here (see the corrections callout).

> [!tip] How to use this note
> Answer every question with a **causal chain**: *what it is → why it behaves that way (mechanism) → how you handle it in production → trade-off/trap*. Interviewers grade the mechanism and the real-project choices, not memorised API names.

## Part A — Dart core & async

### Q1. Dart is single-threaded: how do the event loop, microtask queue and event queue give non-blocking async?
- Each isolate has **one thread + one event loop** with two queues: the **microtask** queue (drained completely after every event) and the **event** queue (timers, I/O, gestures, frames, isolate messages).
- I/O and timers are done by the VM/OS outside Dart. Their completions arrive as **events**, so Dart code never blocks waiting. It yields at `await` and resumes when the result event/microtask runs.
- "Non-blocking" only holds if your own code is short. CPU-heavy sync work blocks the loop, and with it every frame and touch.
- **Trap**: saying Dart "has threads for async". Async ≠ parallel. Parallelism needs isolates (Q3).
- Deep dive: [[Dart - Async, Streams & Isolates]]

### Q2. How do Future, async/await and Stream work underneath? What's the priority of microtask vs event queue?
- `Future(() {})` / `Future.delayed` schedule on the **event** queue (via `Timer`). `Future.microtask`, `scheduleMicrotask` and `then` callbacks of already-completed futures go to the **microtask** queue.
- `async` functions run synchronously until the first `await`, then return a Future. Continuations after `await` are scheduled as microtasks when the awaited future completes.
- `Stream` = sequence of async events. Delivery is scheduled via microtasks by default (`StreamController(sync: true)` is synchronous).
- **Priority**: all pending microtasks run before the next event. Flood the microtask queue and frames stop rendering.
- **Classic output question**: `print` sync lines first, then microtasks in order, then `Future(...)`/timers.

### Q3. How do Isolates differ from native threads? How do isolates achieve memory isolation and two-way communication?
- Threads share one heap (locks, races). **Isolates have separate heaps and their own event loops**, so there's no shared mutable state, no locks and no data races. Each runs on a VM-managed OS thread.
- Communication is **message passing**: `SendPort`/`ReceivePort`. Data is copied, except immutable objects, `TransferableTypedData` and `Isolate.exit` (hands the result over without a copy).
- **Two-way**: pass the main isolate's `SendPort` when spawning, and the worker replies with its own `SendPort` as the first message. Both sides can then send.
- Isolate groups (Dart 2.15+) make spawning cheap and messaging fast (shared program code and heap infrastructure, but still no shared objects).
- Practical: `Isolate.run()` / Flutter `compute()` for one-off CPU work. On Flutter **web**, isolates aren't supported (`compute` runs on the main thread).

### Q4. Mixin vs abstract class inheritance (extends) vs interface implementation (implements): mechanisms and multiple-inheritance semantics?
| Keyword | Gets implementation? | How many? | Semantics |
|---|---|---|---|
| `extends` | Yes (fields + methods) | One superclass | Classic single inheritance, `super` calls |
| `implements` | **No**: must implement every member | Many | Every class is an implicit interface. Pure contract |
| `with` (mixin) | Yes | Many | **Linearization**: `class C extends A with M1, M2` builds `A → A+M1 → A+M1+M2 → C`. Later mixins override earlier ones, and `super` inside a mixin calls the previous layer |
- `mixin M on Base` restricts M to classes extending `Base` (and lets M call `super` methods of Base).
- Dart 3: `mixin class` (usable as both), and class modifiers (`interface`, `base`, `final`, `sealed`) control extend/implement outside the library.
- Flutter examples: `SingleTickerProviderStateMixin`, `AutomaticKeepAliveClientMixin`, `WidgetsBindingObserver`.
- **Trap**: calling mixins "multiple inheritance". It's ordered composition without the diamond problem, because linearization decides the order.
- Deep dive: [[Dart - Type System, Null Safety & Mixins]]

### Q5. const vs final at compile time vs runtime, and the impact on widget rebuild performance?
- `final`: assigned **once at runtime**, and the object itself may be mutable. `const`: a **compile-time constant**, deeply immutable and **canonicalized**, so identical `const` expressions share one instance.
- Rebuild impact: in `Element.updateChild`, if the new widget is the **identical** instance (`identical(oldWidget, newWidget)`), Flutter skips updating that child entirely, so the whole `const` subtree isn't rebuilt. That's why `const Text('Hi')`, `const SizedBox(height: 8)` and `const` constructors matter in hot rebuild paths.
- Also: `const` constructors need all-final fields, and const instances avoid allocations on every build.
- **Trap**: "final makes widgets faster". Only `const` instances get the identity short-circuit.
- Deep dive: [[Dart - Type System, Null Safety & Mixins]] · [[Flutter - Performance & DevTools]]

### Q6. Dart VM memory management in AOT vs JIT, and generational GC (young/old)?
- **JIT** (debug): Kernel → JIT-compiled at runtime, enabling **hot reload**. Slower start and bigger memory. **AOT** (release): precompiled machine-code snapshot, fast startup, no JIT at runtime. Both use the same GC.
- **Generational GC** (per isolate group):
  - **New space (young)**: bump-pointer allocation plus a **parallel scavenger** (semi-space copying). Very fast for short-lived objects, which suits Flutter's thousands of short-lived widgets per frame.
  - **Old space**: objects that survive scavenges are promoted. Collected by **mark-sweep with concurrent marking** and occasional **compaction**.
- The Flutter engine **schedules GC during idle time** between frames to avoid jank.
- Leaks in Dart aren't "unfreed memory" (GC handles that). They're **unwanted reachability**: something long-lived still references the object (see Q7).
- Deep dive: [[Dart - VM, Compilation & Garbage Collection]]

### Q7. How do you avoid memory leaks with StreamSubscription, and handle multiple subscribers with BroadcastStream?
Full worked answer in [Part C](#part-c--q7-fully-worked-interview-answer) below.

### Q8. How does sound null safety guarantee type safety at compile time and runtime?
- Types are **non-nullable by default**. `T?` is a distinct type (`T` is a subtype of `T?`). The compiler rejects using a `T?` where `T` is needed unless it's **promoted**.
- **Flow analysis** promotes local variables after `if (x != null)`, `x ??= …`, early returns, and pattern matches (`case final v?`).
- **Soundness**: because all code (Dart 3: no mixed mode) is null-safe, a non-nullable static type is **guaranteed** at runtime. The compiler can drop null checks and optimize.
- Runtime checks remain only at explicit escape hatches: `!`, `as`, `late` (throws `LateInitializationError` if read before assignment), and dynamic data (`jsonDecode` → `dynamic`).
- **Trap**: overusing `!`/`late` hides bugs until runtime. Model optionality explicitly instead.
- Deep dive: [[Dart - Type System, Null Safety & Mixins]]

## Part B — Rendering pipeline & core architecture

### Q9. Walk through Flutter's pipeline from build to pixels: Build, Layout, Compositing, Paint.
On each **vsync**, `WidgetsBinding.drawFrame` (via `SchedulerBinding`) runs:
1. **Animate**: tickers/animations advance (transient frame callbacks).
2. **Build**: `BuildOwner.buildScope` rebuilds **dirty Elements** (from `setState`, inherited changes) and updates widget→element→render object configs.
3. **Layout**: `PipelineOwner.flushLayout`. RenderObjects marked `needsLayout` lay out: **constraints go down, sizes go up, the parent sets the position**.
4. **Compositing bits**: `flushCompositingBits` decides which render objects need their own **layer** (opacity, clips, transforms, platform views).
5. **Paint**: `flushPaint` records drawing commands into **layers** (display lists). `RepaintBoundary` isolates repaint regions.
6. **Composite**: `compositeFrame` builds the **layer tree / Scene** and sends it to the engine. Semantics is updated for accessibility.
7. **Rasterize**: on the **raster thread**, the engine (Impeller) turns the layer tree into GPU commands.
- Jank diagnosis: UI-thread time (build/layout/paint in Dart) vs raster-thread time (shaders, saveLayer, big images). DevTools' frame chart shows both.
- Deep dive: [[Flutter - Rendering & Widget Lifecycle]]

### Q10. What are the responsibilities of the Widget, Element and RenderObject trees, and why separate them?
| Tree | Role | Lifetime |
|---|---|---|
| **Widget** | Immutable *configuration*/blueprint (`build` returns new ones constantly) | Very short, so creating them is cheap |
| **Element** | The *instance* in the tree: holds the widget, parent/children, `State`, `BuildContext`, inherited-dependency tracking | Long-lived. Reused across rebuilds |
| **RenderObject** | Layout, paint, hit-testing (`RenderBox`, `RenderParagraph`…) | Long-lived, expensive. Only created by `RenderObjectWidget`s |
- **Why**: declarative UI wants you to rebuild *descriptions* freely, while layout/paint objects are expensive. Elements reconcile the cheap new descriptions against the existing expensive objects, so only the minimal changes reach the render tree. It also separates concerns (state lives in Elements/State, not in widgets).

### Q11. After setState, how does the Element tree diff and decide whether to reuse a RenderObject?
1. `setState` → `markNeedsBuild()` → the element is added to the dirty list → `scheduleFrame()`.
2. On the next frame the dirty element rebuilds (`performRebuild` → `build()`), producing new child widgets.
3. For each child, `Element.updateChild(child, newWidget, slot)`:
   - `identical(old, new)` → nothing to do (the `const` win from Q5).
   - `Widget.canUpdate(old, new)` (same **`runtimeType`** and same **`key`**) → **reuse the Element**, call `update(newWidget)`. For `RenderObjectElement`, `updateRenderObject()` patches the existing RenderObject's properties (which may mark it `needsLayout`/`needsPaint`).
   - Otherwise → deactivate/unmount the old Element (and its RenderObject, and **its State**) and **inflate** a new one.
4. Multi-child lists use `updateChildren`, matching by **keys** (top/bottom scans plus a key map). Without keys, reordering items reuses elements by position, so state attaches to the wrong item. `GlobalKey` enables reparenting (moving a subtree with its state).

### Q12. How does RenderBox layout work: BoxConstraints passed down, sizes reported up?
- A parent calls `child.layout(constraints, parentUsesSize: …)`. `BoxConstraints` = `minWidth ≤ w ≤ maxWidth`, `minHeight ≤ h ≤ maxHeight`.
- The child picks a size **within** the constraints (`performLayout` → `size = constraints.constrain(...)`). The parent then positions the child (sets `parentData.offset`). A child never decides its own position.
- **Tight** (min = max, e.g. `SizedBox.expand`, screen root) vs **loose** (min = 0, e.g. `Center`) vs **unbounded** (`maxHeight = ∞` inside `ListView`/`Column`). Unbounded + "expand" causes the classic `RenderFlex overflowed` / "Vertical viewport was given unbounded height" errors.
- **Relayout boundaries**: if a child's size doesn't depend on its parent (tight constraints, `sizedByParent`, or `parentUsesSize: false`), layout changes don't propagate upward, so performance stays local.
- Intrinsic sizing (`IntrinsicHeight`, `IntrinsicWidth`) does an extra layout pass and can go **O(n²)** in nested cases. Avoid it in lists.

## Part C — Q7 fully worked (interview answer)

### 1. The interviewer's angle
- **Core skill tested**: rigour in managing async resources, and depth of understanding of Stream/RxDart reactive patterns.
- **Hidden check**: lifecycle awareness. Do you understand how the event loop and memory references interact, and can you choose the right stream type for cross-component communication?
- **Grading criteria**: mentions `cancel()`, ties it to the `StatefulWidget` lifecycle (`dispose`), and distinguishes **single-subscription vs broadcast**.
- **Common pitfalls**:
  - Only closing the stream (`Sink`/`StreamController.close()`) and forgetting to cancel the **subscription**.
  - Using streams after `dispose`.
  - Confusing the subscription limits of the two stream types.

### 2. Answer framework
- **Opening**: name the root cause. A long-lived object holds a reference to a short-lived one.
- **Body 1 (avoiding leaks)**: when to `cancel()`, `StreamBuilder`'s automatic management, and tools for many subscriptions.
- **Body 2 (broadcast)**: `asBroadcastStream()` / `StreamController.broadcast()`, their semantics, and when to use them.
- **Close**: best-practice principles.
- **Method**: a causal chain (*why it leaks → how to fix → advanced options*), a contrast (single vs broadcast, showing you understand the API design), and real project experience (BehaviorSubject/StreamBuilder choices, not recited docs).

### 3. Strong answer (corrected & verified)
> In Flutter, a `StreamSubscription` that isn't managed keeps its `onData` closure alive. That closure usually captures the `State` (and through it the `BuildContext`/Element subtree). If the stream comes from a longer-lived object (a repository, singleton service, socket or event bus), the chain `service → controller → subscription → closure → State` keeps the disposed page reachable, so GC can't reclaim it. Its callbacks also keep firing (`setState() called after dispose()`, duplicate handling).
>
> **Avoiding leaks**:
> - The baseline is to keep the subscription and call `subscription.cancel()` in `dispose()`.
> - With several subscriptions on a complex page, I collect them in a `List<StreamSubscription>` (or RxDart's `CompositeSubscription`) and cancel them all in `dispose`. For cancellable one-off async work I use `CancelableOperation` from `package:async`.
> - Better still, I let the framework own the lifecycle: `StreamBuilder` subscribes and cancels automatically, as do Riverpod `StreamProvider.autoDispose` and BLoC. One caveat: create the stream once (not inside `build`), or `StreamBuilder` resubscribes every rebuild.
> - I also keep the two responsibilities apart. `StreamController.close()` is the producer closing the input, done by the controller's owner. `subscription.cancel()` is each consumer stopping listening.
>
> **Broadcast**:
> - A normal stream is single-subscription: a second `listen` throws, and it buffers events until its one listener arrives.
> - When several widgets react to one source (an event bus, connection status), I create a broadcast stream with `StreamController.broadcast()` or convert with `asBroadcastStream()`. For the latter I pass `onCancel` to pause the source when nobody listens.
> - Broadcast streams **don't buffer**: subscribers only get events emitted after they subscribe, and events with no listeners are dropped. So each subscriber subscribes on demand and owns its own `cancel`.
> - For critical state (socket connection state, auth, cart), late subscribers need the current value. I use RxDart's `BehaviorSubject` (replays the last value), or model it as state (`ValueNotifier`, Riverpod/BLoC) instead of events.
>
> Finally, I verify with lints (`cancel_subscriptions`, `close_sinks`) and with `leak_tracker` in widget tests and DevTools' memory view.

### 4. Weak answer and why it fails
> "Memory leaks happen because the stream isn't closed. Just call `stream.close()` when the page closes. For multiple listeners, use a broadcast stream via `asBroadcastStream`. I usually use StreamBuilder; it's easy and I don't need to think about leaks."

- **Concepts confused**: `close()` is on the controller/sink (closing the input), while `cancel()` on the subscription is what stops listening. They're different responsibilities, and there's no `stream.close()`.
- **No depth**: it doesn't explain the root cause (the reference chain from a long-lived object).
- **Too vague**: it doesn't explain broadcast semantics (no replay, events dropped without listeners) or the risks with multiple subscribers.
- **Over-trusts StreamBuilder**: it doesn't mention the resubscribe-per-build trap, or that StreamBuilder doesn't close controllers you own.

> [!warning] Corrections to the source material
> - "cancelable_operation plugin" → it's **`CancelableOperation` in `package:async`** (Dart team), not a standalone plugin. RxDart offers `CompositeSubscription` for grouping.
> - The order is **Layout → Compositing bits → Paint → Composite**. Compositing-bit updates come *before* paint, and the actual compositing (building the scene) comes *after* paint (Q9).
> - `asBroadcastStream()` subscribes to the source lazily, but by default **keeps** the source subscription after all listeners cancel. Use `onCancel` to pause it (not cancel it, which ends the stream forever).

## Related
- [[Dart - Async, Streams & Isolates]] — Q1–Q3, Q7 in depth
- [[Flutter - Rendering & Widget Lifecycle]] — Q9–Q12 in depth
- [[Flutter - State Management]] — framework-owned subscriptions
- [[Dart - Type System, Null Safety & Mixins]] · [[Dart - VM, Compilation & Garbage Collection]] — Q4–Q6, Q8
- [[Flutter - Performance & DevTools]] — const, rebuild scope, jank, leaks
- [[Dart]] · [[Flutter]]

## References
- Flutter architectural overview (rendering pipeline, trees): https://docs.flutter.dev/resources/architectural-overview
- Inside Flutter (build/layout/paint internals): https://docs.flutter.dev/resources/inside-flutter
- Dart concurrency & isolates: https://dart.dev/language/concurrency
- Dart streams: https://dart.dev/libraries/async/using-streams
- `Stream.asBroadcastStream` API: https://api.dart.dev/dart-async/Stream/asBroadcastStream.html
- `package:async` (CancelableOperation, StreamGroup): https://pub.dev/packages/async
- leak_tracker: https://github.com/dart-lang/leak_tracker/blob/main/doc/leak_tracking/DETECT.md
