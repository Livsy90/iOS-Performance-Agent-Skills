# Allocation and Layout

Use this reference when a review involves stack vs heap behavior, value storage, class or actor allocation, closure capture contexts, boxed variables, existential storage, object layout, async state lifetime, or unexpected allocations in Swift code.

The goal is not to label types as "fast" or "slow". The goal is to identify where values are likely stored, what lifetime they require, whether allocation is repeated, and whether the storage model matters for the measured performance problem.

## Contents

- [Core model](#core-model)
- [What to inspect first](#what-to-inspect-first)
- [Stack and heap storage](#stack-and-heap-storage)
- [Values inside heap objects](#values-inside-heap-objects)
- [References inside value types](#references-inside-value-types)
- [Closure capture contexts](#closure-capture-contexts)
- [Existential storage](#existential-storage)
- [Reference-backed standard library values](#reference-backed-standard-library-values)
- [Async frames and task storage](#async-frames-and-task-storage)
- [Actors and object graphs](#actors-and-object-graphs)
- [Object layout review](#object-layout-review)
- [Decision rules](#decision-rules)
- [Common mistakes](#common-mistakes)
- [Examples](#examples)
- [Validation](#validation)
- [Output guidance](#output-guidance)

## Core model

Avoid source-level shortcuts:

- `struct` means stack.
- `class` means slow.
- value type means no heap allocation.
- reference type means performance problem.
- generic code means zero allocation.
- existential code always allocates.

Use a storage-and-lifetime model instead.

A Swift value can be stored in a stack frame, heap object, closure capture context, boxed mutable variable, collection buffer, existential container, global/static storage, async state, optimizer-created temporary, or type-erasure wrapper.

The same source-level type can appear in different storage locations depending on context. A value type can be stored inside a heap object. A class reference can be stored inside a value type. A temporary may disappear entirely after optimization.

## What to inspect first

Before proposing a storage or layout change, identify:

1. 1Is the code on a hot path?
2. 2Is allocation repeated frequently?
3. 3Does the value escape its local scope?
4. 4Is the value stored inside another heap object?
5. 5Is it captured by an escaping closure?
6. 6Is a mutable local captured and shared with a closure?
7. 7Is it passed through `any Protocol` or another type-erased boundary?
8. 8Is it stored in reference-backed standard library storage?
9. 9Does a large temporary live across an `await`?
10. 10Is there evidence from Allocations, Time Profiler, optimized SIL, benchmarks, or tests?

Do not rewrite storage models only because a type "looks expensive". Connect the source pattern to repeated allocation, retained object graphs, copying, ARC traffic, cache pressure, or user-visible latency.

## Stack and heap storage

Stack storage is usually associated with local values whose lifetime is known and bounded by the current function activation. It may contain local values, parameters, temporaries, inline fields of local structs or enums, and non-escaping closure context when the optimizer can keep it local.

Stack allocation is cheap, but not literally free. It can still affect frame size, register pressure, copying, and optimizer decisions. Do not optimize for stack placement directly unless allocation or copying appears in measurement.

Heap storage commonly appears when a value needs identity, shared ownership, dynamic lifetime, or storage that cannot be represented as a local temporary after optimization.

Common heap-backed or heap-related cases:

- class and actor instances;
- escaping closure contexts;
- boxed captured variables;
- reference-backed collection and string buffers;
- existential boxes for values that cannot fit inline;
- async state that survives suspension;
- Objective-C and Core Foundation objects;
- type-erasure wrappers;
- objects retained by tasks, callbacks, notifications, or caches.

Heap allocation matters most when it is frequent, large, long-lived, contended, or visible in user-facing latency. The important review question is usually allocation frequency and object graph shape, not the mere presence of a class.

## Values inside heap objects

A struct stored in a class instance is not stored on the stack just because it is a struct. It is stored inline within the class object's heap allocation, unless the struct itself contains references to separate storage.

```swift
struct PlaybackPosition {
    var seconds: Double
    var rate: Double
}

final class PlayerSession {
    var position: PlaybackPosition
}
```

`PlaybackPosition` has value semantics, but `position` is part of the `PlayerSession` heap object.

Review rule: distinguish value semantics from physical storage location, check where the owning value lives, and avoid saying "make this a struct so it goes on the stack" unless lifetime and ownership support that claim.

## References inside value types

A struct can contain class references, standard library buffers, COW storage, closures, type-erased wrappers, and bridged Objective-C values. Copying the struct copies the value container; it may also retain shared storage or references.

Review whether the value contains class references, large COW storage, cross-layer copies, mutation after copying, value-looking syntax over shared mutable state, or unintended shared mutable references.

## Closure capture contexts

Closures are a common source of hidden allocation and lifetime extension.

A non-escaping closure can often remain local, inline, or disappear after optimization. An escaping closure usually needs a capture context whose lifetime extends beyond the current function call.

Look for closures that are created inside loops, stored in collections, assigned to properties, passed to SDK callbacks, cross async or actor boundaries, capture `self` when only one dependency is needed, capture mutable locals, or retain large object graphs longer than intended.

Prefer narrow captures when they express the intended lifetime.

```swift
final class MetricsReporter {
    private let sink: MetricsSink
    private let clock: Clock

    func makeHandler() -> (Event) -> Void {
        let sink = sink
        let clock = clock
        return { event in
            sink.record(event, at: clock.now)
        }
    }
}
```

The escaping closure still needs storage, but it captures only the dependencies it uses instead of retaining the whole reporter.

A mutable local captured by an escaping closure may require boxed storage so the closure and the original scope can share the same variable.

```swift
func makeCounter() -> () -> Int {
    var value = 0
    return {
        value += 1
        return value
    }
}
```

This is correct behavior. It becomes relevant when similar boxes are created repeatedly in hot paths or when boxed mutation makes ownership unclear.

## Existential storage

An existential such as `any Protocol` is a runtime container for a value whose concrete type is not statically exposed at that point.

An existential may involve inline storage, boxed storage, type metadata, witness tables, and dynamic opening when the value is used.

Review whether the existential is used in a hot loop, whether concrete types are homogeneous, whether runtime heterogeneity is required, whether the value is stored long-term, whether boxing appears in Allocations or optimized SIL, and whether a generic, opaque type, enum, or concrete helper would preserve the design.

Do not automatically replace `any Protocol` with generics. Match the storage model to the design. Use generics or `some Protocol` when the concrete type is homogeneous and static specialization matters. Use existentials when runtime heterogeneity is the point.

For deeper dispatch and specialization analysis, read `dispatch-and-specialization.md` and `existentials-generics-opaque-types.md`.

## Reference-backed standard library values

Many standard value types use heap-backed storage internally: `Array`, `Dictionary`, `Set`, `String`, and `Data`.

Copying these values often copies a small value header and shares storage until mutation. This is usually efficient, but it can still involve reference counting, uniqueness checks, allocation, and buffer copies.

Review questions:

- Is mutation performed after copying?
- Is this repeated for many elements?
- Is capacity reserved when growth is expected?
- Can mutation happen at the owning boundary?
- Would a lazy view, iterator, or streaming approach avoid intermediate storage?
- Is a large buffer retained longer than intended?

For detailed COW mutation rules, read `cow-and-large-values.md`.

## Async frames and task storage

Async functions may need to preserve state across suspension points. Values that live across suspension become part of async function state. Depending on optimization and runtime behavior, that state may not behave like an ordinary synchronous stack frame.

Do not assume every `await` creates a heap allocation. Also do not assume async code has the same lifetime behavior as synchronous stack-only code.

Review questions:

- Does a large value live across an `await` even though it is no longer needed?
- Can expensive temporary state be scoped more tightly before suspension?
- Is a task retaining a large object graph longer than intended?
- Is `Task {}` or `Task.detached {}` extending lifetime beyond the caller?
- Is the issue really allocation/lifetime, or is it actor hopping, cancellation, or MainActor responsiveness?

If the main concern is task lifecycle, actor isolation, cancellation, executor behavior, or MainActor responsiveness, route to `swift-concurrency-performance`. Use this file only for storage, allocation, and lifetime aspects.

## Actors and object graphs

Actors are reference types with identity. Use this reference only for actor object graph size, retained state, closure retention, or allocation. Route scheduling, reentrancy, cancellation, priority, and executor behavior to `swift-concurrency-performance`.

## Object layout review

Object layout details can matter in low-level code, but they are usually not the first review target.

Ask how many objects are allocated, how large the retained graph is, whether wrappers add allocation layers, whether bridging adds objects, and whether layout is relevant to cache locality in a measured hot loop.

Avoid overclaiming exact object header size, existential container size, enum layout, or class layout across compiler/runtime versions.

## Decision rules

- Treat storage location as a hypothesis, not a source-level fact.
- Optimize allocation only when it is repeated, large, retained too long, or visible in measurement.
- Prefer lifetime reduction before replacing abstractions.
- Prefer narrowing captures before redesigning closure-heavy APIs.
- Prefer moving mutation to the owning boundary before introducing unsafe buffers.
- Prefer reserving capacity when predictable growth causes repeated collection allocation.
- Prefer concrete or generic hot paths only when runtime heterogeneity is not required.
- Prefer scoping large temporaries before `await` when they do not need to survive suspension.
- Do not replace classes with structs only to chase stack allocation.
- Do not introduce unsafe memory APIs unless safe designs and data layout improvements are insufficient.

## Common mistakes

- Saying "structs live on the stack".
- Saying "classes are slow".
- Treating value semantics as a physical storage guarantee.
- Ignoring references inside value types.
- Ignoring reference-backed storage inside standard library values.
- Treating every closure as an allocation problem.
- Missing escaping closures created repeatedly in hot paths.
- Missing mutable captured variables that require shared storage.
- Treating every existential as boxed.
- Treating every existential as free.
- Assuming every async suspension allocates.
- Keeping large temporary values alive across `await`.
- Using object layout speculation instead of Allocations, SIL, or benchmarks.
- Recommending unsafe memory as the first optimization step.

## Examples

### Value type stored in a heap object

```swift
struct SearchSnapshot {
    var query: String
    var results: [SearchResult]
}

final class SearchScreenState {
    var snapshot: SearchSnapshot
}
```

`SearchSnapshot` is a value, but here it is stored inside a class instance. Its `String` and `Array` fields also use reference-backed storage.

### Repeated closure creation

```swift
func handlers(for items: [Item], tracker: Tracker) -> [() -> Void] {
    items.map { item in { tracker.track(item.id) } }
}
```

This intentionally creates escaping closures. It may be fine for a small list, but suspicious in a frequently rebuilt hot path.

### Collection growth

Reserve capacity where growth is predictable and allocation shows up. Do not add capacity tuning everywhere.

```swift
var rows: [Row] = []
rows.reserveCapacity(models.count)
```

## Validation

Use validation that matches the suspected cost.

Use Instruments Allocations, memory graph inspection, signposts, and repeated before/after runs for allocation and object churn.

Use optimized SIL, not Debug SIL, for compiler/storage hypotheses. Search for `alloc_ref`, `alloc_box`, `partial_apply`, `alloc_stack`, `copy_value`, `destroy_value`, and `open_existential_*`.

Use targeted benchmarks, signposted hot paths, allocation counts during mutation, and buffer-growth checks for copying and COW behavior.

Do not call a storage change successful only because the code looks lower level. Confirm that it reduces allocation, copying, retained graph size, or latency without harming correctness.

## Output guidance

When using this reference in a review, return:

1. 1The likely storage or allocation model.
2. 2The source-level pattern that creates or retains storage.
3. 3Why it may matter in this specific path.
4. 4The smallest safe change.
5. 5Trade-offs in readability, API clarity, ownership, or flexibility.
6. 6The validation method.

Use cautious language when physical layout is not proven: "This may allocate if...", "This is likely heap-backed because...", "Validate with Allocations or optimized SIL...", "The source type alone does not prove the storage location..."
