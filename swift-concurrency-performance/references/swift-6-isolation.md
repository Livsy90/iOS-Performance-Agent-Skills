# Swift 6 Isolation and `@concurrent`

Use this reference when the task involves Swift 6-era isolation behavior, Swift 6.2 default actor isolation, `@concurrent`, `nonisolated`, `nonisolated(nonsending)`, Sendable boundaries, `MainActor` responsiveness, or migration-related performance regressions.

This reference is for performance review. It helps the agent decide where async work actually runs, whether it accidentally remains on `MainActor`, and whether moving work across an isolation boundary is safe.

This is not a general Swift Concurrency guide. For cancellation, actor reentrancy, bounded task groups, blocking legacy APIs, or continuation safety, use the more specific references.

## Contents

* [Core model](#core-model)
* [Review workflow](#review-workflow)
* [Default actor isolation](#default-actor-isolation)
* [`async` does not mean background](#async-does-not-mean-background)
* [`@concurrent`](#concurrent)
* [`nonisolated`](#nonisolated)
* [`nonisolated(nonsending)`](#nonisolatednonsending)
* [`nonisolated` vs `nonisolated(nonsending)` vs `@concurrent`](#nonisolated-vs-nonisolatednonsending-vs-concurrent)
* [Sendable boundary review](#sendable-boundary-review)
* [MainActor refactoring pattern](#mainactor-refactoring-pattern)
* [Migration-related performance regressions](#migration-related-performance-regressions)
* [Common mistakes](#common-mistakes)
* [Diagnostics](#diagnostics)
* [Review checklist](#review-checklist)
* [Source notes](#source-notes)

## Core model

Do not assume that `async` means “background”.

In Swift 6-era code, the execution context of an async function depends on actor isolation, language mode, compiler settings, upcoming feature flags, module settings, and annotations.

A function can be async and still run on the caller’s actor.

For performance review, always ask:

* What actor is the caller isolated to?
* Is the enclosing type explicitly or implicitly `@MainActor`?
* Is default actor isolation enabled for this target or module?
* Is the function actor-isolated, `nonisolated`, `nonisolated(nonsending)`, or `@concurrent`?
* Does the function only suspend, or does it perform meaningful CPU work?
* Does the function run synchronous CPU work before its first suspension?
* Are parameters, captures, dependencies, and return values safe to send across an isolation boundary?
* Is the suspected performance issue caused by where the work runs, by the amount of work, by blocking, or by unbounded parallelism?

Performance model:

```text id="xenk1p"
async != background
await != off-main execution
nonisolated synchronous helper != thread hop
nonisolated(nonsending) async function != actor escape
@concurrent async function == explicitly switch off actor to run
Sendable diagnostics == boundary design signal
```

## Review workflow

1. Identify the user-visible symptom: UI stall, slow interaction, actor queue buildup, new Sendable diagnostics, migration warning, or regression after enabling Swift 6 settings.
2. Locate the caller’s isolation domain.
3. Locate the callee’s isolation behavior.
4. Check whether heavy work is running on `MainActor` or another hot actor.
5. Check whether the slow work is CPU-heavy, blocking, waiting on another actor, or mostly awaiting an already asynchronous API.
6. Check whether moving the work away would cross a Sendable boundary.
7. Prefer immutable snapshots over moving mutable UI-owned reference objects.
8. Use `@concurrent` only when leaving the caller’s actor is intentional and useful.
9. Use `nonisolated` only when the member does not need isolated state.
10. Use `nonisolated(nonsending)` when an async function should preserve caller isolation rather than switch away.
11. Validate the change with traces, signposts, UI responsiveness, compiler diagnostics, or targeted tests.

## Default actor isolation

Swift 6.2 adds project and package settings that can make declarations in a module infer `@MainActor` isolation by default.

This can improve approachability for UI-heavy apps, but it changes the performance review question:

```text id="n92q1o"
Is this code explicitly main-actor isolated, or did it become main-actor isolated because of the target’s default isolation setting?
```

When default `MainActor` isolation is enabled, unannotated declarations in that module may become main-actor isolated unless another rule applies.

If no default isolation setting is specified, the module default remains `nonisolated`.

Default isolation is module-scoped. A UI app target can default to `MainActor`, while a separate package or framework target can remain `nonisolated`. Code imported from another module is not rewritten by the current module’s default isolation setting.

Review these cases carefully:

* view models that combine UI state and data processing;
* services placed in app targets rather than framework targets;
* helper functions near UI code;
* static utilities in UI modules;
* protocol conformances that inherit isolation;
* nested types and helpers inside main-actor-isolated types;
* code that behaved differently after enabling Swift 6.2 settings.

Prefer checking the build setting instead of guessing from the source file alone.

### App-target risk

Risky pattern:

```swift id="z2onqb"
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

```swift id="wuvcf6"
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

Before recommending this, verify that:

* the project supports `@concurrent`;
* the snapshot and result are safe to send;
* the builder does meaningful CPU work;
* the refactor reduces a measured main-actor interval;
* cancellation and stale-result behavior remain correct.

If the helper is cheap, extracting it may improve clarity but may not produce a meaningful responsiveness win.

## `async` does not mean background

An async function may suspend. That does not prove that its synchronous work runs away from the caller’s actor.

Risky:

```swift id="0wg9hm"
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

A better structure is to keep UI mutation small and move heavy pure work behind an explicit boundary:

```swift id="xsax87"
@MainActor
final class DashboardModel {
    private(set) var sections: [DashboardSection] = []

    func refresh() async throws {
        let payload = try await api.dashboard()
        let builtSections = try await makeDashboardSections(from: payload.snapshot)
        sections = builtSections
    }
}

@concurrent
func makeDashboardSections(from snapshot: DashboardSnapshot) async throws -> [DashboardSection] {
    try Task.checkCancellation()
    return DashboardBuilder.makeSections(from: snapshot)
}
```

Use this pattern only when:

* the builder does meaningful work;
* the values crossing the boundary are safe to send;
* the target supports the required language feature;
* the final UI mutation remains on `MainActor`;
* the change improves a measured responsiveness signal.

Do not turn every helper into `async`. The goal is not to add `await`; the goal is to control where meaningful work runs.

## `@concurrent`

Use `@concurrent` when an async function should explicitly switch off an actor to run.

This is useful when:

* the caller is `@MainActor` or another hot actor;
* the callee performs meaningful CPU work;
* the caller’s actor should remain responsive;
* the function does not need actor-isolated state;
* inputs and outputs can safely cross the isolation boundary;
* leaving the actor improves a measured user-visible or throughput signal.

Good use:

```swift id="2nq73s"
@MainActor
final class ReportModel {
    private(set) var report: Report?

    func reload() async throws {
        let data = try await reportService.loadData()
        let compiled = try await compileReport(from: data.snapshot)
        report = compiled
    }
}

@concurrent
func compileReport(from snapshot: ReportSnapshot) async throws -> Report {
    try Task.checkCancellation()
    return ReportCompiler.compile(snapshot)
}
```

`@concurrent` implies `nonisolated`. It cannot be combined with global actor isolation such as `@MainActor`, isolated parameters, or other actor-isolation attributes. It applies to async functions, not synchronous functions.

Do not use `@concurrent` to access actor state from outside the actor. A `@concurrent` method on an actor-isolated type cannot access isolated mutable state unless it explicitly receives a safe snapshot or uses another valid isolation mechanism.

### `@concurrent` is not a magic optimizer

`@concurrent` can move execution away from the caller’s actor. It does not make CPU-heavy work automatically parallel, cancellable, cheap, or memory efficient.

Risky:

```swift id="p72gm9"
@concurrent
func buildHugeIndex(from records: [Record]) async -> SearchIndex {
    SearchIndexBuilder.build(from: records)
}
```

This may avoid blocking `MainActor`, but the work can still monopolize a cooperative worker thread for a long time.

For long-running cooperative CPU work, consider chunking and cancellation:

```swift id="07xm9y"
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

Use `Task.yield()` selectively. It does not make CPU work cheaper and does not move work to a different executor. Too frequent yielding can reduce throughput. Use it only when a long cooperative loop needs to give other work a chance to run, and validate the result.

For blocking legacy work, use the blocking legacy APIs reference. `@concurrent` alone is not a safe blocking adapter.

## `nonisolated`

Use `nonisolated` when a member does not need actor-isolated state.

Good use:

```swift id="rdhptz"
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

```swift id="6jzn66"
actor ImageCache {
    private var storage: [URL: Image] = [:]

    nonisolated func cachedImage(for url: URL) -> Image? {
        storage[url]
    }
}
```

This is not a valid escape from actor isolation. The method tries to access actor-isolated mutable state.

Important performance caveat:

```text id="d6cs0t"
nonisolated on a synchronous method removes actor isolation, but it does not create a background hop.
```

If a synchronous `nonisolated` helper is called from the main actor, the helper’s synchronous work still runs in that caller’s execution. Use this pattern to clarify isolation and prevent accidental actor-state access. Do not present it as a guaranteed responsiveness fix by itself.

## `nonisolated(nonsending)`

Use `nonisolated(nonsending)` for async functions that should be nonisolated but should run on the caller’s actor by default.

This is useful when an async API should preserve caller isolation rather than switch away from it.

Example:

```swift id="qonmjc"
final class FormatterService {
    nonisolated(nonsending)
    func label(for value: DisplayValue) async -> String {
        value.title.trimmingCharacters(in: .whitespacesAndNewlines)
    }
}
```

When called from `@MainActor`, this kind of function can continue on the caller’s actor instead of sending arguments and results across an isolation boundary.

This improves usability for APIs that operate on caller-owned, non-Sendable, or UI-adjacent values. It is not a performance fix for heavy CPU work. If the function performs meaningful CPU work and the caller is `MainActor`, preserving caller isolation can still hurt responsiveness.

Use `nonisolated(nonsending)` when:

* preserving caller isolation is intentional;
* values should not be sent across an isolation boundary;
* the function is cheap, coordination-oriented, or caller-context-sensitive;
* the API should avoid unnecessary Sendable diagnostics;
* responsiveness is not harmed by running in the caller’s context.

Do not use it when the goal is to leave `MainActor` for heavy CPU work. Use an explicit worker boundary or `@concurrent` when appropriate and safe.

## `nonisolated` vs `nonisolated(nonsending)` vs `@concurrent`

They solve different problems.

Use `nonisolated` when:

* the member does not need actor state;
* the operation is synchronous, pure, cheap, or based only on inputs;
* the goal is to avoid unnecessary actor isolation for a member;
* you are clarifying isolation, not trying to hop to another executor.

Use `nonisolated(nonsending)` when:

* the function is async;
* it should preserve caller actor isolation;
* values should not be sent across an isolation boundary by default;
* avoiding unnecessary Sendable boundary crossing is the goal;
* the work is cheap or caller-context-sensitive.

Use `@concurrent` when:

* the function is async;
* it should explicitly leave the caller’s actor;
* the operation does meaningful work;
* moving the work away improves responsiveness or avoids actor contention;
* the boundary is Sendable-safe;
* the toolchain and language mode support the feature.

Do not replace one with another mechanically.

### Decision table

| Need                                                     | Prefer                                                                              | Caveat                                                 |
| -------------------------------------------------------- | ----------------------------------------------------------------------------------- | ------------------------------------------------------ |
| UI state mutation                                        | `@MainActor`                                                                        | Keep the isolated section short.                       |
| Pure synchronous helper that does not access actor state | `nonisolated`                                                                       | Does not move work to another thread.                  |
| Async helper should preserve caller actor isolation      | `nonisolated(nonsending)` or the enabled default behavior                           | Not suitable for heavy CPU work on `MainActor`.        |
| Async helper should explicitly leave caller actor        | `@concurrent`                                                                       | Requires safe boundary and feature support.            |
| CPU-heavy work with large input                          | Worker/service boundary, `@concurrent` when safe, bounded task groups when parallel | Validate memory, cancellation, and responsiveness.     |
| Mutable UI-owned object needs processing                 | Create an immutable snapshot first                                                  | Do not send mutable UI state across isolation domains. |
| Legacy blocking operation                                | Blocking legacy API adapter                                                         | `@concurrent` alone is not enough.                     |
| Many independent items                                   | Bounded task group                                                                  | Priority or `@concurrent` is not a concurrency limit.  |

## Sendable boundary review

When recommending `@concurrent`, check every value crossing the boundary:

* parameters;
* return values;
* captured values;
* stored dependencies;
* closures;
* reference types;
* mutable shared state;
* objects owned by UI state;
* values captured from `self`.

Risky:

```swift id="809bxe"
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

```swift id="13nfm2"
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

The UI model remains isolated. The concurrent work receives immutable data.

Do not silence Sendable diagnostics mechanically. They often indicate that the design is trying to send mutable state across a boundary.

## MainActor refactoring pattern

The safest pattern is usually:

1. Keep UI state and UI mutation on `MainActor`.
2. Extract immutable input data.
3. Perform heavy work in a separate async function that intentionally leaves the caller’s actor when that is safe and useful.
4. Return a safe result.
5. Assign the final result back on `MainActor`.
6. Validate that the main-actor interval or user-visible latency improves.

Before:

```swift id="t52dvi"
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

```swift id="1skftu"
@MainActor
final class InsightsModel {
    private(set) var insights: [InsightRow] = []

    func refresh() async throws {
        let events = try await eventService.loadEvents()
        let rows = try await computeInsightRows(from: EventSnapshot(events))
        insights = rows
    }
}

@concurrent
func computeInsightRows(from snapshot: EventSnapshot) async throws -> [InsightRow] {
    try Task.checkCancellation()
    return InsightEngine.computeRows(from: snapshot)
}
```

For larger workloads, add chunking, cancellation checks, bounded parallelism, or a domain-specific worker service.

Do not assume this refactor is correct until:

* `EventSnapshot` and `[InsightRow]` are safe to cross the boundary;
* cancellation is preserved;
* stale result behavior is preserved;
* the heavy work is large enough to justify the boundary;
* before/after validation shows the desired signal.

## Migration-related performance regressions

Swift 6-era isolation changes can make code safer and easier to migrate, but they can also move work onto `MainActor` more often than expected.

Watch for regressions after:

* enabling default `MainActor` isolation for a target;
* moving code into an app target that has default `MainActor` isolation;
* converting view models or services to `@MainActor`;
* fixing Sendable diagnostics by adding broad actor isolation;
* replacing compiler errors with `Task {}` or `Task.detached`;
* adding `@concurrent` broadly without checking Sendable boundaries;
* changing language mode or upcoming feature flags around nonisolated async behavior.

Common regression pattern:

```swift id="x1oapr"
// Before migration this helper lived in a nonisolated module.
// After migration it lives in a target with default MainActor isolation.

func normalizeTimeline(_ payload: TimelinePayload) -> [TimelineRow] {
    TimelineNormalizer.normalize(payload)
}
```

The helper may now run as part of main-actor-isolated code. If it is heavy, move it to a nonisolated service module or an explicit async boundary with safe inputs.

Prefer this review question:

```text id="wk4o6m"
Did the migration change where this code executes, or only how the compiler describes it?
```

Also check the opposite problem: adding `@concurrent` may restore responsiveness but introduce new Sendable errors or unsafe state transfer. That is not a compiler nuisance; it is a boundary design issue.

## Common mistakes

* Treating `async` as proof that work is not on the main actor.
* Treating `await` as proof of background execution.
* Saying “Swift 6 makes everything `MainActor` by default” instead of checking the module setting.
* Forgetting that default actor isolation is module-scoped.
* Forgetting that imported declarations are not rewritten by the current module’s default isolation setting.
* Adding `@concurrent` everywhere after seeing one main-actor stall.
* Using `Task.detached` instead of understanding isolation.
* Moving mutable UI reference objects across isolation boundaries.
* Marking a whole type `@MainActor` because one property updates UI.
* Keeping data processing methods inside UI-facing `@MainActor` models.
* Using `nonisolated` to escape actor isolation while still needing actor state.
* Using `nonisolated` on a synchronous helper and assuming it moved CPU work off the caller.
* Using `nonisolated(nonsending)` for heavy CPU work that should leave `MainActor`.
* Ignoring default actor isolation settings during review.
* Treating Sendable diagnostics as noise instead of a boundary design signal.
* Calling `Task.yield()` a performance fix without measuring.
* Using `@concurrent` for blocking legacy work without a proper blocking adapter.

## Diagnostics

Use Instruments, signposts, compiler diagnostics, and targeted tests when isolation behavior is uncertain.

Look for:

* UI freezes while an async method is running;
* long main-actor sections after an `await`;
* CPU-heavy work on the main actor;
* actor queue buildup around one UI-facing type;
* Sendable diagnostics after adding `@concurrent`;
* performance regressions after enabling Swift 6.2 settings;
* code that appears async but does not improve responsiveness;
* helper code that moved into a default-main-actor app target;
* synchronous work before the first suspension.

Likely causes:

* heavy work remains actor-isolated;
* an async function preserves caller isolation;
* default actor isolation made a helper `@MainActor`;
* non-Sendable reference state cannot cross the boundary;
* work is serialized through a UI-facing actor;
* `nonisolated` clarified isolation but did not move work;
* the real bottleneck is another actor, service, blocking call, or unbounded fan-out.

Validation options:

* add signposts around the heavy computation and final UI assignment;
* record a trace and inspect main-thread or main-actor activity;
* compare before/after interaction latency;
* inspect compiler diagnostics before and after isolation changes;
* log current task/request identifiers around duplicate work;
* add cancellation tests for long-running computation;
* check memory if snapshots copy large data;
* validate on realistic data and release-like builds.

## Review checklist

Before recommending a Swift 6 isolation change, check:

* [ ] Is the caller isolated to `MainActor` or another actor?
* [ ] Is default actor isolation enabled for this target?
* [ ] Is the callee explicitly isolated, implicitly isolated, `nonisolated`, `nonisolated(nonsending)`, or `@concurrent`?
* [ ] Is the slow work CPU-heavy, blocking, or mostly awaiting another async API?
* [ ] Would `@concurrent` actually move meaningful work off the caller’s actor?
* [ ] Would preserving caller isolation with `nonisolated(nonsending)` be more correct?
* [ ] Would `nonisolated` only clarify a synchronous helper rather than move work?
* [ ] Are parameters, captures, dependencies, and return values safe to send?
* [ ] Can mutable UI state be converted to a safe snapshot?
* [ ] Does the heavy work need cancellation checks?
* [ ] Does long CPU work need chunking or yielding?
* [ ] Is broad `@MainActor` isolation hiding processing work?
* [ ] Is a default-main-actor target hiding the isolation annotation?
* [ ] Would moving code to a separate module change the default isolation model?
* [ ] Is the recommendation validated with a trace, signpost, test, compiler diagnostic, or reproducible UI behavior?

## Source notes

This reference is based on Swift 6.2-era concurrency behavior. Check current Swift Evolution proposals, Swift documentation, and the project’s actual language mode and compiler settings when language rules change.

Useful primary sources:

* SE-0461: Run nonisolated async functions on the caller's actor by default.
* SE-0466: Control default actor isolation inference.
* Swift migration guidance for data-race safety and concurrency adoption.

Key source facts used by this reference:

* SE-0461 is implemented in Swift 6.2.
* `nonisolated(nonsending)` async functions run on the caller’s actor by default.
* `@concurrent` async functions always switch off an actor to run.
* `@concurrent` implies `nonisolated`.
* `@concurrent` cannot be combined with global actor isolation, isolated parameters, or other actor-isolation attributes.
* `@concurrent` cannot be applied to synchronous functions.
* SE-0466 is implemented in Swift 6.2.
* Default actor isolation is module-scoped.
* If no default isolation setting is specified, the module default remains `nonisolated`.
* Imported declarations are unaffected by the current module’s default isolation choice.
