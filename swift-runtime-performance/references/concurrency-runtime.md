# Concurrency Runtime

Use this reference only when a concurrency-related question is specifically about Swift runtime costs such as allocation, ARC traffic, closure capture contexts, task object churn, SIL lowering, executor-related overhead, actor-boundary overhead, or runtime evidence around async code.

Do not use this reference as the main guide for actor design, `MainActor` responsiveness, cancellation, `AsyncSequence` cleanup, continuation correctness, task-group workflow, reentrancy, or structured concurrency architecture. For those topics, use `swift-concurrency-performance` instead.

This file is a runtime-cost appendix. It should help the agent explain why async code may allocate, retain, hop, serialize, or lower into a more expensive shape than expected. It should not encourage weakening concurrency correctness for theoretical micro-optimizations.

## Contents

* [Scope and boundaries](#scope-and-boundaries)
* [Runtime cost model](#runtime-cost-model)
* [What to inspect first](#what-to-inspect-first)
* [Task and closure allocation](#task-and-closure-allocation)
* [Captured state and retained object graphs](#captured-state-and-retained-object-graphs)
* [ARC traffic around async code](#arc-traffic-around-async-code)
* [Executor hops and scheduling overhead](#executor-hops-and-scheduling-overhead)
* [Actor calls as runtime boundaries](#actor-calls-as-runtime-boundaries)
* [SIL patterns for async code](#sil-patterns-for-async-code)
* [Measurement guidance](#measurement-guidance)
* [Decision rules](#decision-rules)
* [Common mistakes](#common-mistakes)
* [Output expectations](#output-expectations)

## Scope and boundaries

Use this reference when the task asks about:

* allocations caused by `Task`, child tasks, task groups, async closures, escaping captures, callback wrappers, or continuation adapters;
* ARC traffic caused by async boundaries, retained `self`, retained services, captured object graphs, or values living across suspension;
* task churn in a hot path, launch path, scroll path, import path, search path, or repeated UI lifecycle path;
* repeated actor calls in a hot loop where boundary overhead or serialization cost is suspected;
* executor hops in a measured hot path;
* optimized SIL for async functions, closures, continuations, actor calls, captures, or retain/release traffic;
* whether concurrency syntax changed the runtime shape of a hot path;
* whether a migration to async/await introduced allocation, capture, isolation, or executor-boundary costs.

Prefer `swift-concurrency-performance` when the task asks about:

* how to structure child tasks;
* whether to use `async let`, task groups, actors, `Task {}`, or `Task.detached`;
* `MainActor` responsiveness as a UI or isolation problem;
* cancellation propagation or cleanup;
* `AsyncSequence` lifecycle and termination;
* continuation safety and exactly-once resume;
* actor reentrancy, logical races, or isolation design;
* blocking calls in the cooperative pool as a concurrency workflow issue.

If both apply, split the answer. Use `swift-concurrency-performance` for the concurrency design and this reference only for runtime-cost evidence.

## Runtime cost model

Swift concurrency is not free, but most costs matter only when they appear in a hot path, high-frequency lifecycle, large fan-out, or user-visible latency path.

Classify the suspected runtime cost before recommending a change:

* task creation or task-group fan-out;
* closure allocation or escaping capture context;
* boxed mutable capture or async state storage;
* retained object graph or unintended lifetime extension;
* ARC traffic across async boundaries;
* values kept alive across suspension points;
* executor hop or repeated actor boundary crossing;
* actor serialization of many tiny operations;
* continuation or callback-wrapper allocation;
* lost specialization behind generic, existential, or closure-heavy abstraction;
* Objective-C or Foundation bridging inside async adapters.

Do not infer these costs from syntax alone. A single `Task`, `await`, actor call, or closure is usually not a performance problem. Repetition, hot-path placement, retained lifetime, and evidence matter.

Prefer this mental model:

1. Concurrency syntax describes suspension, isolation, and task structure.
2. The runtime implements that structure with tasks, jobs, executors, closures, captures, and state that may survive suspension.
3. Runtime cost matters only when it appears frequently enough, retains enough state, serializes enough work, or delays user-visible progress.
4. A cheaper runtime shape is useful only if it preserves isolation, cancellation, priority, error handling, ordering, and ownership semantics.

## What to inspect first

Before proposing a runtime-level change, answer these questions:

1. Is the async code on a hot path, repeated path, launch path, scroll path, import path, search path, or frequently triggered UI event?
2. Is the suspected cost allocation, ARC, closure capture, retained lifetime, executor hopping, actor serialization, task churn, or SIL-level lowering?
3. Is there evidence from Allocations, Time Profiler, Swift Concurrency instrument, signposts, logs, release benchmarks, or optimized SIL?
4. Does the code create many tasks where a sequential loop, batched call, or bounded group would be enough?
5. Does an async closure capture `self` or a large object graph when only a small value, service, ID, or immutable snapshot is needed?
6. Does a loop repeatedly cross the same actor boundary?
7. Does a value live across an `await` even though it could be reduced before suspension?
8. Does a generic, existential, or closure-heavy abstraction become expensive only after being placed inside async work?
9. Would the proposed optimization preserve cancellation, priority, isolation safety, ordering, error handling, and ownership clarity?
10. Is this really a runtime-cost issue, or is it a concurrency design issue that belongs in `swift-concurrency-performance`?

Runtime advice should not weaken concurrency correctness. If the cheaper version changes lifetime, cancellation, actor isolation, sendability, priority, ordering, or error semantics, call that out explicitly.

## Task and closure allocation

A task is a unit of asynchronous work, not a thread. Creating tasks has runtime overhead: task records, jobs, closure contexts, captured state, scheduling, priority propagation, cancellation state, and lifetime management.

This overhead is usually fine for coarse work. It becomes suspicious when code creates many tiny tasks for work that is cheaper than the task machinery around it.

Look for:

* `Task {}` inside frequently called methods;
* `Task {}` inside loops;
* one child task per small input element;
* task groups with very small child operations;
* task bodies that capture large objects;
* async adapters that allocate wrapper objects per call;
* callback-to-async wrappers created repeatedly;
* repeated creation of closures that could be hoisted, batched, or made concrete.

Risky pattern:

```swift
func warmUp(_ ids: [ItemID]) {
    for id in ids {
        Task {
            await cache.prepare(id)
        }
    }
}
```

This creates one unstructured task per element. For a small list this might be harmless. For a large or repeated list, it can create task churn and unclear lifetime.

Prefer a shape that matches ownership and cost:

```swift
func warmUp(_ ids: [ItemID]) async {
    for id in ids {
        await cache.prepare(id)
    }
}
```

This does not introduce parallelism. It reduces unstructured task creation and keeps lifetime under the caller's async operation.

If the work is genuinely independent and expensive enough to parallelize, use a structured and bounded approach in `swift-concurrency-performance`. This reference should only explain the runtime cost that made unbounded task creation suspicious.

## Captured state and retained object graphs

Async closures often extend lifetimes. A task body may keep captured values alive until the task completes, is cancelled and finishes cleanup, or is otherwise released.

Review captures when async code mentions `self`:

* Does the task need the whole object or only a value?
* Can the task capture a service, ID, URL, configuration, or immutable snapshot instead?
* Can the UI-facing object own the task and cancel it?
* Could this task outlive the screen, operation, or feature?
* Is a large object graph retained only to access one small dependency?
* Would a weak capture silently drop required work?
* Would a strong capture intentionally keep the operation alive?

Risky pattern:

```swift
final class FeedViewModel {
    private let service: FeedService
    private var items: [FeedItem] = []

    func refresh() {
        Task {
            let newItems = try await service.loadFeed()
            self.items = newItems
        }
    }
}
```

The task captures `self`, which retains the view model and everything it owns. That may be correct if the view model should stay alive until refresh finishes, but it should be intentional.

A narrow capture is useful when the async work does not need owner identity:

```swift
final class ExportCoordinator {
    private let encoder: ExportEncoder
    private let store: ExportStore
    private var exportTask: Task<Void, Never>?

    func startExport() {
        let encoder = encoder
        let store = store

        exportTask = Task { [encoder, store] in
            do {
                let data = try await encoder.encode()
                try await store.save(data)
            } catch {
                // Handle or report the error intentionally.
            }
        }
    }
}
```

This does not remove task allocation. It reduces the retained object graph by capturing only the dependencies needed by the operation.

If the task must update owner state after the async work, that final update still needs an explicit ownership and isolation decision:

```swift
@MainActor
final class FeedViewModel {
    private let service: FeedService
    private var items: [FeedItem] = []
    private var refreshTask: Task<Void, Never>?

    func refresh() {
        let service = service

        refreshTask = Task { [weak self, service] in
            do {
                let newItems = try await service.loadFeed()

                await MainActor.run { [weak self] in
                    self?.items = newItems
                }
            } catch {
                // Handle or publish the error intentionally.
            }
        }
    }
}
```

This example illustrates capture shape, not a universal replacement. `[weak self]` is correct only if dropping the final update after the owner disappears is acceptable. If refresh must keep the owner alive until completion, a strong capture may be the intended ownership model.

## ARC traffic around async code

Async code can create extra ownership traffic because values cross suspension points, closures retain captures, and state machines preserve values that are needed after `await`.

Look for ARC traffic when:

* a small async function is called very frequently;
* a hot loop awaits on each iteration;
* a closure captures reference-heavy state;
* a task or async adapter retains a large object graph;
* a value must live across an `await` even though it could be reduced before suspension;
* an abstraction layer wraps every async call with closures, boxes, continuations, or type erasure.

Pattern to inspect:

```swift
for model in models {
    let formatter = self.formatter
    let result = await service.convert(model, formatter: formatter)
    output.append(result)
}
```

Questions:

* Is `self` retained longer than needed?
* Is `formatter` reference-backed and retained for every iteration?
* Can the call be batched?
* Can pure preparation happen before the `await`?
* Can the large state be reduced before the suspension point?
* Is ARC traffic visible in Time Profiler or optimized SIL?

Do not rewrite for ARC reduction unless the path is hot and evidence shows retain/release traffic matters.

## Executor hops and scheduling overhead

Executor hops are not OS thread context switches, but they still have runtime cost. The cost is usually small compared with I/O or meaningful CPU work. It can matter when code repeatedly crosses the same actor or executor boundary for tiny operations.

Look for:

* actor calls inside tight loops;
* `await MainActor.run` repeated for each element;
* bouncing between a UI actor, service actor, and storage actor for small operations;
* frequent transitions around tiny getters or setters;
* signposts showing gaps between small async steps;
* Swift Concurrency instrument showing dense task/executor activity around tiny operations.

Risky pattern:

```swift
for id in ids {
    let record = await index.record(for: id)
    records.append(record)
}
```

A batched actor call can reduce repeated boundary crossing and allow the actor to do local work while isolated:

```swift
let records = await index.records(for: ids)
```

This is a runtime optimization only if repeated boundary crossing is actually part of the cost. It is also an API design change, so preserve isolation, ordering, error handling, cancellation, and back-pressure semantics.

Be careful with source-level assumptions about where async code runs. In modern Swift, executor behavior can depend on language mode, feature flags, `nonisolated(nonsending)`, `@concurrent`, actor isolation, custom executors, imported async APIs, and compiler optimization. Do not infer executor hops only from the presence of `async`. Validate with compiler diagnostics, optimized SIL, the Swift Concurrency instrument, signposts, or targeted logging.

Do not claim that an executor hop happened unless you have evidence. It is safer to write: "This may repeatedly cross an actor or executor boundary; validate with the Swift Concurrency instrument or optimized SIL."

## Actor calls as runtime boundaries

Actors are reference types with identity and isolated state. They can be the correct design and still introduce runtime costs when used at the wrong granularity.

Analyze actor costs as separate boundaries:

* actor instance allocation and lifetime;
* queueing and serialization of isolated jobs;
* repeated cross-actor calls;
* values copied or retained across the boundary;
* tiny isolated operations that force many awaits;
* CPU work performed while isolated;
* large values returned from actor state;
* actor methods that call back into other actors repeatedly.

Prefer batching when the caller needs many small reads from the same actor:

```swift
// Repeated boundary crossing.
for key in keys {
    values.append(await store.value(for: key))
}

// One boundary crossing, local work inside the actor.
let values = await store.values(for: keys)
```

Prefer moving pure CPU work outside actor isolation when it does not need isolated state:

```swift
actor SearchIndex {
    private var records: [Record] = []

    func snapshots() -> [RecordSnapshot] {
        records.map(RecordSnapshot.init)
    }
}

let snapshots = await index.snapshots()
let ranked = rank(snapshots, query: query)
```

This is only better if `rank` does not need actor-isolated state and can run safely outside the actor. If moving work changes consistency, ordering, or isolation guarantees, do not make the change as a runtime optimization.

Do not use this reference to decide the actor architecture itself. If the main concern is isolation design, reentrancy, cancellation, or `MainActor` correctness, use `swift-concurrency-performance`.

## SIL patterns for async code

When source-level reasoning is not enough, inspect optimized SIL for the async path.

Useful patterns may include:

* `partial_apply` — closure creation or function capture context;
* `alloc_ref` — class or actor allocation;
* `alloc_box` — boxed mutable capture or storage;
* `strong_retain` / `strong_release` — ARC traffic;
* `retain_value` / `release_value` — ownership traffic;
* `copy_value` / `destroy_value` — ownership movement;
* `hop_to_executor` — executor transition in lowered async code;
* `class_method` — dynamic class dispatch;
* `witness_method` — protocol witness dispatch;
* `open_existential_*` — existential opening;
* async lowering that preserves values across suspension points.

Use optimized builds. Debug SIL is often misleading for performance decisions.

Treat SIL instruction names as implementation-level evidence, not stable API. Exact lowering can change across compiler versions, optimization levels, language modes, build settings, ownership mode, and target platforms.

SIL should answer a narrow question, such as:

* Is this closure allocated repeatedly?
* Is mutable captured state boxed?
* Are retains/releases present in the hot path?
* Does this loop repeatedly hop executors?
* Did specialization happen across this abstraction boundary?
* Is a value kept alive across a suspension point?

Do not paste large SIL dumps into final answers. Summarize the relevant pattern and connect it to the source-level code.

## Measurement guidance

Use the measurement source that matches the suspected cost:

| Suspected cost                              | Useful evidence                                                                                                 |
| ------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| Task churn                                  | Swift Concurrency instrument, Allocations, signposts around task creation, release benchmarks                   |
| Closure context allocation                  | Allocations, optimized SIL `partial_apply`, Time Profiler if churn is CPU-visible                               |
| Retained object graph                       | Memory Graph, Leaks, lifecycle logs, weak-reference tests                                                       |
| ARC traffic                                 | Time Profiler, optimized SIL retain/release patterns, release benchmarks                                        |
| Values live across `await`                  | optimized SIL, memory graph, scoped experiments, signposts around large state creation and release              |
| Repeated actor boundary crossing            | Swift Concurrency instrument, signposts, optimized SIL `hop_to_executor`, before/after batching benchmark       |
| Actor serialization                         | Swift Concurrency instrument, Time Profiler, signposts inside actor methods, queueing delays visible in traces  |
| Lost specialization or abstraction overhead | optimized SIL, Time Profiler, targeted benchmarks                                                               |
| Callback/continuation wrapper churn         | Allocations, Time Profiler, signposts around adapter creation                                                   |
| UI delay caused by runtime overhead         | Instruments trace with signposts, Time Profiler, Swift Concurrency instrument, screen recording for correlation |

Prefer before/after validation. Runtime-level changes often trade clarity for small wins, so require evidence that the win matters.

Use Release or Release-like builds when possible. Debug builds exaggerate abstraction, ownership, and concurrency overhead and can mislead runtime analysis.

Do not claim a runtime cost was measured unless a trace, benchmark, log, SIL inspection, or user-provided artifact supports it.

## Decision rules

* Do not optimize concurrency syntax. Optimize measured runtime cost.
* Treat a task as a runtime object with lifetime and captures, not as a free background marker.
* Treat an async closure like any other closure: inspect escaping behavior, captures, allocation, and retained lifetime.
* Prefer batching when repeated actor or executor boundaries dominate tiny operations.
* Prefer narrower captures when `self` retention is not needed.
* Prefer structured ownership before micro-optimizing task allocation.
* Prefer reducing retained lifetime before replacing concurrency primitives.
* Prefer Release-like measurements over Debug observations.
* Inspect optimized SIL only when source-level reasoning and profiling are insufficient.
* Do not weaken actor isolation, cancellation, priority, ordering, sendability, or error semantics to remove a small runtime cost.
* If the issue is mostly architecture or correctness, route to `swift-concurrency-performance`.

## Common mistakes

* Treating every `Task {}` as a performance bug.
* Treating every actor hop as expensive enough to remove.
* Treating executor hops as OS thread context switches.
* Replacing clear structured concurrency with manual lifetime management for a theoretical allocation win.
* Using `Task.detached` as a performance fix without preserving cancellation, priority, task-local values, and ownership.
* Capturing `self` accidentally in long-running tasks.
* Capturing a whole object graph when a value snapshot would do.
* Adding `[weak self]` even when the task should intentionally keep the operation alive.
* Ignoring that values may be kept alive across `await`.
* Assuming nonisolated async code always runs in the same place across Swift language modes.
* Using Debug behavior or Debug SIL as runtime evidence.
* Reading SIL without connecting it to a measured hot path.
* Solving `MainActor` responsiveness here instead of using `swift-concurrency-performance`.
* Removing actor isolation to reduce a hop without preserving data-race safety.

## Output expectations

When this reference is used, include:

1. The specific runtime cost suspected: task churn, closure allocation, ARC traffic, retained graph, executor hop, actor serialization, boxing, async state lifetime, continuation wrapper, or SIL lowering.
2. Why the async structure may create that cost.
3. Whether the path is hot enough for the cost to matter.
4. The smallest safe change that reduces the cost without weakening concurrency correctness.
5. The trade-off in ownership, isolation, cancellation, priority, ordering, sendability, readability, or API shape.
6. A validation step using Allocations, Time Profiler, Swift Concurrency instrument, signposts, optimized SIL, Memory Graph, Leaks, or a release benchmark.

If the issue is actually about actor design, cancellation, continuations, `AsyncSequence`, `MainActor` responsiveness, reentrancy, or structured concurrency workflow, state that this reference is not the right primary source and route to `swift-concurrency-performance`.

Use cautious language when evidence is incomplete:

* "This may create task churn if the method is called frequently."
* "This may retain the view model longer than intended."
* "This may repeatedly cross an actor boundary; validate with signposts or the Swift Concurrency instrument."
* "The cheaper version changes ownership/cancellation semantics, so it is not a drop-in replacement."
* "There is no reason to optimize this unless it appears in a hot path or trace."
