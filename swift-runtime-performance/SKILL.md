---
name: swift-runtime-performance
description: Use this skill when reviewing Swift code for runtime-level performance costs, including heap allocation, ARC traffic, stack vs heap storage, closure capture contexts, method dispatch, protocol witness dispatch, existentials vs generics, opaque types, copy-on-write, SIL optimizer output, unsafe memory boundaries, or module-boundary optimizer visibility. Do not use it for Swift Concurrency scheduling, SwiftUI rendering, app launch, or profiling workflows unless the question is specifically about Swift runtime costs.
---

# Swift Runtime Performance

## Purpose

Use this skill to review Swift code for runtime-level performance costs without turning every abstraction into a problem. Focus on concrete costs such as allocation, ARC traffic, dispatch, specialization, copying, optimizer visibility, and unsafe memory boundaries.

This skill should help the agent distinguish real hot-path runtime costs from theoretical micro-optimizations.

## When to use this skill

Use this skill when the task involves Swift runtime behavior such as:

- heap allocation, stack vs heap storage, object layout, boxed values, or closure contexts;
- ARC retain/release traffic, ownership, lifetime, weak/unowned references, or closure captures;
- direct dispatch, class dispatch, Objective-C dispatch, witness dispatch, or dynamic dispatch in hot paths;
- `any Protocol`, `some Protocol`, generics, type erasure, specialization, or unspecialized hot code;
- copy-on-write collections, large values, custom COW storage, or repeated copies;
- optimized SIL inspection, compiler optimization, inlining, devirtualization, or specialization evidence;
- unsafe Swift, pointer lifetime, memory binding, aliasing, buffer mutation, or safe wrappers around unsafe regions;
- module boundaries that affect optimizer visibility, `@inlinable`, `@usableFromInline`, `@frozen`, or public API resilience trade-offs.

## When not to use this skill

Do not use this skill for:

- general Swift syntax or API usage questions with no runtime performance concern;
- app launch investigations where the main issue is pre-main, dyld, static initializers, SDK startup, first frame, or first interaction;
- SwiftUI performance issues where the main issue is identity, invalidation, state scope, layout, drawing, animation, or scrolling;
- Swift Concurrency issues where the main issue is task lifetime, actor isolation, MainActor responsiveness, cancellation, AsyncSequence cleanup, reentrancy, or executor behavior;
- profiling workflow questions where the main task is choosing tools, interpreting traces, designing signposts, XCTest metrics, MetricKit, or production signals;
- broad architecture questions unless there is a specific runtime cost in a hot path.

If another skill is more specific, route there first and use this skill only for the runtime subproblem.

## Neighbor skill boundaries

Use these boundaries before applying runtime advice:

- Use `swift-concurrency-performance` when the task centers on actors, tasks, MainActor, cancellation, AsyncSequence, continuations, task groups, executor behavior, or responsiveness under async work.
- Use `ios-launch-performance` when the task centers on cold launch, warm launch, pre-main, dyld, framework loading, static initializers, AppDelegate, SwiftUI `App`, first frame, first interaction, or launch metrics.
- Use `swiftui-performance` when the task centers on SwiftUI body evaluation, invalidation, identity, state ownership, dependency scope, `List`, `LazyVStack`, layout, drawing, animation, or lifecycle work in views.
- Use `ios-performance-profiling` when the task centers on choosing Instruments templates, interpreting traces, XCTest metrics, MetricKit, signposts, memory graphs, hangs, hitches, CPU, allocations, disk I/O, networking, or production telemetry.
- Use this skill when a neighboring task reveals a Swift runtime-level cost such as allocation churn, ARC traffic, existential boxing, witness dispatch, unspecialized generics, repeated COW copies, or unsafe memory boundaries.

## Core principle

Do not optimize based only on how the source code looks.

First identify:

1. whether the code is on a hot path;
2. what runtime cost is suspected;
3. whether the cost is visible in measurement, compiler output, or a small benchmark;
4. whether the proposed change preserves semantics and improves the measured path;
5. what trade-off the change introduces.

A runtime optimization is useful only when it reduces cost in a path that matters.

## Runtime cost taxonomy

Classify the suspected issue before recommending a change.

Use these categories:

- **Allocation** — heap objects, boxes, closure contexts, existential containers, temporary objects, intermediate collections.
- **ARC** — retain/release traffic, closure captures, weak/unowned access, bridged object lifetime, reference-backed value storage.
- **Dispatch** — dynamic dispatch, witness dispatch, Objective-C dispatch, closure calls, missed devirtualization.
- **Existentials and generics** — `any`, `some`, type erasure, generic specialization, unspecialized hot paths, boxing.
- **Copying** — COW storage, large values, repeated collection mutation, defensive copies, bridging copies.
- **Compiler optimization** — inlining, specialization, devirtualization, module visibility, resilience boundaries, optimized SIL output.
- **Unsafe boundary** — pointer lifetime, binding, alignment, aliasing, mutation, escaping buffers, safe wrappers.
- **Module boundary** — public API visibility, `@inlinable`, `@usableFromInline`, `@frozen`, ABI resilience, optimizer visibility.

