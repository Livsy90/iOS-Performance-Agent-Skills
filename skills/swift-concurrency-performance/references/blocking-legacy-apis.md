# Blocking Legacy APIs

Use this reference when the task involves semaphores, synchronous file I/O, blocking networking, locks, callback APIs, old SDKs, or async wrappers around blocking work.

This reference helps distinguish true suspension from blocking, and prevents unsafe fixes such as hiding blocking work behind `async`.

## Contents

- [Core model](#core-model)
- [When this matters](#when-this-matters)
- [Decision rules](#decision-rules)
- [Common blocking patterns](#common-blocking-patterns)
- [Semaphores and synchronous waiting](#semaphores-and-synchronous-waiting)
- [Synchronous file I/O](#synchronous-file-io)
- [Blocking networking and old SDKs](#blocking-networking-and-old-sdks)
- [Locks in async code](#locks-in-async-code)
- [Callback APIs](#callback-apis)
- [Async wrappers around blocking work](#async-wrappers-around-blocking-work)
- [Review checklist](#review-checklist)
- [Validation](#validation)

## Core model

Swift Concurrency is built around suspension.

A suspended task frees the underlying thread so other work can run. A blocked thread does not. This distinction matters because blocking inside async code can starve the cooperative executor, reduce throughput, delay unrelated tasks, and make UI latency worse.

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

## When this matters

Look for blocking legacy API risks when code:

- calls `wait()`, `sync`, `sleep`, synchronous file APIs, or blocking SDK methods from async functions;
- wraps a callback API by blocking until a callback arrives;
- uses a semaphore to turn async or callback work into synchronous work;
- performs large file reads, writes, decoding, or database work inside a task without checking where it runs;
- uses locks around state that may interact with async code;
- bridges old delegate/callback APIs into async APIs;
- tries to “fix” UI latency by adding `Task {}` around blocking work;
- uses `Task.detached` to hide blocking work without controlling lifetime, priority, or cancellation.

## Decision rules

- Prefer suspension over blocking.
- Do not call a wrapper safe just because its function is marked `async`.
- Check what the underlying API does. If it blocks a thread, the async wrapper still blocks unless the work is moved to an appropriate non-cooperative execution context.
- Avoid semaphores as a bridge between async and sync worlds.
- Do not hold locks across `await`.
- Keep critical sections small and synchronous.
- Prefer native async APIs when available.
- For legacy callback APIs, prefer checked continuations when the API is naturally asynchronous.
- For truly blocking APIs, isolate them behind a small boundary and validate thread usage.
- Preserve cancellation semantics when bridging APIs.
- Do not use `Task.detached` as a generic blocking-work escape hatch.
- If blocking work cannot be removed, make the trade-off explicit and measure it.

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

The important part is not the exact wrapper name. The important part is that `profileStore.readProfileData()` has a known execution strategy, cancellation behavior, and validation path.

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

This blocks the current thread until the callback fires. If the callback depends on work scheduled to the same saturated executor or queue, this can also contribute to deadlocks or starvation.

Prefer a continuation for naturally asynchronous callback APIs:

```swift
func fetchUser() async throws -> User {
    try await withCheckedThrowingContinuation { continuation in
        legacyClient.fetchUser { result in
            continuation.resume(with: result)
        }
    }
}
```

Then review continuation safety separately:

- Does every path resume?
- Can any path resume twice?
- What happens on cancellation?
- What happens if the callback never arrives?
- Is the legacy operation cancellable?

Read `references/continuation-safety.md` when the resume contract is non-trivial.

## Synchronous file I/O

Synchronous file I/O can be acceptable for tiny, predictable operations off the critical path. It becomes risky when it happens on the main actor, during launch, inside hot UI interactions, or across many concurrent tasks.

Risky:

```swift
@MainActor
final class ReportViewModel: ObservableObject {
    func openReport(_ url: URL) async throws {
        let data = try Data(contentsOf: url)
        report = try ReportParser.parse(data)
    }
}
```

This can block UI responsiveness because the synchronous read and parse are inside main-actor-isolated code.

Prefer separating UI state from loading/parsing work:

```swift
@MainActor
final class ReportViewModel: ObservableObject {
    func openReport(_ url: URL) async throws {
        let loaded = try await reportLoader.load(url)
        report = loaded
    }
}
```

Then inspect `reportLoader.load(_:)`:

- Does it avoid the main actor?
- Does it handle cancellation?
- Does it bound concurrent file work?
- Does parsing happen outside UI isolation?
- Is the operation measured on realistic files?

## Blocking networking and old SDKs

Some old SDKs expose methods that look simple but block internally.

Risky:

```swift
func refreshRemoteConfig() async throws {
    try legacySDK.refreshSynchronously()
}
```

Wrapping this in `async` does not make it non-blocking.

Better options, in order:

1. Use the SDK's native async or callback API if it has one.
2. Bridge a truly asynchronous callback API with checked continuations.
3. If the SDK only exposes blocking calls, isolate the call behind a narrow adapter and document the execution trade-off.
4. Add cancellation or timeout behavior if the SDK supports it.
5. Validate whether the call blocks cooperative threads, the main thread, or a private SDK thread.

Avoid:

```swift
Task.detached {
    try legacySDK.refreshSynchronously()
}
```

as a default answer. Detached tasks change lifetime, priority inheritance, task-local values, cancellation, and actor isolation. They may be appropriate only when that independent lifetime is intentional and validated.

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

Prefer keeping locked sections small and synchronous:

```swift
lock.lock()
let cached = cache[key]
lock.unlock()

if let cached {
    return cached
}

let value = try await fetchValue()

lock.lock()
cache[key] = value
lock.unlock()

return value
```

This example may still allow duplicate in-flight work. If duplicate work matters, use an actor with in-flight tracking or another explicit coordination mechanism. Read `references/actor-reentrancy.md` when the problem is duplicate work after suspension.

## Callback APIs

Callback APIs fall into two broad categories.

Some are naturally asynchronous:

```swift
legacyClient.fetchUser { result in
    ...
}
```

These are usually good candidates for checked continuations.

Some only appear asynchronous but do heavy synchronous work before returning:

```swift
legacyClient.startExpensiveOperation { result in
    ...
}
```

Before recommending a continuation bridge, check:

- Does the method return quickly?
- Does it perform synchronous setup, file work, networking, or parsing before returning?
- Which queue or thread invokes the callback?
- Is cancellation supported?
- Can the callback be called multiple times?
- Can the callback never be called?
- Does the callback retain the owner or producer?

If the callback may be called repeatedly, it may need `AsyncStream` rather than a continuation. Read `references/asyncsequence-and-stream-cleanup.md` for producer cleanup and buffering.

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
    func image(for key: String) async throws -> UIImage {
        try Task.checkCancellation()
        return try await storage.loadDecodedImage(for: key)
    }
}
```

Then verify that `storage.loadDecodedImage(for:)` has a real implementation strategy:

- native async I/O where available;
- bounded work if many images can be requested;
- parsing or decoding outside main-actor isolation;
- cancellation checks before expensive stages;
- no unbounded `Task.detached` fan-out;
- signposts or measurements for before/after comparison.

## Review checklist

When reviewing code that may block inside Swift Concurrency, check:

- [ ] Is the function marked `async` only because it calls blocking work?
- [ ] Does the underlying API suspend, callback asynchronously, or block a thread?
- [ ] Can the work run on `MainActor`?
- [ ] Can the work run during launch, first interaction, scrolling, typing, or animation?
- [ ] Are semaphores, `wait()`, `sync`, or `Thread.sleep` used?
- [ ] Is synchronous file I/O used on a hot path or inside UI isolation?
- [ ] Does a legacy SDK call block internally?
- [ ] Are locks held across `await`?
- [ ] Does a callback bridge use checked continuations correctly?
- [ ] Does cancellation stop the underlying operation, or only cancel the Swift task?
- [ ] Is parallel blocking work bounded?
- [ ] Is `Task.detached` being used to hide blocking work?
- [ ] Is there a measurement plan before and after the change?

## Validation

Use validation that can reveal blocking behavior directly.

Useful signals:

- main thread stalls during the operation;
- cooperative pool threads blocked in synchronous calls;
- long wall-clock time with low useful CPU progress;
- many tasks waiting behind blocking operations;
- actor queue buildup caused by a blocking isolated method;
- UI latency during file I/O, SDK calls, parsing, or callback setup;
- memory growth from many blocked or pending tasks;
- cancellation requested but underlying work continues.

Recommended validation tools:

- Instruments Time Profiler to see where threads spend time;
- Swift Concurrency instruments when investigating task lifetime, actor hops, and executor behavior;
- Hangs or Main Thread Checker-style investigation for UI stalls;
- signposts around async boundaries and blocking calls;
- cancellation tests for navigation, timeout, and owner deallocation;
- stress tests with many inputs when blocking work can fan out;
- production telemetry for rare tail-latency issues.

A good final recommendation should say what blocking pattern is suspected, why it matters, what the smallest safe change is, and how to prove that the change reduced blocking rather than only moved it somewhere less visible.
