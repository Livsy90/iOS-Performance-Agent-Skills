# Actor Reentrancy

Use this reference when the task involves actor-isolated state, duplicate network requests, cache stampedes, state checks before and after `await`, or actor queue buildup.

This reference is about performance and correctness risks caused by actor reentrancy. It is not a general actor tutorial.

## Contents

- [Core model](#core-model)
- [When this matters](#when-this-matters)
- [Review workflow](#review-workflow)
- [Cache miss duplicate work](#cache-miss-duplicate-work)
- [In-flight task pattern](#in-flight-task-pattern)
- [State checks across await](#state-checks-across-await)
- [Actor contention and queue buildup](#actor-contention-and-queue-buildup)
- [Decision rules](#decision-rules)
- [Common mistakes](#common-mistakes)
- [Validation](#validation)
- [Review checklist](#review-checklist)

## Core model

Actor isolation prevents data races on actor-isolated state. It does not make an entire `async` actor method atomic.

Every `await` inside an actor-isolated method is a reentrancy boundary. While the method is suspended, the actor may run other work. When the original method resumes, actor state may no longer match the assumptions made before suspension.

Review actor methods as isolated synchronous regions separated by suspension points:

```text
read actor state
await external work     <- reentrancy boundary
resume later
use actor state again
```

The key question: what assumptions were made before `await`, and are they still valid after the method resumes?

## When this matters

Look for actor reentrancy when code:

- reads actor state before an `await`;
- assumes that state is still valid after the `await`;
- performs cache lookups around async loads;
- coordinates shared network requests;
- mutates counters, quotas, budgets, or state machines;
- shows duplicate requests, cache stampedes, stale validation, negative counters, out-of-order transitions, or actor queue buildup.

## Review workflow

1. 1Find actor methods that contain `await`.
2. 2Mark state reads before each `await`.
3. 3Mark state mutations after each `await`.
4. 4Ask what other actor calls could run during suspension.
5. 5Check whether two callers can start the same expensive work.
6. 6Check whether validation before suspension must be repeated after suspension.
7. 7Check whether state can be committed before suspension.
8. 8Check whether hot actor calls can be batched.
9. 9Recommend the smallest change that preserves the actor's consistency model.
10. 10Include a validation path.

## Cache miss duplicate work

Risky:

```swift
actor ResourceCatalog {
    private var cache: [ResourceKey: Resource] = [:]
    private let loader: any ResourceLoading

    func resource(for key: ResourceKey) async throws -> Resource {
        if let cached = cache[key] {
            return cached
        }

        let resource = try await loader.loadResource(for: key)
        cache[key] = resource
        return resource
    }
}
```

This has no data race, but it has a reentrancy window. Task A can see a cache miss and suspend while loading. Task B can then enter the actor, see the same miss, and start a second load. Actor isolation protected the dictionary. It did not deduplicate the async operation.

## In-flight task pattern

Use an in-flight table when the actor must deduplicate work across suspension points.

```swift
actor ResourceCatalog {
    private var cache: [ResourceKey: Resource] = [:]
    private var inFlight: [ResourceKey: Task<Resource, Error>] = [:]
    private let loader: any ResourceLoading

    func resource(for key: ResourceKey) async throws -> Resource {
        if let cached = cache[key] {
            return cached
        }

        if let existing = inFlight[key] {
            return try await existing.value
        }

        let loader = self.loader
        let task = Task<Resource, Error> {
            try await loader.loadResource(for: key)
        }

        inFlight[key] = task

        do {
            let resource = try await task.value
            cache[key] = resource
            inFlight[key] = nil
            return resource
        } catch {
            inFlight[key] = nil
            throw error
        }
    }
}
```

The important coordination state is written before suspension. Later callers find `inFlight[key]` and await the same task instead of starting duplicate work.

Use this pattern when duplicate work is expensive, many callers may request the same missing value, the result can be shared safely, and the actor owns the shared operation.

For shared in-flight work, always define cancellation policy:

- what happens if one waiter is cancelled;
- what happens if all waiters are cancelled;
- whether the shared task should complete for future callers;
- whether the in-flight entry is removed on both success and failure;
- whether the task priority is appropriate for shared work.

For cache loads, cancelling the underlying operation when one caller cancels is often wrong because other callers may still need the result. For owner-scoped work, cancellation may need to cancel the underlying task.

## State checks across await

Risky:

```swift
actor DownloadQuota {
    private var remainingMegabytes: Int
    private let audit: AuditLogging

    func approveDownload(size: Int) async throws {
        guard remainingMegabytes >= size else {
            throw QuotaError.notEnoughCapacity
        }

        await audit.logApproval(size)

        remainingMegabytes -= size
    }
}
```

The validation happens before suspension. Another caller may consume quota while `audit.logApproval` is running.

Prefer committing state before suspension when the business rule allows it:

```swift
func approveDownload(size: Int) async throws {
    guard remainingMegabytes >= size else {
        throw QuotaError.notEnoughCapacity
    }

    remainingMegabytes -= size
    await audit.logApproval(size)
}
```

If external work must happen before the state mutation, re-check after suspension:

```swift
func approveAfterVerification(size: Int) async throws {
    guard remainingMegabytes >= size else {
        throw QuotaError.notEnoughCapacity
    }

    try await verifier.verifyDownload(size)

    guard remainingMegabytes >= size else {
        throw QuotaError.notEnoughCapacity
    }

    remainingMegabytes -= size
}
```

The second check is not redundant. It protects the business invariant after the reentrancy boundary.

For transaction-like workflows, split the operation into short actor-isolated state transitions and long external work:

```swift
let token = try await stateMachine.beginSync()

do {
    try await syncEngine.run()
    await stateMachine.finishSync(token: token, result: .success(()))
} catch {
    await stateMachine.finishSync(token: token, result: .failure(error))
}
```

`beginSync()` and `finishSync()` should be short isolated transitions. The long async operation should run outside the actor transaction.

## Actor contention and queue buildup

Reentrancy prevents a suspended actor method from blocking the actor forever, but actors can still become bottlenecks.

Look for many tasks awaiting the same actor, hot actor methods called once per item, expensive synchronous work inside actor methods, repeated actor hops in tight loops, logging/metrics funnels, and CPU-heavy work that does not need actor isolation.

Risky:

```swift
for event in events {
    await analyticsStore.append(event)
}
```

Prefer batching:

```swift
await analyticsStore.append(contentsOf: events)
```

If a method does not read or mutate actor-isolated state, keep it outside actor isolation:

```swift
actor DocumentStore {
    nonisolated func tokenize(_ document: Document) -> [Token] {
        Tokenizer.tokenize(document.text)
    }
}
```

Use `nonisolated` only when the method does not touch actor-isolated state.

Do not split state across many actors just to increase parallelism. Split actors only when the split matches the consistency model.

## Decision rules

- Treat every `await` inside an actor method as a possible state invalidation point.
- Write coordination state before suspension when deduplicating work.
- Commit state before `await` when the business rule allows it.
- Re-check state after `await` when external work must happen first.
- Keep transaction-like state transitions short and synchronous when possible.
- Use an in-flight table when duplicate work is expensive and shareable.
- Make cancellation policy explicit for shared in-flight work.
- Batch actor calls on hot paths.
- Move CPU-heavy work out of actor isolation when it does not need actor state.
- Do not use actor isolation as a substitute for workflow design.

## Common mistakes

- Assuming an actor method is atomic from entry to return.
- Reading state, awaiting, then mutating based on the stale read.
- Using a cache dictionary without tracking in-flight loads.
- Cancelling a shared in-flight task when only one waiter cancels.
- Forgetting to remove in-flight entries on failure.
- Putting expensive CPU work inside an actor because the actor owns related data.
- Calling an actor once per item in a tight loop.
- Splitting one consistency domain across multiple actors without a clear invariant.
- Adding locks inside actors before understanding the reentrancy problem.
- Treating actor queue buildup as a reason to remove isolation rather than narrowing isolated work.

## Validation

For duplicate work, add request identifiers in logs, count requests per cache key, stress test many concurrent callers requesting the same key, and verify only one underlying load starts for a cache miss.

For stale state after `await`, write tests with concurrent callers, add controlled suspension points using test doubles, and verify counters, quotas, and state transitions cannot go negative or out of order.

For actor contention, use Instruments to inspect actor timelines and task waiting, compare per-item actor calls with batched calls, and use signposts around actor APIs on hot paths.

Do not call the fix successful only because the data race is gone. Actor reentrancy problems are usually correctness, duplication, latency, or throughput problems.

## Review checklist

Use this checklist for actor methods:

- [ ] Does the method contain `await`?
- [ ] Is actor state read before `await` and trusted after `await`?
- [ ] Can another caller enter during suspension and change the state?
- [ ] Could two callers start the same expensive work?
- [ ] Should in-flight work be tracked?
- [ ] Is in-flight cleanup handled on success and failure?
- [ ] Is the cancellation policy explicit?
- [ ] Can state be committed before suspension?
- [ ] If not, is state re-validated after suspension?
- [ ] Is the actor doing expensive work that does not need isolation?
- [ ] Are hot actor calls batched?
- [ ] Is the actor protecting one clear consistency domain?
- [ ] Would a smaller isolated section fit better?
- [ ] Is the proposed fix validated with a stress test, logs, signposts, or Instruments?
