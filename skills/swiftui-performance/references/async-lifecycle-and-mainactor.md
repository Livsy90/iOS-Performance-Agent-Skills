# Async Lifecycle and MainActor

Use this reference when reviewing SwiftUI code that starts or coordinates async work from `.task`, `.task(id:)`, `.refreshable`, `.onAppear`, row lifecycle callbacks, explicit user actions, or UI-facing view models.

Use it for SwiftUI-specific lifecycle risks: duplicate loading, unstable task restarts, stale result commits, cancellation before UI updates, row-driven pagination triggers, and heavy work accidentally running on the main actor.

Do not use this file as the primary guide for general actor design, task groups, continuations, `AsyncSequence` producer cleanup, executor internals, or Swift runtime costs. Route those topics to the dedicated Swift Concurrency references. This file is about how async work interacts with SwiftUI view lifetime and UI responsiveness.

## Contents

- [Core model](#core-model)
- [Review workflow](#review-workflow)
- [Choose the right lifecycle trigger](#choose-the-right-lifecycle-trigger)
- [Never start async work from `body`](#never-start-async-work-from-body)
- [Use `.task(id:)` for semantic restarts](#use-taskid-for-semantic-restarts)
- [Use `.task` without `id` for one lifecycle run](#use-task-without-id-for-one-lifecycle-run)
- [Use `.onAppear` narrowly](#use-onappear-narrowly)
- [Row lifecycle and pagination](#row-lifecycle-and-pagination)
- [Cancellation and stale commits](#cancellation-and-stale-commits)
- [MainActor work](#mainactor-work)
- [Batch UI state updates](#batch-ui-state-updates)
- [Search, filters, and debounce](#search-filters-and-debounce)
- [Long-lived streams](#long-lived-streams)
- [Validation](#validation)
- [Review checklist](#review-checklist)
- [Common mistakes](#common-mistakes)
- [Agent response guidance](#agent-response-guidance)

## Core model

Async SwiftUI code should have:

- a clear lifecycle boundary;
- a stable semantic trigger;
- idempotent loading behavior;
- cooperative cancellation;
- stale-result protection;
- minimal heavy work on the main actor;
- compact final UI state mutation.

Do not call the issue generic SwiftUI slowness. Explain which task starts, what causes it to restart, whether duplicate work can happen, whether an old result can overwrite a newer one, and whether the main actor is doing too much work.

## Review workflow

1. Identify the user-visible symptom: delayed loading, duplicate calls, stale content, typing lag, scrolling hitch, repeated pagination, or delayed interaction.
2. Find the trigger that starts work: `.task`, `.task(id:)`, `.refreshable`, `.onAppear`, row callback, button action, timer, stream, or model method.
3. Check whether the trigger is stable and semantic.
4. Check whether the operation is idempotent.
5. Check cancellation before committing visible state.
6. Check whether older tasks can commit after newer tasks.
7. Separate UI state mutation from CPU-heavy parsing, filtering, sorting, formatting, or render-model generation.
8. Recommend the smallest lifecycle or isolation refactor.
9. Provide a validation path when performance or duplicate work is only suspected.

## Choose the right lifecycle trigger

Prefer triggers that match the lifetime of the work:

- `.task(id:)` — work belongs to the view and restarts for a semantic input.
- `.task` — work runs once for the current view identity.
- `.refreshable` — user-initiated pull-to-refresh.
- Button/action closures — explicit user-initiated operations.
- `.onAppear` — appearance side effects, not default async loading.
- Model-owned task — work intentionally outlives one view appearance.

A trigger should not restart because of incidental rendering, random IDs, timestamps, or values that the task itself mutates.

## Never start async work from `body`

`body` can be evaluated many times. Starting async work during body evaluation turns rendering into a side-effect source and can create duplicate tasks.

Risky:

```swift
var body: some View {
    let _ = Task { await model.refresh() }
    AccountSummaryContent(state: model.state)
}
```

Prefer a lifecycle modifier or explicit action:

```swift
var body: some View {
    AccountSummaryContent(state: model.state)
        .task(id: accountID) {
            await model.load(accountID: accountID)
        }
}
```

## Use `.task(id:)` for semantic restarts

Use `.task(id:)` when work should cancel and restart for a meaningful input:

- entity ID;
- selected tab;
- search query;
- filter or sort option;
- currently selected account, chat, document, or route.

Good:

```swift
.task(id: transactionID) {
    await model.load(transactionID: transactionID)
}
```

Risky:

```swift
.task(id: UUID()) { await model.reload() }
```

Risky:

```swift
.task(id: model.lastRefreshDate) {
    await model.refresh()
}
```

If `refresh()` updates `lastRefreshDate`, the task can restart for its own side effect. Prefer a stable semantic input or an explicit user action.

## Use `.task` without `id` for one lifecycle run

Use `.task` without `id` when work should run for the current view identity and should not restart for changing inputs.

```swift
.task {
    await model.loadInitialContentIfNeeded()
}
```

The model method should still be idempotent:

```swift
@MainActor
func loadInitialContentIfNeeded() async {
    guard !didLoad else { return }
    didLoad = true

    state = .loading
    do {
        let content = try await service.loadWelcomeContent()
        try Task.checkCancellation()
        state = .loaded(content)
    } catch is CancellationError {
        // Usually do not commit visible state from a canceled task.
        // A newer task or view disappearance may have replaced this lifecycle run.
    } catch {
        state = .failed(error)
    }
}
```

Idempotency matters because SwiftUI identity can change during navigation, conditional rendering, parent restructuring, or explicit `.id` boundaries.

If cancellation should reset visible state, guard that reset with the same semantic input or request token used by the current operation. Otherwise, an older canceled task can overwrite newer visible state.

## Use `.onAppear` narrowly

Use `.onAppear` for appearance-related side effects:

- analytics exposure events;
- starting a local animation flag;
- focusing a field after appearance;
- notifying a parent that a child became visible;
- lightweight UI-only effects.

Do not use `.onAppear { Task { ... } }` as the default loading pattern when `.task` expresses lifecycle and cancellation better.

Risky:

```swift
.onAppear {
    Task { await model.load() }
}
```

Prefer:

```swift
.task {
    await model.loadIfNeeded()
}
```

If `.onAppear` must own async work, store the `Task` handle and cancel it in `.onDisappear`. Prefer `.task` when manual ownership is not required.

## Row lifecycle and pagination

Rows in `List`, `LazyVStack`, and lazy containers can appear multiple times during scrolling, navigation, refresh, filtering, and identity changes. A row `.onAppear` is not a reliable “first time visible” event.

Risky:

```swift
.onAppear {
    if row.id == model.rows.last?.id {
        Task { await model.loadNextPage() }
    }
}
```

Problems:

- row appearance can repeat;
- the task is unstructured;
- duplicate page loads are possible;
- slow older requests can race with newer requests;
- layout changes can re-trigger the last row.

Prefer a stable prefetch boundary and an idempotent model method:

```swift
.task(id: row.id) {
    guard model.shouldPrefetch(after: row.id) else { return }
    await model.loadNextPageIfNeeded(trigger: row.id)
}
```

The model should guard:

- already loading;
- `nextPage != nil` or `hasMore`;
- already requested page keys;
- refresh and pagination overlap;
- cancellation before committing rows;
- stale trigger IDs when needed.

For very large lists, prefer a footer or sentinel view so every row does not create its own lifecycle task. Use row-level `.task(id:)` for pagination only when the guard is cheap and the model fully handles duplicate attempts.

## Cancellation and stale commits

Cancellation is cooperative. It marks a task as canceled, but work may continue until it observes cancellation.

Check cancellation before committing visible state:

```swift
@MainActor
func search(query: String) async {
    state = .loading
    do {
        let results = try await service.search(query)
        try Task.checkCancellation()
        state = .loaded(results)
    } catch is CancellationError {
        // A newer query or view disappearance canceled this task.
    } catch {
        state = .failed(error)
    }
}
```

When an old task can finish after a newer one, also validate the active input before committing:

```swift
try Task.checkCancellation()
guard activeAccountID == accountID else { return }
state = .loaded(details)
```

Use stale-commit protection for search, fast filter changes, tab switching, navigation details, and pagination.

## MainActor work

A `@MainActor` view model is often a good fit for SwiftUI-facing state. The problem is not UI isolation itself; the problem is doing large synchronous work while isolated to the main actor.

Risky inside a `@MainActor` model during a user interaction:

```swift
rows = response.entries
    .sorted { $0.date > $1.date }
    .map(StatementRowModel.init)
```

Prefer preparing render-ready data away from main-actor UI mutation when the data is safe to transfer:

```swift
let response = try await service.loadStatement()
let preparedRows = try await rowBuilder.makeRows(from: response.entries)
try Task.checkCancellation()
rows = preparedRows
```

Keep UI state mutation on `MainActor`. Move pure, CPU-heavy parsing, sorting, filtering, formatting, and render-model generation out of the main-actor update path when it is safe and meaningful.

This only helps when the preparation method is not isolated to the main actor and does not call main-actor-isolated APIs. Extracting code into an `async` function or adding `await` does not automatically move CPU-heavy work off the main actor.

Do not assume `Task { ... }` is a background-thread escape hatch. A task created from a main-actor context may still execute main-actor-isolated work.

Use `Task.detached` sparingly. It creates an unstructured detached top-level task and requires explicit data-transfer boundaries; captured values must be safe to use outside the current actor context. It should not be the default fix for slow UI.

## Batch UI state updates

Repeated main-actor state updates can invalidate visible SwiftUI views repeatedly.

Risky:

```swift
for item in items {
    rows.append(await importer.importRow(item))
}
```

Prefer batching when the UI does not need per-item progress:

```swift
let importedRows = try await importer.importRows(items)
try Task.checkCancellation()
rows = importedRows
```

If progress is required, update a lightweight progress value, coalesce updates, or throttle updates instead of rebuilding the full visible collection on every item.

## Search, filters, and debounce

For async search, `.task(id:)` can express restart-on-query-change behavior.

```swift
.task(id: query) {
    let trimmed = query.trimmingCharacters(in: .whitespacesAndNewlines)
    guard !trimmed.isEmpty else {
        await model.clearResults()
        return
    }

    do {
        try await Task.sleep(for: .milliseconds(300))
        try Task.checkCancellation()
        await model.search(query: trimmed)
    } catch is CancellationError {
        // A newer query replaced this one.
    } catch {
        await model.handleSearchError(error)
    }
}
```

Keep debounce, cancellation, and final commit behavior explicit. Do not keep many independent search tasks alive unless manual ownership is required.

## Long-lived streams

When a view consumes an `AsyncSequence`, tie the consuming loop to a lifecycle boundary so it cancels when the view disappears.

```swift
.task {
    await model.observePrices()
}
```

For high-frequency streams, do not update full UI state for every event unless the UI needs that frequency. Coalesce, throttle, diff, or update only changed values.

Keep producer lifetime, buffering, and `onTermination` details in the dedicated `AsyncSequence` reference. In this file, focus on the SwiftUI consumer lifetime and UI update frequency.

## Validation

Use validation when the issue is not obvious from code or when claiming a performance improvement.

Good validation options:

- log task start, cancellation, completion, and commit events;
- count duplicate network calls for the same semantic input;
- add signposts around user action, request start, response arrival, render-model build, and final state commit;
- use Time Profiler to see whether heavy transformation work runs on the main thread during interaction;
- use the SwiftUI instrument to inspect repeated body updates after async commits;
- use Animation Hitches when the symptom is visible stutter;
- use XCTest performance tests for isolated row-building, filtering, sorting, or formatting pipelines;
- use MetricKit or production logs to detect repeated hangs or slow interactions.

Separate static review risks from measured results. Do not claim a numeric cost unless the user provided measurements or a trace/log/benchmark supports it.

## Review checklist

Check:

- Is work started from `body`?
- Is `.task(id:)` using a stable semantic ID?
- Does the task mutate the same value used as its ID?
- Is `.task` without `id` guarded by an idempotent model method?
- Is `.onAppear` used only where repeated appearance is acceptable or guarded?
- Can row lifecycle callbacks start duplicate work?
- Are pagination loads guarded by loading state, next-page state, and requested-page tracking?
- Can refresh and pagination commit conflicting state?
- Is cancellation handled separately from real errors?
- Is cancellation checked before visible state commits?
- Can an older result overwrite a newer result?
- Is heavy transformation work accidentally running on the main actor?
- Are UI state mutations compact and batched?
- Is `Task {}` being used as a fake background queue?
- Is `Task.detached` avoided unless isolation and data-transfer boundaries are explicit?
- Are high-frequency streams coalesced before updating UI?

## Common mistakes

- Starting `Task {}` during `body` evaluation.
- Using `.task(id: UUID())` or another unstable ID.
- Using a timestamp or `lastUpdatedAt` as a task ID when the task mutates it.
- Treating `.onAppear` as a reliable one-time event.
- Starting pagination from row appearance without duplicate-load guards.
- Catching cancellation as a normal user-facing error.
- Committing results after cancellation or after the active input changed.
- Doing sorting, filtering, parsing, or render-model building on a `@MainActor` hot path.
- Updating a visible collection once per item when one batched update would work.
- Assuming `Task {}` automatically moves work off the main actor.
- Using `Task.detached` without a clear reason and safe data-transfer boundary.

## Agent response guidance

When reviewing async SwiftUI code, respond with:

1. The lifecycle trigger that starts the work.
2. Whether the trigger is stable.
3. Whether duplicate work is possible.
4. Whether cancellation and stale commits are handled.
5. Whether heavy work runs on the main actor.
6. The smallest refactor that fixes the issue.
7. A validation step when confirmation is needed.

Prefer precise language:

```md
This `.onAppear` can run repeatedly as rows enter and leave the lazy list. Move the trigger to `.task(id:)` or a sentinel view, and make the model method idempotent with `isLoadingNextPage`, `nextPage`, requested-page tracking, and cancellation checks before committing rows.
```

Avoid unsupported claims such as “this definitely causes a 300 ms hitch” unless a trace, signpost, benchmark, or user-provided measurement proves that number.

## Final rule

Async lifecycle performance in SwiftUI is mostly about preventing accidental work: duplicate tasks, unstable restarts, stale commits, excessive main-actor transformation, and too many UI state updates. Make the trigger stable, make the operation idempotent, make cancellation explicit, and keep the final UI mutation small.
