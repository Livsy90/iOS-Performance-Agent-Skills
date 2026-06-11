# AsyncSequence and Stream Cleanup

Use this reference when the task involves `AsyncSequence`, `AsyncStream`, `AsyncThrowingStream`, long-running streams, buffering, producer lifetime, `onTermination`, or `for await` loops.

This reference is about performance, responsiveness, memory, cancellation, and lifetime safety. It is not a general introduction to `AsyncSequence`.

## Contents

- [Core model](#core-model)
- [Review workflow](#review-workflow)
- [Decision rules](#decision-rules)
- [Producer lifetime](#producer-lifetime)
- [Termination cleanup](#termination-cleanup)
- [Buffering and backpressure](#buffering-and-backpressure)
- [`for await` loop cancellation](#for-await-loop-cancellation)
- [AsyncThrowingStream error paths](#asyncthrowingstream-error-paths)
- [Bridging callbacks and delegates](#bridging-callbacks-and-delegates)
- [Common mistakes](#common-mistakes)
- [Validation](#validation)

## Core model

An async sequence is a pull-based consumer interface over values that may be produced over time.

That does not mean the producer is automatically lazy, bounded, cancellable, or short-lived.

When reviewing stream code, always separate:

- the consumer lifetime;
- the stream object lifetime;
- the producer lifetime;
- the buffering policy;
- the cancellation and termination path;
- the work done for each element.

The most common performance problem is not the `AsyncSequence` abstraction itself. It is a producer that keeps running after the consumer has gone away, or a stream that buffers faster than the consumer can process values.

## Review workflow

1. Identify who creates the stream.
2. Identify who consumes the stream.
3. Check what starts the producer.
4. Check what stops the producer.
5. Check whether `onTermination` releases callbacks, delegates, observers, timers, tasks, sockets, monitors, or other producer resources.
6. Check whether the stream can buffer unbounded values.
7. Check whether each `for await` loop observes cancellation during expensive per-element work.
8. Check whether error, completion, cancellation, and owner deallocation paths all finish or terminate the stream.
9. Check whether values are produced on a reasonable executor or accidentally force work onto the main actor.
10. Recommend the smallest change that makes lifetime, buffering, or cancellation explicit.

## Decision rules

- Treat stream termination as part of the API contract.
- Prefer explicit cleanup with `continuation.onTermination`.
- Avoid unbounded buffering for high-frequency or long-running producers.
- Keep producer ownership obvious. If the producer is created inside the stream builder, make sure it remains alive for the stream lifetime and is released on termination.
- Do not assume breaking out of a `for await` loop automatically stops the underlying producer unless the stream implementation handles termination.
- Do not do heavy per-element work on `MainActor` unless the work is UI-only and small.
- Check cancellation inside long-running loops, especially when processing each element is expensive.
- Prefer finishing the stream explicitly when the producer naturally ends.
- For throwing streams, make error and cancellation paths explicit.
- Validate stream fixes with lifetime, memory, and cancellation evidence.

## Producer lifetime

A stream often wraps another system: a delegate, notification observer, socket, file monitor, timer, SDK callback, sensor, or database listener.

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
```

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

- callback closures;
- delegates;
- notification observers;
- timers;
- file or network monitors;
- Combine subscriptions;
- child tasks;
- sockets or long-lived connections;
- database listeners;
- SDK handles.

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
    AsyncThrowingStream { continuation in
        uploader.onProgress = { event in
            continuation.yield(event)
        }

        uploader.onComplete = {
            continuation.finish()
        }

        uploader.onFailure = { error in
            continuation.finish(throwing: error)
        }

        continuation.onTermination = { _ in
            uploader.cancel()
            uploader.onProgress = nil
            uploader.onComplete = nil
            uploader.onFailure = nil
        }

        uploader.start()
    }
}
```

## Buffering and backpressure

`AsyncStream` can hide memory growth when the producer emits values faster than the consumer can process them.

Review buffering when the stream is:

- high frequency;
- long running;
- driven by sensors, sockets, notifications, logs, progress events, UI events, or database changes;
- consumed on the main actor;
- processed with expensive per-element work.

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
    AsyncStream(bufferingPolicy: .bufferingNewest(100)) { continuation in
        telemetry.onEvent = { event in
            continuation.yield(event)
        }

        continuation.onTermination = { _ in
            telemetry.stop()
            telemetry.onEvent = nil
        }

        telemetry.start()
    }
}
```

Choose the policy based on semantics:

- use `.bufferingNewest(_:)` when stale values can be dropped;
- use `.bufferingOldest(_:)` when early values matter more than newer values;
- avoid unbounded buffering unless the producer is naturally small or bounded.

Do not pick a buffer size as a magic number. Explain what happens when the buffer fills and why dropping values is acceptable for that stream.

## `for await` loop cancellation

A `for await` loop exits when the sequence ends or when the consuming task is cancelled and the sequence observes cancellation. But expensive work inside the loop can still keep running unless it checks cancellation.

Risky:

```swift
for await update in updates {
    await processExpensiveUpdate(update)
}
```

Prefer cancellation-aware processing for expensive per-element work:

```swift
for await update in updates {
    try Task.checkCancellation()
    await processExpensiveUpdate(update)
}
```

If the loop owns resources, use structured cleanup:

```swift
func observeUpdates() async {
    do {
        for try await update in updates {
            try Task.checkCancellation()
            await apply(update)
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

- success completion;
- failure completion;
- cancellation;
- timeout;
- owner deallocation;
- invalid input;
- delegate invalidation;
- producer shutdown.

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
    AsyncThrowingStream { continuation in
        downloader.onEvent = { event in
            continuation.yield(event)
        }

        downloader.onComplete = {
            continuation.finish()
        }

        downloader.onFailure = { error in
            continuation.finish(throwing: error)
        }

        continuation.onTermination = { _ in
            downloader.cancel()
            downloader.onEvent = nil
            downloader.onComplete = nil
            downloader.onFailure = nil
        }

        downloader.start()
    }
}
```

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

If the underlying API supports only one delegate or callback, call out the limitation. The stream may not be safe for multiple consumers.

Also check executor assumptions. If a callback arrives on a background queue but updates UI state in the consumer, the consumer should hop to the correct isolation boundary deliberately.

## Common mistakes

- Creating a stream that starts a producer but never stops it.
- Forgetting to nil out callbacks in `onTermination`.
- Assuming consumer cancellation automatically cancels the underlying SDK or delegate source.
- Creating a producer inside the stream builder without a clear owner.
- Using unbounded buffering for high-frequency streams.
- Ignoring `yield` results when dropped values or termination should affect producer behavior.
- Doing heavy work inside a `for await` loop without cancellation checks.
- Catching `CancellationError` as a generic error and reporting it as failure.
- Forgetting success completion for `AsyncThrowingStream`.
- Forgetting failure completion for streams that can fail.
- Allowing multiple consumers to overwrite a single callback slot.
- Updating UI state from stream callbacks without clear actor isolation.
- Holding `self` strongly from a long-lived stream without a deliberate lifetime reason.

## Validation

Use validation that matches the risk.

For producer lifetime:

- start consuming the stream;
- cancel the consuming task;
- verify the producer stops;
- verify callbacks, delegates, observers, or tokens are removed;
- verify the owner can deallocate.

For buffering:

- simulate a producer faster than the consumer;
- watch memory growth;
- verify the chosen buffering policy drops or keeps values according to the expected semantics;
- log dropped values when that affects correctness.

For cancellation:

- cancel the parent task, navigate away, deallocate the owner, or trigger timeout;
- verify the `for await` loop exits;
- verify expensive per-element work does not continue;
- verify cancellation is not reported as an ordinary failure.

For Instruments:

- look for long-lived tasks after the owner disappears;
- look for producer objects retained after stream cancellation;
- look for memory growth during high-frequency streams;
- look for main-thread or main-actor work inside per-element processing;
- look for blocked cooperative threads caused by synchronous producer work.

For tests:

- write a cancellation test for streams owned by views, view models, requests, or sessions;
- use a fake producer that records `start`, `stop`, handler registration, and handler removal;
- test success, failure, cancellation, and early-exit paths separately.

A stream cleanup fix is successful only when the producer lifetime, termination behavior, and buffering semantics are explicit and observable.
