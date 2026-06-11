# Concurrency Runtime

Use this reference only when a concurrency-related question is specifically about Swift runtime costs such as allocation, ARC traffic, closure capture contexts, task object churn, SIL lowering, executor-related overhead, or runtime evidence around async code.

Do not use this reference as the main guide for actor design, `MainActor` responsiveness, cancellation, `AsyncSequence` cleanup, continuation correctness, task-group workflow, reentrancy, or structured concurrency architecture. For those topics, use `swift-concurrency-performance` instead.

## Contents

- [Scope and boundaries](#scope-and-boundaries)
- [Runtime cost model](#runtime-cost-model)
- [What to inspect first](#what-to-inspect-first)
- [Task and closure allocation](#task-and-closure-allocation)
- [Captured state and retained object graphs](#captured-state-and-retained-object-graphs)
- [ARC traffic around async code](#arc-traffic-around-async-code)
- [Executor hops and scheduling overhead](#executor-hops-and-scheduling-overhead)
- [Actor calls as runtime boundaries](#actor-calls-as-runtime-boundaries)
- [SIL patterns for async code](#sil-patterns-for-async-code)
- [Measurement guidance](#measurement-guidance)
- [Decision rules](#decision-rules)
- [Common mistakes](#common-mistakes)
- [Output expectations](#output-expectations)

## Scope and boundaries

This is a runtime appendix for Swift concurrency. It should help the agent reason about the cost created by async implementation details, not redesign an application's concurrency architecture.

Use this reference when the task asks about:

- allocations caused by `Task`, `TaskGroup`, async closures, escaping captures, or callback wrappers;
- ARC traffic caused by async boundaries, retained `self`, retained services, or captured object graphs;
- repeated actor calls in a hot loop where hop overhead or serialization cost is suspected;
- executor hops in a measured hot path;
- optimized SIL for async functions, closures, continuations, actor calls, or retain/release traffic;
- whether concurrency syntax changed the runtime shape of a hot path.

Prefer `swift-concurrency-performance` when the task asks about:

- how to structure child tasks;
- whether to use `async let`, task groups, actors, or `Task.detached`;
- `MainActor` responsiveness as a UI problem;
- cancellation propagation or cleanup;
- `AsyncSequence` lifecycle and termination;
- continuation safety and exactly-once resume;
- actor reentrancy, logical races, or isolation design;
- blocking calls in the cooperative pool as a concurrency workflow issue.

If both apply, split the answer: use `swift-concurrency-performance` for the concurrency design and this reference only for the runtime-cost evidence.

## Runtime cost model

Swift concurrency is not free, but most costs matter only when they appear in a hot path, high-frequency lifecycle, large fan-out, or user-visible latency path.

Classify the suspected runtime cost before recommending a change:


Do not infer these costs from syntax alone. A single `Task` or actor call is usually not a performance problem. Repetition, hot-path placement, retained lifetime, and evidence matter.

## What to inspect first

Before proposing a runtime-level change, answer these questions:

1. 1Is the async code on a hot path, repeated path, launch path, scroll path, or frequently triggered UI event?
2. 2Is the suspected cost allocation, ARC, closure capture, executor hopping, actor serialization, or SIL-level lowering?
3. 3Is there evidence from Allocations, Time Profiler, Swift Concurrency instrument, signposts, logs, or optimized SIL?
4. 4Does the code create many tasks where a sequential loop or batched call would be enough?
5. 5Does an async closure capture `self` or a large object graph when only a small value is needed?
6. 6Does a loop repeatedly cross the same actor boundary?
7. 7Does a generic, existential, or closure-heavy abstraction become expensive only after being placed inside async work?
8. 8Would the proposed optimization preserve cancellation, isolation safety, and ownership clarity?

Runtime advice should not weaken concurrency correctness. If the cheaper version changes lifetime, cancellation, actor isolation, or error semantics, call that out explicitly.

## Task and closure allocation

A task is a unit of asynchronous work, not a thread. Creating tasks has runtime overhead: task records, closure contexts, captured state, scheduling, and lifetime management.

This overhead is usually fine for coarse work. It becomes suspicious when code creates many tiny tasks for work that is cheaper than the task machinery around it.

Look for:

- `Task {}` inside frequently called methods;
- `Task {}` inside loops;
- one child task per small input element;
- task bodies that capture large objects;
- task groups with very small child operations;
- async adapters that allocate wrapper objects per call;
- repeated creation of closures that could be hoisted, batched, or made concrete.

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

Prefer a design that matches ownership and cost:

```swift
func warmUp(_ ids: [ItemID]) async {
    for id in ids {
        await cache.prepare(id)
    }
}
```

If the work is genuinely independent and expensive enough to parallelize, use a structured and bounded approach in the concurrency skill. This reference should only explain the runtime cost that made unbounded task creation suspicious.

## Captured state and retained object graphs

Async closures often extend lifetimes. A task body may keep captured values alive until the task completes or is cancelled and released.

Review captures when async code mentions `self`:

- Does the task need the whole object or only a value?
- Can the task capture a service, ID, URL, configuration, or immutable snapshot instead?
- Can the UI-facing object own the task and cancel it?
- Could this task outlive the screen, operation, or feature?
- Is a large object graph retained only to access one small dependency?

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

The task captures `self`, which retains the view model and everything it owns. That may be correct if the view model owns the task lifetime, but it should be intentional.

A narrower capture can reduce lifetime and ARC risk:

```swift
func refresh() {
    let service = service

    Task { [service] in
        let newItems = try await service.loadFeed()
        await apply(newItems)
    }
}
```

This example is not a universal replacement. It changes ownership and isolation shape. Validate whether `apply` is correctly isolated and whether the task has the right lifetime.

## ARC traffic around async code

Async code can create extra ownership traffic because values cross suspension points, closures retain captures, and state machines preserve values that are needed after `await`.

Look for ARC traffic when:

- a small async function is called very frequently;
- a hot loop awaits on each iteration;
- a closure captures reference-heavy state;
- a value must live across an `await` even though it could be reduced before suspension;
- an abstraction layer wraps every async call with closures, boxes, or type erasure.

Pattern to inspect:

```swift
for model in models {
    let formatter = self.formatter
    let result = await service.convert(model, formatter: formatter)
    output.append(result)
}
```

Questions:

- Is `self` retained longer than needed?
- Is `formatter` reference-backed and retained for every iteration?
- Can the call be batched?
- Can pure preparation happen before the `await`?
- Is ARC traffic visible in Time Profiler or SIL?

Do not rewrite for ARC reduction unless the path is hot and evidence shows retain/release traffic matters.

## Executor hops and scheduling overhead

Executor hops are not OS thread context switches, but they still have runtime cost. The cost is usually small compared with I/O or meaningful CPU work. It can matter when code repeatedly crosses the same actor or executor boundary for tiny operations.

Look for:

- actor calls inside tight loops;
- `await MainActor.run` repeated for each element;
- bouncing between a UI actor, service actor, and storage actor for small operations;
- frequent transitions around tiny getters/setters;
- signposts showing gaps between small async steps.

Risky pattern:

```swift
for id in ids {
    let record = await index.record(for: id)
    records.append(record)
}
```

A batched actor call can reduce repeated hops and allow the actor to do local work while isolated:

```swift
let records = await index.records(for: ids)
```

This is a runtime optimization only if repeated boundary crossing is actually part of the cost. It is also an API design change, so preserve isolation, ordering, error handling, and cancellation semantics.

## Actor calls as runtime boundaries

Actors are reference types with identity and isolated state. They can be the correct design and still introduce runtime costs when used at the wrong granularity.

Analyze actor costs as boundaries:

- allocation and lifetime of the actor instance;
- queueing and serialization of isolated jobs;
- repeated cross-actor calls;
- values copied or retained across the boundary;
- work performed while isolated that could be pure and nonisolated;
- actor state exposed through large value returns.

Prefer batching when the caller needs many small reads from the same actor. Prefer moving pure CPU work outside actor isolation when it does not need isolated state.

Do not use this reference to decide the actor architecture itself. If the main concern is isolation design, reentrancy, cancellation, or `MainActor` correctness, use `swift-concurrency-performance`.

## SIL patterns for async code

When source-level reasoning is not enough, inspect optimized SIL for the async path.

Useful patterns:

- `partial_apply` — closure creation or function capture context;
- `alloc_ref` — class or actor allocation;
- `alloc_box` — boxed mutable capture or storage;
- `strong_retain` / `strong_release` — ARC traffic;
- `copy_value` / `destroy_value` — ownership movement;
- `hop_to_executor` — executor transition in lowered async code;
- `class_method` — dynamic class dispatch;
- `witness_method` — protocol witness dispatch;
- `open_existential_*` — existential opening;
- async function lowering that preserves values across suspension points.

Use optimized builds. Debug SIL is often misleading for performance decisions.

SIL should answer a narrow question, such as:

- Is this closure allocated repeatedly?
- Is mutable captured state boxed?
- Are retains/releases present in the hot path?
- Does this loop repeatedly hop executors?
- Did specialization happen across this abstraction boundary?

Do not paste large SIL dumps into final answers. Summarize the relevant pattern and connect it to the source-level code.

## Measurement guidance

Use the measurement source that matches the suspected cost:


Prefer before/after validation. Runtime-level changes often trade clarity for small wins; require evidence that the win matters.

## Decision rules

- Do not optimize concurrency syntax. Optimize measured runtime cost.
- Treat a task as a runtime object with lifetime and captures, not as a free background marker.
- Treat an async closure like any other closure: inspect escaping behavior, captures, allocation, and retained lifetime.
- Prefer batching when repeated actor or executor boundaries dominate tiny operations.
- Prefer narrower captures when `self` retention is not needed.
- Prefer structured ownership before micro-optimizing task allocation.
- Inspect optimized SIL only when source-level reasoning and profiling are insufficient.
- Do not weaken actor isolation, cancellation, or error semantics to remove a small runtime cost.
- If the issue is mostly architecture or correctness, route to `swift-concurrency-performance`.

## Common mistakes

- Treating every `Task {}` as a performance bug.
- Treating every actor hop as expensive enough to remove.
- Replacing clear structured concurrency with manual lifetime management for a theoretical allocation win.
- Using `Task.detached` as a performance fix without preserving cancellation, priority, and ownership.
- Capturing `self` accidentally in long-running tasks.
- Capturing a whole object graph when a value snapshot would do.
- Ignoring that values may be kept alive across `await`.
- Using Debug behavior or Debug SIL as runtime evidence.
- Reading SIL without connecting it to a measured hot path.
- Solving `MainActor` responsiveness here instead of using `swift-concurrency-performance`.

## Output expectations

When this reference is used, include:

1. 1The specific runtime cost suspected: task churn, closure allocation, ARC traffic, retained graph, executor hop, actor serialization, boxing, or SIL lowering.
2. 2Why the async structure may create that cost.
3. 3Whether the path is hot enough for the cost to matter.
4. 4The smallest safe change that reduces the cost without weakening concurrency correctness.
5. 5The trade-off in ownership, isolation, cancellation, readability, or API shape.
6. 6A validation step using Allocations, Time Profiler, Swift Concurrency instrument, signposts, optimized SIL, or a release benchmark.

If the issue is actually about actor design, cancellation, continuations, `AsyncSequence`, `MainActor` responsiveness, or structured concurrency workflow, state that this reference is not the right primary source and route to `swift-concurrency-performance`.
