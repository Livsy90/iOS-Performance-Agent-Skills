# AsyncSequence and Stream Cleanup

Use this reference when the task involves `AsyncSequence`, `AsyncStream`, `AsyncThrowingStream`, long-running streams, buffering, producer lifetime, `onTermination`, cancellation, callback/delegate bridging, or `for await` loops.

This reference is about performance, responsiveness, memory, cancellation, and lifetime safety. It is not a general introduction to `AsyncSequence`.

## Contents

* [Core model](#core-model)
* [Review workflow](#review-workflow)
* [Decision rules](#decision-rules)
* [Producer lifetime](#producer-lifetime)
* [Termination cleanup](#termination-cleanup)
* [Buffering and backpressure](#buffering-and-backpressure)
* [`yield` results](#yield-results)
* [`for await` loop cancellation](#for-await-loop-cancellation)
* [AsyncThrowingStream error paths](#asyncthrowingstream-error-paths)
* [Bridging callbacks and delegates](#bridging-callbacks-and-delegates)
* [Actor isolation and executor assumptions](#actor-isolation-and-executor-assumptions)
* [Captures and owner lifetime](#captures-and-owner-lifetime)
* [Common mistakes](#common-mistakes)
* [Validation](#validation)
* [Review checklist](#review-checklist)

## Core model

An async sequence gives the consumer a pull-style iteration interface through `next()` and `for await`.

That does not mean the underlying producer is automatically lazy, bounded, cancellable, backpressured, or short-lived.

`AsyncStream` and `AsyncThrowingStream` often bridge push-based systems into a pull-style consumer API. A delegate, callback, timer, notification observer, socket, sensor, SDK listener, or file monitor may continue producing values independently of the consumer.

When reviewing stream code, always separate:

* the consumer lifetime;
* the stream object lifetime;
* the producer lifetime;
* the buffering policy;
* the cancellation and termination path;
* the executor or actor isolation used by callbacks and cleanup;
* the work done for each element.

The most common performance problem is not the `AsyncSequence` abstraction itself. It is a producer that keeps running after the consumer has gone away, or a stream that buffers faster than the consumer can process values.

A safe stream design answers these questions:

```text
What starts the producer?
What stops the producer?
What happens when the consumer cancels?
What happens when the producer completes?
What happens when the producer fails?
What happens when the owner deallocates?
What happens when values arrive faster than the consumer can process them?
```

## Review workflow

1. Identify who creates the stream.
2. Identify who consumes the stream.
3. Check what starts the producer.
4. Check what stops the producer.
5. Check whether `onTermination` releases callbacks, delegates, observers, timers, tasks, sockets, monitors, listeners, or other producer resources.
6. Check whether cleanup is safe for the executor or actor isolation it runs on.
7. Check whether the stream can buffer unbounded values.
8. Check whether `yield` results matter for dropped values or terminated streams.
9. Check whether each `for await` loop observes cancellation during expensive per-element work.
10. Check whether expensive per-element work is cancellation-aware internally.
11. Check whether success, failure, cancellation, timeout, early exit, and owner deallocation paths all finish or terminate the stream.
12. Check whether values are produced on a reasonable executor or accidentally force work onto the main actor.
13. Check whether callback/delegate APIs support multiple listeners.
14. Recommend the smallest change that makes lifetime, buffering, or cancellation explicit.

## Decision rules

* Treat stream termination as part of the API contract.
* Prefer explicit cleanup with `continuation.onTermination` when the stream starts or registers producer work.
* Do not assume `onTermination` runs on the main actor or on the producer's original executor.
* Keep producer ownership obvious. If the producer is created inside the stream builder, make sure it remains alive for the stream lifetime and is released on termination.
* Do not assume breaking out of a `for await` loop automatically stops the underlying producer unless the stream implementation handles termination.
* Avoid unbounded buffering for high-frequency or long-running producers.
* Choose buffering policy from stream semantics, not from a magic number.
* Check `yield` results when dropped values, termination, or producer throttling matter.
* Do not do heavy per-element work on `MainActor` unless the work is UI-only and small.
* Check cancellation inside long-running loops.
* Make expensive per-element work cancellation-aware internally.
* Prefer finishing the stream explicitly when the producer naturally ends.
* For throwing streams, make success, failure, cancellation, and cleanup paths explicit.
* If the underlying API supports only one callback or delegate, make single-consumer limitations explicit or introduce a shared broadcaster.
* Validate stream fixes with lifetime, memory, cancellation, and buffering evidence.

## Producer lifetime

A stream often wraps another system:

* delegate;
* notification observer;
* socket;
* file monitor;
* timer;
* SDK callback;
* sensor;
* database listener;
* Combine subscription;
* child task;
* network connection.

The stream consumer may stop at any time. The producer must not continue indefinitely unless that is intentional.

Risky:

```swift
func paymentEvents() -> AsyncStream<PaymentEvent> {
    AsyncStream { continuation in
        paymentMonitor.onEvent = { event in
            continuation.yield(event)
        }

        paymentMonitor.start()
    }
}
```

The consumer can stop listening, but `paymentMonitor` may keep running and may keep retaining its callback.

Prefer making producer cleanup explicit:

```swift
func paymentEvents() -> AsyncStream<PaymentEvent> {
    let monitor = paymentMonitor

    return AsyncStream { continuation in
        monitor.onEvent = { event in
            continuation.yield(event)
        }

        continuation.onTermination = { _ in
            monitor.stop()
            monitor.onEvent = nil
        }

        monitor.start()
    }
}
```

Capturing `monitor` explicitly makes the ownership relationship easier to see than implicitly capturing `self`.

If the producer is created inside the stream builder, make its lifetime deliberate.

Risky:

```swift
func locationUpdates() -> AsyncStream<Location> {
    AsyncStream { continuation in
        let manager = LocationMonitor()

        manager.onUpdate = { location in
            continuation.yield(location)
        }

        manager.start()
    }
}
```

This code is ambiguous. The monitor may be released too early, or it may be retained indirectly in a way that is hard to reason about.

Prefer an explicit holder when the producer must live with the stream:

```swift
func locationUpdates() -> AsyncStream<Location> {
    final class Holder {
        let monitor = LocationMonitor()

        func stop() {
            monitor.stop()
            monitor.onUpdate = nil
        }
    }

    let holder = Holder()

    return AsyncStream { continuation in
        holder.monitor.onUpdate = { location in
            continuation.yield(location)
        }

        continuation.onTermination = { _ in
            holder.stop()
        }

        holder.monitor.start()
    }
}
```

Use this pattern carefully. The goal is not to add wrapper objects everywhere. The goal is to make lifetime obvious when the producer is not owned elsewhere.

## Termination cleanup

Use `onTermination` when the stream starts work, registers callbacks, or retains external resources.

Cleanup usually needs to remove or stop:

* callback closures;
* delegates;
* notification observers;
* timers;
* file or network monitors;
* Combine subscriptions;
* child tasks;
* sockets or long-lived connections;
* database listeners;
* SDK handles;
* retained producer objects.

A good cleanup path is idempotent. It should be safe if cancellation, completion, and owner deallocation happen close together.

Prefer:

```swift
func notifications(named name: Notification.Name) -> AsyncStream<Notification> {
    AsyncStream { continuation in
        let token = NotificationCenter.default.addObserver(
            forName: name,
            object: nil,
            queue: nil
        ) { notification in
            continuation.yield(notification)
        }

        continuation.onTermination = { _ in
            NotificationCenter.default.removeObserver(token)
        }
    }
}
```

If a stream has a natural end, finish it explicitly:

```swift
func uploadProgress() -> AsyncThrowingStream<ProgressEvent, Error> {
    let currentUploader = uploader

    return AsyncThrowingStream { continuation in
        currentUploader.onProgress = { event in
            continuation.yield(event)
        }

        currentUploader.onComplete = {
            continuation.finish()
        }

        currentUploader.onFailure = { error in
            continuation.finish(throwing: error)
        }

        continuation.onTermination = { _ in
            currentUploader.cancel()
            currentUploader.onProgress = nil
            currentUploader.onComplete = nil
            currentUploader.onFailure = nil
        }

        currentUploader.start()
    }
}
```

`onTermination` should not be the only terminal path. If the producer naturally completes or fails, call `finish()` or `finish(throwing:)` explicitly.

## Buffering and backpressure

`AsyncStream` can hide memory growth when the producer emits values faster than the consumer can process them.

Review buffering when the stream is:

* high frequency;
* long running;
* driven by sensors, sockets, notifications, logs, progress events, UI events, or database changes;
* consumed on the main actor;
* processed with expensive per-element work;
* produced by a source that cannot slow down.

Risky:

```swift
func telemetryEvents() -> AsyncStream<TelemetryEvent> {
    AsyncStream { continuation in
        telemetry.onEvent = { event in
            continuation.yield(event)
        }

        telemetry.start()
    }
}
```

If telemetry emits faster than the consumer handles values, memory may grow.

Prefer an explicit buffering policy when only recent values matter:

```swift
func telemetryEvents() -> AsyncStream<TelemetryEvent> {
    let source = telemetry

    return AsyncStream(bufferingPolicy: .bufferingNewest(100)) { continuation in
        source.onEvent = { event in
            continuation.yield(event)
        }

        continuation.onTermination = { _ in
            source.stop()
            source.onEvent = nil
        }

        source.start()
    }
}
```

Choose the policy based on semantics:

* use `.bufferingNewest(_:)` when stale values can be dropped;
* use `.bufferingOldest(_:)` when early values matter more than newer values;
* avoid unbounded buffering unless the producer is naturally small or bounded.

Do not pick a buffer size as a magic number. Explain what happens when the buffer fills and why dropping values is acceptable for that stream.

Examples:

* telemetry: newest values may be enough;
* progress updates: newest values are usually enough;
* audit events: dropping values may be incorrect;
* chat messages: dropping values is usually incorrect;
* sensor samples: dropping may be acceptable if the UI only displays the latest state;
* database changes: dropping may be incorrect unless a later snapshot supersedes earlier changes.

## `yield` results

`continuation.yield(_:)` can report whether the value was enqueued, dropped, or the stream had already terminated.

Ignoring the result is acceptable when dropped values have no semantic meaning and the producer cost is small.

Do not ignore the result when:

* dropped values affect correctness;
* dropped values should be counted;
* termination should stop the producer;
* the producer is expensive and should not keep generating values for a terminated stream;
* buffer pressure should affect diagnostics.

Example:

```swift
source.onEvent = { event in
    let result = continuation.yield(event)

    switch result {
    case .enqueued(_):
        break

    case .dropped(_):
        metrics.droppedTelemetryEvents += 1

    case .terminated:
        source.stop()
        source.onEvent = nil
    }
}
```

Use this carefully. Do not add heavy logging or metrics work to a high-frequency callback unless that work is cheap and safe for the callback executor.

## `for await` loop cancellation

Cancellation is cooperative. A generic `AsyncSequence` is not guaranteed to stop immediately unless its iterator observes cancellation, returns `nil`, or throws `CancellationError`.

A `for await` loop exits when:

* the sequence finishes;
* a throwing sequence throws;
* the loop breaks;
* the consuming task is cancelled and the sequence or loop body cooperates with cancellation.

Risky:

```swift
for await update in updates {
    await processExpensiveUpdate(update)
}
```

This loop may continue expensive per-element processing even after cancellation unless the sequence or processing cooperates.

For non-throwing contexts, use `Task.isCancelled`:

```swift
for await update in updates {
    guard !Task.isCancelled else {
        break
    }

    await processExpensiveUpdate(update)
}
```

For throwing contexts, use `Task.checkCancellation()`:

```swift
func observeUpdates() async throws {
    for await update in updates {
        try Task.checkCancellation()
        try await processExpensiveUpdate(update)
    }
}
```

Checking before the call is not enough if `processExpensiveUpdate(_:)` is itself long-running. Expensive per-element work should also check cancellation internally.

Example:

```swift
func processExpensiveUpdate(_ update: Update) async throws {
    try Task.checkCancellation()

    let prepared = try await prepare(update)

    try Task.checkCancellation()

    try await apply(prepared)
}
```

If the loop owns resources, use structured cleanup and handle cancellation separately:

```swift
func observeUpdates() async {
    do {
        for try await update in updates {
            try Task.checkCancellation()
            try await apply(update)
        }
    } catch is CancellationError {
        // Expected when the owner stops observing.
    } catch {
        await report(error)
    }
}
```

Avoid swallowing cancellation accidentally:

```swift
do {
    for try await event in events {
        try await handle(event)
    }
} catch {
    // Risk: cancellation is treated like an ordinary failure.
    await report(error)
}
```

Prefer handling cancellation separately when it is an expected lifecycle path:

```swift
do {
    for try await event in events {
        try await handle(event)
    }
} catch is CancellationError {
    // Normal lifecycle end.
} catch {
    await report(error)
}
```

## AsyncThrowingStream error paths

For `AsyncThrowingStream`, review all terminal paths:

* success completion;
* failure completion;
* cancellation;
* timeout;
* owner deallocation;
* invalid input;
* delegate invalidation;
* producer shutdown.

A throwing stream should not leave consumers suspended forever.

Risky:

```swift
func downloadEvents() -> AsyncThrowingStream<DownloadEvent, Error> {
    AsyncThrowingStream { continuation in
        downloader.onEvent = { event in
            continuation.yield(event)
        }

        downloader.onFailure = { error in
            continuation.finish(throwing: error)
        }

        downloader.start()
    }
}
```

This handles failure but not success, cancellation, or cleanup.

Prefer explicit terminal paths:

```swift
func downloadEvents() -> AsyncThrowingStream<DownloadEvent, Error> {
    let currentDownloader = downloader

    return AsyncThrowingStream { continuation in
        currentDownloader.onEvent = { event in
            continuation.yield(event)
        }

        currentDownloader.onComplete = {
            continuation.finish()
        }

        currentDownloader.onFailure = { error in
            continuation.finish(throwing: error)
        }

        continuation.onTermination = { _ in
            currentDownloader.cancel()
            currentDownloader.onEvent = nil
            currentDownloader.onComplete = nil
            currentDownloader.onFailure = nil
        }

        currentDownloader.start()
    }
}
```

If cancellation should be reported as cancellation rather than ordinary failure, make that explicit in the stream or in the consumer.

Do not leave a stream open when:

* the request was cancelled;
* the owner deallocated;
* the delegate was invalidated;
* the producer cannot emit any more values;
* input was invalid before the producer started.

## Bridging callbacks and delegates

When bridging callback or delegate APIs into streams, check whether the original API supports multiple listeners.

Risky:

```swift
func keyboardEvents() -> AsyncStream<KeyboardEvent> {
    AsyncStream { continuation in
        keyboardObserver.onEvent = { event in
            continuation.yield(event)
        }
    }
}
```

This may overwrite an existing callback if `keyboardObserver` only has one `onEvent` slot.

Prefer APIs that return a registration token when possible:

```swift
func keyboardEvents() -> AsyncStream<KeyboardEvent> {
    AsyncStream { continuation in
        let token = keyboardObserver.addHandler { event in
            continuation.yield(event)
        }

        continuation.onTermination = { _ in
            keyboardObserver.removeHandler(token)
        }
    }
}
```

If every call to the stream function installs a single-slot callback, two independent consumers can overwrite each other.

In that case, choose one of these designs:

* expose a single shared stream owner;
* use a multicast or broadcast abstraction;
* return an explicit single-consumer stream;
* document that only one consumer is supported;
* redesign the underlying observer API to support registration tokens.

Also check executor assumptions. If a callback arrives on a background queue but updates UI state in the consumer, the consumer should hop to the correct isolation boundary deliberately.

## Actor isolation and executor assumptions

Do not assume `onTermination` runs on the main actor or on the producer's original executor.

If cleanup touches main-actor-isolated objects, hop explicitly:

```swift
continuation.onTermination = { _ in
    Task { @MainActor in
        viewModel.stopObserving()
    }
}
```

Use this pattern only when asynchronous cleanup is acceptable. If the resource must be released synchronously, design the producer owner so cleanup can happen safely from `onTermination`.

If cleanup touches non-sendable state, keep the stream creation and cleanup inside the same isolation domain, or wrap the producer in an actor-safe owner.

Risky:

```swift
@MainActor
final class LocationViewModel {
    private let monitor = LocationMonitor()

    func locations() -> AsyncStream<Location> {
        AsyncStream { continuation in
            monitor.onUpdate = { location in
                continuation.yield(location)
            }

            continuation.onTermination = { _ in
                monitor.stop()
                monitor.onUpdate = nil
            }

            monitor.start()
        }
    }
}
```

The model is `@MainActor`, but `onTermination` should not be assumed to run on `MainActor`.

Prefer making the isolation explicit:

```swift
@MainActor
final class LocationViewModel {
    private let monitor = LocationMonitor()

    func locations() -> AsyncStream<Location> {
        let monitor = self.monitor

        return AsyncStream { continuation in
            monitor.onUpdate = { location in
                continuation.yield(location)
            }

            continuation.onTermination = { _ in
                Task { @MainActor in
                    monitor.stop()
                    monitor.onUpdate = nil
                }
            }

            monitor.start()
        }
    }
}
```

If strict concurrency warnings appear, do not silence them mechanically. They often indicate that producer ownership, sendability, or actor isolation is unclear.

## Captures and owner lifetime

Long-lived streams can easily extend object lifetimes.

Risky:

```swift
final class PaymentService {
    private let paymentMonitor = PaymentMonitor()

    func paymentEvents() -> AsyncStream<PaymentEvent> {
        AsyncStream { continuation in
            paymentMonitor.onEvent = { event in
                continuation.yield(event)
            }

            continuation.onTermination = { _ in
                paymentMonitor.stop()
                paymentMonitor.onEvent = nil
            }

            paymentMonitor.start()
        }
    }
}
```

This may implicitly capture `self` through `paymentMonitor`.

Prefer capturing the specific producer when that producer is the intended lifetime dependency:

```swift
final class PaymentService {
    private let paymentMonitor = PaymentMonitor()

    func paymentEvents() -> AsyncStream<PaymentEvent> {
        let monitor = paymentMonitor

        return AsyncStream { continuation in
            monitor.onEvent = { event in
                continuation.yield(event)
            }

            continuation.onTermination = { _ in
                monitor.stop()
                monitor.onEvent = nil
            }

            monitor.start()
        }
    }
}
```

Do not use `[weak self]` as a reflex. It can be correct, but it may also make the stream silently stop yielding without cleaning up the producer.

Ask:

* should the stream keep the producer alive?
* should the owner keep the stream alive?
* should owner deallocation finish the stream?
* who is responsible for stopping the producer?
* is the capture needed for the whole stream lifetime or only for setup?

If owner deallocation should end the stream, model that explicitly.

## Common mistakes

* Creating a stream that starts a producer but never stops it.
* Forgetting to nil out callbacks in `onTermination`.
* Assuming consumer cancellation automatically cancels the underlying SDK or delegate source.
* Assuming `onTermination` runs on the main actor.
* Creating a producer inside the stream builder without a clear owner.
* Capturing `self` strongly from a long-lived stream without a deliberate lifetime reason.
* Using `[weak self]` to hide an unclear lifetime model.
* Using unbounded buffering for high-frequency streams.
* Ignoring `yield` results when dropped values or termination should affect producer behavior.
* Doing heavy work inside a `for await` loop without cancellation checks.
* Checking cancellation before expensive processing but not inside the expensive processing itself.
* Catching `CancellationError` as a generic error and reporting it as failure.
* Forgetting success completion for `AsyncThrowingStream`.
* Forgetting failure completion for streams that can fail.
* Leaving consumers suspended forever on timeout, cancellation, invalid input, or owner deallocation.
* Allowing multiple consumers to overwrite a single callback slot.
* Updating UI state from stream callbacks without clear actor isolation.
* Treating buffering size as a magic constant rather than a semantic choice.
* Silencing strict concurrency warnings instead of clarifying sendability or isolation.

## Validation

Use validation that matches the risk.

For producer lifetime:

* start consuming the stream;
* cancel the consuming task;
* verify the producer stops;
* verify callbacks, delegates, observers, or tokens are removed;
* verify the owner can deallocate;
* verify cancellation does not leave timers, sockets, listeners, or monitors alive.

For buffering:

* simulate a producer faster than the consumer;
* watch memory growth;
* verify the chosen buffering policy drops or keeps values according to the expected semantics;
* log dropped values when that affects correctness;
* confirm that dropped values do not break downstream state.

For cancellation:

* cancel the parent task;
* navigate away;
* deallocate the owner;
* trigger timeout;
* verify the `for await` loop exits;
* verify expensive per-element work does not continue;
* verify cancellation is not reported as an ordinary failure.

For terminal paths:

* test success completion;
* test failure completion;
* test cancellation;
* test early loop exit;
* test producer shutdown;
* test owner deallocation;
* test invalid input before producer start.

For Instruments:

* look for long-lived tasks after the owner disappears;
* look for producer objects retained after stream cancellation;
* look for memory growth during high-frequency streams;
* look for main-thread or main-actor work inside per-element processing;
* look for blocked cooperative threads caused by synchronous producer work;
* look for timers, sockets, monitors, or listeners that remain active after navigation.

For tests:

* write a cancellation test for streams owned by views, view models, requests, or sessions;
* use a fake producer that records `start`, `stop`, handler registration, and handler removal;
* test success, failure, cancellation, timeout, and early-exit paths separately;
* test multiple consumers if the API is expected to support them;
* test that single-consumer APIs fail clearly or document the limitation.

A stream cleanup fix is successful only when the producer lifetime, termination behavior, buffering semantics, cancellation behavior, and isolation assumptions are explicit and observable.

## Review checklist

Use this checklist for stream code:

* [ ] Who creates the stream?
* [ ] Who consumes the stream?
* [ ] What starts the producer?
* [ ] What stops the producer?
* [ ] Does `onTermination` release callbacks, delegates, observers, timers, tasks, sockets, monitors, or listeners?
* [ ] Is cleanup idempotent?
* [ ] Is cleanup safe for the executor or actor isolation where it runs?
* [ ] Does the stream finish when the producer naturally completes?
* [ ] Does the stream finish or throw when the producer fails?
* [ ] Does cancellation stop the underlying producer when appropriate?
* [ ] Can the stream buffer unbounded values?
* [ ] Is the buffering policy explicit for high-frequency streams?
* [ ] Are dropped values acceptable?
* [ ] Does the code inspect `yield` results when drops or termination matter?
* [ ] Does the `for await` loop check cancellation during expensive work?
* [ ] Is expensive per-element work cancellation-aware internally?
* [ ] Are cancellation errors handled separately from real failures?
* [ ] Does the underlying callback/delegate API support multiple listeners?
* [ ] Can multiple consumers overwrite one callback slot?
* [ ] Does the stream capture `self` strongly?
* [ ] Is strong capture intentional?
* [ ] Are strict concurrency warnings about sendability or isolation addressed rather than hidden?
* [ ] Is the fix validated with tests, logs, memory inspection, Instruments, or a fake producer?
