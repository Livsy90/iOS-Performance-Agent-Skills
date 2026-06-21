# Blocking Legacy APIs

Use this reference when the task involves semaphores, synchronous file I/O, blocking networking, locks, callback APIs, old SDKs, blocking SDK calls, or async wrappers around blocking work.

This reference helps distinguish true suspension from blocking, and prevents unsafe fixes such as hiding blocking work behind `async`, `Task {}`, or `Task.detached`.

## Contents

* [Core model](#core-model)
* [When this matters](#when-this-matters)
* [Decision rules](#decision-rules)
* [Common blocking patterns](#common-blocking-patterns)
* [Semaphores and synchronous waiting](#semaphores-and-synchronous-waiting)
* [Synchronous file I/O](#synchronous-file-io)
* [Blocking networking and old SDKs](#blocking-networking-and-old-sdks)
* [Bounded blocking adapters](#bounded-blocking-adapters)
* [Locks in async code](#locks-in-async-code)
* [Callback APIs](#callback-apis)
* [Async wrappers around blocking work](#async-wrappers-around-blocking-work)
* [Review checklist](#review-checklist)
* [Validation](#validation)

## Core model

Swift Concurrency is built around suspension.

A suspended task frees the underlying thread so other work can run. A blocked thread does not. This distinction matters because blocking inside async code can starve cooperative executor threads, reduce throughput, delay unrelated tasks, and make UI latency worse.

Do not treat this as equivalent:

```swift
try await Task.sleep(for: .seconds(1))
```

and this:

```swift
Thread.sleep(forTimeInterval: 1)
```

The first suspends the task. The second blocks a thread.

The same principle applies to semaphores, synchronous I/O, blocking SDK calls, synchronous networking wrappers, and locks held across async boundaries.

An `async` function can still block:

```swift
func loadProfile() async throws -> Profile {
    let data = try Data(contentsOf: profileURL)
    return try JSONDecoder().decode(Profile.self, from: data)
}
```

The signature tells the caller that the function may suspend. It does not prove that the implementation avoids blocking.

## When this matters

Look for blocking legacy API risks when code:

* calls `wait()`, `sync`, `sleep`, synchronous file APIs, or blocking SDK methods from async functions;
* wraps a callback API by blocking until a callback arrives;
* uses a semaphore to turn async or callback work into synchronous work;
* performs large file reads, writes, decoding, parsing, compression, hashing, or database work inside a task without checking where it runs;
* uses locks around state that may interact with async code;
* bridges old delegate or callback APIs into async APIs;
* tries to “fix” UI latency by adding `Task {}` around blocking work;
* uses `Task.detached` to hide blocking work without controlling lifetime, priority, cancellation, or fan-out;
* allows many Swift tasks to call the same blocking API concurrently;
* calls blocking code from an actor-isolated method;
* calls blocking code from `MainActor`;
* performs blocking work during launch, first interaction, scrolling, typing, animation, or other latency-sensitive flows.

## Decision rules

* Prefer suspension over blocking.
* Do not call a wrapper safe just because its function is marked `async`.
* Check what the underlying API does. If it blocks a thread, the async wrapper still blocks unless the work is moved behind an explicit execution boundary designed for blocking work.
* Prefer native async APIs when available.
* For legacy callback APIs, prefer checked continuations only when the API is naturally asynchronous and one-shot.
* Avoid semaphores as a bridge between async and sync worlds.
* Do not hold locks across `await`.
* Keep critical sections small and synchronous.
* For truly blocking APIs, isolate them behind a narrow adapter and validate thread usage.
* If blocking work cannot be removed, bound concurrency explicitly.
* Do not allow one blocking call per Swift task without a limit.
* Preserve cancellation semantics when bridging APIs.
* Remember that Swift task cancellation does not automatically interrupt a blocking system call or synchronous SDK method.
* Do not use `Task.detached` as a generic blocking-work escape hatch.
* If blocking work cannot be removed, make the trade-off explicit and measure it.

## Common blocking patterns

Risky:

```swift
func loadProfile() async throws -> Profile {
    let data = try Data(contentsOf: profileURL)
    return try JSONDecoder().decode(Profile.self, from: data)
}
```

The function is `async`, but `Data(contentsOf:)` is still synchronous. If this runs on a cooperative executor thread, it can block that thread until file I/O completes.

Prefer using an API or boundary that makes the blocking behavior explicit:

```swift
func loadProfile() async throws -> Profile {
    let data = try await profileStore.readProfileData()
    return try decoder.decode(Profile.self, from: data)
}
```

The important part is not the exact wrapper name. The important part is that `profileStore.readProfileData()` has a known execution strategy, cancellation behavior, concurrency limit, and validation path.

A good review should ask:

```text
Does this implementation suspend, callback asynchronously, or block a thread?
If it blocks, where does it block?
How many such blocking calls can run at once?
What happens on cancellation?
What happens under slow disk, slow network, or old device conditions?
```

## Semaphores and synchronous waiting

Semaphores are one of the most common unsafe bridges between callback code and async code.

Risky:

```swift
func fetchUser() async throws -> User {
    let semaphore = DispatchSemaphore(value: 0)
    var result: Result<User, Error>?

    legacyClient.fetchUser { response in
        result = response
        semaphore.signal()
    }

    semaphore.wait()

    return try result!.get()
}
```

This blocks the current thread until the callback fires. If the callback depends on work scheduled to the same saturated executor or queue, this can also contribute to starvation or deadlock-like failures.

Prefer a continuation for naturally asynchronous one-shot callback APIs:

```swift
func fetchUser() async throws -> User {
    try await withCheckedThrowingContinuation { continuation in
        legacyClient.fetchUser { result in
            continuation.resume(with: result)
        }
    }
}
```

A continuation bridge is correct only when the callback contract is clear.

Review continuation safety separately:

* Does every path resume?
* Can any path resume twice?
* What happens on cancellation?
* What happens if the callback never arrives?
* Is the legacy operation cancellable?
* Which queue or actor invokes the callback?
* Does the callback retain the owner?
* Does the API call the callback synchronously in some cases?
* Is the API one-shot, or can it emit multiple values?

Read `references/continuation-safety.md` when the resume contract is non-trivial.

Do not use a semaphore to call async code from synchronous code unless there is no viable alternative and the boundary is explicitly isolated, measured, and documented. In most application code, redesign the API boundary instead.

## Synchronous file I/O

Synchronous file I/O can be acceptable for tiny, predictable operations off the critical path. It becomes risky when it happens on the main actor, during launch, inside hot UI interactions, or across many concurrent tasks.

Risky:

```swift
@MainActor
final class ReportViewModel: ObservableObject {
    @Published private(set) var report: Report?

    func openReport(_ url: URL) async throws {
        let data = try Data(contentsOf: url)
        report = try ReportParser.parse(data)
    }
}
```

This can block UI responsiveness because the synchronous read and parse are inside main-actor-isolated code.

Prefer separating UI state from loading and parsing work:

```swift
@MainActor
final class ReportViewModel: ObservableObject {
    @Published private(set) var report: Report?

    private let reportLoader: ReportLoading

    init(reportLoader: ReportLoading) {
        self.reportLoader = reportLoader
    }

    func openReport(_ url: URL) async throws {
        let loaded = try await reportLoader.load(url)
        report = loaded
    }
}
```

Then inspect `reportLoader.load(_:)`:

* Does it avoid the main actor?
* Does it handle cancellation?
* Does it bound concurrent file work?
* Does parsing happen outside UI isolation?
* Does it avoid unbounded `Task.detached` fan-out?
* Is the operation measured on realistic files and devices?

Synchronous file I/O is especially risky when:

* the file size is user-controlled;
* the file is on slow or remote-backed storage;
* many files are loaded concurrently;
* decoding or parsing follows the read;
* the operation runs during launch or first screen construction;
* the result is awaited by UI before first feedback.

## Blocking networking and old SDKs

Some old SDKs expose methods that look simple but block internally.

Risky:

```swift
func refreshRemoteConfig() async throws {
    try legacySDK.refreshSynchronously()
}
```

Wrapping this in `async` does not make it non-blocking. A blocking network or SDK call can occupy a thread for an unpredictable amount of time.

Better options, in order:

1. Use the SDK's native async API if it has one.
2. Use the SDK's callback API if it is truly asynchronous.
3. Bridge a truly asynchronous one-shot callback API with checked continuations.
4. If the SDK only exposes blocking calls, isolate the call behind a narrow adapter and document the execution trade-off.
5. Add cancellation or timeout behavior if the SDK supports it.
6. Bound concurrency if multiple calls can run at once.
7. Validate whether the call blocks cooperative executor threads, the main thread, or a private SDK thread.

Avoid this as a default answer:

```swift
Task.detached {
    try legacySDK.refreshSynchronously()
}
```

Detached tasks change lifetime, priority inheritance, task-local values, cancellation, and actor isolation. They may be appropriate only when independent lifetime is intentional and validated.

If a detached task is considered, require an explicit answer to these questions:

* Who owns the task?
* Who cancels it?
* What priority should it run at?
* How many detached tasks can run at once?
* Does it access non-sendable state?
* Does it need task-local values?
* Does the result need to return to a specific actor?
* What happens if the user navigates away?

## Bounded blocking adapters

If blocking work cannot be removed, isolate it behind a small boundary and bound concurrency explicitly.

The goal is not to make blocking “good.” The goal is to prevent unbounded blocking from spreading through the app.

Risky:

```swift
func thumbnails(for urls: [URL]) async throws -> [UIImage] {
    try await withThrowingTaskGroup(of: UIImage.self) { group in
        for url in urls {
            group.addTask {
                try UIImageLoader.loadSynchronously(from: url)
            }
        }

        var images: [UIImage] = []

        for try await image in group {
            images.append(image)
        }

        return images
    }
}
```

This can create one blocking operation per input. With many inputs, it may overload disk I/O, memory, or executor threads.

Prefer an adapter with an explicit concurrency limit. The exact implementation may use an operation queue, a bounded worker, an actor that schedules limited work, or a native async API.

Example shape:

```swift
protocol ThumbnailLoading {
    func thumbnail(for url: URL) async throws -> UIImage
}

struct ThumbnailService {
    private let loader: ThumbnailLoading

    func thumbnails(for urls: [URL]) async throws -> [UIImage] {
        var results: [UIImage] = []
        results.reserveCapacity(urls.count)

        for chunk in urls.chunked(into: 4) {
            try Task.checkCancellation()

            let chunkImages = try await withThrowingTaskGroup(of: UIImage.self) { group in
                for url in chunk {
                    group.addTask {
                        try Task.checkCancellation()
                        return try await loader.thumbnail(for: url)
                    }
                }

                var images: [UIImage] = []

                for try await image in group {
                    images.append(image)
                }

                return images
            }

            results.append(contentsOf: chunkImages)
        }

        return results
    }
}
```

This example shows the policy, not a universal implementation. The limit should come from real constraints: file size, device class, SDK behavior, memory pressure, UI latency, and measurement.

A blocking adapter should document:

* what blocks;
* where it blocks;
* maximum concurrency;
* timeout behavior;
* cancellation behavior;
* priority policy;
* whether it runs on a private queue, SDK queue, operation queue, or native async API;
* how it was validated.

Task cancellation does not magically interrupt a blocking system call or synchronous SDK method. If the underlying API has no cancellation mechanism, document that cancellation only affects the Swift task before the call starts or after the blocking call returns.

## Locks in async code

Locks are not forbidden in Swift code, but they are dangerous when mixed casually with async suspension.

Never hold a lock across `await`.

Risky:

```swift
lock.lock()
let cached = cache[key]

if cached == nil {
    let value = try await fetchValue()
    cache[key] = value
}

lock.unlock()
```

This keeps a synchronous lock held while the task suspends. That can block other threads and create priority or progress problems.

Prefer keeping locked sections small and synchronous.

If the project has a `withLock` helper, prefer it:

```swift
let cached = lock.withLock {
    cache[key]
}

if let cached {
    return cached
}

let value = try await fetchValue()

lock.withLock {
    cache[key] = value
}

return value
```

If using manual locking, keep `defer` inside a short synchronous scope:

```swift
let cached: Value? = {
    lock.lock()
    defer { lock.unlock() }

    return cache[key]
}()

if let cached {
    return cached
}

let value = try await fetchValue()

do {
    lock.lock()
    defer { lock.unlock() }

    cache[key] = value
}

return value
```

Do not stretch a `defer { lock.unlock() }` scope across an `await`.

This pattern may still allow duplicate in-flight work. If duplicate work matters, use an actor with in-flight tracking or another explicit coordination mechanism. Read `references/actor-reentrancy.md` when the problem is duplicate work after suspension.

Also check whether the lock is needed at all. If the state belongs to one concurrency domain, an actor or a smaller isolated owner may express the consistency model more clearly.

## Callback APIs

Callback APIs fall into two broad categories.

Some are naturally asynchronous:

```swift
legacyClient.fetchUser { result in
    ...
}
```

These are usually good candidates for checked continuations when they are one-shot.

Some only appear asynchronous but do heavy synchronous work before returning:

```swift
legacyClient.startExpensiveOperation { result in
    ...
}
```

Before recommending a continuation bridge, check:

* Does the method return quickly?
* Does it perform synchronous setup, file work, networking, parsing, or locking before returning?
* Which queue or thread invokes the callback?
* Is cancellation supported?
* Can the callback be called multiple times?
* Can the callback be called synchronously before the function returns?
* Can the callback never be called?
* Does the callback retain the owner or producer?
* Is the callback API single-listener or multi-listener?

If the callback may be called repeatedly, it may need `AsyncStream` rather than a continuation. Read `references/asyncsequence-and-stream-cleanup.md` for producer cleanup and buffering.

If the callback is one-shot but the operation is cancellable, pair the continuation with a cancellation strategy. If the operation is not cancellable, document that cancellation cannot stop the underlying work once it has started.

## Async wrappers around blocking work

A common mistake is creating an async API that hides blocking work.

Risky:

```swift
struct ImageCache {
    func image(for key: String) async throws -> UIImage {
        try loadImageSynchronously(for: key)
    }
}
```

The caller sees an async function and may assume it is safe to call from many tasks. In reality, it may block executor threads, overload disk I/O, and create memory pressure.

A better wrapper makes the strategy explicit:

```swift
struct ImageCache {
    private let storage: ImageStorage

    func image(for key: String) async throws -> UIImage {
        try Task.checkCancellation()

        return try await storage.loadDecodedImage(for: key)
    }
}
```

Then verify that `storage.loadDecodedImage(for:)` has a real implementation strategy:

* native async API where available;
* dedicated SDK background mechanism where appropriate;
* bounded work if many images can be requested;
* parsing or decoding outside main-actor isolation;
* cancellation checks before expensive stages;
* timeout behavior if the underlying operation can hang;
* no unbounded `Task.detached` fan-out;
* signposts or measurements for before/after comparison.

Do not hide blocking work behind a nicer name.

Bad:

```swift
func performAsyncRefresh() async throws {
    try refreshSynchronously()
}
```

Better:

```swift
func refresh() async throws {
    try await refreshAdapter.refreshWithBoundedBlockingBoundary()
}
```

The name is less important than the documented behavior. The wrapper should make the blocking boundary visible to maintainers and reviewers.

## Review checklist

When reviewing code that may block inside Swift Concurrency, check:

* [ ] Is the function marked `async` only because it calls blocking work?
* [ ] Does the underlying API suspend, callback asynchronously, or block a thread?
* [ ] Can the work run on `MainActor`?
* [ ] Can the work run inside an actor-isolated method?
* [ ] Can the work run during launch, first interaction, scrolling, typing, or animation?
* [ ] Are semaphores, `wait()`, `sync`, or `Thread.sleep` used?
* [ ] Is synchronous file I/O used on a hot path or inside UI isolation?
* [ ] Does a legacy SDK call block internally?
* [ ] Can blocking networking wait for an unpredictable amount of time?
* [ ] Are locks held across `await`?
* [ ] Are lock scopes small, synchronous, and exception-safe?
* [ ] Does a callback bridge use checked continuations correctly?
* [ ] Is the callback one-shot, repeated, synchronous, or optional?
* [ ] Does cancellation stop the underlying operation, or only cancel the Swift task?
* [ ] If underlying cancellation is impossible, is that limitation documented?
* [ ] Is parallel blocking work bounded?
* [ ] Is `Task.detached` being used to hide blocking work?
* [ ] Is task lifetime, priority, cancellation, and ownership explicit?
* [ ] Is there a measurement plan before and after the change?

## Validation

Use validation that can reveal blocking behavior directly.

Useful signals:

* main thread stalls during the operation;
* cooperative pool threads blocked in synchronous calls;
* long wall-clock time with low useful CPU progress;
* many tasks waiting behind blocking operations;
* actor queue buildup caused by a blocking isolated method;
* UI latency during file I/O, SDK calls, parsing, or callback setup;
* memory growth from many blocked or pending tasks;
* thread growth or excessive worker creation;
* cancellation requested but underlying work continues;
* old devices show worse tail latency than newer devices;
* low power or thermal pressure makes the issue more visible.

Recommended validation tools:

* Instruments Time Profiler to see where threads spend time;
* Swift Concurrency instruments when investigating task lifetime, actor hops, and executor behavior;
* Hangs or Main Thread Checker-style investigation for UI stalls;
* signposts around async boundaries and blocking calls;
* cancellation tests for navigation, timeout, and owner deallocation;
* stress tests with many inputs when blocking work can fan out;
* tests with slow fake SDKs or slow fake storage;
* production telemetry for rare tail-latency issues.

For blocking adapters, validate:

* maximum concurrent blocking operations;
* behavior under cancellation;
* behavior under timeout;
* behavior when the underlying SDK never returns;
* memory use under many requests;
* UI responsiveness during the operation;
* whether the call blocks cooperative executor threads, a private queue, or the main thread.

A good final recommendation should say what blocking pattern is suspected, why it matters, what the smallest safe change is, what trade-off remains, and how to prove that the change reduced blocking rather than only moved it somewhere less visible.
