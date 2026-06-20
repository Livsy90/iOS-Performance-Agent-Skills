# Dispatch and Specialization

Use this reference when the task involves direct dispatch, class dispatch, Objective-C dispatch, witness dispatch, closure dispatch, devirtualization, inlining, or generic specialization.

Use it to decide whether an abstraction creates measurable runtime cost, whether the dynamic behavior is required by the design, and whether optimized builds can inline, devirtualize, or specialize the path.

Do not use this reference to ban protocols, inheritance, closures, or generics. Use it for hot paths where dispatch, specialization, optimizer visibility, or abstraction overhead is a plausible cost.

Prefer `existentials-generics-opaque-types.md` when the main issue is API shape around `any`, `some`, generics, type erasure, or heterogeneous storage.

Prefer `arc-and-ownership.md` when the main issue is retain/release traffic, closure capture lifetime, reference cycles, weak/unowned ownership, or Objective-C bridging lifetime.

Prefer `allocation-and-layout.md` when the main issue is physical storage, object layout, closure boxes, existential containers, or unexpected allocation.

If one of these files does not exist yet, treat the link as intended routing and do not invent details from a missing reference.

## Contents

* [Core model](#core-model)
* [Dispatch categories](#dispatch-categories)
* [Review workflow](#review-workflow)
* [Direct dispatch](#direct-dispatch)
* [Class dispatch and `final`](#class-dispatch-and-final)
* [Protocol witness dispatch](#protocol-witness-dispatch)
* [Existentials and type erasure](#existentials-and-type-erasure)
* [Protocol extension dispatch](#protocol-extension-dispatch)
* [Closure dispatch and captures](#closure-dispatch-and-captures)
* [Objective-C message dispatch](#objective-c-message-dispatch)
* [Generics and specialization](#generics-and-specialization)
* [Inlining and devirtualization](#inlining-and-devirtualization)
* [Module boundaries](#module-boundaries)
* [SIL signs to check](#sil-signs-to-check)
* [Common mistakes](#common-mistakes)
* [Validation](#validation)
* [Output guidance](#output-guidance)

## Core model

Swift dispatch performance questions usually reduce to three checks:

1. Can the compiler know the exact call target?
2. Is dynamic behavior required by the API or architecture?
3. Can optimized builds inline, devirtualize, or specialize the path?

Source syntax is not enough. A call that looks dynamic may be devirtualized. A generic function that looks static may remain unspecialized across a module boundary. A closure that looks small may allocate if it escapes or captures reference-heavy state.

Treat dispatch cost as one possible contributor. In many paths, allocation, ARC, collection work, I/O, formatting, decoding, layout, synchronization, actor hopping, or algorithmic complexity dominates the cost.

Do not optimize dispatch in isolation. Optimize only when the call is repeated enough, appears in a measured path, or blocks a meaningful compiler optimization.

## Dispatch categories

Use these as review categories, not as guaranteed final machine behavior. Optimized SIL is the better place to confirm what survived optimization.

| Category                     | Typical source shape                                                                              | What to verify                                                                   |
| ---------------------------- | ------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| Direct dispatch              | free functions, static methods, concrete value-type methods, `final` methods, devirtualized calls | whether the exact target is known and whether inlining/devirtualization happened |
| Class dispatch               | non-final class methods, overridable members, open/public extension points                        | whether subclassing or overriding is part of the API contract                    |
| Objective-C dispatch         | selectors, target-action, KVO, swizzling, `dynamic`, Objective-C runtime APIs                     | whether Objective-C runtime behavior is required                                 |
| Witness dispatch             | protocol requirement calls                                                                        | whether specialization removes the witness call in the hot path                  |
| Existential dispatch         | calls through `any Protocol` or type-erased wrappers                                              | whether runtime heterogeneity is required                                        |
| Closure dispatch             | stored, escaping, or repeated closures                                                            | whether closure allocation, captures, or lost inlining matter                    |
| Generic specialized dispatch | generic calls with concrete visible types                                                         | whether optimized SIL shows specialization                                       |
| Module-boundary call         | public API, package/framework boundary, resilient library boundary                                | whether implementation visibility limits optimization                            |

These categories can overlap. For example, a generic function can call a protocol requirement, a closure can capture an existential, and a public generic helper can fail to specialize across a module boundary.

## Review workflow

1. Identify the hot path.
2. Classify the call shape.
3. Ask whether the dynamic behavior is intentional.
4. Check whether the implementation is visible to the optimizer.
5. Check whether the call is repeated enough to matter.
6. Check whether dispatch is actually significant compared with the work inside the callee.
7. Prefer the smallest change that preserves the design.
8. Validate with profiling, benchmark, or optimized SIL.

Do not rewrite abstractions only because they have a runtime mechanism. Rewrite them only when the abstraction is unnecessary for the path, the path is hot, and the change gives the optimizer useful information.

## Direct dispatch

Direct dispatch is possible when the compiler can resolve the target without runtime lookup.

Common cases:

* free functions with visible implementations;
* functions on concrete value types;
* `final` class methods;
* static methods;
* private functions and internal helpers whose concrete implementation is visible to the optimizer;
* non-requirement protocol extension methods called through a statically known type;
* calls devirtualized during optimization.

```swift id="h3a1bu"
struct ChecksumCombiner {
    func combine(_ lhs: UInt64, _ rhs: UInt64) -> UInt64 {
        lhs &* 31 &+ rhs
    }
}
```

Review questions:

* Is the concrete implementation known at the use site?
* Is the implementation visible in the same module or safely exposed for optimization?
* Is the function small enough to inline profitably?
* Is this call inside a hot loop or repeated path?
* Does the optimized output still contain a dynamic call?

Visibility helps the optimizer, but it does not guarantee inlining or devirtualization. Check optimized SIL when the performance claim depends on it.

Do not force direct dispatch if runtime polymorphism is part of the design or the path is not performance-sensitive.

## Class dispatch and `final`

Non-final class methods may require dynamic dispatch because subclasses can override them.

Use non-final classes when inheritance is part of the contract:

* UIKit/AppKit subclassing;
* framework extension points;
* public or `open` APIs designed for subclassing;
* test seams that intentionally use subclassing;
* domain models that genuinely require polymorphic class behavior;
* Objective-C interoperability that requires dynamic behavior.

Prefer `final` for app-level classes that are not designed for inheritance.

```swift id="v1eq4s"
final class ReceiptFormatter {
    func title(for receipt: Receipt) -> String {
        "\(receipt.storeName) · \(receipt.total)"
    }
}
```

Review rules:

* Do not require a measured hot path just to mark a non-inheritable app class as `final`.
* Require evidence when presenting `final` as a performance optimization.
* Do not flatten a useful hierarchy only because dynamic dispatch exists.
* Consider concrete final implementations internally while preserving public abstraction at module boundaries.
* Do not mark framework extension points `final` when subclassing is part of the API contract.

`final` is first an API and inheritance decision. It can also help optimization by proving that overriding cannot occur, but do not present it as a guaranteed speedup without evidence.

## Protocol witness dispatch

A protocol requirement call can use witness table dispatch. The witness table maps each protocol requirement to the implementation for a concrete conforming type.

```swift id="6yv5mq"
protocol EventEncoder {
    func encode(_ event: AnalyticsEvent) -> Data
}

func encodeAll<E: EventEncoder>(
    _ events: [AnalyticsEvent],
    using encoder: E
) -> [Data] {
    events.map { encoder.encode($0) }
}
```

A generic call to a protocol requirement may be represented through witness information unless the optimizer specializes the generic function for a concrete conforming type and removes the indirect call.

Review questions:

* Is the requirement called in a hot loop?
* Is the concrete conforming type known at the call site?
* Is the generic function visible to the optimizer?
* Does optimized SIL show a specialized version?
* Does the witness call remain in the hot path?
* Would a concrete helper, generic overload, or specialized fast path preserve the design while reducing runtime cost?

Do not assume every protocol call remains dynamic in final optimized code. Also do not assume generic code always becomes direct or specialized.

## Existentials and type erasure

An existential such as `any EventEncoder` hides the concrete type at that boundary. Calls through the existential may require opening the existential and dispatching through witness information.

Use an existential when:

* values of different conforming types must be stored together;
* the concrete type is selected at runtime;
* the API boundary intentionally hides implementation choices;
* type erasure simplifies ownership or architecture enough to justify the cost.

Prefer a generic or concrete path when:

* the call path is homogeneous;
* the same concrete type is used repeatedly in a hot loop;
* specialization and inlining materially affect the measured path;
* the existential was introduced only to avoid spelling a generic parameter.

Do not automatically replace `any` with generics. First check whether runtime heterogeneity is the design.

This section focuses on dispatch and specialization. For existential container storage, boxing, and type-erasure object layout, use the existential/storage reference if available.

## Protocol extension dispatch

Protocol extension methods that are not protocol requirements are dispatched based on the static type.

```swift id="xbtm5x"
protocol BadgeProvider {
    var badge: String { get }
}

extension BadgeProvider {
    func decoratedBadge() -> String {
        "[\(badge)]"
    }
}
```

A conforming type can define a method with the same name, but that does not make it an override unless the method is a protocol requirement.

A protocol extension can also provide a default implementation for a protocol requirement. In that case, calls through the protocol requirement use the witness chosen for the conformance. The static-dispatch pitfall applies to extension-only methods that are not protocol requirements.

Review rules:

* If conforming types must customize the behavior dynamically, make the method a protocol requirement.
* If static extension behavior is intended, keep it as an extension-only helper.
* Treat this primarily as a correctness and API-design issue, not only a performance issue.

## Closure dispatch and captures

A closure can be cheap, but it is not automatically free.

Check whether the closure:

* escapes;
* captures `self` or a large object graph;
* is created repeatedly inside a hot path;
* prevents inlining of a tiny operation;
* stores type-erased behavior long term;
* introduces ARC traffic through captured references.

If a hot closure captures a large owner only to use one dependency, consider narrowing the capture:

```swift id="xc6bn1"
func ranked(_ items: [FeedItem]) -> [FeedItem] {
    let scorer = scorer

    return items.sorted {
        scorer.score($0) > scorer.score($1)
    }
}
```

This does not guarantee a win. It may reduce captured lifetime and ARC traffic when the original closure retained more state than needed.

Do not replace closures with types by default. Closures are often the simplest and fastest representation when they are non-escaping and optimized well.

If closure allocation, capture contexts, or retained object graphs are the central issue, route to `arc-and-ownership.md` or `allocation-and-layout.md`.

## Objective-C message dispatch

Objective-C interoperability can require a more dynamic dispatch model.

Common cases:

* selectors and target-action;
* KVO-compatible members;
* Objective-C protocols;
* `NSObject` subclass boundaries;
* method swizzling;
* APIs marked `dynamic`;
* APIs intentionally exposed with `@objc`;
* Objective-C runtime APIs.

Be precise:

* `@objc` exposes a declaration to Objective-C.
* `dynamic` forces dynamic dispatch through a runtime mechanism.
* Selectors, KVO, swizzling, target-action, and Objective-C runtime APIs require dynamic behavior.
* A Swift-to-Swift call involving an `@objc` member is not automatically a performance problem.
* Do not assume `@objc` alone means every Swift call must go through Objective-C message dispatch. Check the actual call path and optimized output when the claim matters.

Keep dynamic interop boundaries small when useful:

```swift id="2n5y5t"
final class SearchButtonHandler: NSObject {
    @objc func didTapSearchButton(_ sender: Any) {
        submitSearch()
    }

    private func submitSearch() {}
}
```

Do not remove Objective-C interop from required UIKit/AppKit or framework boundaries.

## Generics and specialization

Generics let code be written once while allowing the optimizer to produce concrete specialized versions when it has enough visibility and decides the specialization is profitable.

Specialization is not guaranteed. It depends on:

* optimization level;
* concrete type visibility;
* module boundaries;
* generic function visibility;
* code size heuristics;
* whether closures or type-erased wrappers hide the implementation;
* whether the optimizer considers specialization profitable.

Review questions:

* Is the generic function called from a hot path?
* Is it in the same module as its hot call sites?
* Is it public across a module boundary?
* Is it called with a small number of concrete types?
* Does specialization create unacceptable code size growth?
* Does optimized SIL show specialized versions?

Do not say “generic means static dispatch” unless specialization was confirmed or the performance claim does not depend on it.

## Inlining and devirtualization

Inlining replaces a call with the function body. Devirtualization converts a dynamic call into a direct call when the compiler can prove the target.

Inlining can help by removing call overhead and exposing constant propagation, ARC elimination, specialization, and simplification.

Inlining can hurt by increasing code size, harming instruction-cache behavior, increasing compile time, or duplicating large cold code into multiple call sites.

Review rules:

* Do not add `@inline(__always)` as a default fix.
* Consider `@inline(never)` only for specific measurement, debugging, or code-size reasons, not as a general cold-code annotation.
* Treat inlining attributes as evidence-driven tools, not style preferences.
* Prefer making the code easier for the optimizer to reason about before forcing attributes.

## Module boundaries

Module boundaries can limit inlining, devirtualization, and specialization.

Check this when:

* a hot generic function lives in a separate module;
* a small hot helper is public only because of package structure;
* a protocol-heavy API crosses framework boundaries;
* the implementation is hidden from the client optimizer;
* the code uses `@inlinable`, `@usableFromInline`, or `@frozen`;
* library evolution mode or ABI resilience is relevant.

Guidance:

* Prefer keeping hot implementation details internal when possible.
* Use `@inlinable` only for small, stable, performance-critical public APIs whose body can safely become part of the module interface.
* Use `@usableFromInline` only when inlinable code needs ABI-visible implementation details.
* Use `@frozen` only for public structs or enums in library-evolution contexts when you are willing to commit to layout-related ABI constraints. It is not a local performance decoration for app code.
* Do not add ABI attributes only because a function is slow.

First identify the module-boundary problem. Then choose whether to internalize, move code, add a specialized overload, expose an inlinable body, or accept the abstraction cost.

## SIL signs to check

Use optimized SIL when source-level reasoning is not enough.

Useful signs may include:

* `function_ref` — direct function reference;
* `class_method` — class method dispatch;
* `objc_method` or Objective-C thunk patterns — Objective-C dispatch boundary;
* `witness_method` — protocol witness dispatch;
* `open_existential_*` — existential opening;
* `partial_apply` — closure creation or capture context;
* `alloc_ref` / `alloc_box` — allocation or boxed mutable capture;
* `strong_retain` / `strong_release` — ARC traffic;
* `retain_value` / `release_value` — ownership traffic;
* specialized function names or generated specialized variants — evidence that specialization happened.

Use SIL to answer a narrow question:

* Did the witness call remain?
* Was the class call devirtualized?
* Did the generic function specialize?
* Did closure creation remain?
* Did an existential boundary remain in the hot loop?
* Did ARC traffic change around the call path?

Guardrails:

* Inspect optimized builds, not Debug SIL.
* Treat SIL instruction names as implementation-level evidence, not stable API.
* Exact lowering can change across compiler versions, optimization levels, build settings, language modes, ownership modes, and target platforms.
* Do not use SIL as a replacement for profiling.
* Use SIL to verify a specific hypothesis.
* Prefer `references/sil-inspection.md` for deeper SIL workflows if that file exists.

## Common mistakes

* Treating every protocol call as a performance bug.
* Saying generics are always statically dispatched without checking specialization.
* Replacing useful runtime polymorphism with concrete code in cold paths.
* Using `some Protocol` where runtime heterogeneity is required.
* Adding `@inline(__always)` before measuring or inspecting optimized output.
* Using `@inlinable`, `@usableFromInline`, or `@frozen` as local app-level performance decorations.
* Marking framework extension points `final` when subclassing is part of the API contract.
* Removing `@objc` or `dynamic` from code that relies on selectors, KVO, swizzling, or framework integration.
* Confusing protocol extension-only methods with protocol requirements that have default implementations.
* Ignoring closure captures while focusing only on method dispatch.
* Assuming dispatch matters more than the work inside the called function.
* Treating SIL patterns as proof without a hot path or validation.

## Validation

Use one or more validation paths depending on the claim.

For runtime impact:

* Time Profiler for hot call stacks;
* Allocations for closure contexts, boxes, wrappers, and type-erased storage;
* XCTest performance tests or microbenchmarks for stable hot loops;
* before/after traces with the same input and build settings.

For optimizer behavior:

* inspect optimized SIL;
* check whether witness calls remain;
* check whether class calls were devirtualized;
* check whether Objective-C dispatch remains on the path;
* check whether generic functions specialized;
* check whether closure creation remains;
* check whether ARC traffic changed around the call path.

For design safety:

* confirm that inheritance, Objective-C runtime behavior, or runtime heterogeneity was not part of the required API contract;
* confirm that public ABI attributes do not overcommit the API;
* confirm that code size and maintainability did not regress.

When the path is not hot, say so. Prefer preserving clarity over low-level dispatch rewrites.

## Output guidance

When this reference is used, include:

1. The call shape: direct, class, Objective-C, witness, existential, closure, generic, or module-boundary call.
2. Why dynamic behavior may or may not be required.
3. Whether the path is hot enough for dispatch or specialization to matter.
4. What optimized SIL, profiling, or benchmark evidence should confirm.
5. The smallest safe change.
6. Trade-offs in API flexibility, binary compatibility, code size, testability, readability, ownership, or runtime heterogeneity.

Use cautious language when evidence is incomplete:

* "This may remain dynamically dispatched unless the optimizer devirtualizes it."
* "This is only worth changing if the call is in a hot path."
* "A generic version helps only if the concrete type is visible and specialization happens."
* "This existential may be the correct design if runtime heterogeneity is required."
* "Do not add ABI attributes unless the module-boundary trade-off is intentional."
