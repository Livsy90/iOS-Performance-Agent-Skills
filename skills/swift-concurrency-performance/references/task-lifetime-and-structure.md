# Task Lifetime and Structure

Use this reference when the task involves structured concurrency, unstructured tasks, task ownership, `Task {}`, `Task.detached`, view/view-model lifetimes, or tasks that outlive their owner.

## Contents

- [Core model](#core-model)
- [Structured concurrency first](#structured-concurrency-first)
- [Task ownership](#task-ownership)
- [`Task {}`](#task-)
- [`Task.detached`](#taskdetached)
- [View and view-model lifetimes](#view-and-view-model-lifetimes)
- [Long-lived service tasks](#long-lived-service-tasks)
- [Common mistakes](#common-mistakes)
- [Review checklist](#review-checklist)
- [Validation](#validation)

## Core model

A task is not only a way to run code later. It has a lifetime, priority, cancellation behavior, task-local values, and isolation context.

When reviewing task structure, ask:

1. Who owns this work?
2. When should it start?
3. When should it stop?
4. What happens if the user navigates away?
5. What happens if the parent task is cancelled?
6. Is the result still relevant when the task completes?

The common performance risk is uncontrolled lifetime. A task that outlives its owner can keep doing work, retain objects, duplicate requests, update stale state, increase memory pressure, or compete with current user-visible work.

## Structured concurrency first

Prefer structured concurrency when the parent operation owns the child work.

Structured concurrency makes the lifetime model clear:

- child tasks are scoped to the parent operation;
- cancellation can propagate from parent to children;
- errors can be collected or propagated deliberately;
- the caller cannot accidentally lose track of the work;
- traces and tests are easier to reason about.

Prefer:

```swift
func loadDashboard() async throws -> Dashboard {
    async let profile = profileService.loadProfile()
    async let feed = feedService.loadFeed()
    async let notifications = notificationService.loadUnreadCount()

    return try await Dashboard(
        profile: profile,
        feed: feed,
        unreadNotifications: notifications
    )
}
```

This is appropriate when all results are needed for the operation and should be cancelled if `loadDashboard()` is cancelled.

Avoid unstructured tasks just to make work happen in parallel:

```swift
func loadDashboard() async {
    Task { await profileService.loadProfile() }
    Task { await feedService.loadFeed() }
    Task { await notificationService.loadUnreadCount() }
}
```

The caller cannot naturally await the whole operation, handle failure, or cancel all work as one unit.

## Task ownership

Every unstructured task should have an owner.

Common owners:

- a SwiftUI view through `.task`;
- a view model that stores a `Task` handle;
- a service that owns a long-lived observation loop;
- an app-level coordinator for app-session work;
- a request object for request-scoped work.

A useful review question:

> If this task is still running in 30 seconds, who is responsible for cancelling it?

If the answer is vague, prefer a structured task, a stored task handle, or a clearer lifetime boundary.

## `Task {}`

`Task {}` creates an unstructured task. It can inherit useful context from the current task, including priority, task-local values, and actor context.

Use `Task {}` when there is an explicit lifetime owner.

Reasonable:

```swift
@MainActor
final class SearchViewModel: ObservableObject {
    @Published private(set) var results: [SearchResult] = []

    private var searchTask: Task<Void, Never>?

    func search(query: String) {
        searchTask?.cancel()

        searchTask = Task { [searchService] in
            do {
                let results = try await searchService.search(query)
                try Task.checkCancellation()
                self.results = results
            } catch is CancellationError {
                // Expected when a newer search replaces this one.
            } catch {
                self.results = []
            }
        }
    }

    deinit {
        searchTask?.cancel()
    }
}
```

This task has a clear owner. New searches cancel old searches, and deinit cancels remaining work.

Risky:

```swift
func refresh() {
    Task {
        await syncService.refreshEverything()
    }
}
```

This may be acceptable for app-session fire-and-forget work, but it should be justified. Otherwise, the caller loses cancellation, error handling, and completion semantics.

Prefer returning an async operation when the caller owns the work:

```swift
func refresh() async throws {
    try await syncService.refreshEverything()
}
```

## `Task.detached`

`Task.detached` creates a task that is independent from the current task hierarchy and does not inherit actor isolation in the same way as `Task {}`.

It should be rare in app code.

Use it only when the work is intentionally independent from the current task and actor context.

Possible cases:

- independent app-level maintenance work;
- CPU-heavy work that must not inherit main-actor isolation;
- explicitly isolated background processing with its own cancellation and priority policy;
- bridging to a subsystem with a separate lifetime model.

Risky:

```swift
@MainActor
func didTapExport() {
    Task.detached {
        let file = try await exporter.exportLargeReport()
        await self.showExportResult(file)
    }
}
```

Review the risks:

- `self` may be captured across an independent task lifetime;
- cancellation is unclear;
- priority is unclear;
- the task may outlive the screen;
- UI state must be crossed back to the main actor;
- errors may be lost if not handled.

Use `Task.detached` only when detachment is part of the design, not as a workaround for isolation errors.

## View and view-model lifetimes

SwiftUI views are value types and can be recreated often. Avoid treating view initialization as the lifetime owner for async work.

Prefer `.task` when the work is tied to view presence:

```swift
struct ProfileScreen: View {
    let userID: User.ID
    @State private var model: ProfileModel?

    var body: some View {
        ProfileContent(model: model)
            .task(id: userID) {
                model = await loadProfile(userID)
            }
    }
}
```

Use `.task(id:)` when the identity of the work matters. The task restarts when the `id` changes.

Be careful with `.onAppear { Task { ... } }` for loading work. It can create repeated unstructured tasks when the view appears multiple times, and cancellation/restart behavior is less explicit than `.task(id:)`.

For view models, store task handles when work can outlive a single method call.

## Long-lived service tasks

Some tasks are intentionally long-lived: listening to notifications, observing sockets, syncing state, or consuming streams.

Long-lived tasks still need ownership.

A service that starts an observation task should usually have:

- a stored task handle;
- idempotent `start()`;
- explicit `stop()`;
- cancellation in `deinit`;
- cancellation checks inside long loops;
- stream cleanup when the consumer stops.

Review long-lived tasks for duplicate starts, missing stop paths, missing cancellation checks, streams that do not terminate, retained producers, unbounded buffering, or work continuing after logout/navigation.

## Common mistakes

### Fire-and-forget without ownership

```swift
func save() {
    Task {
        await database.saveChanges()
    }
}
```

This hides failure, cancellation, and completion. Prefer `async throws` if the caller needs the result, or store the task if the object owns it.

### Using `Task.detached` to avoid `MainActor`

```swift
@MainActor
func process() {
    Task.detached {
        await self.processor.run()
    }
}
```

This may create a data-safety and lifetime problem. Prefer moving CPU-heavy work into a non-main-actor dependency and calling it from an owned task.

### Starting tasks from high-frequency callbacks

```swift
func cellDidAppear(item: Item) {
    Task {
        await imageLoader.preload(item.imageURL)
    }
}
```

This can create a task explosion during scrolling. Prefer caching, deduplication, backpressure, or a bounded prefetching pipeline.

### Ignoring replacement semantics

```swift
func search(query: String) {
    Task {
        results = await service.search(query)
    }
}
```

Older searches may finish after newer ones. Store the task, cancel old work, and check cancellation before applying results.

### Assuming deallocation cancels everything

Objects do not automatically cancel every task they started. If a task captures `self`, it may also keep `self` alive longer than expected.

## Review checklist

When reviewing task lifetime and structure, check:

- [ ] Is the task structured under a parent operation when possible?
- [ ] If the task is unstructured, is the owner explicit?
- [ ] Is the task handle stored when later cancellation is required?
- [ ] Is there a cancellation point when newer work replaces older work?
- [ ] Does deinit or flow teardown cancel owned tasks?
- [ ] Does the task update state only if the result is still relevant?
- [ ] Are errors handled intentionally instead of disappearing?
- [ ] Is `Task.detached` justified by a real independence requirement?
- [ ] Could repeated calls create many overlapping tasks?
- [ ] Could scrolling, typing, or lifecycle callbacks create task explosions?
- [ ] Are long-lived tasks protected against duplicate starts?
- [ ] Is there a clear stop path for long-lived observation loops?
- [ ] Does validation prove tasks stop when the owner disappears?

## Validation

Use validation that matches the suspected lifetime issue.

For cancellation and navigation:

- log task start, cancellation, and completion;
- navigate away and verify the operation stops;
- verify stale results are not applied after cancellation;
- add cancellation tests for view-model-owned work.

For task explosions:

- add temporary counters for task creation and completion;
- reproduce with fast typing, rapid navigation, or fast scrolling;
- inspect Instruments for high task counts or long-lived tasks;
- verify bounded concurrency or deduplication reduces task count.

For retained owners:

- use memory graph debugging;
- add temporary `deinit` logs during investigation;
- verify the owner deallocates after navigation;
- check whether task closures capture `self` longer than intended.

For detached or long-lived tasks:

- verify explicit start and stop behavior;
- verify logout, cancellation, or app-session teardown stops the work;
- check that priority and cancellation are intentional;
- inspect whether the task keeps running after the triggering UI disappears.

A task lifetime refactor is successful only if the work now has a clear owner, a clear cancellation path, and observable evidence that stale or unnecessary work stops.
