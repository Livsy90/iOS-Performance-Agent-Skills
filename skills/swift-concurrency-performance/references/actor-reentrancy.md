# Actor Reentrancy

Use this reference when the task involves actor-isolated state, duplicate network requests, cache stampedes, state checks before and after `await`, shared in-flight work, or actor queue buildup.

This reference is about performance and correctness risks caused by actor reentrancy. It is not a general actor tutorial.

## Contents

* [Core model](#core-model)
* [When this matters](#when-this-matters)
* [Review workflow](#review-workflow)
* [Cache miss duplicate work](#cache-miss-duplicate-work)
* [In-flight task pattern](#in-flight-task-pattern)
* [State checks across await](#state-checks-across-await)
* [Transaction-like workflows](#transaction-like-workflows)
* [Actor contention and queue buildup](#actor-contention-and-queue-buildup)
* [Decision rules](#decision-rules)
* [Common mistakes](#common-mistakes)
* [Validation](#validation)
* [Review checklist](#review-checklist)

## Core model

Actor isolation prevents data races on actor-isolated state. It does not make an entire `async` actor method atomic from entry to return.

An `await` inside an actor-isolated method is a possible reentrancy boundary. While the method is suspended, the actor may run other work. When the original method resumes, actor state may no longer match the assumptions made before suspension.

Review actor methods as isolated synchronous regions separated by suspension points:

```text
read actor state
await external work     <- possible reentrancy boundary
resume later
use actor state again
```

The key question is:

```text
What assumptions were made before await, and are they still valid after the method resumes?
```

This is not primarily a data-race problem. Actor isolation may protect memory access correctly while the workflow still has duplicate work, stale validation, out-of-order state transitions, extra latency, or poor throughput.

## When this matters

Look for actor reentrancy when code:

* reads actor state before an `await`;
* assumes that state is still valid after the `await`;
* performs cache lookups around async loads;
* coordinates shared network requests;
* stores shared in-flight work;
* mutates counters, quotas, budgets, limits, or state machines;
* uses actors as service funnels for many concurrent tasks;
* shows duplicate requests, cache stampedes, stale validation, negative counters, out-of-order transitions, or actor queue buildup.

Common symptoms:

* the same resource is loaded multiple times under concurrent access;
* a cache works in simple tests but stampedes under load;
* quota, inventory, balance, or counter checks pass but later commit invalid state;
* cancellation by one caller unexpectedly cancels shared work needed by other callers;
* many tasks spend time waiting for the same actor;
* a hot loop performs one actor call per item.

## Review workflow

1. Find actor methods that contain `await`.
2. Mark actor-state reads before each `await`.
3. Mark actor-state mutations after each `await`.
4. Ask what other actor calls could run during suspension.
5. Check whether two callers can start the same expensive work.
6. Check whether validation before suspension must be repeated after suspension.
7. Check whether state can be committed before suspension.
8. Check whether coordination state is written before suspension.
9. Check whether hot actor calls can be batched.
10. Recommend the smallest change that preserves the actor's consistency model.
11. Include a validation path.

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

This has no data race on `cache`, but it has a reentrancy window.

Task A can see a cache miss and suspend while loading. Task B can then enter the actor, see the same miss, and start a second load. Actor isolation protected the dictionary. It did not deduplicate the async operation.

This can cause:

* duplicate network requests;
* duplicate disk reads;
* duplicate decoding;
* unnecessary battery and bandwidth usage;
* extra latency under load;
* inconsistent side effects if the load is not idempotent.

## In-flight task pattern

Use an in-flight table when the actor must deduplicate shareable work across suspension points.

```swift
protocol ResourceLoading: Sendable {
    func loadResource(for key: ResourceKey) async throws -> Resource
}

struct ResourceKey: Hashable, Sendable {
    let rawValue: String
}

struct Resource: Sendable {
    let data: Data
}

actor ResourceCatalog {
    private var cache: [ResourceKey: Resource] = [:]
    private var inFlight: [ResourceKey: Task<Resource, Error>] = [:]
    private let loader: any ResourceLoading

    init(loader: any ResourceLoading) {
        self.loader = loader
    }

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

Use this pattern when:

* duplicate work is expensive;
* many callers may request the same missing value;
* the result can be shared safely;
* the load is idempotent or safe to share;
* the actor owns the shared operation;
* the cache or operation has a clear lifetime.

Do not use this pattern mechanically for every async call. It is most useful when duplicate work is a real cost or correctness problem.

### Swift 6 and `Sendable`

In Swift 6 strict concurrency, this pattern assumes that values crossing concurrency domains are safe to share.

Prefer:

* `Sendable` keys;
* `Sendable` results;
* `Sendable` loader protocols or dependencies;
* value types for cache keys and render-ready results;
* explicit actor boundaries for non-sendable mutable services.

Avoid hiding `Sendable` problems with unchecked annotations until the ownership model is clear.

If a dependency is not safely shareable, do not move it into an unstructured task just to satisfy the in-flight pattern. Keep the dependency isolated, redesign the boundary, or make the shared work return a sendable snapshot.

### Task choice

The shared task should not perform expensive synchronous work on the actor executor before its first suspension.

If loading includes heavy parsing, decoding, compression, hashing, or transformation, keep that work outside actor isolation and make the concurrency boundary explicit.

For CPU-heavy work, consider whether a task group, bounded parallelism, or a dedicated worker abstraction is more appropriate than storing one `Task` per key.

### Cleanup

Always remove in-flight entries on both success and failure.

Prefer `defer` when it keeps cleanup simple:

```swift
inFlight[key] = task

do {
    defer {
        inFlight[key] = nil
    }

    let resource = try await task.value
    cache[key] = resource
    return resource
} catch {
    throw error
}
```

Be careful with `defer` if cleanup depends on whether the result should be cached, retried, or preserved for future waiters.

### Cancellation policy

For shared in-flight work, always define cancellation policy:

* what happens if one waiter is cancelled;
* what happens if all waiters are cancelled;
* whether the underlying shared task should continue for future callers;
* whether failed loads should be retried immediately or throttled;
* whether cancellation should remove the in-flight entry;
* whether task priority is appropriate for shared work.

For cache loads, cancelling the underlying operation when one caller cancels is often wrong because other callers may still need the result.

For owner-scoped work, cancellation may need to cancel the underlying task.

Document the policy near the code. Cancellation behavior is part of the consistency model, not an implementation detail.

## State checks across await

Risky:

```swift
actor DownloadQuota {
    private var remainingMegabytes: Int
    private let audit: AuditLogging

    init(remainingMegabytes: Int, audit: AuditLogging) {
        self.remainingMegabytes = remainingMegabytes
        self.audit = audit
    }

    func approveDownload(size: Int) async throws {
        guard remainingMegabytes >= size else {
            throw QuotaError.notEnoughCapacity
        }

        await audit.logApproval(size)

        remainingMegabytes -= size
    }
}
```

The validation happens before suspension. Another caller may consume quota while `audit.logApproval(_:)` is running. When the original method resumes, the earlier check may be stale.

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

This makes the quota reservation part of the short actor-isolated section.

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

The second check is not redundant. It protects the business invariant after the possible reentrancy boundary.

Use this rule for:

* quotas;
* balances;
* inventory;
* counters;
* rate limits;
* permission checks;
* state machines;
* “only if still current” workflows.

## Transaction-like workflows

For transaction-like workflows, split the operation into short actor-isolated state transitions and long external work.

Prefer this shape:

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

The token should represent the state transition that was approved by the actor. It can help reject stale completions:

```swift
actor SyncStateMachine {
    private var currentSyncID: UUID?

    func beginSync() throws -> UUID {
        guard currentSyncID == nil else {
            throw SyncError.alreadyRunning
        }

        let id = UUID()
        currentSyncID = id
        return id
    }

    func finishSync(token: UUID, result: Result<Void, Error>) {
        guard currentSyncID == token else {
            return
        }

        currentSyncID = nil

        // Commit result-specific state here.
    }
}
```

This makes the consistency model explicit:

* starting work is a short actor transition;
* external work does not hold the actor;
* finishing work is validated against the current state;
* stale completions do not overwrite newer state.

## Actor contention and queue buildup

Reentrancy prevents a suspended actor method from blocking the actor forever, but actors can still become bottlenecks.

Look for:

* many tasks awaiting the same actor;
* hot actor methods called once per item;
* expensive synchronous work inside actor methods;
* repeated actor hops in tight loops;
* logging, metrics, or analytics funnels;
* CPU-heavy work that does not need actor isolation;
* overly broad actors that protect unrelated state;
* small actor methods called at very high frequency.

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

Risky:

```swift
for item in items {
    let status = await store.status(for: item.id)
    rows.append(RowModel(item: item, status: status))
}
```

Prefer one actor hop that returns a snapshot:

```swift
let statuses = await store.statuses(for: items.map(\.id))

let rows = items.map { item in
    RowModel(item: item, status: statuses[item.id])
}
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

Do not use `nonisolated` as a performance escape hatch for logic that depends on actor state. If the method needs actor state, pass an explicit snapshot into a nonisolated helper:

```swift
actor DocumentStore {
    private var documents: [Document.ID: Document] = [:]

    func tokens(for id: Document.ID) throws -> [Token] {
        guard let document = documents[id] else {
            throw DocumentError.notFound
        }

        return Self.tokenize(document)
    }

    private nonisolated static func tokenize(_ document: Document) -> [Token] {
        Tokenizer.tokenize(document.text)
    }
}
```

Do not split state across many actors just to increase parallelism. Split actors only when the split matches the consistency model.

A good actor protects one clear consistency domain. A poor actor either protects too much unrelated state or protects too little state to maintain an invariant.

## Decision rules

* Treat every `await` inside an actor method as a possible state invalidation point.
* Do not assume an `async` actor method is atomic from entry to return.
* Write coordination state before suspension when deduplicating work.
* Use an in-flight table when duplicate work is expensive and shareable.
* Make cancellation policy explicit for shared in-flight work.
* Commit state before `await` when the business rule allows it.
* Re-check state after `await` when external work must happen first.
* Keep transaction-like state transitions short and synchronous when possible.
* Use tokens or operation IDs to reject stale completions.
* Batch actor calls on hot paths.
* Move CPU-heavy work out of actor isolation when it does not need actor state.
* Use `nonisolated` only for logic that does not access actor-isolated state.
* Do not split one consistency domain across multiple actors without a clear invariant.
* Do not use actor isolation as a substitute for workflow design.

## Common mistakes

* Assuming an actor method is atomic from entry to return.
* Reading state, awaiting, then mutating based on the stale read.
* Using a cache dictionary without tracking in-flight loads.
* Writing `inFlight[key]` after the first suspension.
* Cancelling a shared in-flight task when only one waiter cancels.
* Forgetting to remove in-flight entries on failure.
* Ignoring Swift 6 `Sendable` requirements around shared tasks and dependencies.
* Putting expensive CPU work inside an actor because the actor owns related data.
* Calling an actor once per item in a tight loop.
* Returning non-sendable mutable state from an actor.
* Splitting one consistency domain across multiple actors without a clear invariant.
* Adding locks inside actors before understanding the reentrancy problem.
* Treating actor queue buildup as a reason to remove isolation rather than narrowing isolated work.
* Using `nonisolated` to bypass actor isolation for state-dependent logic.

## Validation

For duplicate work:

* add request identifiers in logs;
* count requests per cache key;
* stress test many concurrent callers requesting the same key;
* verify only one underlying load starts for a cache miss;
* test success, failure, retry, and cancellation paths;
* verify the in-flight entry is removed after success and failure.

For stale state after `await`:

* write tests with concurrent callers;
* add controlled suspension points using test doubles;
* verify counters, quotas, balances, and state transitions cannot go negative or out of order;
* test stale completion cases;
* test cancellation between validation and commit.

For actor contention:

* use Instruments to inspect actor timelines and task waiting when available;
* compare per-item actor calls with batched calls;
* use signposts around actor APIs on hot paths;
* measure before and after batching;
* inspect whether CPU-heavy work is running inside actor-isolated methods.

Do not call the fix successful only because the data race is gone. Actor reentrancy problems are usually correctness, duplication, latency, cancellation, or throughput problems.

## Review checklist

Use this checklist for actor methods:

* [ ] Does the method contain `await`?
* [ ] Is actor state read before `await` and trusted after `await`?
* [ ] Can another caller enter during suspension and change the state?
* [ ] Could two callers start the same expensive work?
* [ ] Should in-flight work be tracked?
* [ ] Is coordination state written before suspension?
* [ ] Is in-flight cleanup handled on success and failure?
* [ ] Is the cancellation policy explicit?
* [ ] Are Swift 6 `Sendable` requirements satisfied?
* [ ] Can state be committed before suspension?
* [ ] If not, is state re-validated after suspension?
* [ ] Does the workflow need an operation token to reject stale completions?
* [ ] Is the actor doing expensive work that does not need isolation?
* [ ] Are hot actor calls batched?
* [ ] Is the actor protecting one clear consistency domain?
* [ ] Would a smaller isolated section fit better?
* [ ] Is `nonisolated` used only for code that does not touch actor-isolated state?
* [ ] Is the proposed fix validated with a stress test, logs, signposts, tests, or Instruments?
