# Task Lifetime and Structure

Use this reference when the task involves structured concurrency, unstructured tasks, task ownership, `Task {}`, `Task.detached`, SwiftUI `.task`, view/view-model lifetimes, replacement operations, long-lived service tasks, or tasks that outlive their owner.

This reference is about task lifetime and ownership. It is not a general Swift Concurrency guide. For cancellation internals, bounded task groups, blocking legacy APIs, actor reentrancy, or `MainActor` responsiveness, use the more specific references.

## Contents

* [Core model](#core-model)
* [Structured concurrency first](#structured-concurrency-first)
* [Task ownership](#task-ownership)
* [Decision rules](#decision-rules)
* [`Task {}`](#task-)
* [`Task.detached`](#taskdetached)
* [View and view-model lifetimes](#view-and-view-model-lifetimes)
* [Replacement operations](#replacement-operations)
* [Long-lived service tasks](#long-lived-service-tasks)
* [Common mistakes](#common-mistakes)
* [Review checklist](#review-checklist)
* [Validation](#validation)
* [Related references](#related-references)

## Core model

A task is not only a way to run code later. It has:

* a lifetime;
* priority;
* cancellation behavior;
* task-local values;
* isolation context;
* result or error semantics;
* ownership implications.

When reviewing task structure, ask:

1. Who owns this work?
2. When should it start?
3. When should it stop?
4. What happens if the user navigates away?
5. What happens if the parent task is cancelled?
6. Who observes errors?
7. Is the result still relevant when the task completes?
8. Can repeated calls create overlapping work?

The common performance risk is uncontrolled lifetime. A task that outlives its owner can keep doing work, retain objects, duplicate requests, update stale state, increase memory pressure, consume battery/network, or compete with current user-visible work.

A task lifetime refactor is successful only if the work has:

```text id="5i1poh"
clear owner
clear cancellation path
clear result/error handling
clear stale-result policy
observable validation
```

## Structured concurrency first

Prefer structured concurrency when the parent operation owns the child work.

Structured concurrency makes the lifetime model clear:

* child tasks are scoped to the parent operation;
* child tasks cannot arbitrarily outlive the scope that created them;
* cancellation can propagate from parent to children;
* errors can be collected or propagated deliberately;
* the caller cannot accidentally lose track of the work;
* traces and tests are easier to reason about.

Prefer:

```swift id="ryh4k3"
func loadDashboard() async throws -> Dashboard {
    async let profile = profileService.loadProfile()
    async let feed = feedService.loadFeed()
    async let notifications = notificationService.loadUnreadCount()

    let loaded = try await (profile, feed, notifications)

    return Dashboard(
        profile: loaded.0,
        feed: loaded.1,
        unreadNotifications: loaded.2
    )
}
```

This is appropriate when all results are needed for the operation and should be cancelled if `loadDashboard()` is cancelled.

Avoid unstructured tasks just to make work happen in parallel:

```swift id="qvhtl4"
func loadDashboard() async {
    Task { await profileService.loadProfile() }
    Task { await feedService.loadFeed() }
    Task { await notificationService.loadUnreadCount() }
}
```

The caller cannot naturally await the whole operation, handle failure, aggregate results, or cancel all work as one unit.

Use `async let` for a small fixed number of independent child operations.

Use task groups for a dynamic number of child operations, and bound concurrency when input size can grow.

## Task ownership

Every unstructured task should have an owner.

Common owners:

* a SwiftUI view through `.task` or `.task(id:)`;
* a view model that stores a `Task` handle;
* a view controller that stores a `Task` handle;
* a service that owns a long-lived observation loop;
* an app-level coordinator for app-session work;
* a request object for request-scoped work;
* a feature coordinator for work that outlives one screen but not the app.

A useful review question:

```text id="g243gs"
If this task is still running in 30 seconds, who is responsible for cancelling it?
```

If the answer is vague, prefer a structured task, a stored task handle, or a clearer lifetime boundary.

Ownership is not only about cancellation. The owner is also responsible for:

* deciding whether errors are surfaced or ignored;
* deciding whether stale results can update state;
* releasing retained resources;
* preventing duplicate starts;
* defining teardown behavior.

## Decision rules

* Prefer structured concurrency when the caller owns the work.
* Use `.task(id:)` when SwiftUI view identity owns the work.
* Use a stored task handle when an object owns replaceable or long-running work.
* Cancel previous work before starting replacement work when older results are no longer relevant.
* Do not use `Task {}` only to create parallelism; use `async let` or task groups when the parent owns the work.
* Do not use `Task {}` as a background boundary from `@MainActor`.
* If an unstructured task can throw, handle the error inside the task or ensure someone awaits the task result.
* Use `Task.detached` only for intentionally independent work with explicit cancellation, priority, ownership, Sendable boundaries, and result delivery.
* Treat fire-and-forget as a design decision, not a default.
* Avoid starting unbounded tasks from scrolling, typing, lifecycle callbacks, or retry loops.
* Validate that owned tasks stop when the owner disappears.
* Validate that stale results do not update current UI state.

## `Task {}`

`Task {}` creates an unstructured task.

It can inherit useful context from the current task, including priority, task-local values, and actor context. When created from actor-isolated code, the task closure may inherit that actor isolation.

This is useful, but it has two important consequences:

1. `Task {}` is not a structured child task of the current async scope.
2. `Task {}` created from `@MainActor` code is not a background boundary.

Do not use `Task {}` from UI code as a way to move CPU-heavy work off the main actor. Move heavy work behind an explicit non-main-actor boundary and keep only UI mutation on `MainActor`.

Use `Task {}` when there is an explicit lifetime owner.

Reasonable:

```swift id="du38ej"
@MainActor
final class SearchViewModel: ObservableObject {
    @Published private(set) var results: [SearchResult] = []
    @Published private(set) var error: Error?

    private let searchService: SearchService
    private var searchTask: Task<Void, Never>?

    init(searchService: SearchService) {
        self.searchService = searchService
    }

    func search(query: String) {
        searchTask?.cancel()

        searchTask = Task { [weak self, searchService] in
            do {
                let results = try await searchService.search(query)
                try Task.checkCancellation()

                guard let self else {
                    return
                }

                self.results = results
                self.error = nil
            } catch is CancellationError {
                // Expected when a newer search replaces this one.
            } catch {
                guard let self else {
                    return
                }

                self.results = []
                self.error = error
            }
        }
    }

    deinit {
        searchTask?.cancel()
    }
}
```

This task has a clear owner. New searches cancel old searches, and `deinit` cancels remaining work.

The task closure uses `[weak self]` so the task does not keep the view model alive while waiting for the service. A strong capture may still be intentional in some designs, but then the extended lifetime must be accepted explicitly.

Risky:

```swift id="sk61oe"
func refresh() {
    Task {
        await syncService.refreshEverything()
    }
}
```

This may be acceptable for app-session fire-and-forget work, but it should be justified. Otherwise, the caller loses cancellation, error handling, completion semantics, and lifetime visibility.

Prefer returning an async operation when the caller owns the work:

```swift id="4xm16x"
func refresh() async throws {
    try await syncService.refreshEverything()
}
```

### Errors in unstructured tasks

If an unstructured task can throw, someone must observe the task result or handle errors inside the task.

Risky:

```swift id="svdlyu"
func startUpload() {
    Task {
        try await uploader.upload()
    }
}
```

The task may fail, but the caller does not await the result and the error is easy to lose.

Prefer handling errors inside the owned task:

```swift id="ct8d8q"
func startUpload() {
    uploadTask?.cancel()

    uploadTask = Task { [weak self, uploader] in
        do {
            try await uploader.upload()
            try Task.checkCancellation()

            await self?.markUploadFinished()
        } catch is CancellationError {
            // Expected cancellation.
        } catch {
            await self?.markUploadFailed(error)
        }
    }
}
```

Or return an async throwing API when the caller owns the operation:

```swift id="n7xrlt"
func upload() async throws {
    try await uploader.upload()
}
```

## `Task.detached`

`Task.detached` creates an unstructured task that is independent from the current task hierarchy and does not inherit actor isolation, task-local values, or priority in the same way as `Task {}`.

It should be rare in app code.

Use it only when the work is intentionally independent from the current task and actor context.

Possible cases:

* independent app-level maintenance work;
* explicitly isolated background processing with its own cancellation and priority policy;
* bridging to a subsystem with a separate lifetime model;
* work that truly should not inherit caller actor isolation and has safe inputs.

Risky:

```swift id="9q6vk4"
@MainActor
func didTapExport() {
    Task.detached {
        let file = try await exporter.exportLargeReport()
        await self.showExportResult(file)
    }
}
```

Review the risks:

* `self` may be captured across an independent task lifetime;
* `exporter` and `self` may be main-actor-isolated or non-Sendable;
* Swift 6 strict concurrency may report isolation or Sendable diagnostics;
* cancellation is unclear;
* priority is unclear;
* task-local context is not inherited as expected;
* the task may outlive the screen;
* UI state must be crossed back to the main actor;
* errors may be lost if not handled.

Use `Task.detached` only when detachment is part of the design, not as a workaround for isolation errors.

Prefer an owned task plus a safe service boundary:

```swift id="pi0vnj"
@MainActor
final class ExportViewModel: ObservableObject {
    @Published private(set) var state: ExportState = .idle

    private let exporter: ReportExporting
    private var exportTask: Task<Void, Never>?

    init(exporter: ReportExporting) {
        self.exporter = exporter
    }

    func export() {
        exportTask?.cancel()

        exportTask = Task { [weak self, exporter] in
            do {
                let file = try await exporter.exportLargeReport()
                try Task.checkCancellation()

                guard let self else {
                    return
                }

                self.state = .finished(file)
            } catch is CancellationError {
                // Expected if the user cancels or leaves.
            } catch {
                guard let self else {
                    return
                }

                self.state = .failed(error)
            }
        }
    }

    func cancelExport() {
        exportTask?.cancel()
        exportTask = nil
    }

    deinit {
        exportTask?.cancel()
    }
}
```

If export CPU work must not inherit `MainActor`, make that true inside `ReportExporting` with a documented non-main-actor execution model. Do not make the UI layer detached by default.

## View and view-model lifetimes

SwiftUI views are value types and can be recreated often. Avoid treating view initialization as the lifetime owner for async work.

Prefer `.task` when the work is tied to view presence.

Prefer `.task(id:)` when the identity of the work matters and old work should be cancelled when the identity changes:

```swift id="y2d8ou"
struct ProfileScreen: View {
    let userID: User.ID

    @State private var model: ProfileModel?
    @State private var error: Error?

    var body: some View {
        ProfileContent(model: model)
            .task(id: userID) {
                do {
                    let loaded = try await loadProfile(userID)
                    try Task.checkCancellation()

                    model = loaded
                    error = nil
                } catch is CancellationError {
                    // View disappeared or userID changed.
                } catch {
                    self.error = error
                }
            }
    }
}
```

SwiftUI cancels the `.task` when the view task is no longer valid, such as when the view disappears or the `id` changes. Cancellation is still cooperative. The underlying operation must observe cancellation, and the task should avoid applying stale results after cancellation or identity changes.

Be careful with `.onAppear { Task { ... } }` for loading work. It can create repeated unstructured tasks when the view appears multiple times, and cancellation/restart behavior is less explicit than `.task(id:)`.

For UIKit, AppKit, or manually owned objects, store task handles and cancel them in the relevant lifecycle boundary.

For view models, store task handles when work can outlive a single method call or when newer operations replace older ones.

## Replacement operations

Replacement operations are tasks where a newer operation makes older results irrelevant.

Common examples:

* search-as-you-type;
* autocomplete;
* filter changes;
* sort changes;
* route recalculation;
* image request for a reused cell;
* screen reload after changing an identifier;
* user switching accounts or organizations.

Rules:

* cancel the previous task before starting a new one;
* check cancellation before applying results;
* avoid showing cancellation as a user-visible error;
* decide whether old work should be cancelled, ignored, or coalesced;
* track request identity when cancellation cannot stop underlying work.

If cancellation cannot stop the underlying operation, use an identity token or request id to ignore stale results:

```swift id="q8mcbt"
@MainActor
final class SearchViewModel: ObservableObject {
    @Published private(set) var results: [SearchResult] = []

    private let searchService: SearchService
    private var searchTask: Task<Void, Never>?
    private var currentSearchID = UUID()

    init(searchService: SearchService) {
        self.searchService = searchService
    }

    func search(query: String) {
        searchTask?.cancel()

        let searchID = UUID()
        currentSearchID = searchID

        searchTask = Task { [weak self, searchService] in
            do {
                let results = try await searchService.search(query)
                try Task.checkCancellation()

                guard let self, self.currentSearchID == searchID else {
                    return
                }

                self.results = results
            } catch is CancellationError {
                // Expected replacement.
            } catch {
                guard let self, self.currentSearchID == searchID else {
                    return
                }

                self.results = []
            }
        }
    }
}
```

Use identity checks when underlying cancellation is delayed, ignored, or not supported by the legacy API.

## Long-lived service tasks

Some tasks are intentionally long-lived: listening to notifications, observing sockets, syncing state, polling, or consuming streams.

Long-lived tasks still need ownership.

A service that starts an observation task should usually have:

* a stored task handle;
* idempotent `start()`;
* explicit `stop()`;
* cancellation in `deinit`;
* cancellation checks inside long loops;
* stream cleanup when the consumer stops;
* duplicate-start protection;
* a clear isolation model for `start()` and `stop()`.

Idempotent `start()` and `stop()` must be protected by the service’s isolation model. If multiple callers can start the service concurrently, use an actor, lock, serial queue, or main-actor isolation to prevent duplicate tasks.

Example shape:

```swift id="uvjg9r"
actor NotificationObserverService {
    private var observationTask: Task<Void, Never>?

    func start() {
        guard observationTask == nil else {
            return
        }

        observationTask = Task { [notificationCenter] in
            for await event in notificationCenter.events {
                if Task.isCancelled {
                    break
                }

                await handle(event)
            }
        }
    }

    func stop() {
        observationTask?.cancel()
        observationTask = nil
    }

    deinit {
        observationTask?.cancel()
    }

    private func handle(_ event: NotificationEvent) async {
        // Process event or forward it to another owned component.
    }
}
```

Review long-lived tasks for:

* duplicate starts;
* missing stop paths;
* missing cancellation checks;
* streams that do not terminate;
* retained producers;
* unbounded buffering;
* work continuing after logout, navigation, or app-session teardown;
* callbacks or delegates that survive longer than the service.

## Common mistakes

### Fire-and-forget without ownership

```swift id="npkq8v"
func save() {
    Task {
        await database.saveChanges()
    }
}
```

This hides failure, cancellation, and completion. Prefer `async throws` if the caller needs the result, or store the task if the object owns it.

### Using `Task {}` as background execution

```swift id="z7m7r5"
@MainActor
func process() {
    Task {
        expensiveCPUWork()
        state = .done
    }
}
```

A task created from `@MainActor` code can inherit main-actor isolation. This does not reliably move CPU work away from UI isolation.

### Using `Task.detached` to avoid `MainActor`

```swift id="q8nd3r"
@MainActor
func process() {
    Task.detached {
        await self.processor.run()
    }
}
```

This may create a data-safety and lifetime problem. Prefer moving CPU-heavy work into a non-main-actor dependency and calling it from an owned task.

### Starting tasks from high-frequency callbacks

```swift id="lk83bt"
func cellDidAppear(item: Item) {
    Task {
        await imageLoader.preload(item.imageURL)
    }
}
```

This can create a task explosion during scrolling. Prefer caching, deduplication, backpressure, or a bounded prefetching pipeline.

### Ignoring replacement semantics

```swift id="611vpw"
func search(query: String) {
    Task {
        results = await service.search(query)
    }
}
```

Older searches may finish after newer ones. Store the task, cancel old work, and check cancellation or request identity before applying results.

### Assuming deallocation cancels everything

Objects do not automatically cancel every task they started. If a task captures `self`, it may also keep `self` alive longer than expected, which can prevent `deinit` from running when you expect it.

### Losing errors in unstructured tasks

```swift id="rb9wcb"
func sync() {
    Task {
        try await service.sync()
    }
}
```

If nobody awaits the task value or handles errors inside the task, failure is easy to lose. Make error handling part of the task ownership model.

## Review checklist

When reviewing task lifetime and structure, check:

* [ ] Is the task structured under a parent operation when possible?
* [ ] If the task is unstructured, is the owner explicit?
* [ ] Is the task handle stored when later cancellation is required?
* [ ] Is there a cancellation point when newer work replaces older work?
* [ ] Does deinit or flow teardown cancel owned tasks?
* [ ] Could the task retain its owner and prevent `deinit`?
* [ ] Does the task update state only if the result is still relevant?
* [ ] Are errors handled intentionally instead of disappearing?
* [ ] Is `Task {}` being used as a fake background boundary?
* [ ] Is `Task.detached` justified by a real independence requirement?
* [ ] Are Swift 6 Sendable and actor-isolation diagnostics addressed correctly?
* [ ] Could repeated calls create many overlapping tasks?
* [ ] Could scrolling, typing, or lifecycle callbacks create task explosions?
* [ ] Are long-lived tasks protected against duplicate starts?
* [ ] Is there a clear stop path for long-lived observation loops?
* [ ] Does validation prove tasks stop when the owner disappears?
* [ ] Does validation prove stale results do not apply?

## Validation

Use validation that matches the suspected lifetime issue.

### Cancellation and navigation

* log task start, cancellation, and completion;
* navigate away and verify the operation stops;
* verify stale results are not applied after cancellation;
* add cancellation tests for view-model-owned work;
* verify underlying work observes cancellation when possible.

### Replacement operations

* start request A;
* start request B before A completes;
* verify A is cancelled or ignored safely;
* verify only B can update state;
* verify cancellation from A is not shown as a user-visible error;
* verify errors from stale requests do not overwrite current state.

### Task explosions

* add temporary counters for task creation and completion;
* reproduce with fast typing, rapid navigation, or fast scrolling;
* inspect Instruments for high task counts or long-lived tasks;
* verify bounded concurrency, coalescing, caching, or deduplication reduces task count.

### Retained owners

* use memory graph debugging;
* add temporary `deinit` logs during investigation;
* verify the owner deallocates after navigation;
* check whether task closures capture `self` longer than intended;
* check whether underlying work ignores cancellation and keeps the task alive.

### Detached or long-lived tasks

* verify explicit start and stop behavior;
* verify logout, cancellation, or app-session teardown stops the work;
* check that priority and cancellation are intentional;
* inspect whether the task keeps running after the triggering UI disappears;
* verify duplicate starts do not create multiple loops.

A task lifetime refactor is successful only if the work now has a clear owner, a clear cancellation path, intentional error handling, stale-result protection, and observable evidence that unnecessary work stops.

## Related references

* `cancellation-and-task-lifetime.md` — use for cancellation propagation, swallowed cancellation, navigation cancellation, and cancellation tests.
* `mainactor-responsiveness.md` — use for `MainActor`, UI state, main-thread stalls, and moving CPU-heavy work away from UI isolation.
* `bounded-task-groups.md` — use for task groups, fan-out work, and limiting concurrency.
* `blocking-legacy-apis.md` — use for synchronous I/O, semaphores, blocking SDKs, and async wrappers around legacy APIs.
* `asyncsequence-and-stream-cleanup.md` — use for stream lifetime, buffering, producer cleanup, and `onTermination`.
* `continuation-safety.md` — use for callback/delegate bridges, exactly-once resume, and cancellation races.
