# Swift 6.2 Isolation and `@concurrent`

Use this reference when the task involves Swift 6.2 isolation behavior, default actor isolation, `@concurrent`, `nonisolated`, Sendable boundaries, or migration-related performance regressions.

This reference is for performance review. It helps the agent decide where async work actually runs, whether it accidentally remains on `MainActor`, and whether moving work across an isolation boundary is safe.

## Contents

- [Core model](#core-model)
- [Review workflow](#review-workflow)
- [Default actor isolation](#default-actor-isolation)
- [`async` does not mean background](#async-does-not-mean-background)
- [`@concurrent`](#concurrent)
- [`nonisolated`](#nonisolated)
- [`nonisolated` vs `@concurrent`](#nonisolated-vs-concurrent)
- [Sendable boundary review](#sendable-boundary-review)
- [MainActor refactoring pattern](#mainactor-refactoring-pattern)
- [Migration-related performance regressions](#migration-related-performance-regressions)
- [Common mistakes](#common-mistakes)
- [Diagnostics](#diagnostics)
- [Review checklist](#review-checklist)
- [Source notes](#source-notes)

## Core model

Do not assume that `async` means “background”.

In Swift 6.2-era code, the execution context of an async function depends on actor isolation, compiler settings, and annotations. A function can be async and still run on the caller’s actor.

For performance review, always ask:

- What actor is the caller isolated to?
- Is the enclosing type explicitly or implicitly `@MainActor`?
- Is default actor isolation enabled for this target or module?
- Is the function actor-isolated, `nonisolated`, or `@concurrent`?
- Does the function only suspend, or does it perform meaningful CPU work?
- Are parameters, captures, and return values safe to send across an isolation boundary?
- Is the suspected performance issue caused by where the work runs, or by the amount of work itself?

## Review workflow

1. 1Identify the user-visible symptom: UI stall, slow interaction, actor queue buildup, new Sendable diagnostics, or regression after enabling Swift 6.2 settings.
2. 2Locate the caller’s isolation domain.
3. 3Locate the callee’s isolation behavior.
4. 4Check whether heavy work is running on `MainActor` or another hot actor.
5. 5Check whether moving the work away would cross a Sendable boundary.
6. 6Prefer immutable snapshots over moving mutable UI-owned reference objects.
7. 7Use `@concurrent` only when leaving the caller’s actor is intentional and useful.
8. 8Use `nonisolated` only when the member does not need isolated state.
9. 9Validate the change with traces, signposts, UI responsiveness, or targeted tests.

## Default actor isolation

Swift 6.2 adds project and package settings that can make declarations in a module infer `@MainActor` isolation by default.

This can improve approachability for UI-heavy apps, but it changes the performance review question:

> Is this code explicitly main-actor isolated, or did it become main-actor isolated because of the target’s default isolation setting?

When default `MainActor` isolation is enabled, unannotated declarations may become main-actor isolated unless another rule applies.

Review these cases carefully:

- view models that combine UI state and data processing;
- services placed in app targets rather than framework targets;
- helper functions near UI code;
- static utilities in UI modules;
- protocol conformances that inherit isolation;
- code that behaved differently after enabling Swift 6.2 settings.

Prefer checking the build setting instead of guessing from the source file alone.

### App-target risk

Risky pattern:

```swift
// Target built with default MainActor isolation.

final class SearchModel {
    private(set) var rows: [SearchRow] = []

    func apply(_ payload: SearchPayload) {
        rows = SearchRowBuilder.buildRows(from: payload)
    }
}
```

The code may look like a plain class, but in a target with default `MainActor` isolation it can become main-actor isolated. The row-building work may run on the main actor.

Prefer separating UI coordination from processing:

```swift
@MainActor
final class SearchModel {
    private(set) var rows: [SearchRow] = []

    func apply(_ payload: SearchPayload) async {
        rows = await buildSearchRows(from: payload.snapshot)
    }
}

@concurrent
func buildSearchRows(from snapshot: SearchPayloadSnapshot) async -> [SearchRow] {
    SearchRowBuilder.buildRows(from: snapshot)
}
```

Before recommending this, verify that the snapshot and result are safe to send.

## `async` does not mean background

An async function may suspend. That does not prove that its synchronous work runs away from the caller’s actor.

Risky:

```swift
@MainActor
final class DashboardModel {
    private(set) var sections: [DashboardSection] = []

    func refresh() async throws {
        let payload = try await api.dashboard()
        sections = DashboardBuilder.makeSections(from: payload)
    }
}
```

The network call suspends, but the section-building work after the `await` still happens inside a main-actor-isolated method.

A better structure is to keep UI mutation small:

```swift
@MainActor
final class DashboardModel {
    private(set) var sections: [DashboardSection] = []

    func refresh() async throws {
        let payload = try await api.dashboard()
        sections = try await makeDashboardSections(from: payload.snapshot)
    }
}

@concurrent
func makeDashboardSections(from snapshot: DashboardSnapshot) async throws -> [DashboardSection] {
    try Task.checkCancellation()
    return DashboardBuilder.makeSections(from: snapshot)
}
```

Use this pattern only when the builder does meaningful work and the values crossing the boundary are safe.

## `@concurrent`

Use `@concurrent` when an async function should explicitly leave the caller’s actor.

This is useful when:

- the caller is `@MainActor`;
- the callee performs meaningful CPU work;
- the caller’s actor should remain responsive;
- the function does not need actor-isolated state;
- inputs and outputs can safely cross the isolation boundary.

Good use:

```swift
@MainActor
final class ReportModel {
    private(set) var report: Report?

    func reload() async throws {
        let data = try await reportService.loadData()
        report = try await compileReport(from: data.snapshot)
    }
}

@concurrent
func compileReport(from snapshot: ReportSnapshot) async throws -> Report {
    try Task.checkCancellation()
    return ReportCompiler.compile(snapshot)
}
```

### `@concurrent` is not a magic optimizer

`@concurrent` can move execution away from the caller’s actor. It does not make CPU-heavy work automatically parallel, cancellable, or cheap.

Risky:

```swift
@concurrent
func buildHugeIndex(from records: [Record]) async -> SearchIndex {
    SearchIndexBuilder.build(from: records)
}
```

This may avoid blocking `MainActor`, but the work can still monopolize a cooperative worker thread for a long time.

For long-running work, consider chunking and cancellation:

```swift
@concurrent
func buildHugeIndex(from records: [Record]) async throws -> SearchIndex {
    var builder = SearchIndexBuilder()

    for batch in records.chunked(into: 500) {
        try Task.checkCancellation()
        builder.add(batch)
        await Task.yield()
    }

    return builder.build()
}
```

Use `Task.yield()` selectively. It is a responsiveness tool for long cooperative loops, not a substitute for good algorithms or measurement.

## `nonisolated`

Use `nonisolated` when a member does not need actor-isolated state.

Good use:

```swift
actor SymbolStore {
    private var symbols: [String: Symbol] = [:]

    nonisolated func canonicalSymbol(_ raw: String) -> String {
        raw.trimmingCharacters(in: .whitespacesAndNewlines).uppercased()
    }
}
```

The method is pure and does not read or mutate `symbols`.

Do not use `nonisolated` to bypass actor isolation for convenience. If the code needs actor state, it should stay isolated or receive a safe snapshot.

Risky:

```swift
actor ImageCache {
    private var storage: [URL: Image] = [:]

    nonisolated func cachedImage(for url: URL) -> Image? {
        storage[url]
    }
}
```

This is not a valid escape from actor isolation. The method tries to access actor-isolated mutable state.

## `nonisolated` vs `@concurrent`

They solve different problems.

Use `nonisolated` when:

- the member does not need actor state;
- the operation is pure, cheap, or based only on inputs;
- the goal is to avoid unnecessary actor isolation for a member.

Use `@concurrent` when:

- the function is async;
- it should explicitly leave the caller’s actor;
- the operation does meaningful work;
- moving the work away improves responsiveness or avoids actor contention;
- the boundary is Sendable-safe.

Do not replace one with the other mechanically.

### Decision table


## Sendable boundary review

When recommending `@concurrent`, check every value crossing the boundary:

- parameters;
- return values;
- captured values;
- stored dependencies;
- closures;
- reference types;
- mutable shared state;
- objects owned by UI state.

Risky:

```swift
@MainActor
final class ExportModel {
    private let draft: MutableExportDraft

    func preview() async -> ExportPreview {
        await makePreview(from: draft)
    }
}

@concurrent
func makePreview(from draft: MutableExportDraft) async -> ExportPreview {
    ExportPreview(draft: draft)
}
```

A mutable UI-owned reference object should usually stay on `MainActor`.

Prefer a Sendable snapshot:

```swift
struct ExportDraftSnapshot: Sendable {
    let title: String
    let pages: [ExportPage]
    let options: ExportOptions
}

@MainActor
final class ExportModel {
    private let draft: MutableExportDraft

    func preview() async -> ExportPreview {
        let snapshot = draft.snapshot()
        return await makePreview(from: snapshot)
    }
}

@concurrent
func makePreview(from snapshot: ExportDraftSnapshot) async -> ExportPreview {
    ExportPreview(snapshot: snapshot)
}
```

The UI model remains isolated. The background work receives immutable data.

## MainActor refactoring pattern

The safest pattern is usually:

1. 1keep UI state and UI mutation on `MainActor`;
2. 2extract immutable input data;
3. 3perform heavy work in a separate async function that intentionally leaves the caller’s actor;
4. 4return a Sendable result;
5. 5assign the final result back on `MainActor`.

Before:

```swift
@MainActor
final class InsightsModel {
    private(set) var insights: [InsightRow] = []

    func refresh() async throws {
        let events = try await eventService.loadEvents()
        insights = InsightEngine.computeRows(from: events)
    }
}
```

After:

```swift
@MainActor
final class InsightsModel {
    private(set) var insights: [InsightRow] = []

    func refresh() async throws {
        let events = try await eventService.loadEvents()
        insights = try await computeInsightRows(from: EventSnapshot(events))
    }
}

@concurrent
func computeInsightRows(from snapshot: EventSnapshot) async throws -> [InsightRow] {
    try Task.checkCancellation()
    return InsightEngine.computeRows(from: snapshot)
}
```

For larger workloads, add chunking and cancellation checks.

## Migration-related performance regressions

Swift 6.2 isolation changes can make code safer and easier to migrate, but they can also move work onto `MainActor` more often than expected.

Watch for regressions after:

- enabling default `MainActor` isolation for a target;
- moving code into an app target that has default `MainActor` isolation;
- converting view models or services to `@MainActor`;
- fixing Sendable diagnostics by adding broad actor isolation;
- replacing compiler errors with `Task {}` or `Task.detached`;
- adding `@concurrent` broadly without checking Sendable boundaries.

Common regression pattern:

```swift
// Before migration this helper lived in a nonisolated module.
// After migration it lives in a target with default MainActor isolation.

func normalizeTimeline(_ payload: TimelinePayload) -> [TimelineRow] {
    TimelineNormalizer.normalize(payload)
}
```

The helper may now run as part of main-actor-isolated code. If it is heavy, move it to a nonisolated service module or an explicit `@concurrent` async boundary with Sendable inputs.

Prefer this review question:

> Did the migration change where this code executes, or only how the compiler describes it?

## Common mistakes

- Treating `async` as proof that work is not on the main actor.
- Adding `@concurrent` everywhere after seeing one main-actor stall.
- Using `Task.detached` instead of understanding isolation.
- Moving mutable UI reference objects across isolation boundaries.
- Marking a whole type `@MainActor` because one property updates UI.
- Keeping data processing methods inside UI-facing `@MainActor` models.
- Using `nonisolated` to escape actor isolation while still needing actor state.
- Ignoring default actor isolation settings during review.
- Treating Sendable diagnostics as noise instead of a boundary design signal.
- Calling `Task.yield()` a performance fix without measuring.

## Diagnostics

Use Instruments or signposts when isolation behavior is uncertain.

Look for:

- UI freezes while an async method is running;
- long main-actor sections after an `await`;
- CPU-heavy work on the main actor;
- actor queue buildup around one UI-facing type;
- Sendable diagnostics after adding `@concurrent`;
- performance regressions after enabling Swift 6.2 settings;
- code that appears async but does not improve responsiveness.

Likely causes:

- heavy work remains actor-isolated;
- an async function does not leave the caller’s actor;
- default actor isolation made a helper `@MainActor`;
- non-Sendable reference state cannot cross the boundary;
- work is serialized through a UI-facing actor;
- the real bottleneck is another actor or service.

Validation options:

- add signposts around the heavy computation and final UI assignment;
- record a trace and inspect main-thread or main-actor activity;
- compare before/after interaction latency;
- log current task/request identifiers around duplicate work;
- add cancellation tests for long-running computation;
- check memory if snapshots copy large data.

## Review checklist

Before recommending a Swift 6.2 isolation change, check:

- [ ] Is the caller isolated to `MainActor` or another actor?
- [ ] Is default actor isolation enabled for this target?
- [ ] Is the callee explicitly isolated, implicitly isolated, `nonisolated`, or `@concurrent`?
- [ ] Is the slow work CPU-heavy, blocking, or mostly awaiting another async API?
- [ ] Would `@concurrent` actually move meaningful work off the caller’s actor?
- [ ] Are parameters, captures, dependencies, and return values safe to send?
- [ ] Can mutable UI state be converted to a Sendable snapshot?
- [ ] Would `nonisolated` be more appropriate than `@concurrent`?
- [ ] Does the heavy work need cancellation checks?
- [ ] Does long CPU work need chunking or yielding?
- [ ] Is broad `@MainActor` isolation hiding processing work?
- [ ] Is the recommendation validated with a trace, signpost, test, or reproducible UI behavior?

## Source notes

This reference is based on Swift 6.2-era concurrency behavior and should be checked against current Swift documentation when language rules change.

Useful primary sources:

- Swift Evolution SE-0461: Run nonisolated async functions on the caller's actor by default.
- Swift Evolution SE-0466: Control default actor isolation inference.
- Swift migration guidance for data-race safety and concurrency adoption.
