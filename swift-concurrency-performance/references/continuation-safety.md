# Continuation Safety

Use this reference when the task involves `withCheckedContinuation`, `withCheckedThrowingContinuation`, delegate bridges, callback wrappers, timeout paths, cancellation paths, callback-after-cancel races, or exactly-once resume guarantees.

Continuations are a bridge between Swift Concurrency and non-async APIs. Review them as correctness, lifetime, isolation, and performance boundaries. A broken continuation can create stuck tasks, double resumes, leaked work, ignored cancellation, retained owners, or UI flows that never complete.

## Contents

* [Core model](#core-model)
* [When continuations are appropriate](#when-continuations-are-appropriate)
* [Decision rules](#decision-rules)
* [Exactly-once resume contract](#exactly-once-resume-contract)
* [Checked vs unsafe continuations](#checked-vs-unsafe-continuations)
* [Callback wrappers](#callback-wrappers)
* [Delegate bridges](#delegate-bridges)
* [Timeout and cancellation paths](#timeout-and-cancellation-paths)
* [Swift 6, Sendable, and isolation](#swift-6-sendable-and-isolation)
* [Lifetime and retention](#lifetime-and-retention)
* [Performance notes](#performance-notes)
* [Common mistakes](#common-mistakes)
* [Validation](#validation)
* [Review checklist](#review-checklist)

## Core model

A continuation represents a suspended async task waiting for an external event.

The external event may come from:

* a completion handler;
* a delegate callback;
* a notification;
* a legacy SDK;
* a timeout;
* cancellation;
* an operation queue;
* a manually managed service.

The review question is not only:

```text
Does this compile?
```

The review question is:

```text
Can every possible path resume the continuation exactly once, release owned resources, and stop work that is no longer needed?
```

A continuation bridge is safe only when these paths are explicit:

* success;
* failure;
* cancellation;
* timeout;
* invalid input;
* owner deallocation;
* callback-after-cancel;
* duplicate callback;
* callback never arrives.

## When continuations are appropriate

Use continuations when adapting a one-shot callback-style API into an async function.

Good candidates:

* a completion handler that returns once;
* a delegate-driven operation with a single final success or failure event;
* a legacy SDK method that has a clear terminal callback;
* a wrapper around a one-time authorization, export, import, upload, or fetch operation.

Be careful when the source can produce multiple values. A multi-event source usually belongs to `AsyncStream` or `AsyncThrowingStream`, not a one-shot continuation.

Prefer:

```swift
func loadUser(id: User.ID) async throws -> User {
    try await withCheckedThrowingContinuation { continuation in
        api.loadUser(id: id) { result in
            continuation.resume(with: result)
        }
    }
}
```

Avoid using a continuation for a stream of values:

```swift
func observePaymentEvents() async -> PaymentEvent {
    await withCheckedContinuation { continuation in
        monitor.onEvent = { event in
            continuation.resume(returning: event)
        }
    }
}
```

If more than one event can arrive, use an async stream instead.

Read `references/asyncsequence-and-stream-cleanup.md` when the source is long-lived, multi-value, buffered, or producer-owned.

## Decision rules

* Prefer `withCheckedContinuation` and `withCheckedThrowingContinuation` by default.
* Resume every continuation exactly once.
* Make success, failure, timeout, cancellation, early-return, invalid-input, and owner-deallocation paths explicit.
* Prefer `Result`-based callback wrappers when possible.
* Use `continuation.resume(with: result)` when the callback already provides `Result`.
* Do not resume from multiple independent `if` branches unless they are mutually exclusive and exhaustive.
* Do not store a continuation without a clear owner and terminal cleanup path.
* Do not use a continuation for multi-value streams.
* Do not use unsafe continuations unless profiling proves checked continuation overhead matters.
* Treat cancellation as part of the bridge design, not an afterthought.
* Treat timeout as a terminal path that must cancel or clean up the underlying operation when possible.
* Protect against callback-after-cancel and duplicate callback races.
* Keep delegate or bridge objects alive until a terminal event.
* Do not silence Swift 6 `Sendable` diagnostics without proving the bridge is safe.
* Do not mutate `@MainActor` state or non-thread-safe state from an arbitrary callback queue.

## Exactly-once resume contract

A continuation must be resumed exactly once.

Risky:

```swift
func exportReport() async throws -> ExportedReport {
    try await withCheckedThrowingContinuation { continuation in
        exporter.start { output, failure in
            if let output {
                continuation.resume(returning: output)
            }

            if let failure {
                continuation.resume(throwing: failure)
            }
        }
    }
}
```

This is risky because both values may be present, or neither may be present. The continuation may resume twice or never resume.

Prefer a single terminal result:

```swift
func exportReport() async throws -> ExportedReport {
    try await withCheckedThrowingContinuation { continuation in
        exporter.start { result in
            switch result {
            case .success(let report):
                continuation.resume(returning: report)

            case .failure(let error):
                continuation.resume(throwing: error)
            }
        }
    }
}
```

Even better, when the callback already returns `Result`:

```swift
func exportReport() async throws -> ExportedReport {
    try await withCheckedThrowingContinuation { continuation in
        exporter.start { result in
            continuation.resume(with: result)
        }
    }
}
```

For callback shapes with optional success and optional failure, make the missing-result path explicit:

```swift
func requestToken() async throws -> Token {
    try await withCheckedThrowingContinuation { continuation in
        auth.requestToken { token, error in
            switch (token, error) {
            case let (token?, _):
                continuation.resume(returning: token)

            case let (nil, error?):
                continuation.resume(throwing: error)

            case (nil, nil):
                continuation.resume(throwing: AuthError.missingResult)
            }
        }
    }
}
```

Do not let a malformed callback shape suspend the task forever.

## Checked vs unsafe continuations

Use checked continuations by default:

```swift
withCheckedContinuation { continuation in
    // bridge callback API
}

withCheckedThrowingContinuation { continuation in
    // bridge throwing callback API
}
```

Checked continuations help catch incorrect resume behavior during development.

Use unsafe continuations only when all of these are true:

* the bridge is on a measured hot path;
* checked continuation overhead is visible in profiling;
* the callback contract is simple and proven;
* tests cover success, failure, cancellation, timeout, duplicate callback, and owner-deallocation behavior;
* the code documents why unsafe continuation is justified.

Do not start with unsafe continuations as a style preference.

Continuation overhead is usually not the first performance suspect. Check blocking work, callback queue, duplicate operations, missing cancellation, and retained bridge objects before replacing checked continuations.

## Callback wrappers

For callback wrappers, inspect the callback contract before writing the async API.

Ask:

* Is the callback guaranteed to be called?
* Can it be called synchronously before the wrapping function returns?
* Can it be called more than once?
* Can success and failure both be represented at the same time?
* Can success and failure both be missing?
* Is there a cancellation token or operation handle?
* Which queue or actor invokes the callback?
* Does the callback retain `self`, a delegate, or a producer object?
* Does cancellation stop the underlying work or only cancel the Swift task?

Risky callback wrapper:

```swift
func requestToken() async throws -> Token {
    try await withCheckedThrowingContinuation { continuation in
        auth.requestToken { token, error in
            if let token {
                continuation.resume(returning: token)
            } else if let error {
                continuation.resume(throwing: error)
            }
        }
    }
}
```

If both `token` and `error` are nil, the task never resumes.

Prefer explicit fallback behavior:

```swift
func requestToken() async throws -> Token {
    try await withCheckedThrowingContinuation { continuation in
        auth.requestToken { token, error in
            switch (token, error) {
            case let (token?, _):
                continuation.resume(returning: token)

            case let (nil, error?):
                continuation.resume(throwing: error)

            case (nil, nil):
                continuation.resume(throwing: AuthError.missingResult)
            }
        }
    }
}
```

If the callback can be called repeatedly, do not use a one-shot continuation. Use `AsyncStream` or `AsyncThrowingStream`.

If the callback can be called synchronously, make sure bridge state is initialized before the call that can trigger the callback.

## Delegate bridges

Delegate-based APIs often need extra care because the final callback may arrive through a separate object.

Review:

* where the continuation is stored;
* who owns the delegate or proxy object;
* whether the SDK stores delegates strongly or weakly;
* what happens on success;
* what happens on failure;
* what happens on cancellation;
* what happens on timeout;
* what happens if the owner is deallocated;
* whether the delegate can deliver multiple terminal callbacks.

A common pattern is to use a small bridge object that owns the continuation and clears it after resume.

Example shape:

```swift
final class ExportBridge: NSObject, ExporterDelegate {
    private var continuation: CheckedContinuation<ExportedReport, Error>?

    init(continuation: CheckedContinuation<ExportedReport, Error>) {
        self.continuation = continuation
    }

    func exporterDidFinish(_ report: ExportedReport) {
        resume(.success(report))
    }

    func exporterDidFail(_ error: Error) {
        resume(.failure(error))
    }

    private func resume(_ result: Result<ExportedReport, Error>) {
        guard let continuation else {
            return
        }

        self.continuation = nil
        continuation.resume(with: result)
    }
}
```

This shape protects against repeated terminal callbacks by clearing the continuation before resuming.

A delegate bridge must be retained until a terminal event. If the SDK stores delegates weakly, the async wrapper must keep the bridge alive through an operation owner, dictionary of active bridges, service-level storage, or another explicit lifetime owner.

Example ownership shape:

```swift
final class ExportService {
    private var activeBridges: [UUID: ExportBridge] = [:]

    func export(_ document: Document) async throws -> ExportedReport {
        let id = UUID()

        return try await withCheckedThrowingContinuation { continuation in
            let bridge = ExportBridge(
                id: id,
                continuation: continuation,
                onFinish: { [weak self] id in
                    self?.activeBridges[id] = nil
                }
            )

            activeBridges[id] = bridge

            exporter.delegate = bridge
            exporter.startExport(document)
        }
    }
}
```

This is only a shape. Real code must handle cancellation, timeout, duplicate callbacks, and exporter ownership.

Do not rely on accidental retention through closures or delegates. Make bridge lifetime explicit and remove the bridge after terminal cleanup.

## Timeout and cancellation paths

Cancellation does not automatically cancel a callback-based API. The bridge must connect Swift task cancellation to the underlying operation when the legacy API supports cancellation.

For cancellable legacy APIs, prefer a bridge that keeps the operation handle and cancels it from `withTaskCancellationHandler`.

Example shape:

```swift
func upload(_ file: File) async throws -> UploadResult {
    let operation = UploadOperation(file: file)

    return try await withTaskCancellationHandler {
        try await withCheckedThrowingContinuation { continuation in
            operation.start { result in
                continuation.resume(with: result)
            }
        }
    } onCancel: {
        operation.cancel()
    }
}
```

This shape is only safe if the operation guarantees one final callback after cancellation, or if the bridge has a separate exactly-once guard.

Cancellation can race with callback registration. The bridge must handle cancellation:

* before the operation starts;
* while the callback is being installed;
* after the callback is installed;
* while the callback is executing;
* after the callback already completed.

If cancellation can stop callbacks entirely, the bridge must resume the continuation on cancellation as well. That requires synchronized state, because the callback and cancellation path may race.

Use a one-shot guard when there are multiple possible terminal sources, such as callback, timeout, cancellation, and owner deallocation.

Conceptual shape:

```swift
final class OneShotBox<Success>: @unchecked Sendable {
    private let lock = NSLock()
    private var continuation: CheckedContinuation<Success, Error>?

    init(_ continuation: CheckedContinuation<Success, Error>) {
        self.continuation = continuation
    }

    func resume(_ result: Result<Success, Error>) {
        let continuation = takeContinuation()
        continuation?.resume(with: result)
    }

    private func takeContinuation() -> CheckedContinuation<Success, Error>? {
        lock.lock()
        defer { lock.unlock() }

        let continuation = self.continuation
        self.continuation = nil
        return continuation
    }
}
```

This box clears the continuation under a lock and resumes outside the lock.

Use `@unchecked Sendable` only when synchronization really protects all mutable state. Keep the box small, auditable, and covered by tests.

Do not add timeout behavior with a racing task unless the losing path is cancelled or guarded so it cannot later resume the same continuation.

When timeout wins, cancel or clean up the underlying operation according to the API contract. A timeout that only resumes the continuation but leaves the legacy operation running may leak work or later race with the callback.

Conceptual timeout rule:

```text
timeout wins -> resume once with timeout error -> cancel/cleanup operation -> ignore late callback safely
callback wins -> resume once with callback result -> cancel timeout path -> cleanup operation/bridge
cancellation wins -> resume or cancel operation according to contract -> ignore late callback safely
```

## Swift 6, Sendable, and isolation

In Swift 6 strict concurrency, cancellation handlers and callback closures may be `@Sendable`. If they capture operation handles, bridge objects, or continuations, those captured values must be safe to share across concurrency domains.

Do not silence diagnostics mechanically.

Prefer one of these designs:

* a small lock-protected `Sendable` bridge box;
* an actor that owns bridge state;
* a service that keeps callbacks on a known isolation domain;
* immutable `Sendable` values crossing the async boundary;
* explicit `@MainActor` isolation when the legacy API is main-thread-only.

Risky:

```swift
func load() async throws -> Model {
    let operation = NonSendableOperation()

    return try await withTaskCancellationHandler {
        try await withCheckedThrowingContinuation { continuation in
            operation.start {
                continuation.resume(with: $0)
            }
        }
    } onCancel: {
        operation.cancel()
    }
}
```

This may be conceptually correct but unsafe under strict concurrency if `operation` is non-sendable and used from callback and cancellation contexts without synchronization or isolation.

Resuming a continuation from a callback is fine, but do not mutate `@MainActor` state or non-thread-safe bridge state from an arbitrary callback queue.

Risky:

```swift
api.load { result in
    self.isLoading = false
    continuation.resume(with: result)
}
```

If `self` is `@MainActor` or otherwise not thread-safe, hop or isolate correctly:

```swift
api.load { result in
    continuation.resume(with: result)

    Task { @MainActor in
        self.isLoading = false
    }
}
```

Better still, keep UI state changes outside the bridge when possible. Let the async caller update UI state after `await` on the correct actor.

## Lifetime and retention

Continuations often hide lifetime bugs.

Check:

* Does the legacy operation stay alive until it completes?
* Does the bridge object stay alive until the terminal event?
* Does the continuation retain objects longer than expected?
* Does cancellation release the callback, delegate, producer, or operation?
* Does the wrapper capture `self` strongly, and is that intended?
* Can the owner deallocate while the continuation is still pending?
* Does timeout remove the bridge and stop the operation?
* Does callback-after-cancel release retained resources?

Risky:

```swift
func loadImage() async throws -> Image {
    try await withCheckedThrowingContinuation { continuation in
        let request = imageLoader.load { result in
            continuation.resume(with: result)
        }

        request.start()
    }
}
```

If `request` is not retained by `imageLoader` or another owner, it may deallocate before completion.

Prefer explicit ownership through the underlying service, a bridge object, or an operation handle with a clear lifetime.

Risky strong capture:

```swift
func load() async throws -> Model {
    try await withCheckedThrowingContinuation { continuation in
        sdk.load { result in
            self.lastResult = result
            continuation.resume(with: result)
        }
    }
}
```

Ask whether `self` should really stay alive until the callback arrives. If not, use an explicit owner, cancellation path, or weak capture with a defined missing-owner result:

```swift
func load() async throws -> Model {
    try await withCheckedThrowingContinuation { [weak self] continuation in
        guard let self else {
            continuation.resume(throwing: BridgeError.ownerDeallocated)
            return
        }

        sdk.load { result in
            continuation.resume(with: result)
        }
    }
}
```

Do not use `[weak self]` merely to silence retention concerns. It must still resume the continuation on the missing-owner path.

## Performance notes

Continuation overhead is usually not the first performance suspect.

Look first for:

* callback APIs that call back on the main thread and then do heavy parsing;
* excessive bridging in tight loops;
* continuations that never resume and keep tasks alive;
* duplicate operations caused by multiple callers awaiting separate bridges;
* blocking legacy APIs wrapped in async functions;
* missing cancellation that lets expensive work continue after the result is no longer needed;
* bridge objects retained after the operation should have ended;
* callback queues that create main-thread stalls.

Do not replace checked continuations with unsafe continuations as a first optimization. If a continuation bridge appears in a hot path, profile first and confirm the actual cost.

If many callers may request the same underlying work, continuation safety is not enough. You may also need in-flight task deduplication or actor-owned coordination.

Read `references/actor-reentrancy.md` when duplicate in-flight work or cache stampedes are involved.

## Common mistakes

* Using a continuation for a multi-value event source.
* Missing the nil/nil result path in callback wrappers.
* Resuming from multiple independent branches.
* Forgetting timeout, cancellation, owner-deallocation, or early-return paths.
* Storing a continuation without clearing it after resume.
* Keeping a delegate bridge alive accidentally rather than explicitly.
* Letting a delegate bridge deallocate before the terminal callback.
* Assuming task cancellation cancels the underlying callback API.
* Racing timeout/cancellation/callback paths without a one-shot guard.
* Resuming a continuation under a lock while holding shared state.
* Wrapping a blocking API in a continuation and calling it “async.”
* Switching to unsafe continuations without measurement.
* Ignoring which queue or actor invokes the callback.
* Mutating `@MainActor` state from an arbitrary callback queue.
* Capturing `self` strongly in a bridge without checking lifetime.
* Using `[weak self]` but failing to resume when `self` is gone.
* Silencing Swift 6 `Sendable` warnings instead of fixing ownership or isolation.

## Validation

Use validation based on the risk.

For correctness:

* test success;
* test failure;
* test invalid input;
* test nil or malformed result if the callback shape allows it;
* test cancellation before operation start;
* test cancellation after callback registration;
* test cancellation while callback is racing;
* test timeout if timeout is supported;
* test duplicate callback delivery if the legacy API can misbehave;
* test callback-after-cancel;
* test owner deallocation while the operation is pending.

For lifetime:

* verify the operation stays alive until terminal completion;
* verify the delegate bridge stays alive until terminal completion;
* verify the bridge is released after terminal completion;
* verify cancellation releases callbacks, delegates, producers, and operation handles;
* verify timeout removes active bridge entries;
* verify late callbacks do not retain owners forever.

For performance and responsiveness:

* use Instruments to look for stuck tasks, blocked threads, main-thread callbacks, and retained bridge objects;
* add signposts around the async wrapper and the underlying callback;
* log operation identifiers to detect duplicate in-flight work;
* use memory tools when pending continuations may retain large values or long-lived producers;
* verify that cancellation stops the underlying work, not only the Swift task waiting for it;
* verify callback queues do not perform heavy work on the main thread.

A continuation bridge is safe only when every terminal path is explicit, exactly-once resume is enforced, resource cleanup is observable, and isolation assumptions are correct.

## Review checklist

When reviewing continuation code, check:

* [ ] Is the underlying API one-shot?
* [ ] Would `AsyncStream` be more appropriate?
* [ ] Is every success path resumed?
* [ ] Is every failure path resumed?
* [ ] Is invalid or missing result handled?
* [ ] Is cancellation handled?
* [ ] Is timeout handled if needed?
* [ ] Can callback and cancellation race?
* [ ] Can callback and timeout race?
* [ ] Can the callback arrive after cancellation?
* [ ] Can the callback arrive more than once?
* [ ] Is exactly-once resume enforced?
* [ ] Is the continuation cleared after resume?
* [ ] Is the bridge or delegate retained until terminal completion?
* [ ] Is the operation retained until terminal completion?
* [ ] Is cleanup performed after success, failure, cancellation, and timeout?
* [ ] Are callback queue and actor isolation understood?
* [ ] Are Swift 6 `Sendable` diagnostics addressed correctly?
* [ ] Is `self` captured intentionally?
* [ ] Does `[weak self]` still resume on the missing-owner path?
* [ ] Is there a validation path for duplicate callback, cancellation, timeout, and deallocation?
