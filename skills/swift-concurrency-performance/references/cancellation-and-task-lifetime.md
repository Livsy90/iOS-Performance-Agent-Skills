# Cancellation and Task Lifetime

Use this reference when the task involves navigation cancellation, long-running work, cancellation propagation, cancellation swallowed by `catch`, task groups, streams, or cancellation tests.

## Contents

- [Core model](#core-model)
- [What cancellation means in Swift Concurrency](#what-cancellation-means-in-swift-concurrency)
- [Task lifetime ownership](#task-lifetime-ownership)
- [Navigation and view lifetime](#navigation-and-view-lifetime)
- [Cancellation propagation](#cancellation-propagation)
- [Cancellation in loops and long-running work](#cancellation-in-loops-and-long-running-work)
- [Cancellation and task groups](#cancellation-and-task-groups)
- [Cancellation and AsyncSequence](#cancellation-and-asyncsequence)
- [Cancellation and continuations](#cancellation-and-continuations)
- [Cancellation swallowed by catch](#cancellation-swallowed-by-catch)
- [Decision rules](#decision-rules)
- [Common mistakes](#common-mistakes)
- [Code review checklist](#code-review-checklist)
- [Validation](#validation)

## Core model

Cancellation in Swift Concurrency is cooperative.

Calling `cancel()` marks a task as cancelled. It does not forcibly stop arbitrary work, close legacy callbacks, interrupt synchronous CPU work, or automatically clean up resources that live outside the task tree.

A cancellation-safe design answers three questions:

1. Who owns this task?
2. When should the work stop?
3. Does the code actually observe cancellation and clean up external resources?

If those answers are unclear, the code may keep doing work after the user navigates away, after a parent operation fails, after a timeout, or after the result is no longer useful.

## What cancellation means in Swift Concurrency

A task can become cancelled when:

- its parent is cancelled;
- a task group cancels remaining children;
- the caller explicitly calls `task.cancel()`;
- a timeout wrapper cancels work;
- a SwiftUI `.task` is cancelled when the view identity changes or disappears;
- an async sequence stops being consumed and its producer receives termination;
- test code cancels a task to verify lifetime behavior.

Cancellation is not the same as failure, although many APIs report it by throwing `CancellationError`.

A cancelled task should usually stop as soon as it safely can. It may still run cleanup code, release resources, notify producers, or finish a short critical section.

## Task lifetime ownership

Every task should have an owner.

Common owners:

- a SwiftUI view;
- a view model;
- a request handler;
- a service operation;
- an app-level session;
- an actor method;
- a task group parent;
- an async stream consumer.

Prefer structured concurrency when the parent operation owns the child work:

```swift
func loadScreen() async throws -> ScreenModel {
    async let user = userService.loadUser()
    async let items = itemService.loadItems()

    return try await ScreenModel(
        user: user,
        items: items
    )
}
```

Here the lifetime is clear. If the parent task is cancelled, the child work is cancelled too.

Use stored unstructured tasks only when work must outlive the current async scope, and make the owner explicit:

```swift
@MainActor
final class SearchViewModel {
    private var searchTask: Task<Void, Never>?

    func search(query: String) {
        searchTask?.cancel()

        searchTask = Task {
            do {
                let results = try await service.search(query)
                try Task.checkCancellation()
                self.results = results
            } catch is CancellationError {
                // Expected when the user types a new query.
            } catch {
                self.error = error
            }
        }
    }

    deinit {
        searchTask?.cancel()
    }
}
```

This is appropriate only if the view model owns the task and cancels it when the task is no longer useful.

## Navigation and view lifetime

Navigation cancellation problems happen when work continues after the screen that requested it is gone.

Risky:

```swift
func onAppear() {
    Task {
        let details = try await service.loadDetails()
        self.details = details
    }
}
```

The task is unstructured. If the screen disappears, this task may continue unless something else cancels it.

Prefer SwiftUI `.task` when the work belongs to the view identity:

```swift
struct DetailView: View {
    let id: Item.ID
    @State private var details: Details?

    var body: some View {
        content
            .task(id: id) {
                do {
                    details = try await service.loadDetails(id: id)
                } catch is CancellationError {
                    // View disappeared or id changed.
                } catch {
                    // Present real failure.
                }
            }
    }
}
```

Use `.task(id:)` when a change in identity should cancel old work and start new work.

For UIKit or manually owned objects, store and cancel the task:

```swift
final class DetailViewController: UIViewController {
    private var loadTask: Task<Void, Never>?

    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)

        loadTask = Task { [weak self] in
            guard let self else { return }

            do {
                let details = try await service.loadDetails()
                try Task.checkCancellation()
                await MainActor.run {
                    self.render(details)
                }
            } catch is CancellationError {
                // Screen disappeared.
            } catch {
                await MainActor.run {
                    self.showError(error)
                }
            }
        }
    }

    override func viewDidDisappear(_ animated: Bool) {
        super.viewDidDisappear(animated)
        loadTask?.cancel()
        loadTask = nil
    }
}
```

## Cancellation propagation

Structured child tasks inherit cancellation from their parent.

This is usually good:

```swift
func refresh() async throws {
    try await withThrowingTaskGroup(of: Void.self) { group in
        for id in ids {
            group.addTask {
                try await service.refresh(id)
            }
        }

        try await group.waitForAll()
    }
}
```

If the parent is cancelled, children are cancelled too.

Be careful when moving work into unstructured tasks:

```swift
func refresh() async {
    Task {
        await service.refreshEverything()
    }
}
```

This breaks the parent-child relationship. The caller can no longer reliably cancel the work by cancelling the original task.

Do not use unstructured tasks to escape cancellation unless that is the explicit design.

## Cancellation in loops and long-running work

Long-running loops should observe cancellation.

Risky:

```swift
func process(_ items: [Item]) async throws {
    for item in items {
        try await process(item)
    }
}
```

This may be fine if `process(item)` suspends through cancellation-aware APIs. But if the loop does expensive synchronous work between awaits, cancellation may be delayed.

Prefer explicit checks around expensive work:

```swift
func process(_ items: [Item]) async throws {
    for item in items {
        try Task.checkCancellation()
        let output = expensiveTransform(item)
        try Task.checkCancellation()
        try await upload(output)
    }
}
```

For non-throwing APIs, use `Task.isCancelled`:

```swift
func warmCache(_ items: [Item]) async {
    for item in items {
        if Task.isCancelled { return }
        await cache.prepare(item)
    }
}
```

Do not overuse cancellation checks in tiny loops. Add them where work is long-running, expensive, high-volume, or user-cancellable.

## Cancellation and task groups

Task groups cancel remaining children when the parent task is cancelled.

Throwing task groups also cancel remaining children when one child throws and the error is propagated.

This is usually good:

```swift
func loadSections(_ ids: [Section.ID]) async throws -> [Section] {
    try await withThrowingTaskGroup(of: Section.self) { group in
        for id in ids {
            group.addTask {
                try await service.loadSection(id)
            }
        }

        var sections: [Section] = []

        for try await section in group {
            sections.append(section)
        }

        return sections
    }
}
```

Watch for two risks:

1. Child tasks must cooperate with cancellation.
2. Large input collections still need bounded concurrency.

Cancellation does not fix unbounded fan-out. If the group creates thousands of children, the app may already suffer memory pressure or scheduling overhead before cancellation helps.

Read `references/bounded-task-groups.md` when the issue involves fan-out work, memory spikes, or limiting concurrency.

## Cancellation and AsyncSequence

A `for await` loop exits when the consuming task is cancelled, but producer cleanup depends on the sequence implementation.

For `AsyncStream`, define termination behavior:

```swift
func events() -> AsyncStream<Event> {
    AsyncStream { continuation in
        let token = eventSource.observe { event in
            continuation.yield(event)
        }

        continuation.onTermination = { _ in
            eventSource.removeObserver(token)
        }
    }
}
```

Without `onTermination`, the producer may continue running after the consumer stops.

Inside a long-running stream consumer, check cancellation when per-element work is expensive:

```swift
for await event in events {
    try Task.checkCancellation()
    let model = expensiveTransform(event)
    try Task.checkCancellation()
    await render(model)
}
```

For high-frequency streams, cancellation and buffering are connected. If the consumer stops or falls behind, the producer should not keep accumulating unbounded values.

Read `references/asyncsequence-and-stream-cleanup.md` when the task involves stream lifetime, buffering, producer retention, or `onTermination`.

## Cancellation and continuations

A continuation bridge does not automatically make the underlying callback API cancellation-aware.

Risky:

```swift
func loadImage() async throws -> UIImage {
    try await withCheckedThrowingContinuation { continuation in
        imageLoader.start { result in
            continuation.resume(with: result)
        }
    }
}
```

If the task is cancelled, the image loader may keep running unless the bridge cancels or ignores the callback safely.

A cancellation-aware bridge usually needs:

- a handle to the underlying operation;
- exactly-once resume behavior;
- cancellation cleanup;
- protection against callback-after-cancel races.

Sketch:

```swift
func loadImage() async throws -> UIImage {
    let operation = ImageLoadOperation()

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

This sketch is not enough if the callback can still fire after cancellation and resume the continuation twice. Real bridges need an exactly-once state machine or synchronization around the continuation.

Read `references/continuation-safety.md` when the task involves delegate bridges, callback wrappers, timeout paths, cancellation paths, or exactly-once resume guarantees.

## Cancellation swallowed by catch

A common bug is treating cancellation like a normal error.

Risky:

```swift
do {
    try await refresh()
} catch {
    showError(error)
}
```

If `refresh()` throws `CancellationError`, this may show an error for expected cancellation.

Prefer handling cancellation separately:

```swift
do {
    try await refresh()
} catch is CancellationError {
    // Expected cancellation. Do not show an error.
} catch {
    showError(error)
}
```

Another common bug is swallowing cancellation and continuing work:

```swift
do {
    try await refresh()
} catch {
    logger.info("Refresh failed: \(error)")
}

await loadFallback()
```

If the first operation was cancelled, fallback work may run even though the user already left the flow.

Prefer rethrowing cancellation when the caller owns the lifetime:

```swift
do {
    try await refresh()
} catch is CancellationError {
    throw CancellationError()
} catch {
    logger.info("Refresh failed: \(error)")
    await loadFallback()
}
```

In many cases, `catch` should preserve cancellation rather than converting it into success, fallback work, or UI error state.

## Decision rules

- Prefer structured concurrency when the parent operation owns the work.
- Store and cancel unstructured tasks when an object owns the lifetime.
- Use `.task(id:)` when SwiftUI view identity should control cancellation.
- Check cancellation around expensive synchronous work inside async functions.
- Keep cancellation separate from user-visible failure unless the product intentionally treats it as failure.
- Do not use broad `catch` blocks that hide `CancellationError`.
- Cancellation should stop work that is no longer useful, not merely suppress the final UI update.
- Async streams should clean up producers when the consumer stops.
- Continuation bridges should cancel underlying work or safely ignore callbacks after cancellation.
- Task groups should still be bounded when input size can grow.

## Common mistakes

- Creating a `Task` in `onAppear` and never cancelling it.
- Starting a new search request without cancelling the previous search task.
- Assuming task cancellation interrupts synchronous CPU work.
- Assuming cancellation automatically stops a callback-based SDK.
- Catching all errors and showing cancellation as an alert.
- Catching all errors and continuing fallback work after cancellation.
- Forgetting `onTermination` in `AsyncStream`.
- Creating unbounded task groups and relying on cancellation to save memory.
- Updating UI after cancellation without checking whether the result is still relevant.
- Treating `Task.detached` as a way to make cancellation problems disappear.

## Code review checklist

When reviewing cancellation and lifetime, check:

- [ ] Who owns each task?
- [ ] When should each task stop?
- [ ] Is the task structured under a parent, or stored by an explicit owner?
- [ ] Are unstructured tasks cancelled in `deinit`, `viewDidDisappear`, replacement operations, or equivalent lifecycle events?
- [ ] Does SwiftUI code use `.task(id:)` when identity changes should cancel old work?
- [ ] Are long-running loops cancellation-aware?
- [ ] Are expensive synchronous sections surrounded by cancellation checks where appropriate?
- [ ] Do task groups propagate cancellation and avoid unbounded fan-out?
- [ ] Do streams clean up producers on termination?
- [ ] Do continuation bridges handle cancellation and exactly-once resume?
- [ ] Are `CancellationError` paths separated from real failures?
- [ ] Does cancellation stop the underlying work, not just suppress UI rendering?
- [ ] Is there a test or trace that proves cancelled work stops?

## Validation

Use validation that matches the lifetime risk.

For navigation cancellation:

1. Start the operation.
2. Navigate away before it completes.
3. Verify the task is cancelled.
4. Verify no UI update happens after cancellation.
5. Verify the underlying request, stream, or producer stops if it should.

For replacement operations such as search:

1. Start request A.
2. Start request B before A completes.
3. Verify A is cancelled or ignored safely.
4. Verify only B can update state.
5. Verify errors from A cancellation are not shown to the user.

For long-running loops:

1. Start processing a large input.
2. Cancel midway.
3. Verify the loop stops near the next cancellation checkpoint.
4. Verify no unnecessary fallback work continues.

For task groups:

1. Cancel the parent task.
2. Verify child tasks observe cancellation.
3. Verify memory and task count return to normal.
4. Verify partial results do not incorrectly update state.

For streams:

1. Start consuming the stream.
2. Cancel the consumer.
3. Verify `onTermination` runs.
4. Verify the producer, observer, delegate, timer, or callback source is removed.
5. Verify buffered values do not keep growing.

For continuations:

1. Test success, failure, cancellation, timeout, and callback-after-cancel paths.
2. Verify the continuation resumes exactly once.
3. Verify the underlying operation is cancelled or cleaned up.
4. Verify cancellation is not converted into a user-visible failure unless intended.

Useful signals:

- Instruments shows fewer long-lived tasks after navigation.
- Logs show cancellation at the expected owner boundary.
- Network or SDK logs show cancelled underlying work when appropriate.
- Memory returns to baseline after streams or tasks are cancelled.
- UI does not render stale results from cancelled work.
