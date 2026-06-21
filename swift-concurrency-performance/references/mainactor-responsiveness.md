# MainActor Responsiveness

Use this reference when the task involves `MainActor`, `@MainActor` types, UI state, view models, main-thread stalls, `@concurrent`, `nonisolated`, SwiftUI responsiveness, UIKit/AppKit isolation, or moving CPU-heavy work away from UI isolation.

This reference is about responsiveness and isolation boundaries. It is not a general guide to Swift Concurrency.

## Contents

* [Core model](#core-model)
* [When MainActor is appropriate](#when-mainactor-is-appropriate)
* [When MainActor becomes a performance problem](#when-mainactor-becomes-a-performance-problem)
* [Review workflow](#review-workflow)
* [Decision rules](#decision-rules)
* [Important caveats](#important-caveats)

  * [`async` is not a background boundary](#async-is-not-a-background-boundary)
  * [`nonisolated` does not automatically move synchronous work](#nonisolated-does-not-automatically-move-synchronous-work)
  * [`@concurrent` is an explicit isolation decision](#concurrent-is-an-explicit-isolation-decision)
  * [`MainActor.run` should be small](#mainactorrun-should-be-small)
  * [`Task.detached` is not a default escape hatch](#taskdetached-is-not-a-default-escape-hatch)
* [Refactoring patterns](#refactoring-patterns)

  * [Keep UI mutation on MainActor, move pure work behind an explicit boundary](#keep-ui-mutation-on-mainactor-move-pure-work-behind-an-explicit-boundary)
  * [Avoid making an entire service MainActor-isolated](#avoid-making-an-entire-service-mainactor-isolated)
  * [Use nonisolated helpers for isolation cleanup, not as a thread hop](#use-nonisolated-helpers-for-isolation-cleanup-not-as-a-thread-hop)
  * [Use @concurrent deliberately](#use-concurrent-deliberately)
  * [Avoid chatty MainActor hops](#avoid-chatty-mainactor-hops)
* [Common mistakes](#common-mistakes)
* [Code review checklist](#code-review-checklist)
* [Validation](#validation)
* [Output guidance](#output-guidance)

## Core model

`MainActor` is an isolation domain for work that must coordinate with the main thread, especially UI state and main-thread-only framework APIs.

It is not a performance optimization by itself.

A function isolated to `MainActor` runs as part of the main-actor execution context. That is useful for protecting UI state, navigation state, and framework interactions, but dangerous when the isolated function performs CPU-heavy work, synchronous I/O, parsing, image processing, large formatting passes, large collection transformations, or long loops.

The key review question is:

```text
Is this code on MainActor because it must touch UI state, or because the type was marked @MainActor for convenience?
```

When a type such as a view model is marked `@MainActor`, instance methods and mutable instance state normally inherit that isolation. This can be correct for UI-facing state, but it can also accidentally pull non-UI work onto the main actor.

The review goal is not to remove `MainActor`. The goal is to keep the UI-critical mutation on `MainActor` and keep expensive non-UI work out of the main-actor interval when it is safe to do so.

## When MainActor is appropriate

`MainActor` is usually appropriate for:

* mutating UI state observed by SwiftUI, UIKit, or AppKit;
* presenting UI, navigation, alerts, sheets, and other main-thread-only actions;
* interacting with UIKit, AppKit, or SwiftUI APIs that require main-thread access;
* coordinating short state transitions before or after async work;
* exposing a UI-facing view model API that should be called from views;
* protecting invariants that belong to UI state.

Good use:

```swift
@MainActor
final class ProfileViewModel: ObservableObject {
    @Published private(set) var state: State = .idle

    func load() async {
        state = .loading

        do {
            let profile = try await service.fetchProfile()
            state = .loaded(profile)
        } catch is CancellationError {
            state = .idle
        } catch {
            state = .failed(error)
        }
    }
}
```

This is usually acceptable if:

* `service.fetchProfile()` is truly asynchronous or otherwise does not block the main actor;
* the state transitions are short;
* heavy parsing, mapping, sorting, decoding, or formatting does not happen inside the main-actor-isolated section;
* cancellation and stale-result behavior are handled according to the feature’s lifetime rules.

## When MainActor becomes a performance problem

`MainActor` becomes suspicious when isolated code performs work that does not need UI isolation.

Look for:

* JSON decoding or encoding inside a `@MainActor` type;
* image decoding, resizing, compression, hashing, or file processing;
* large sorting, filtering, diffing, grouping, formatting, or mapping;
* database reads or synchronous file I/O;
* loops over large collections;
* expensive computed properties read by views;
* repeated calls from scrolling, typing, gestures, or animation paths;
* broad `@MainActor` annotations on services, repositories, clients, formatters, decoders, mappers, or caches;
* `await MainActor.run` blocks that contain more than the final UI mutation;
* traces showing long main-thread slices under async functions;
* async helpers that still run synchronous work before the first suspension;
* result aggregation from task groups or streams happening on `MainActor`.

The performance issue is not the annotation itself. The issue is accidentally serializing heavy non-UI work through the main actor.

## Review workflow

1. Identify the user-visible symptom: stall, delayed tap response, slow screen transition, typing lag, scrolling hitch, animation hitch, delayed loading indicator, or delayed state update.
2. Locate the `MainActor` boundary: `@MainActor` type, method, property, closure, `MainActor.run`, SwiftUI `.task`, `onAppear`, UIKit callback, or UI-facing view model method.
3. Separate UI state mutation from pure computation, parsing, formatting, I/O, data transformation, or aggregation.
4. Check whether broad type-level isolation pulled helper methods onto the main actor.
5. Check whether async calls actually suspend or whether they wrap blocking or CPU-heavy work.
6. Check whether the first screen, current interaction, or loading feedback awaits the result.
7. Check cancellation and stale-result behavior before changing task lifetime.
8. Propose the smallest refactor that keeps UI mutation isolated but moves safe non-UI work away from the main-actor interval.
9. Validate with a before/after signal.

## Decision rules

* Keep UI state mutations on `MainActor`.
* Keep main-actor-isolated sections short.
* Do not make a type `@MainActor` just because one property or method is UI-facing.
* Do not put parsing, image work, database work, synchronous I/O, or large transformations in main-actor-isolated methods.
* Treat `await` as a possible suspension point, not proof that work moved away from the main actor.
* Treat `async` as an API/lifetime model, not as a background execution guarantee.
* Prefer moving pure work into a separate non-main-actor service, pure value helper, worker, or explicit execution boundary.
* Use `MainActor.run` only around the UI mutation, not around the whole operation.
* If a method belongs to a `@MainActor` type but does not need isolated state, consider `nonisolated` to clarify the isolation boundary.
* Remember that `nonisolated` on a synchronous helper does not by itself move work to another executor or thread.
* If a method should explicitly switch away from actor isolation, consider `@concurrent` only when the compiler, language mode, deployment target, `Sendable` requirements, and lifetime model support it.
* Avoid many tiny `MainActor` hops in a hot loop. Batch, throttle, or coalesce updates when incremental rendering is not required.
* Do not move work away from `MainActor` if it reads or mutates UI state, touches main-thread-only APIs, or relies on main-actor-protected invariants.
* Do not use `Task.detached` as the default way to escape `MainActor`.
* Validate the refactor with traces, signposts, UI behavior, or controlled tests.

## Important caveats

### `async` is not a background boundary

An async function can still run synchronous work before its first suspension.

Risky assumption:

```swift
@MainActor
func reload() async {
    let model = await builder.buildModel()
    self.model = model
}
```

This is only safe for responsiveness if `builder.buildModel()` does not run heavy synchronous work on the main actor before suspending or if it has an explicit execution boundary that keeps the heavy work out of the main-actor interval.

Do not use `async` as proof that CPU work moved off the main actor.

Depending on Swift language mode, compiler settings, and isolation annotations, a nonisolated async helper may run on the caller’s actor by default or may explicitly switch off actor isolation. If the goal is main-actor responsiveness, make the execution boundary explicit and validate it.

### `nonisolated` does not automatically move synchronous work

`nonisolated` removes actor isolation from a declaration when the declaration does not need isolated state.

It does not automatically move synchronous work to another thread.

Example:

```swift
@MainActor
final class OrdersViewModel: ObservableObject {
    @Published private(set) var sections: [OrderSection] = []

    func reload() async {
        let orders = try? await repository.loadOrders()
        sections = Self.buildSections(from: orders ?? [])
    }

    nonisolated private static func buildSections(from orders: [Order]) -> [OrderSection] {
        Dictionary(grouping: orders, by: \.day)
            .map(OrderSection.init)
            .sorted { $0.day > $1.day }
    }
}
```

This improves the isolation model because `buildSections` no longer depends on `MainActor` state.

But because `buildSections` is synchronous and called directly from `reload()`, the computation still runs during that main-actor call. This pattern is useful for correctness, reuse, and testability. It is not enough by itself for CPU-heavy work that must be kept out of the main-actor interval.

For heavy work, combine this with an execution strategy that actually keeps the work away from `MainActor`.

### `@concurrent` is an explicit isolation decision

`@concurrent` can be used on async functions that should always switch off an actor to run.

Use it deliberately. It implies a stronger isolation boundary than ordinary helper extraction.

Before recommending `@concurrent`, check:

* the project’s Swift version and language mode support it;
* the method does not access actor-isolated mutable state;
* the method does not touch UI APIs;
* values crossing the boundary are safe to send;
* arguments and results satisfy the required data-race safety rules;
* cancellation behavior remains clear;
* the method’s lifetime is still owned by the caller;
* the result is applied back to UI state in a short main-actor section.

Do not use `@concurrent` to silence isolation problems. Use it to express an intentional execution boundary.

### `MainActor.run` should be small

`MainActor.run` is useful when non-main-actor code needs to perform a short UI mutation.

Prefer:

```swift
let model = try await worker.buildModel()

await MainActor.run {
    state = .loaded(model)
}
```

Avoid:

```swift
await MainActor.run {
    let model = expensiveMapping(response)
    state = .loaded(model)
}
```

If the current function is already `@MainActor`, wrapping code in `MainActor.run` is usually unnecessary. The work is already main-actor-isolated.

Use `MainActor.run` mainly to return to UI isolation for a compact state update.

### `Task.detached` is not a default escape hatch

`Task.detached` creates independent work outside the current structured task and actor context.

That can be correct for truly independent app-level work, but it changes important semantics:

* lifetime ownership;
* cancellation propagation;
* priority expectations;
* task-local values;
* actor isolation;
* `Sendable` requirements;
* result delivery.

Do not replace a main-actor performance problem with an unowned detached task. Prefer a clear worker/service boundary, structured concurrency, bounded work, and explicit cancellation.

## Refactoring patterns

### Keep UI mutation on MainActor, move pure work behind an explicit boundary

Risky:

```swift
@MainActor
final class SearchViewModel: ObservableObject {
    @Published private(set) var results: [SearchResult] = []

    func applyResponse(_ response: SearchResponse) {
        let mapped = response.items
            .filter { $0.isVisible }
            .sorted { $0.score > $1.score }
            .map(SearchResult.init)

        results = mapped
    }
}
```

The mapping may be harmless for small input, but it is suspicious when the response is large or this runs often.

Better boundary:

```swift
@MainActor
final class SearchViewModel: ObservableObject {
    @Published private(set) var results: [SearchResult] = []

    func applyResponse(_ response: SearchResponse) async {
        let mapped = await searchMappingService.visibleResults(from: response)
        results = mapped
    }

    private let searchMappingService: SearchMappingService

    init(searchMappingService: SearchMappingService) {
        self.searchMappingService = searchMappingService
    }
}

struct SearchMappingService {
    func visibleResults(from response: SearchResponse) async -> [SearchResult] {
        response.items
            .filter { $0.isVisible }
            .sorted { $0.score > $1.score }
            .map(SearchResult.init)
    }
}
```

This separates UI mutation from transformation, but do not assume this alone proves the work runs away from the main actor in every Swift language mode and compiler setting.

When the transformation is CPU-heavy and must not run during the main-actor interval, make the execution behavior explicit. Options depend on the project:

* use a separate worker/service with a documented isolation model;
* use an explicitly concurrent async function when supported and safe;
* use bounded task groups for large independent work;
* use a measured legacy boundary for blocking work;
* keep the work synchronous but call it outside main-actor-isolated code;
* validate with signposts or Instruments.

The important review point is not “make it async.” The important point is “keep expensive non-UI work out of the main-actor interval and prove that it moved.”

### Avoid making an entire service MainActor-isolated

Risky:

```swift
@MainActor
final class ImageService {
    func thumbnail(for data: Data) -> UIImage? {
        decodeAndResize(data)
    }

    func update(imageView: UIImageView, image: UIImage) {
        imageView.image = image
    }
}
```

This mixes UI work and CPU-heavy image work in one isolation domain.

Prefer separating responsibilities:

```swift
struct ImageDecoder {
    func thumbnail(for data: Data) -> UIImage? {
        decodeAndResize(data)
    }
}

@MainActor
final class ImagePresenter {
    func update(imageView: UIImageView, image: UIImage) {
        imageView.image = image
    }
}
```

The service that performs image processing should not become main-actor isolated only because another method touches UIKit.

If image decoding or resizing is expensive, also validate where it executes and whether it blocks a user-visible path.

### Use nonisolated helpers for isolation cleanup, not as a thread hop

When a type is `@MainActor`, pure helpers may accidentally inherit main-actor isolation.

Risky:

```swift
@MainActor
final class OrdersViewModel: ObservableObject {
    @Published private(set) var sections: [OrderSection] = []

    func reload() async {
        let orders = try? await repository.loadOrders()
        sections = buildSections(from: orders ?? [])
    }

    private func buildSections(from orders: [Order]) -> [OrderSection] {
        Dictionary(grouping: orders, by: \.day)
            .map(OrderSection.init)
            .sorted { $0.day > $1.day }
    }
}
```

Better isolation boundary:

```swift
@MainActor
final class OrdersViewModel: ObservableObject {
    @Published private(set) var sections: [OrderSection] = []

    func reload() async {
        let orders = try? await repository.loadOrders()
        sections = Self.buildSections(from: orders ?? [])
    }

    nonisolated private static func buildSections(from orders: [Order]) -> [OrderSection] {
        Dictionary(grouping: orders, by: \.day)
            .map(OrderSection.init)
            .sorted { $0.day > $1.day }
    }
}
```

This is useful only when the helper does not access isolated state and its inputs/outputs are safe to use across the intended boundary.

However, this synchronous helper still runs where it is called. If `reload()` is `@MainActor`, the call to `Self.buildSections(...)` still happens during the main-actor operation.

Use this pattern to make isolation explicit and prevent accidental state access. For CPU-heavy work, add a real execution boundary and validate that the main-actor interval is reduced.

### Use @concurrent deliberately

`@concurrent` can be useful when an async method on an actor-isolated type should explicitly run away from actor isolation.

Do not add it as a reflex.

Pattern:

```swift
@MainActor
final class ReportViewModel: ObservableObject {
    @Published private(set) var summary: ReportSummary?

    func reload() async throws {
        let data = try await repository.loadReportData()
        let summary = await makeSummary(from: data)

        self.summary = summary
    }

    @concurrent
    private func makeSummary(from data: ReportData) async -> ReportSummary {
        ReportSummaryBuilder.build(from: data)
    }
}
```

Use this pattern only when all of these are true:

* the project supports the required Swift language mode and compiler behavior;
* `makeSummary` does not access `self`, `summary`, or other main-actor-isolated state;
* `makeSummary` does not touch UI APIs;
* `ReportData` and `ReportSummary` are safe to send across the isolation boundary;
* cancellation remains owned by the caller;
* the final UI mutation remains short and main-actor-isolated.

If there is uncertainty, prefer a separate non-main-actor helper or service with a clearer isolation model.

Do not recommend `@concurrent` when the real issue is blocking legacy work, unbounded parallelism, or a missing cancellation model. Use the more specific reference for those cases.

### Avoid chatty MainActor hops

Risky:

```swift
for item in items {
    let value = await worker.process(item)

    await MainActor.run {
        results.append(value)
    }
}
```

This crosses to the main actor once per item and may cause repeated UI updates.

Prefer batching when the UI does not need incremental updates:

```swift
let values = await worker.process(items)

await MainActor.run {
    results = values
}
```

Incremental updates may still be correct for progressive rendering. In that case, throttle, batch, or coalesce updates so the UI is not invalidated excessively.

Good questions:

* Does the user need every intermediate value?
* Is this progressive rendering or accidental churn?
* Can updates be delivered every N items or every short interval?
* Does each update trigger layout, diffing, animation, or expensive SwiftUI body work?
* Would a single final assignment feel better?

## Common mistakes

* Marking a whole view model `@MainActor` and then doing all parsing, mapping, sorting, and formatting inside it.
* Treating `@MainActor` as a safety blanket for every UI-adjacent object.
* Moving code off `MainActor` without checking whether it touches UI state or main-thread-only APIs.
* Using `Task.detached` to escape the main actor instead of fixing isolation boundaries.
* Wrapping the whole async operation in `MainActor.run`.
* Calling `await MainActor.run` inside a tight loop.
* Adding `nonisolated` and assuming synchronous work moved to a background thread.
* Adding `@concurrent` without checking Swift version, Sendable requirements, state access, and lifetime.
* Assuming an async function called from a `@MainActor` context automatically runs away from the main actor.
* Hiding blocking or CPU-heavy work behind an async helper.
* Ignoring first-interaction latency because the work eventually suspends.
* Moving result aggregation from a task group onto `MainActor`.
* Optimizing based on one local Debug run instead of a repeatable before/after signal.

## Code review checklist

When reviewing `MainActor` responsiveness, check:

* [ ] What user-visible symptom is being explained?
* [ ] Which type, method, property, closure, or lifecycle callback is main-actor isolated?
* [ ] Does the isolated code touch UI state or main-thread-only APIs?
* [ ] Is there CPU-heavy work inside the isolated region?
* [ ] Is there synchronous I/O, blocking waiting, or legacy callback bridging inside the isolated region?
* [ ] Is a broad `@MainActor` annotation pulling helper methods onto the main actor?
* [ ] Is an `async` helper being treated as a background boundary without proof?
* [ ] Is `nonisolated` being used only to clarify isolation, or incorrectly as a thread hop?
* [ ] Would `@concurrent` be valid under the project’s Swift version and data-race safety requirements?
* [ ] Are arguments and results crossing isolation boundaries safe to send?
* [ ] Can pure work become a static helper, separate service, worker, or explicit concurrent boundary?
* [ ] Would moving work away from the main actor violate UI state safety?
* [ ] Are actor hops batched rather than repeated in a hot loop?
* [ ] Does the first screen or current interaction await this work?
* [ ] Is cancellation preserved after the refactor?
* [ ] Is stale-result handling preserved after the refactor?
* [ ] Is there a validation plan?

## Validation

Choose validation based on the suspected issue.

Use Instruments when checking:

* long main-thread slices;
* UI stalls during tap, scroll, typing, or navigation;
* actor hops or executor behavior;
* high task counts;
* blocked cooperative threads;
* repeated updates caused by many small main-actor crossings;
* CPU-heavy work before the first visible UI update.

Use signposts when comparing:

* tap-to-response latency;
* request-to-state-update latency;
* screen appear to first visible feedback;
* decode/transform time before UI update;
* time spent inside main-actor-isolated code;
* before/after cost of moving work out of main-actor isolation.

Use logs when checking:

* whether work starts on the expected path;
* whether cancellation happens after navigation;
* whether stale results are ignored;
* whether results are applied once or repeatedly;
* whether batched UI updates replaced per-item updates;
* whether helper code still runs before the first suspension.

Use UI behavior when checking:

* first interaction responsiveness;
* whether loading indicators appear promptly;
* whether scrolling or typing remains smooth;
* whether progressive rendering still feels correct after batching;
* whether state updates remain correct and predictable.

Use Release or release-like builds for timing comparisons. Debug traces are useful for verifying signposts and logic, but they are not final performance evidence because optimization level can significantly affect timing, allocation behavior, SwiftUI overhead, and concurrency overhead.

A successful refactor should show one or more of these signals:

* shorter main-thread blocking intervals;
* less time spent in main-actor-isolated code;
* fewer main-actor crossings in the hot path;
* less work before the first visible UI update;
* smoother interaction during async work;
* no lost cancellation;
* no stale UI updates;
* no UI state mutation outside the main actor.

## Output guidance

When giving advice, be precise about the boundary.

Prefer:

```text
Keep the final `state = .loaded(...)` assignment on `MainActor`, but move the response mapping behind a non-main-actor execution boundary because it does not read UI state and may run over a large collection. Validate with a signpost from response received to state applied and confirm the main-actor interval shrinks.
```

Avoid:

```text
Move this off the main thread.
```

Explain:

* what work should stay on `MainActor`;
* what work can move away;
* whether `async`, `nonisolated`, `@concurrent`, a worker service, or a different boundary is appropriate;
* why the move is safe;
* what `Sendable`, isolation, lifetime, and cancellation requirements apply;
* what trade-off it introduces;
* how to validate the result.

When uncertain, say what evidence is missing. Do not claim a responsiveness improvement until the main-actor interval, first-feedback latency, UI smoothness, or another relevant signal improves.
