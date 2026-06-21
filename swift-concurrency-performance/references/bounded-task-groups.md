# Bounded Task Groups

Use this reference when the task involves `withTaskGroup`, `withThrowingTaskGroup`, parallel mapping, fan-out work, batch processing, memory spikes, backend rate limits, CPU contention, downstream bottlenecks, or limiting concurrency.

This reference is about safe and intentional task-group parallelism. It is not a general introduction to Swift Concurrency.

## Contents

* [Core model](#core-model)
* [When to use a task group](#when-to-use-a-task-group)
* [When not to parallelize](#when-not-to-parallelize)
* [Risky unbounded fan-out](#risky-unbounded-fan-out)
* [Bounded concurrency pattern](#bounded-concurrency-pattern)
* [Preserving input order](#preserving-input-order)
* [Cancellation](#cancellation)
* [Error behavior](#error-behavior)
* [Swift 6 and Sendable](#swift-6-and-sendable)
* [MainActor and task groups](#mainactor-and-task-groups)
* [Choosing the limit](#choosing-the-limit)
* [Diagnostics](#diagnostics)
* [Review checklist](#review-checklist)
* [Output guidance](#output-guidance)

## Core model

A task group gives structured concurrency for a dynamic number of child tasks.

It does not automatically make the amount of concurrency safe.

Creating one child task per item is acceptable when the input is small or externally bounded. When the input can be large, untrusted, user-controlled, or backed by a limited downstream resource, unbounded task creation can cause:

* too many live tasks;
* memory pressure;
* scheduler overhead;
* backend rate-limit failures;
* CPU contention;
* worse UI responsiveness;
* database, actor, or network contention;
* many pending operations after cancellation;
* worse tail latency.

The core review question is:

```text
Is the amount of concurrent work intentionally bounded?
```

If the answer is no, treat the task group as a performance risk until proven otherwise.

Task priority is not a concurrency limit. Priority can influence scheduling, but it does not bound the number of active child tasks, network requests, database calls, file reads, or CPU-heavy operations.

## When to use a task group

Use a task group when:

* the number of child operations is dynamic;
* child operations are independent;
* the parent operation owns the lifetime of all child work;
* cancellation should propagate from parent to children;
* the parent needs to aggregate child results;
* the result can be produced from all children or from controlled partial success.

Use `async let` instead when the number of child operations is small, fixed, and known in the code.

Use ordinary sequential `await` when operations depend on each other.

Use an actor, queue, semaphore-like async limiter, operation queue, or service-specific throttling layer when the main problem is controlling access to a limited shared resource.

## When not to parallelize

Do not replace sequential awaits mechanically.

Sequential awaits are correct when:

* each step depends on the previous result;
* ordering is required;
* the resource is intentionally serial;
* the operation is already internally concurrent;
* parallel work would overload a service;
* task creation overhead would dominate the useful work;
* predictable memory usage matters more than throughput;
* the bottleneck is a downstream actor, database lock, SDK, or backend rate limit;
* the work runs on `MainActor` and would serialize there anyway.

Risky rewrite:

```swift
async let session = createSession()
async let token = refreshToken()
async let data = fetchData(using: token)
```

This is wrong if `refreshToken` needs the session or `fetchData` needs the refreshed token.

Prefer sequential code when the dependency is real:

```swift
let session = try await createSession()
let token = try await refreshToken(for: session)
let data = try await fetchData(using: token)
```

Concurrency is useful only when the operations are actually independent or when controlled overlap improves latency without violating resource limits.

## Risky unbounded fan-out

This pattern creates one child task per item:

```swift
try await withThrowingTaskGroup(of: EncodedClip.self) { group in
    for clip in clips {
        group.addTask {
            try await encoder.encode(clip)
        }
    }

    var output: [EncodedClip] = []

    for try await encoded in group {
        output.append(encoded)
    }

    return output
}
```

This can be fine for a handful of clips. It is risky when `clips` can contain hundreds or thousands of items.

Common places to check:

* batch uploads or downloads;
* image resizing;
* video transcoding;
* OCR;
* PDF rendering;
* thumbnail generation;
* file hashing;
* database imports;
* search indexing;
* analytics backfills;
* migration or sync jobs.

The problem is not task groups themselves. The problem is unbounded fan-out.

Also check whether each child task captures large values. A large captured image, document, data blob, model graph, or non-sendable service can multiply memory and correctness risks across many child tasks.

## Bounded concurrency pattern

Start a limited number of child tasks. Each time one child completes, add one more.

```swift
try await withThrowingTaskGroup(of: EncodedClip.self) { group in
    let limit = max(1, maxConcurrentJobs)
    var iterator = clips.makeIterator()
    var output: [EncodedClip] = []

    for _ in 0..<limit {
        guard let clip = iterator.next() else { break }

        group.addTask {
            try Task.checkCancellation()
            return try await encoder.encode(clip)
        }
    }

    while let encoded = try await group.next() {
        output.append(encoded)

        if let nextClip = iterator.next() {
            group.addTask {
                try Task.checkCancellation()
                return try await encoder.encode(nextClip)
            }
        }
    }

    return output
}
```

This keeps at most `limit` child tasks active at once.

This version returns results in completion order. If the caller needs input order, use the indexed variant below.

Review details:

* validate or normalize the limit before starting the group;
* avoid a zero limit that silently returns no work;
* avoid capturing large values inside the child closure;
* avoid capturing mutable non-sendable services into many child tasks;
* avoid heavy aggregation on `MainActor`;
* avoid choosing a magic limit without explanation;
* measure before claiming the limit is optimal;
* check downstream limits, not only CPU count.

Prefer normalizing invalid limits for app-facing APIs:

```swift
let limit = max(1, maxConcurrentJobs)
```

Prefer failing fast for internal APIs where an invalid limit indicates a programming error:

```swift
precondition(maxConcurrentJobs > 0)
```

## Preserving input order

Task group results arrive in completion order, not input order.

If output order matters, include the index in the child result and store each result at its original position.

```swift
let indexedPages = Array(pages.enumerated())
var results = Array<Image?>(repeating: nil, count: indexedPages.count)
var iterator = indexedPages.makeIterator()
let limit = max(1, maxConcurrentJobs)

try await withThrowingTaskGroup(of: (Int, Image).self) { group in
    for _ in 0..<limit {
        guard let (index, page) = iterator.next() else { break }

        group.addTask {
            try Task.checkCancellation()
            let image = try await render(page)
            return (index, image)
        }
    }

    while let (index, image) = try await group.next() {
        results[index] = image

        if let (nextIndex, nextPage) = iterator.next() {
            group.addTask {
                try Task.checkCancellation()
                let image = try await render(nextPage)
                return (nextIndex, image)
            }
        }
    }
}

guard results.allSatisfy({ $0 != nil }) else {
    throw RenderingError.missingResult
}

return results.map { $0! }
```

Do not use `compactMap` silently if a missing result would hide a correctness bug.

For large inputs, avoid `Array(input.enumerated())` if it creates unnecessary memory pressure. Prefer indexing the original collection when possible, or create a lightweight iterator that carries the index.

## Cancellation

Task groups are structured, so parent cancellation propagates to child tasks.

But cancellation is cooperative. Child tasks must reach suspension points or explicitly check cancellation during expensive work.

```swift
group.addTask {
    try Task.checkCancellation()
    let decoded = try await decoder.decode(file)
    try Task.checkCancellation()
    return try await optimizer.optimize(decoded)
}
```

If child work loops internally, cancellation checks should be inside the loop.

```swift
for timestamp in video.timestamps {
    try Task.checkCancellation()
    frames.append(try await video.frame(at: timestamp))
}
```

Check cancellation when:

* the user navigates away;
* a search query changes;
* the parent operation times out;
* one child fails;
* child work performs CPU-heavy loops with few suspension points;
* child work calls blocking synchronous APIs;
* the group processes many inputs.

Do not assume cancellation stops blocking synchronous work. If a child task enters a blocking SDK call, synchronous file read, synchronous network call, or CPU loop without checks, cancellation may not take effect until that work returns.

For blocking legacy APIs, use a bounded adapter and document cancellation limits.

## Error behavior

Use `withThrowingTaskGroup` when one child failure should fail the whole operation.

When `withThrowingTaskGroup` exits by throwing, remaining child tasks are cancelled as part of structured cleanup. Cancellation is still cooperative. CPU-heavy, blocking, or poorly written child work may continue until it reaches a cancellation check, suspension point, or returns.

Example all-or-nothing behavior:

```swift
try await withThrowingTaskGroup(of: ImportedFile.self) { group in
    for file in files {
        group.addTask {
            try Task.checkCancellation()
            return try await importFile(file)
        }
    }

    var imported: [ImportedFile] = []

    while let file = try await group.next() {
        imported.append(file)
    }

    return imported
}
```

Use a non-throwing group with per-item `Result` when partial success is valid:

```swift
await withTaskGroup(of: ImportResult.self) { group in
    for file in files {
        group.addTask {
            do {
                return .success(try await importFile(file))
            } catch is CancellationError {
                return .cancelled(file)
            } catch {
                return .failure(file, error)
            }
        }
    }

    for await result in group {
        report.add(result)
    }
}
```

For large input, combine per-item results with bounded concurrency.

Do not use partial success accidentally. Decide whether one failure should cancel the whole operation or whether each item should report its own result.

## Swift 6 and Sendable

In Swift 6 strict concurrency, values captured by child tasks should be safe to use across concurrency domains.

Prefer:

* `Sendable` inputs;
* `Sendable` results;
* immutable value snapshots;
* actor-isolated services with explicit async APIs;
* thread-safe service objects with clear ownership;
* small IDs instead of large mutable models.

Risky:

```swift
try await withThrowingTaskGroup(of: EncodedClip.self) { group in
    for clip in clips {
        group.addTask {
            try await encoder.encode(clip)
        }
    }
}
```

This may be unsafe or rejected by strict concurrency if `encoder` is a mutable non-sendable class or if `clip` contains non-sendable mutable state.

Prefer making the boundary explicit:

```swift
struct ClipJob: Sendable {
    let id: Clip.ID
    let sourceURL: URL
}

let jobs: [ClipJob] = clips.map {
    ClipJob(id: $0.id, sourceURL: $0.sourceURL)
}

try await withThrowingTaskGroup(of: EncodedClip.self) { group in
    for job in jobs {
        group.addTask {
            try await encoderService.encode(job)
        }
    }
}
```

Do not silence `Sendable` diagnostics mechanically. They often indicate that task-group fan-out is crossing an unclear ownership or synchronization boundary.

If the service is actor-isolated, remember that many child tasks may still serialize through that actor. That may be correct, but it can also mean the task group adds overhead without throughput improvement.

## MainActor and task groups

Creating a task group inside a `@MainActor` function does not make heavy work safe for the main actor.

Keep heavy aggregation and result processing out of main-actor-isolated code.

Risky:

```swift
@MainActor
func buildRows(files: [File]) async throws {
    rows = try await withThrowingTaskGroup(of: RowModel.self) { group in
        for file in files {
            group.addTask {
                try await makeRowModel(file)
            }
        }

        var result: [RowModel] = []

        while let row = try await group.next() {
            result.append(row)
        }

        return result
    }
}
```

Even if child tasks do work off the main actor, the surrounding function and aggregation are main-actor-isolated. If aggregation is large, it can hurt UI responsiveness.

Prefer preparing results away from `MainActor`, then applying a compact UI state update:

```swift
func buildRows(files: [File]) async throws -> [RowModel] {
    try await withThrowingTaskGroup(of: RowModel.self) { group in
        for file in files {
            group.addTask {
                try await makeRowModel(file)
            }
        }

        var result: [RowModel] = []
        result.reserveCapacity(files.count)

        while let row = try await group.next() {
            result.append(row)
        }

        return result
    }
}

@MainActor
func applyRows(_ newRows: [RowModel]) {
    rows = newRows
}
```

Child tasks should not call `@MainActor` APIs for CPU-heavy work. If each child quickly hops to `MainActor`, the group may create concurrency overhead while the real bottleneck remains serialized on the main actor.

## Choosing the limit

There is no universal concurrency limit.

Choose a starting point based on:

* CPU count;
* memory footprint per child task;
* API rate limits;
* server-side limits;
* network behavior;
* database or actor contention;
* priority of the user action;
* whether the work is CPU-bound or I/O-bound;
* whether the work is blocking or suspending;
* device class;
* Low Power Mode and thermal behavior;
* whether the UI needs headroom.

Typical starting points:

* CPU-heavy image or video work: start with a small limit and measure;
* CPU-heavy work on iOS: do not automatically use all cores; leave headroom for UI responsiveness;
* network requests to the same backend: respect backend, API, and retry limits;
* database work: avoid parallelism that only creates lock contention;
* actor-backed services: check whether work serializes through the actor;
* blocking SDK calls: use a bounded adapter and document cancellation limits.

Do not hard-code a magic number without a reason. A conservative default plus measurement is usually better than unbounded concurrency.

Do not claim the limit is optimal without measuring:

* total wall-clock time;
* memory peak;
* UI responsiveness;
* error and retry rate;
* backend rate-limit behavior;
* cancellation latency;
* old-device behavior.

## Diagnostics

Use Instruments, signposts, logging, and memory tools when task-group behavior is unclear.

| Symptom                         | Likely cause                                             |
| ------------------------------- | -------------------------------------------------------- |
| Many live tasks                 | Unbounded group or input size not constrained            |
| Memory spike                    | Too many active child tasks or large captured values     |
| Rate-limit failures             | Concurrency limit too high                               |
| No speedup                      | Bottleneck is serialized elsewhere                       |
| UI stalls                       | Child work or result aggregation is hitting `MainActor`  |
| High CPU with poor throughput   | Oversubscription, contention, or blocking work           |
| Work continues after navigation | Parent task lifetime is wrong or cancellation is ignored |
| Slow cancellation               | Child tasks do not check cancellation or are blocked     |
| Actor queue buildup             | Too many child tasks call the same actor                 |
| Tail latency gets worse         | Limit is too high or downstream resource is saturated    |

When the user provides a trace, connect each recommendation to an observable signal.

Useful instrumentation:

* signpost the parent operation;
* log input count and concurrency limit;
* log child start/end counts;
* log cancellation events;
* log retry and rate-limit events;
* measure memory peak;
* compare wall-clock time across several limits;
* test on older devices when UI responsiveness matters.

## Review checklist

Before recommending a task-group change, check:

* [ ] Is the input size bounded?
* [ ] Is each child operation independent?
* [ ] Is output order important?
* [ ] Does the code create one task per item?
* [ ] Should concurrency be limited?
* [ ] Is the selected limit greater than zero?
* [ ] Is the selected limit justified?
* [ ] Is task priority being mistaken for a concurrency limit?
* [ ] Is cancellation checked inside expensive child work?
* [ ] Can child work enter blocking synchronous APIs?
* [ ] Are errors all-or-nothing or per-item?
* [ ] Are large captures avoided in child task closures?
* [ ] Are captured values safe under Swift 6 strict concurrency?
* [ ] Does the group call one actor, database, SDK, or backend so heavily that it only creates contention?
* [ ] Does result aggregation happen off `MainActor` when heavy?
* [ ] Is there a validation plan?

## Output guidance

When this reference applies, the answer should include:

1. whether the current task group is bounded or unbounded;
2. why the input size or downstream dependency makes that safe or risky;
3. whether operations are independent enough to parallelize;
4. whether output order, cancellation, and error behavior are correct;
5. whether Swift 6 `Sendable` or actor-isolation issues affect the design;
6. the smallest safe refactor;
7. a validation step.