## Core workflow

1. **Locate the user-visible symptom.** Identify whether the concern is latency, scrolling, repeated work, memory growth, CPU use, binary size, launch impact, or theoretical code review risk.
2. **Confirm the hot path.** Ask whether the code runs frequently, touches many elements, blocks interaction, runs during startup, or appears in measurements.
3. **Classify the suspected cost.** Use the runtime cost taxonomy instead of saying the code is vaguely “slow.”
4. **Look for evidence.** Prefer Instruments, Allocations, Time Profiler, optimized SIL, benchmark output, XCTest performance tests, or production signals.
5. **Separate semantics from mechanics.** Do not remove an abstraction only because it has a possible cost. Check what design purpose it serves.
6. **Propose the smallest safe change.** Prefer local changes that reduce allocation, ARC traffic, dispatch, copying, or missed specialization without damaging API clarity.
7. **Explain trade-offs.** Mention readability, API flexibility, testability, binary size, ABI stability, build time, or maintenance cost.
8. **Validate the result.** Do not call the optimization successful without a before/after validation path.

## Decision rules

### If the code is not on a hot path

Avoid low-level rewrites.

Explain that the concern may be theoretically valid but unlikely to matter without evidence. Suggest measurement only if the path is suspected to affect user-visible performance.

### If the issue is allocation

Check whether allocation comes from:

- class instances;
- closure contexts;
- boxed variables;
- existential storage;
- type erasure wrappers;
- intermediate arrays, dictionaries, sets, strings, or data buffers;
- bridging between Swift and Objective-C/Foundation types.

Prefer reducing repeated allocation over replacing every reference type.

### If the issue is ARC

Check ownership and lifetime before recommending changes.

Look for repeated retain/release traffic in loops, closure captures of large owners, unnecessary weak access in hot paths, and reference-backed value storage. Do not treat `weak` and `unowned` as performance fixes. They are ownership tools first.

### If the issue is dispatch

Ask whether dynamic dispatch is intentional.

Prefer `final` for app-level classes that are not designed for inheritance. Use generics, concrete types, or internal implementation details only when they preserve the intended abstraction and matter in the measured path.

Do not flatten useful polymorphism without evidence.

### If the issue is `any Protocol`

Ask whether runtime heterogeneity is required.

`any Protocol` is not automatically wrong. It is appropriate when values of different concrete types must be stored or passed uniformly. Consider generics or opaque types when the hot path can remain statically typed.

### If the issue is generics

Check whether specialization actually happens.

Generics help most when the optimizer can see enough implementation detail to specialize the hot path. Module boundaries, public resilience, large functions, or type erasure can limit that.

### If the issue is copy-on-write

Check mutation patterns and uniqueness boundaries.

Repeated mutation of COW values can be cheap when storage is uniquely referenced and expensive when it repeatedly copies. Custom COW must preserve value semantics and document thread-safety assumptions.

### If the issue is SIL or compiler output

Use optimized SIL, not Debug SIL, for performance conclusions.

Look for evidence such as allocation instructions, retain/release traffic, witness dispatch, existential opening, closure creation, missed specialization, and missed devirtualization. Treat SIL as evidence, not as an excuse to overfit source code to one compiler version.

### If the issue is unsafe Swift

Do not recommend unsafe code as a first step.

Use unsafe APIs only when safe APIs cannot express the operation efficiently enough, measurement shows the abstraction cost matters, and the unsafe region can be kept small behind a safe wrapper.

### If the issue is module boundaries

Separate runtime optimization from architecture and build-time concerns.

Use `@inlinable`, `@usableFromInline`, and `@frozen` only when the API commitment is acceptable. These attributes are not generic “make it faster” switches.

## Common gotchas

- `struct` does not guarantee stack allocation.
- `class` does not automatically mean a performance bug.
- Value semantics can still involve heap storage and ARC.
- `Array`, `String`, `Dictionary`, `Set`, and `Data` can hide reference-backed storage.
- `any Protocol` is a useful abstraction with runtime cost, not a mistake.
- `some Protocol` is not a universal replacement for `any Protocol`.
- Generics help most when specialization happens.
- `final` is a good default for app-level classes that are not designed for inheritance, but it should not be oversold as a standalone performance fix.
- `@inline(__always)` can increase code size and should not be the first fix.
- `@inlinable` is an API and ABI commitment.
- `weak` and `unowned` are ownership tools, not performance tools.
- Debug-build behavior is not reliable evidence for optimized runtime performance.
- Unsafe code can be slower, less optimizable, or incorrect if used casually.
- Do not replace readable architecture with low-level code unless the measured path justifies it.

## Reference routing

Read references selectively. Do not load all reference files by default.

