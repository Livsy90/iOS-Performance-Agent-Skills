# Concurrency Runtime

Use this reference when the task needs the mental model for tasks, suspension, cooperative executors, actor executors, priorities, structured concurrency, task lifetime, or why blocking async code is harmful.

This file explains the runtime model behind Swift Concurrency enough to support performance reviews. It is not a complete language guide and should not replace more focused references for cancellation, continuations, `AsyncSequence`, actor reentrancy, task groups, blocking legacy APIs, or `MainActor` responsiveness.

## Contents

* [Core model](#core-model)
* [Tasks are units of async execution](#tasks-are-units-of-async-execution)
* [Suspension is not blocking](#suspension-is-not-blocking)
* [Jobs and executors](#jobs-and-executors)
* [Actors and actor executors](#actors-and-actor-executors)
* [Structured concurrency](#structured-concurrency)
* [Priorities](#priorities)
* [Why blocking async code is harmful](#why-blocking-async-code-is-harmful)
* [Decision rules](#decision-rules)
* [Common mistakes](#common-mistakes)
* [Review examples](#review-examples)
* [Validation](#validation)
* [Related references](#related-references)
* [Source notes](#source-notes)

## Core model

Swift Concurrency is task-based.

A task is the runtime unit that runs asynchronous code. It is not the same thing as a thread. A task may run on a thread for a while, suspend at an `await`, and later resume on an appropriate executor.

The important performance model is:

```text
task != thread
await != background work
suspension != blocking
actor isolation != parallel execution
more tasks != automatically faster
priority != concurrency limit
```

When reviewing performance, first identify which of these concepts the code is relying on incorrectly.

Common wrong assumptions:

* “This function is `async`, so it cannot block.”
* “There is an `await`, so the work moved off the main actor.”
* “This code creates many tasks, so it must be faster.”
* “This actor protects state, so the whole method is atomic.”
* “This detached task is a safe background task.”
* “Raising priority fixes throughput.”

## Tasks are units of async execution

A task represents an asynchronous operation.

Every async function runs as part of some task. A single task executes one piece of code at a time. Calling another async function does not automatically create another concurrent operation. It continues the same logical task unless the code explicitly creates child or unstructured tasks.

Sequential async code is still sequential:

```swift
let profile = await loadProfile()
let settings = await loadSettings()
let recommendations = await loadRecommendations()
```

This code has suspension points, but the operations are still ordered. It may be responsive because it does not block a thread while waiting, but it is not parallel.

Parallelism requires a concurrency construct:

```swift
async let profile = loadProfile()
async let settings = loadSettings()
async let recommendations = loadRecommendations()

let loaded = await (profile, settings, recommendations)

let result = HomeData(
    profile: loaded.0,
    settings: loaded.1,
    recommendations: loaded.2
)
```

If the child operations can throw, make the await point explicit:

```swift
async let profile = loadProfile()
async let settings = loadSettings()
async let recommendations = loadRecommendations()

let loaded = try await (profile, settings, recommendations)

let result = HomeData(
    profile: loaded.0,
    settings: loaded.1,
    recommendations: loaded.2
)
```

Use parallelism only when the operations are independent and the additional scheduling, cancellation, memory, and error-handling complexity is justified.

Do not parallelize code only because functions are async. Async allows suspension. It does not imply independence.

## Suspension is not blocking

An `await` marks a possible suspension point.

At suspension, the current async function may stop running. The thread that was executing it can be returned to the system so other work can run. Later, when the awaited operation is ready, the task can resume.

That is the key difference between suspension and blocking:

```text
Suspension:
task pauses, thread can run other work

Blocking:
thread waits and cannot run other cooperative work
```

This is why async code can scale better than one-thread-per-operation designs. Many tasks can be suspended without requiring one blocked thread for each pending operation.

However, `await` does not mean:

* the code moved to a background thread;
* the work is parallel;
* the work is cheap;
* the current actor is no longer relevant;
* cancellation is automatically observed;
* the UI cannot still be delayed by the awaited result;
* the underlying operation is non-blocking.

If the UI needs the value before it can render or respond, the user may still experience latency even though the code is asynchronous.

If the awaited operation is implemented with synchronous file I/O, blocking SDK calls, semaphores, locks held too long, or CPU-heavy work, the async call may still hurt responsiveness.

## Jobs and executors

At runtime, executable pieces of async work are scheduled onto executors.

For performance review, use this simplified model:

```text
Task
  owns async operation state, priority, cancellation, and task-local context

Job
  a schedulable piece of work that can run part of a task

Executor
  decides where and when a job runs
```

You usually do not interact with jobs directly in app code. They are useful as a mental model because they explain why blocking a cooperative executor thread is harmful: the executor expected the job to run until it suspends or completes, not to occupy a thread while waiting on a semaphore or synchronous operation.

Executors are allowed to schedule many async jobs across a limited set of threads. That model depends on async work suspending instead of blocking.

Do not rely on a specific thread count or exact scheduling behavior. Executor scheduling is an implementation detail. Validate performance with traces instead of assuming a fixed runtime shape.

For review purposes, ask:

```text
Does this job suspend quickly, complete quickly, or block a thread?
Does it accidentally run long synchronous work on the main actor?
Does it create many more jobs than the system or downstream resource can usefully handle?
```

## Actors and actor executors

Actors protect isolated mutable state.

An actor does not make its internal work parallel. Actor isolation means that actor-isolated mutable state is accessed through the actor’s executor. Only one actor-isolated synchronous region can execute on that actor at a time.

Actor isolation serializes access to actor-isolated state only while a job is actively running on the actor. When an actor-isolated method suspends at `await`, other actor-isolated work may run before the original method resumes.

This is good for data-race safety, but it can become a correctness or performance bottleneck.

Risk pattern:

```swift
actor ImageCache {
    private var storage: [URL: UIImage] = [:]

    func image(for url: URL) async throws -> UIImage {
        if let image = storage[url] {
            return image
        }

        let image = try await downloadAndDecode(url)
        storage[url] = image
        return image
    }
}
```

The actor protects `storage`, but this method suspends during `downloadAndDecode`. While it is suspended, other calls may enter the actor. That can be correct, but it means actor isolation alone does not prevent duplicate in-flight work.

Also check for hot actors:

```text
many callers -> one actor -> long isolated work -> actor queue buildup
```

Common causes:

* large CPU work inside actor-isolated methods;
* chatty APIs that require many small actor hops;
* storing unrelated state in one broad actor;
* doing I/O or decoding inside actor-isolated methods;
* repeatedly entering the same actor from a task group;
* assuming actor isolation is a throughput optimization.

Actors are primarily a correctness and isolation tool. They can improve performance when they remove locks or simplify state coordination, but they can also serialize too much work.

Use `references/actor-reentrancy.md` when the issue involves state before and after `await`, duplicate requests, in-flight work, cache stampedes, or stale validation.

## Structured concurrency

Structured concurrency gives async work a parent-child shape.

Child tasks created in structured scopes are bounded by that scope. The parent must wait for the child work before the scope exits. This makes lifetime, cancellation, priority, and error propagation easier to reason about.

Use structured concurrency when:

* the parent owns the work;
* the work is part of the same operation;
* cancellation should flow from parent to children;
* errors should be collected or propagated through the parent;
* the code should not leave orphaned tasks behind.

Examples:

```swift
async let user = loadUser()
async let permissions = loadPermissions()

let loaded = try await (user, permissions)

let model = UserModel(
    user: loaded.0,
    permissions: loaded.1
)
```

```swift
try await withThrowingTaskGroup(of: Thumbnail.self) { group in
    for item in items {
        group.addTask {
            try await makeThumbnail(for: item)
        }
    }

    var thumbnails: [Thumbnail] = []

    for try await thumbnail in group {
        thumbnails.append(thumbnail)
    }

    return thumbnails
}
```

The second example may still be risky for large `items` because it creates all child tasks eagerly. Use the bounded task groups reference when input size can grow.

Unstructured tasks have different semantics:

```swift
Task {
    await service.refresh()
}
```

This creates work that is not automatically awaited by the current scope. That can be correct for UI-triggered work, app-level services, or fire-and-forget style side effects, but the lifetime owner must be explicit.

Detached tasks are even more independent:

```swift
Task.detached {
    await backgroundIndexer.rebuild()
}
```

Use detached tasks only when intentionally creating independent work outside the current structured task and actor context. Make priority, cancellation, task-local needs, ownership, and result delivery explicit.

Detached tasks are not a default way to “make something background.” They can hide lifetime, cancellation, isolation, and priority problems.

## Priorities

Tasks carry priority information.

Priority is a scheduling signal, not a correctness mechanism, not a cancellation mechanism, and not a guarantee that work runs immediately.

Priority is also not a concurrency limit. A high-priority task group can still create too many child tasks and overload CPU, memory, a database, or a backend.

Review priority when:

* UI work waits behind background work;
* many background tasks compete with user-initiated work;
* detached tasks do not have the priority behavior the caller expected;
* task groups perform expensive work without considering user-visible urgency;
* a high-priority task waits on lower-priority or serialized work;
* priority changes are being used instead of reducing unnecessary work.

Do not “fix” performance by randomly raising priorities. Higher priority can make the current operation faster by making other work slower. It can also increase energy use or worsen contention.

Prefer to reduce unnecessary work, bound parallelism, narrow actor isolation, and avoid blocking before changing priority.

If priority matters, make it part of the design:

```text
What is user-visible?
What can run later?
What can be cancelled?
What is the maximum amount of concurrent work?
Which resource is the bottleneck?
```

## Why blocking async code is harmful

Swift Concurrency is designed around cooperative suspension.

Blocking calls break that model because they occupy a thread while the task is not making progress.

Risky patterns inside async paths:

```swift
semaphore.wait()
```

```swift
Thread.sleep(forTimeInterval: 1)
```

```swift
let data = try Data(contentsOf: url)
```

```swift
queue.sync {
    expensiveWork()
}
```

```swift
lock.lock()
defer { lock.unlock() }
legacyBlockingCall()
```

Locks are not automatically wrong. The risk is holding a lock while running long blocking work, while crossing async boundaries, or inside hot async paths where it blocks cooperative progress.

The issue is not only that the current operation is slow. The bigger issue is that cooperative executor threads are a shared resource. If enough async jobs block threads, unrelated tasks may stop making progress.

Symptoms may look like:

* UI hangs even though code is “async”;
* low CPU utilization but poor responsiveness;
* many tasks waiting without useful work happening;
* timeouts under load;
* actor queues growing;
* thread pool pressure;
* priority inversions;
* intermittent stalls that are hard to reproduce locally.

Prefer real async APIs when available. If a legacy API is truly blocking and cannot be replaced, isolate it deliberately and validate the impact. Do not hide blocking behavior behind an `async` function name.

Risky wrapper:

```swift
func loadFile() async throws -> Data {
    try Data(contentsOf: fileURL)
}
```

This is syntactically async if called from async code, but the file read is still synchronous and can block the executing thread.

Better options depend on the API and platform:

* use a native async API when one exists;
* move the blocking work behind a carefully managed boundary;
* limit concurrency around blocking work;
* avoid running blocking work on the main actor;
* measure under load before assuming the wrapper is safe.

Use `references/blocking-legacy-apis.md` for detailed bridging guidance.

## Decision rules

* First ask whether the code needs concurrency, suspension, isolation, or parallelism. These are different needs.
* Treat `await` as a possible suspension point, not as proof of background execution.
* Treat `Task` creation as a lifetime decision, not just a scheduling decision.
* Prefer structured concurrency when the current scope owns the work.
* Bound dynamic parallelism when the number of child tasks depends on input size.
* Do not treat priority as a concurrency limit.
* Keep actor-isolated sections small on hot paths.
* Batch actor calls when repeated hops dominate latency.
* Move CPU-heavy work out of actor isolation when it does not need isolated state.
* Avoid blocking calls inside async contexts.
* Do not use `Task.detached` unless independent lifetime and escaping inherited context are intentional.
* Do not change priority before checking work volume, blocking, actor contention, and task fan-out.
* Do not rely on exact executor thread counts or scheduling behavior.
* Validate with traces or tests when the performance effect is not obvious.

## Common mistakes

### Mistake: Treating async as parallel

```swift
let a = await loadA()
let b = await loadB()
let c = await loadC()
```

This is async but sequential. It may be correct, especially if each step depends on the previous one. Parallelize only when independence is clear.

### Mistake: Treating await as a background boundary

```swift
@MainActor
func refresh() async {
    let model = await buildLargeModel()
    self.model = model
}
```

If `buildLargeModel()` is also main-actor-isolated or does synchronous work before suspending, the UI may still stall. `await` alone does not guarantee background execution.

### Mistake: Creating more tasks than the system can usefully run

```swift
for item in items {
    group.addTask {
        await process(item)
    }
}
```

For a small collection this may be fine. For thousands of items, it can create memory pressure, scheduling overhead, and contention. Use bounded concurrency.

### Mistake: Hiding blocking work behind async APIs

```swift
func refresh() async {
    semaphore.wait()
    defer { semaphore.signal() }

    legacyRefresh()
}
```

The function looks async to callers, but it blocks a cooperative thread.

### Mistake: Using an actor as a performance queue

```swift
actor WorkQueue {
    func process(_ item: Item) {
        expensiveCPUWork(item)
    }
}
```

This serializes expensive work through the actor. If the work does not need isolated actor state, move it out of the actor-isolated section.

### Mistake: Assuming actor methods are atomic

```swift
actor TokenStore {
    private var token: Token?

    func token() async throws -> Token {
        if let token {
            return token
        }

        let loaded = try await loadToken()
        token = loaded
        return loaded
    }
}
```

This is data-race safe, but another caller may enter while `loadToken()` is suspended and start duplicate work. Actor isolation does not make the async method atomic from entry to return.

### Mistake: Assuming unstructured tasks clean themselves up

```swift
Task {
    await pollForever()
}
```

This task has no obvious owner. It may survive longer than the screen, request, or object that created it. Store, cancel, or structure it.

### Mistake: Using priority instead of reducing work

```swift
Task(priority: .high) {
    await processThousandsOfItems()
}
```

Priority may make this work compete more aggressively with other work. It does not reduce total work, limit fan-out, remove blocking, or fix actor contention.

## Review examples

### Example: async but still blocking

Risky:

```swift
func loadConfiguration() async throws -> Configuration {
    let data = try Data(contentsOf: configurationURL)
    return try JSONDecoder().decode(Configuration.self, from: data)
}
```

Review finding:

The function is async by signature, but the file read and decode are synchronous. If this runs on a cooperative executor thread during a user-visible path, it can reduce responsiveness and delay unrelated async work.

Possible recommendation:

Use a real async loading API where available, move the blocking boundary out of the user-visible path, keep decoding off the main actor, and validate with a trace that shows thread blocking removed or reduced.

### Example: actor hop on a hot path

Risky:

```swift
for id in ids {
    let value = await store.value(for: id)
    output.append(value)
}
```

Review finding:

This creates one actor hop per item. If this is a hot path, hop overhead and actor queueing may dominate.

Possible recommendation:

Add a batched actor API:

```swift
let values = await store.values(for: ids)
```

The actor can then enter isolation once, copy the required state, and return the result.

### Example: unbounded child tasks

Risky:

```swift
try await withThrowingTaskGroup(of: Result.self) { group in
    for request in requests {
        group.addTask {
            try await client.send(request)
        }
    }

    for try await result in group {
        results.append(result)
    }
}
```

Review finding:

This creates one child task per request. For a small fixed number this may be fine. For user-controlled or server-controlled input, it can create excessive fan-out.

Possible recommendation:

Use a bounded task group pattern and validate memory, request concurrency, and latency under realistic input sizes.

### Example: detached task hides lifetime

Risky:

```swift
func startIndexing() {
    Task.detached {
        await indexer.rebuild()
    }
}
```

Review finding:

The detached task has independent lifetime and does not clearly belong to the caller, screen, or service operation. It also makes cancellation and result delivery unclear.

Possible recommendation:

Use structured concurrency if the caller owns the operation. If indexing is app-level work, store the task in an explicit owner, define cancellation behavior, set priority intentionally, and validate that it does not compete with user-visible work.

## Validation

Choose validation based on the suspected runtime issue.

### UI stall or main-thread responsiveness

Use Instruments and signposts around the async boundary. Check whether the main thread or `MainActor` is doing long work, whether the UI awaits a slow result, and whether work moved away from the main actor actually stopped blocking user interaction.

### Blocked cooperative threads

Use Instruments to look for blocking calls, thread waits, semaphores, synchronous I/O, locks held during long work, or legacy APIs running inside async paths. Validate under load, not only with one local run.

### Actor contention

Look for long actor-isolated work, repeated actor hops, actor queue buildup, duplicate in-flight work, and callers waiting on one hot actor. Add signposts around actor APIs if the trace does not make the queueing obvious.

### Task explosion

Track task counts, memory growth, and input sizes. Validate bounded fan-out with realistic worst-case input.

### Priority issues

Check whether high-priority user-visible work is waiting behind lower-priority work, whether detached tasks have unclear priority, or whether priority is being used instead of fixing contention. Prefer reducing work and bounding concurrency before raising priority.

### Lifetime issues

Use logs, cancellation tests, memory graphs, or deinit probes to confirm that tasks stop when the owner disappears.

## Related references

* `mainactor-responsiveness.md` — use for UI isolation, `@MainActor`, main-thread stalls, and moving CPU-heavy work away from UI state.
* `cancellation-and-task-lifetime.md` — use for cancellation propagation, swallowed cancellation, navigation cancellation, and cancellation tests.
* `bounded-task-groups.md` — use for task groups, fan-out work, and limiting concurrency.
* `actor-reentrancy.md` — use for actor state after `await`, duplicate work, and cache stampedes.
* `blocking-legacy-apis.md` — use for semaphores, synchronous I/O, blocking SDKs, and async wrappers around legacy APIs.
* `asyncsequence-and-stream-cleanup.md` — use for `AsyncSequence`, `AsyncStream`, buffering, producer lifetime, and `onTermination`.
* `continuation-safety.md` — use for checked continuations, delegate/callback bridging, cancellation races, timeout paths, and exactly-once resume.
* `diagnostics-and-instruments.md` — use for trace interpretation and measurement workflows.

## Source notes

This reference follows the Swift Concurrency model described by Swift Evolution proposals and the Swift language documentation:

* SE-0296: async/await — https://github.com/swiftlang/swift-evolution/blob/main/proposals/0296-async-await.md
* SE-0304: Structured Concurrency — https://github.com/swiftlang/swift-evolution/blob/main/proposals/0304-structured-concurrency.md
* SE-0306: Actors — https://github.com/swiftlang/swift-evolution/blob/main/proposals/0306-actors.md
* The Swift Programming Language: Concurrency — https://docs.swift.org/swift-book/documentation/the-swift-programming-language/concurrency/
