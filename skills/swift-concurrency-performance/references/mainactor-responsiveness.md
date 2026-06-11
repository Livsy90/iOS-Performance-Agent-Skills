# MainActor Responsiveness

Use this reference when the task involves `MainActor`, `@MainActor` types, UI state, view models, main-thread stalls, `@concurrent`, or moving CPU-heavy work away from UI isolation.

## Contents

- [Core model](#core-model)
- [When MainActor is appropriate](#when-mainactor-is-appropriate)
- [When MainActor becomes a performance problem](#when-mainactor-becomes-a-performance-problem)
- [Review workflow](#review-workflow)
- [Decision rules](#decision-rules)
- [Refactoring patterns](#refactoring-patterns)
  - [Keep UI mutation on MainActor, move CPU work out](#keep-ui-mutation-on-mainactor-move-cpu-work-out)
  - [Avoid making an entire service MainActor-isolated](#avoid-making-an-entire-service-mainactor-isolated)
  - [Use nonisolated helpers for pure work](#use-nonisolated-helpers-for-pure-work)
  - [Use @concurrent deliberately](#use-concurrent-deliberately)
  - [Avoid chatty MainActor hops](#avoid-chatty-mainactor-hops)
- [Common mistakes](#common-mistakes)
- [Code review checklist](#code-review-checklist)
- [Validation](#validation)
- [Output guidance](#output-guidance)

## Core model

`MainActor` is an isolation domain for work that must coordinate with the main thread, especially UI state and main-thread-only framework APIs.

It is not a performance optimization by itself.

A function isolated to `MainActor` runs as part of the main-actor execution context. That is useful for protecting UI state, but dangerous when the isolated function performs CPU-heavy work, synchronous I/O, parsing, image processing, large formatting passes, or long loops.

The key review question is:

> Is this code on `MainActor` because it must touch UI state, or because the type was marked `@MainActor` for convenience?

When a type such as a view model is marked `@MainActor`, all instance methods and mutable state usually inherit that isolation. This can be correct for UI-facing state, but it can also accidentally pull non-UI work onto the main actor.

## When MainActor is appropriate

`MainActor` is usually appropriate for:

- mutating UI state that is observed by SwiftUI or UIKit;
- presenting UI, navigation, alerts, sheets, and other main-thread-only actions;
- interacting with UIKit, AppKit, or SwiftUI APIs that require main-thread access;
- coordinating short state transitions before or after async work;
- exposing a UI-facing view model API that should be called from views.

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
        } catch {
            state = .failed(error)
        }
    }
}
```

This is usually acceptable if `service.fetchProfile()` does not perform heavy synchronous work on the main actor and if the state transitions are short.

## When MainActor becomes a performance problem

`MainActor` becomes suspicious when the isolated code performs work that does not need UI isolation.

Look for:

- JSON decoding or encoding inside a `@MainActor` type;
- image decoding, resizing, compression, hashing, or file processing;
- large sorting, filtering, diffing, grouping, formatting, or mapping;
- database reads or synchronous file I/O;
- loops over large collections;
- expensive computed properties read by views;
- repeated calls from scrolling, typing, gestures, or animation paths;
- broad `@MainActor` annotations on services, repositories, clients, formatters, or caches;
- `await MainActor.run` blocks that contain more than the final UI mutation;
- traces showing long main-thread slices under async functions.

The performance issue is not the annotation itself. The issue is accidentally serializing heavy non-UI work through the main actor.

## Review workflow

1. Identify the user-visible symptom: stall, delayed tap response, slow screen transition, typing lag, scrolling hitch, animation hitch, or delayed state update.
2. Locate the `MainActor` boundary: `@MainActor` type, method, property, closure, `MainActor.run`, or SwiftUI lifecycle callback.
3. Separate UI state mutation from pure computation, parsing, formatting, I/O, or data transformation.
4. Check whether broad type-level isolation pulled helper methods onto the main actor.
5. Check whether async calls actually suspend or whether they wrap blocking work.
6. Check whether the first screen or current interaction awaits the result.
7. Propose the smallest refactor that keeps UI mutation isolated but moves heavy non-UI work away.
8. Validate with a before/after signal.

## Decision rules

- Keep UI state mutations on `MainActor`.
- Keep main-actor isolated sections short.
- Do not make a type `@MainActor` just because one property is UI-facing.
- Do not put parsing, image work, database work, or large transformations in main-actor isolated methods.
- Prefer moving pure work into a nonisolated helper, a separate non-main-actor service, or an async API that does not block the main actor.
- Use `MainActor.run` only around the UI mutation, not around the whole operation.
- If a method belongs to a `@MainActor` type but does not need isolated state, consider `nonisolated`.
- If a method should run concurrently instead of inheriting actor isolation, consider `@concurrent` only after checking Sendable and lifetime requirements.
- Avoid many tiny `MainActor` hops in a hot loop. Batch the result and cross isolation once.
- Do not move work away from `MainActor` if it reads or mutates UI state, touches main-thread-only APIs, or relies on main-actor protected invariants.

## Refactoring patterns

### Keep UI mutation on MainActor, move CPU work out

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

Prefer:

```swift
@MainActor
final class SearchViewModel: ObservableObject {
    @Published private(set) var results: [SearchResult] = []

    func applyResponse(_ response: SearchResponse) async {
        let mapped = await SearchMapper.visibleResults(from: response)
        results = mapped
    }
}

enum SearchMapper {
    static func visibleResults(from response: SearchResponse) async -> [SearchResult] {
        response.items
            .filter { $0.isVisible }
            .sorted { $0.score > $1.score }
            .map(SearchResult.init)
    }
}
```

This keeps the final state mutation on the main actor and makes the transformation easier to move, test, and measure.

If the work is purely synchronous and CPU-heavy, the exact execution strategy depends on the app. The important part for review is not to hide the cost inside a main-actor method.

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

### Use nonisolated helpers for pure work

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

Prefer making pure work independent from UI isolation:

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

This is useful only when the helper does not access isolated state and its inputs/outputs are safe to use across the isolation boundary.

### Use @concurrent deliberately

`@concurrent` can be useful when a method on an actor-isolated type should run concurrently instead of inheriting the actor's isolation.

Do not add it as a reflex.

Before recommending `@concurrent`, check:

- the method does not access actor-isolated mutable state;
- values crossing the boundary are safe to send;
- cancellation behavior remains clear;
- the method's lifetime is still owned by the caller;
- the method does not touch UI APIs;
- the result is applied back to UI state in a short main-actor section.

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

Use this pattern only when the compiler model, deployment target, and project settings support it. If there is uncertainty, prefer a separate non-main-actor helper or service.

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

## Common mistakes

- Marking a whole view model `@MainActor` and then doing all parsing, mapping, sorting, and formatting inside it.
- Treating `@MainActor` as a safety blanket for every UI-adjacent object.
- Moving code off `MainActor` without checking whether it touches UI state or main-thread-only APIs.
- Using `Task.detached` to escape the main actor instead of fixing isolation boundaries.
- Wrapping the whole async operation in `MainActor.run`.
- Calling `await MainActor.run` inside a tight loop.
- Adding `nonisolated` or `@concurrent` without checking Sendable and state access.
- Assuming an async function called from a `@MainActor` context automatically runs away from the main actor.
- Ignoring first-interaction latency because the work eventually suspends.
- Optimizing based on one local run instead of a repeatable before/after signal.

## Code review checklist

When reviewing `MainActor` responsiveness, check:

- [ ] What user-visible symptom is being explained?
- [ ] Which type, method, property, or closure is main-actor isolated?
- [ ] Does the isolated code touch UI state or main-thread-only APIs?
- [ ] Is there CPU-heavy work inside the isolated region?
- [ ] Is there synchronous I/O, blocking waiting, or legacy callback bridging inside the isolated region?
- [ ] Is a broad `@MainActor` annotation pulling helper methods onto the main actor?
- [ ] Can pure work become `nonisolated`, static, or part of a separate service?
- [ ] Would moving work away from the main actor violate UI state safety?
- [ ] Are actor hops batched rather than repeated in a hot loop?
- [ ] Does the first screen or current interaction await this work?
- [ ] Is cancellation preserved after the refactor?
- [ ] Is there a validation plan?

## Validation

Choose validation based on the suspected issue.

Use Instruments when checking:

- long main-thread slices;
- UI stalls during tap, scroll, typing, or navigation;
- actor hops or executor behavior;
- high task counts;
- blocked cooperative threads;
- repeated updates caused by many small main-actor crossings.

Use signposts when comparing:

- tap-to-response latency;
- request-to-state-update latency;
- decode/transform time before UI update;
- before/after cost of moving work out of main-actor isolation.

Use logs when checking:

- whether work starts on the expected path;
- whether cancellation happens after navigation;
- whether results are applied once or repeatedly;
- whether batched UI updates replaced per-item updates.

Use UI behavior when checking:

- first interaction responsiveness;
- whether loading indicators appear promptly;
- whether scrolling or typing remains smooth;
- whether progressive rendering still feels correct after batching.

A successful refactor should show one or more of these signals:

- shorter main-thread blocking intervals;
- fewer main-actor crossings in the hot path;
- less work before the first visible UI update;
- smoother interaction during async work;
- no lost cancellation;
- no UI state mutation outside the main actor.

## Output guidance

When giving advice, be precise about the boundary.

Prefer:

> Keep the final `state = .loaded(...)` assignment on `MainActor`, but move the response mapping into a nonisolated helper because it does not read UI state and may run over a large collection.

Avoid:

> Move this off the main thread.

Explain:

- what work should stay on `MainActor`;
- what work can move away;
- why the move is safe;
- what trade-off it introduces;
- how to validate the result.
