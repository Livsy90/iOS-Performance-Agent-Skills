# Bounded Task Groups

Use this reference when the task involves `withTaskGroup`, `withThrowingTaskGroup`, parallel mapping, fan-out work, batch processing, memory spikes, backend rate limits, or limiting concurrency.

## Contents

- [Core model](#core-model)
- [When to use a task group](#when-to-use-a-task-group)
- [When not to parallelize](#when-not-to-parallelize)
- [Risky unbounded fan-out](#risky-unbounded-fan-out)
- [Bounded concurrency pattern](#bounded-concurrency-pattern)
- [Preserving input order](#preserving-input-order)
- [Cancellation](#cancellation)
- [Error behavior](#error-behavior)
- [Choosing the limit](#choosing-the-limit)
- [Diagnostics](#diagnostics)
- [Review checklist](#review-checklist)
- [Output guidance](#output-guidance)

## Core model

A task group gives structured concurrency for a dynamic number of child tasks.

It does not automatically make the amount of concurrency safe.

Creating one child task per item is acceptable when the input is small or externally bounded. When the input can be large, untrusted, or user-controlled, unbounded task creation can cause:

- too many live tasks;
- memory pressure;
- scheduler overhead;
- backend rate-limit failures;
- CPU contention;
- worse UI responsiveness;
- downstream actor, database, or network contention.

The core review question is:

> Is the amount of concurrent work intentionally bounded?

If the answer is no, treat the task group as a performance risk until proven otherwise.

## When to use a task group

Use a task group when:

- the number of child operations is dynamic;
- child operations are independent;
- the parent operation owns the lifetime of all child work;
- cancellation should propagate from parent to children;
- the parent needs to aggregate child results.

Use `async let` instead when the number of child operations is small, fixed, and known at compile time.

Use ordinary sequential `await` when the operations depend on each other.

## When not to parallelize

Do not replace sequential awaits mechanically.

Sequential awaits are correct when:

- each step depends on the previous result;
- ordering is required;
- the resource is intentionally serial;
- the operation is already internally concurrent;
- parallel work would overload a service;
- task creation overhead would dominate the useful work;
- predictable memory usage matters more than throughput.

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

Concurrency is useful only when the operations are actually independent.

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

- batch uploads or downloads;
- image resizing or video transcoding;
- OCR and PDF rendering;

The problem is not task groups themselves. The problem is unbounded fan-out.

## Bounded concurrency pattern

Start a limited number of child tasks. Each time one child completes, add one more.

```swift
try await withThrowingTaskGroup(of: EncodedClip.self) { group in
    var iterator = clips.makeIterator()
    var output: [EncodedClip] = []

    for _ in 0..<maxConcurrentJobs {
        guard let clip = iterator.next() else { break }

        group.addTask {
            try await encoder.encode(clip)
        }
    }

    while let encoded = try await group.next() {
        output.append(encoded)

        if let nextClip = iterator.next() {
            group.addTask {
                try await encoder.encode(nextClip)
            }
        }
    }

    return output
}
```

This keeps at most `maxConcurrentJobs` child tasks active at once.

Review details:

- validate that the limit is greater than zero;
- avoid capturing large values inside the child closure;
- avoid heavy aggregation on the `MainActor`;
- avoid choosing a magic limit without explanation;
- measure before claiming the limit is optimal.

## Preserving input order

Task group results arrive in completion order, not input order.

If output order matters, include the index in the child result and store each result at its original position.

```swift
group.addTask {
    let image = try await render(page)
    return (index, image)
}

while let (index, image) = try await group.next() {
    results[index] = image
    addNextTaskIfNeeded()
}
```

Validate missing results before returning. Do not use `compactMap` silently if a missing result would hide a correctness bug.

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

Check cancellation when the user navigates away, a search query changes, the parent operation times out, one child fails, or child work performs CPU-heavy loops with few suspension points.

Do not assume cancellation stops blocking synchronous work.

## Error behavior

Use `withThrowingTaskGroup` when one child failure should fail the whole operation.

Use a non-throwing group with per-item `Result` when partial success is valid:

```swift
await withTaskGroup(of: ImportResult.self) { group in
    for file in files {
        group.addTask {
            do {
                return .success(try await importFile(file))
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

## Choosing the limit

There is no universal concurrency limit.

Choose a starting point based on:

- CPU count;
- memory footprint per child task;
- API rate limits;
- server-side limits;
- network behavior;
- database or actor contention;
- priority of the user action;
- whether the work is CPU-bound or I/O-bound;

Typical starting points:

- CPU-heavy image or video work: use a small limit;
- network requests to the same backend: respect backend and API limits;
- database work: avoid parallelism that only creates lock contention;

Do not hard-code a magic number without a reason. A conservative default plus measurement is usually better than unbounded concurrency.

## Diagnostics

Use Instruments, signposts, logging, and memory tools when task-group behavior is unclear.

| Symptom | Likely cause |
|---|---|
| Many live tasks | Unbounded group or input size not constrained |
| Memory spike | Too many active child tasks or large captured values |
| Rate-limit failures | Concurrency limit too high |
| No speedup | Bottleneck is serialized elsewhere |
| UI stalls | Child work or result aggregation is hitting `MainActor` |
| High CPU with poor throughput | Oversubscription, contention, or blocking work |
| Work continues after navigation | Parent task lifetime is wrong or cancellation is ignored |

When the user provides a trace, connect each recommendation to an observable signal.

## Review checklist

Before recommending a task-group change, check:

- [ ] Is the input size bounded?
- [ ] Is each child operation independent?
- [ ] Is output order important?
- [ ] Does the code create one task per item?
- [ ] Should concurrency be limited?
- [ ] Is the selected limit justified?
- [ ] Is cancellation checked inside expensive child work?
- [ ] Are errors all-or-nothing or per-item?
- [ ] Are large captures avoided in child task closures?
- [ ] Does result aggregation happen off the `MainActor` when heavy?

## Output guidance

When this reference applies, the answer should include:

1. whether the current task group is bounded or unbounded;
2. why the input size or downstream dependency makes that safe or risky;
3. whether operations are independent enough to parallelize;
4. whether output order, cancellation, and error behavior are correct;
5. the smallest safe refactor;
6. a validation step.