- `references/allocation-and-layout.md` — read when the task involves stack vs heap behavior, object layout, closure boxes, existential storage, temporary allocations, or hidden heap storage inside value types.
- `references/arc-and-ownership.md` — read when the task involves retain/release traffic, closure captures, weak/unowned references, object lifetime, reference cycles, COW ownership, or bridging lifetime.
- `references/dispatch-and-specialization.md` — read when the task involves direct dispatch, class dispatch, Objective-C dispatch, witness dispatch, closure dispatch, devirtualization, inlining, or generic specialization.
- `references/existentials-generics-opaque-types.md` — read when the task involves `any Protocol`, `some Protocol`, generics, type erasure, protocol witness dispatch, opaque result types, or replacing existential-heavy hot paths.
- `references/cow-and-large-values.md` — read when the task involves copy-on-write collections, large structs, repeated collection mutation, custom COW storage, uniqueness checks, or value-semantic API design.
- `references/sil-inspection.md` — read when source-level reasoning is not enough and the task needs optimized SIL evidence for allocation, ARC, dispatch, existential opening, closure creation, specialization, or inlining.
- `references/unsafe-swift.md` — read when the task involves unsafe pointers, buffer access, memory binding, alignment, aliasing, manual lifetime, unsafe wrappers, or replacing safe APIs with unsafe code.
- `references/modularization-and-linking.md` — read when the task involves module-boundary optimizer visibility, public API resilience, `@inlinable`, `@usableFromInline`, `@frozen`, static vs dynamic libraries, or runtime trade-offs from modularization.
- `references/concurrency-runtime.md` — read only when a concurrency-related question is specifically about Swift runtime costs such as allocation, ARC, closure captures, SIL lowering, or executor-related overhead. For actor design, MainActor responsiveness, cancellation, AsyncSequence cleanup, continuations, or structured concurrency workflow, use `swift-concurrency-performance` instead.

## Evidence and validation

Prefer evidence appropriate to the suspected cost:

- Use **Allocations** when the concern is object churn, boxes, closure contexts, existential storage, or temporary collections.
- Use **Time Profiler** when the concern is CPU cost, dispatch overhead, copying, bridging, or hot function calls.
- Use **optimized SIL** when the concern is compiler optimization, specialization, devirtualization, ARC insertion, existential opening, closure allocation, or inlining.
- Use **small benchmarks** when isolating a tight loop, collection operation, dispatch pattern, or generic/existential comparison.
- Use **XCTest performance tests** when the operation can be reproduced deterministically.
- Use **before/after traces** when the performance claim affects a real user path.

Do not validate with Debug builds unless the task is specifically about Debug-only development performance.

## Output expectations

When reviewing code, respond with:

1. **Summary** — state whether the concern is likely real, theoretical, or impossible to judge from the provided code.
2. **Suspected runtime cost** — classify the issue as allocation, ARC, dispatch, existential/generic, copying, compiler optimization, unsafe boundary, or module-boundary cost.
3. **Hot-path assessment** — explain whether this code likely matters for user-visible performance.
4. **Evidence to check** — name the measurement, SIL output, benchmark, or trace that would confirm the hypothesis.
5. **Finding** — explain the concrete source pattern and why it may produce runtime work.
6. **Recommended change** — propose the smallest safe change. Include code only when it clarifies the recommendation.
7. **Trade-offs** — mention readability, abstraction, API flexibility, binary size, ABI stability, build time, or maintainability.
8. **Validation** — explain how to confirm the result with before/after evidence.

## Response style

Be precise and restrained.

Prefer:

- “This may allocate because the closure escapes and captures `self`.”
- “This existential is probably fine unless this loop is hot or allocation shows up in Instruments.”
- “Use optimized SIL to check whether the generic function specializes across this module boundary.”
- “A concrete enum may help here if the set of cases is closed and this path is measured as hot.”

Avoid:

- “Structs are always faster.”
- “Classes are slow.”
- “Protocols are bad for performance.”
- “Always replace `any` with generics.”
- “Use unsafe pointers for speed.”
- “Add `@inline(__always)`.”
- “Mark everything `final` for performance.”

## Positive trigger examples

This skill should activate for prompts like:

- “Review this Swift hot path for ARC and allocation overhead.”
- “This feed stores `any FeedItem` values. Could that cause runtime cost?”
- “Should this protocol-heavy code use generics instead?”
- “Can this custom COW type accidentally copy too much?”
- “Can you inspect this optimized SIL and explain the retains and allocations?”
- “Is `@inlinable` justified for this small generic function used across module boundaries?”
- “This closure-heavy parser allocates a lot. What should I check?”
- “Is this unsafe buffer optimization justified?”

## Negative trigger examples

This skill should not activate for prompts like:

- “My SwiftUI list stutters when rows update.”
- “The app takes too long to show the first screen after cold launch.”
- “How should I structure cancellation in this task group?”
- “Which Instruments template should I use to diagnose hangs?”
- “How do I create a Swift protocol?”
- “How do I make this SwiftUI screen look better?”
- “Should I move this SDK initialization out of AppDelegate?”

Route those tasks to the more specific skill unless the user explicitly asks about Swift runtime-level cost.
