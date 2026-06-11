# Continuation Safety

Use this reference when the task involves `withCheckedContinuation`, `withCheckedThrowingContinuation`, delegate bridges, callback wrappers, timeout paths, cancellation paths, or exactly-once resume guarantees.

Continuations are a bridge between Swift Concurrency and non-async APIs. Review them as correctness, lifetime, and performance boundaries. A broken continuation can create stuck tasks, double resumes, leaked work, ignored cancellation, or UI flows that never complete.

## Contents

- [Core model](#core-model)
- [When continuations are appropriate](#when-continuations-are-appropriate)
- [Decision rules](#decision-rules)
- [Exactly-once resume contract](#exactly-once-resume-contract)
- [Checked vs unsafe continuations](#checked-vs-unsafe-continuations)
- [Callback wrappers](#callback-wrappers)
- [Delegate bridges](#delegate-bridges)
- [Timeout and cancellation paths](#timeout-and-cancellation-paths)
- [Lifetime and retention](#lifetime-and-retention)
- [Performance notes](#performance-notes)
- [Common mistakes](#common-mistakes)
- [Validation](#validation)

## Core model

A continuation represents a suspended async task waiting for an external event.

The external event may come from:

- a completion handler;
- a delegate callback;
- a notification;
- a legacy SDK;
- a timeout;
- cancellation;
- an operation queue;
- a manually managed service.

The review question is not only “does this compile?” The review question is:

> Can every possible path resume the continuation exactly once, release owned resources, and stop work that is no longer needed?

## When continuations are appropriate

Use continuations when adapting a one-shot callback-style API into an async function.

Good candidates:

- a completion handler that returns once;
- a delegate-driven operation with a single final success or failure event;
- a legacy SDK method that has a clear terminal callback;
- a wrapper around a one-time authorization, export, import, upload, or fetch operation.

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

## Decision rules

- Prefer `withCheckedContinuation` and `withCheckedThrowingContinuation` by default.
- Resume every continuation exactly once.
- Make success, failure, timeout, cancellation, early-return, and invalid-input paths explicit.
- Prefer `Result`-based callback wrappers when possible.
- Use `continuation.resume(with: result)` when the callback already provides `Result`.
- Do not resume from multiple independent `if` branches unless they are mutually exclusive.
- Do not store a continuation without a clear owner and terminal cleanup path.
- Do not use a continuation for multi-value streams.
- Do not use unsafe continuations unless profiling proves checked continuation overhead matters.
- Treat cancellation as part of the bridge design, not an afterthought.

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

- the bridge is on a measured hot path;
- checked continuation overhead is visible in profiling;
- the callback contract is simple and proven;
- tests cover success, failure, cancellation, timeout, and duplicate callback behavior;
- the code documents why unsafe continuation is justified.

Do not start with unsafe continuations as a style preference.

## Callback wrappers

For callback wrappers, inspect the callback contract before writing the async API.

Ask:

- Is the callback guaranteed to be called?
- Can it be called synchronously before the wrapping function returns?
- Can it be called more than once?
- Can success and failure both be represented at the same time?
- Is there a cancellation token or operation handle?
- Which queue or actor invokes the callback?
- Does the callback retain `self`, a delegate, or a producer object?

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

## Delegate bridges

Delegate-based APIs often need extra care because the final callback may arrive through a separate object.

Review:

- where the continuation is stored;
- who owns the delegate/proxy object;
- what happens on success;
- what happens on failure;
- what happens on cancellation;
- what happens if the owner is deallocated;
- whether the delegate can deliver multiple terminal callbacks.

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

If the bridge must also keep the exporter alive, make ownership explicit. Do not rely on accidental retention through closures or delegates.

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

If cancellation can stop callbacks entirely, the bridge must resume the continuation on cancellation as well. That requires synchronized state, because the callback and cancellation path may race.

Conceptual shape:

```swift
final class OneShotBox<Success>: @unchecked Sendable {
    private let lock = NSLock()
    private var continuation: CheckedContinuation<Success, Error>?

    init(_ continuation: CheckedContinuation<Success, Error>) {
        self.continuation = continuation
    }

    func resume(_ result: Result<Success, Error>) {
        lock.lock()
        let continuation = self.continuation
        self.continuation = nil
        lock.unlock()

        continuation?.resume(with: result)
    }
}
```

Use this kind of guard when there are multiple possible terminal sources, such as callback, timeout, and cancellation.

Do not add timeout behavior with a racing task unless the losing path is cancelled and cannot later resume the same continuation.

## Lifetime and retention

Continuations often hide lifetime bugs.

Check:

- Does the legacy operation stay alive until it completes?
- Does the bridge object stay alive until the terminal event?
- Does the continuation retain objects longer than expected?
- Does cancellation release the callback, delegate, producer, or operation?
- Does the wrapper capture `self` strongly, and is that intended?
- Can the owner deallocate while the continuation is still pending?

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

## Performance notes

Continuation overhead is usually not the first performance suspect.

Look first for:

- callback APIs that call back on the main thread and then do heavy parsing;
- excessive bridging in tight loops;
- continuations that never resume and keep tasks alive;
- duplicate operations caused by multiple callers awaiting separate bridges;
- blocking legacy APIs wrapped in async functions;
- missing cancellation that lets expensive work continue after the result is no longer needed.

Do not replace checked continuations with unsafe continuations as a first optimization. If a continuation bridge appears in a hot path, profile first and confirm the actual cost.

## Common mistakes

- Using a continuation for a multi-value event source.
- Missing the nil/nil result path in callback wrappers.
- Resuming from multiple independent branches.
- Forgetting timeout, cancellation, or early-return paths.
- Storing a continuation without clearing it after resume.
- Keeping a delegate bridge alive accidentally rather than explicitly.
- Assuming task cancellation cancels the underlying callback API.
- Racing timeout/cancellation/callback paths without a one-shot guard.
- Wrapping a blocking API in a continuation and calling it “async”.
- Switching to unsafe continuations without measurement.
- Ignoring which queue or actor invokes the callback.
- Capturing `self` strongly in a bridge without checking lifetime.

## Validation

Use validation based on the risk.

For correctness:

- test success;
- test failure;
- test invalid input;
- test nil or malformed result if the callback shape allows it;
- test cancellation before completion;
- test timeout if timeout is supported;
- test duplicate callback delivery if the legacy API can misbehave;
- test owner deallocation while the operation is pending.

For performance and responsiveness:

- use Instruments to look for stuck tasks, blocked threads, main-thread callbacks, and retained bridge objects;
- add signposts around the async wrapper and the underlying callback;
- log operation identifiers to detect duplicate in-flight work;
- use memory tools when pending continuations may retain large values or long-lived producers;
- verify that cancellation stops the underlying work, not only the Swift task waiting for it.

A continuation bridge is safe only when every terminal path is explicit, exactly-once resume is enforced, and resource cleanup is observable.
