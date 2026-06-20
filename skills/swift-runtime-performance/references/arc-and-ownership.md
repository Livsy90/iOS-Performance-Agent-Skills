# ARC and Ownership

Use this reference when the task involves retain/release traffic, closure captures, weak or unowned references, object lifetime, reference cycles, copy-on-write ownership, Objective-C bridging lifetime, or ownership-sensitive performance issues.

The goal is not to remove ARC from Swift code. The goal is to identify whether ownership is causing a correctness issue, memory growth, or measured runtime cost, then recommend the smallest safe change that preserves the intended lifetime model.

Do not use this reference for general memory optimization when there is no ownership, lifetime, leak, retain/release, closure capture, copy-on-write, or bridging signal. If the issue is only that the code "looks reference-heavy", treat it as a hypothesis and ask for evidence.

## Contents

* [Core model](#core-model)
* [When to use this reference](#when-to-use-this-reference)
* [What to inspect first](#what-to-inspect-first)
* [Ownership semantics](#ownership-semantics)
* [Closure captures and cycles](#closure-captures-and-cycles)
* [ARC traffic in hot paths](#arc-traffic-in-hot-paths)
* [Value types, COW, and hidden ownership](#value-types-cow-and-hidden-ownership)
* [Objective-C bridging and autorelease behavior](#objective-c-bridging-and-autorelease-behavior)
* [Observed object lifetime](#observed-object-lifetime)
* [Evidence to check](#evidence-to-check)
* [Decision rules](#decision-rules)
* [Common refactor patterns](#common-refactor-patterns)
* [Validation](#validation)
* [Output guidance](#output-guidance)

## Core model

ARC manages the lifetime of reference-counted objects.

ARC-related cost can appear through:

* references to class and actor instances;
* closure capture contexts;
* reference-backed standard library storage;
* type-erased wrappers that store references;
* Objective-C and Core Foundation bridging;
* values that contain copy-on-write storage.

Value types are not reference-counted as values, but their stored properties can carry ownership cost. `String`, `Array`, `Dictionary`, `Set`, and `Data` may use reference-backed storage. A `struct` can therefore participate in ARC traffic even when the source has no explicit class reference.

Use this model:

1. Ownership defines lifetime.
2. ARC implements that lifetime.
3. ARC traffic matters when it affects correctness, memory growth, hot-path CPU work, locality, or responsiveness.
4. Ownership changes are semantic changes first and performance changes second.

Avoid simple rules such as:

* ARC is always the bottleneck.
* Value types have no ownership cost.
* `weak` is faster or safer by default.
* `unowned` is a performance optimization.
* `[weak self]` belongs in every closure.
* Replacing classes with structs automatically removes ownership cost.
* Exact `deinit` timing is a stable correctness mechanism.
* Every visible retain or release in compiler output is a bug.

## When to use this reference

Use this file for questions about:

* retain/release traffic in a hot path;
* closure capture lists and captured object lifetime;
* strong reference cycles;
* `weak`, `unowned`, or `unowned(unsafe)`;
* task, timer, subscription, observer, or callback ownership;
* memory growth caused by retained object graphs;
* copy-on-write storage and value types that hide references;
* Foundation, Objective-C, or Core Foundation bridging lifetime;
* optimized SIL signs of ARC traffic.

Prefer another reference when the main issue is different:

* `allocation-and-layout.md` — storage location, object layout, closure boxes, existential storage, or unexpected heap allocation.
* `dispatch-and-specialization.md` — dynamic dispatch, witness dispatch, inlining, or specialization.
* `existentials-generics-opaque-types.md` — `any`, `some`, generics, type erasure, or heterogeneous collections.
* `cow-and-large-values.md` — large-value copying, mutation strategy, collection buffers, or custom COW implementation.
* `unsafe-swift.md` — pointers, `Unmanaged`, memory binding, aliasing, or unsafe lifetime boundaries.

If one of these files does not exist yet, treat the link as intended routing and do not invent details from a missing reference.

## What to inspect first

Before proposing ownership changes, identify:

1. Is the issue correctness, memory growth, CPU cost, or only a theoretical concern?
2. Which object or value is retained longer than intended?
3. Who owns it intentionally?
4. Is there a strong reference cycle?
5. Is the closure escaping, stored, or long-lived?
6. Does the closure need the whole owner or only a dependency?
7. Is ARC traffic visible in Time Profiler, Allocations, Memory Graph, Leaks, or optimized SIL?
8. Is COW storage copied, retained, or mutated repeatedly?
9. Is Objective-C bridging, Core Foundation ownership, or autorelease behavior involved?
10. Would changing ownership alter program behavior?
11. How will the change be validated?

Do not change ownership only because code looks reference-heavy. Runtime ownership advice is useful only when it preserves semantics and addresses a real lifetime or performance signal.

## Ownership semantics

### Strong ownership

A strong reference keeps an object alive. Use strong ownership when the current object requires another object to function correctly.

Prefer strong references for required dependencies, direct ownership relationships, state that is part of the owner invariant, and objects that must not disappear while the owner is alive.

Do not weaken required dependencies to reduce retain counts. That turns a clear invariant into optional state and can introduce silent failures.

### Weak ownership

A weak reference does not keep an object alive and is automatically set to `nil` when the referenced object is deallocated.

Because ARC must be able to set a weak reference to `nil`, a weak reference is normally declared as an optional `var`, not as a `let`.

Use `weak` when:

* the referenced object may disappear independently;
* `nil` is a valid state;
* the relationship is observational or back-pointing;
* the relationship would otherwise create a cycle;
* a callback may outlive the object and should not keep it alive.

Common examples include delegates, parent pointers, coordinator back references, observer relationships, and callbacks where the owner may disappear before the callback fires.

Recommend `weak` for lifetime semantics, not for speed. A weak reference introduces optionality and a different ownership model; it is not a harmless performance annotation.

### Unowned ownership

An unowned reference does not keep an object alive and is expected to refer to a valid object whenever it is accessed.

Unlike `weak`, an unowned reference is not automatically set to `nil` when the referenced object is deallocated. Accessing an unowned reference after the referenced object has been deallocated is a programmer error and can trap at runtime.

For normal non-optional `unowned` references, use them only when:

* the referenced object is guaranteed to outlive the reference;
* `nil` is not meaningful;
* optional access would misrepresent the model;
* the lifetime relationship is simple and easy to prove.

This is valid only if the child can never outlive the parent. If the child can be detached, cached, moved, stored globally, retained by a task, or used after the parent is gone, use a different ownership model.

Optional `unowned` references exist, but they are rare and still require explicit lifetime guarantees because ARC does not zero them like `weak`.

Avoid `unowned(unsafe)` unless the code is a carefully isolated low-level boundary with documented lifetime guarantees, tests, and a reason why safe ownership forms are insufficient.

## Closure captures and cycles

Closure capture lists define how referenced values are captured and how long those values may stay alive.

First ask whether the closure is:

* non-escaping;
* escaping but short-lived;
* stored by the owner;
* stored by another object retained by the owner;
* retained by a task, timer, display link, subscription, observer, or callback;
* executed after cancellation, dismissal, teardown, or owner deallocation.

Use `[weak self]` when a closure may outlive `self` and should not keep `self` alive.

Do not use `[weak self]` reflexively when:

* the closure is non-escaping;
* the closure is executed immediately;
* the closure should keep the object alive until completion;
* silently dropping the work would be incorrect;
* there is no cycle or unwanted lifetime-extension risk.

Prefer narrow captures when the closure only needs dependencies, not owner identity. For example, capture `encoder` and `store` instead of the entire exporter when the closure does not need exporter identity.

Common cycle chains:

```text
Owner -> stored closure -> captured owner
Owner -> subscription token -> subscription closure -> captured owner
Owner -> task handle -> task closure -> captured owner
Parent -> child -> delegate/closure -> parent
Timer/display link -> closure/target -> owner -> timer/display link
Notification/observer token -> callback -> owner -> token
```

Break the smallest correct edge:

* make a back reference `weak`;
* capture `self` weakly when the closure should not keep the owner alive;
* capture a dependency instead of `self`;
* invalidate a timer, display link, observer, subscription, or callback;
* cancel a long-lived task;
* clear a stored closure;
* move callback ownership to an object with a clearer lifetime.

Do not break cycles by weakening required dependencies. Fix the relationship that is actually cyclic.

## ARC traffic in hot paths

ARC traffic becomes important when retain/release operations are frequent enough to affect a measured hot path.

Common sources:

* arrays of class instances processed in tight loops;
* repeated creation of short-lived reference wrappers;
* closure creation inside loops;
* escaping closures created during scrolling, parsing, rendering, import, or search;
* type-erased wrappers around reference-heavy objects;
* repeated bridging between Swift and Objective-C types;
* reference-backed value types copied through many layers;
* generic or existential abstraction that prevents optimization from removing ownership traffic.

An array of reference boxes in a tight loop can be fine. If the loop is extremely hot and profiling shows ARC or pointer-chasing cost, a compact value representation may improve locality and reduce ownership traffic. Do not make this rewrite unless the representation still matches the domain and the cost matters.

Treat ARC traffic as one possible cost among others. In many real performance problems, algorithmic work, allocation churn, cache locality, synchronization, actor hopping, I/O, or UI invalidation will dominate retain/release cost.

## Value types, COW, and hidden ownership

A value type can carry ARC cost through its fields.

Passing or assigning a value such as `RenderCommand(title: String, payload: Data, metadata: [String: String])` copies the value container, but the fields may share reference-backed storage. Passing many such values across layers can involve retain/release traffic even without explicit classes.

Review questions:

* Which fields are reference-backed?
* Is the value copied frequently?
* Is mutation performed after copying?
* Are large buffers retained longer than needed?
* Would scoping, borrowing, streaming, or in-place mutation reduce ownership pressure?
* Should deeper COW behavior be reviewed in `cow-and-large-values.md`?

Copy-on-write is neither free nor automatically expensive. It is usually cheap while storage is shared, but mutation may require uniqueness checks, new buffers, or element-level work.

Do not replace a value type with a class only to avoid copying unless the ownership and mutation model truly wants identity. Do not replace a class with a value type only to "remove ARC" if the value still contains reference-backed storage or if copying would become more expensive.

## Objective-C bridging and autorelease behavior

Swift code that crosses Objective-C, Foundation, or Core Foundation boundaries may introduce ownership behavior that is not obvious from pure Swift source.

Watch for:

* repeated bridging between Swift and Objective-C collection or string types;
* Foundation APIs returning autoreleased objects;
* Objective-C APIs that copy or retain blocks;
* callbacks that retain blocks longer than expected;
* Core Foundation create/copy/get ownership conventions;
* `Unmanaged` usage;
* back-and-forth conversion inside inner loops.

A Foundation boundary can be fine. If called repeatedly in a Swift hot path, check whether bridging, autorelease behavior, or Foundation dispatch contributes to the cost.

Prefer:

* keeping bridging at API boundaries;
* avoiding back-and-forth conversion inside inner loops;
* moving conversion outside repeated work when possible;
* using `autoreleasepool {}` only when autoreleased temporaries are the measured cause;
* reviewing Core Foundation ownership rules explicitly.

Do not add `autoreleasepool {}` as a generic optimization. Use it when profiling or memory behavior shows autoreleased temporaries accumulating in a loop or long-running operation.

## Observed object lifetime

Do not rely on the exact moment when an object is destroyed unless the language or API explicitly guarantees the lifetime.

The compiler may shorten or extend lifetimes as long as program semantics are preserved. Optimization level and code shape can change when `deinit` runs.

Avoid correctness patterns that depend on incidental deinitialization timing.

If cleanup timing matters, prefer an explicit lifetime scope such as:

* `defer`;
* explicit `invalidate()` or `close()`;
* a scoped helper such as `withTemporaryFile`;
* a well-defined owner that performs cleanup at a known point.

Use `withExtendedLifetime` only when code must guarantee that a value remains alive through a specific synchronous operation and the lifetime is otherwise invisible to Swift, usually at unsafe, C, or Objective-C interop boundaries.

Do not use `withExtendedLifetime` as a general ownership-management tool. It ensures the value is not destroyed before the closure returns; it does not create an asynchronous ownership model and does not replace explicit ownership for callbacks, tasks, timers, observers, or long-lived resources.

## Evidence to check

Use evidence before recommending ownership rewrites.

In optimized SIL, ARC-related signs may include:

* `strong_retain` / `strong_release`;
* `retain_value` / `release_value`;
* `copy_value` / `destroy_value`;
* `partial_apply`;
* `alloc_ref` / `alloc_box`;
* `load_weak` / `store_weak`;
* `strong_copy_unowned_value`;
* `ref_to_unowned` / `unowned_to_ref`.

Treat SIL names as implementation-level signals, not stable API. They are useful for investigation, but exact instruction names and optimization output can vary by compiler version, optimization level, ownership mode, and build settings.

Use SIL as supporting evidence. Do not treat every visible retain or release as a bug. The question is whether ownership traffic remains in optimized output and matters in a measured path.

In Time Profiler, look for retain/release functions near the hot path, closure allocation or destruction around repeated operations, Objective-C retain/release activity, type-erased wrapper churn, and object-heavy loops with poor locality.

In Allocations, look for repeated short-lived objects, closure context allocation, retained object graphs, unexpected Foundation objects, temporary buffers, and spikes during scrolling, parsing, rendering, search, import, or background processing.

In Memory Graph or Leaks, look for closure cycles, delegate cycles, long-lived tasks retaining owners, subscription/token cycles, timers or display links retaining owners, caches without eviction, and observers not removed or invalidated.

## Decision rules

### If you see `[weak self]`

Ask:

* Is the closure escaping?
* Is it stored by `self` or by something retained by `self`?
* Can it outlive `self`?
* Is silently skipping work correct if `self` is gone?
* Would capturing a specific dependency better express the lifetime?

Use `[weak self]` when it prevents an unwanted lifetime extension or reference cycle. Do not use it as a universal closure style.

### If you see `[unowned self]`

Ask:

* Can the closure outlive `self`?
* Is the lifetime invariant obvious and documented?
* Can the closure be retained by a task, timer, observer, subscription, or callback?
* Is a crash acceptable if the invariant is violated?
* Is `weak` more honest about the relationship?

Use `[unowned self]` only when the lifetime guarantee is strong. Do not use it to avoid optional handling.

### If you see a stored closure

Ask:

* Who owns the closure?
* What does it capture?
* Can it capture its owner?
* Is there an explicit invalidation or clearing path?
* Does it need the whole owner or only a dependency?

### If you see a task stored by an object

Ask:

* Does the task closure capture the same object strongly?
* Can the task run longer than the owner’s intended lifetime?
* Is cancellation explicit?
* Is capturing a service, value, or ID enough?
* Does the broader issue belong in `swift-concurrency-performance`?

Do not assume every task capture is a leak. A task may intentionally keep work alive. The problem is an unintended lifetime extension, missing cancellation, or retained object graph.

### If you see a memory leak

Ask whether there is:

* a strong cycle;
* a task, subscription, timer, display link, observer, or callback keeping the object alive;
* a cache without eviction;
* a stored closure capturing `self`;
* Objective-C or Core Foundation ownership mismatch;
* cleanup tied to unreliable `deinit` timing.

## Common refactor patterns

Prefer these when they match the ownership model:

* Capture a dependency instead of `self`.
* Make back references `weak` when `nil` is valid.
* Use `unowned` only for local, provable lifetime invariants.
* Add explicit invalidation for timers, observers, display links, subscriptions, and callbacks.
* Cancel long-lived tasks when their owner or scenario ends.
* Clear stored closures when the owner no longer needs them.
* Keep bridging at boundaries instead of repeatedly converting in loops.
* Use explicit lifetime scopes for resources.
* Replace reference-heavy representations in hot data paths only when the value model still fits.
* Reduce repeated closure allocation in hot paths.
* Inspect optimized SIL when source-level reasoning is ambiguous.

Avoid:

* using `weak` everywhere;
* using `unowned` to avoid optionals;
* using `[weak self]` reflexively;
* relying on `deinit` timing for important side effects;
* replacing all classes with structs;
* rewriting clear ownership for theoretical ARC savings;
* ignoring COW storage inside value types;
* ignoring Objective-C bridging in Foundation-heavy paths;
* treating every retain/release in SIL as a bug;
* adding `autoreleasepool {}` without evidence of autorelease pressure;
* using `withExtendedLifetime` to paper over unclear ownership.

## Validation

For leaks or retained object graphs:

* Use Xcode Memory Graph or Leaks.
* Add a targeted lifecycle test with weak references.
* Verify observers, subscriptions, timers, display links, callbacks, and tasks are invalidated or cancelled.

For ARC traffic or closure churn:

* Measure an optimized build.
* Use Time Profiler around the suspected hot path.
* Use Allocations to check short-lived objects and closure contexts.
* Inspect optimized SIL if profiling points to abstraction overhead.

For bridging or autorelease pressure:

* Use Allocations to identify Foundation temporaries.
* Use `autoreleasepool {}` only when autoreleased temporaries are the measured cause.
* Move conversion outside inner loops and remeasure.

For COW ownership pressure:

* Check whether copies are followed by mutation.
* Check buffer growth and temporary allocation behavior.
* Compare before/after with representative data sizes.

Do not call an ownership change successful without a before/after signal.

## Output guidance

When this reference is used, include:

```markdown
## Ownership model

Describe who owns what and which lifetimes are intentional.

## Suspected ARC/lifetime issue

Identify retain cycle, lifetime extension, ARC traffic, weak/unowned misuse, COW storage, bridging, or closure capture.

## Why it matters

Tie the issue to correctness, memory growth, CPU overhead, memory pressure, or user-visible performance.

## Recommended change

Suggest the smallest ownership change that preserves semantics.

## Trade-offs

Explain whether the change affects optionality, lifetime guarantees, readability, safety, or API design.

## Validation

Recommend Memory Graph, Leaks, Allocations, Time Profiler, optimized SIL, or a targeted lifetime test.
```

If ARC traffic is only theoretical and there is no hot path, memory issue, or lifetime bug, say so directly and avoid low-level rewrites.
