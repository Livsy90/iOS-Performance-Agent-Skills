# Body Cost and Render Models

Use this reference when a SwiftUI performance issue involves expensive work during rendering: sorting, filtering, grouping, formatting, mapping, parsing, image preparation, expensive computed properties, repeated render model construction, or user-visible delay caused by data preparation on the update path.

Do not use this reference as the primary source for Observation dependency scope, `ForEach` identity, pagination structure, closure-heavy rows, or layout/drawing cost. Use the dedicated references for those topics. This file is about the amount of work performed when a SwiftUI view updates.

## Contents

* [Core model](#core-model)
* [Red flags in `body`](#red-flags-in-body)
* [Body cost vs dependency scope](#body-cost-vs-dependency-scope)
* [Render-ready models](#render-ready-models)
* [Risky pattern](#risky-pattern)
* [Preferred pattern](#preferred-pattern)
* [Derived collections](#derived-collections)
* [Formatting and parsing](#formatting-and-parsing)
* [Expensive computed properties](#expensive-computed-properties)
* [MainActor preparation](#mainactor-preparation)
* [Validation](#validation)
* [Review checklist](#review-checklist)
* [Common mistakes](#common-mistakes)

## Core model

A SwiftUI `body` is a description of UI. It should not be a data preparation pipeline.

`body` can be evaluated frequently. The exact frequency depends on state changes, dependency tracking, identity, and reconciliation, but the safe review rule is simple: repeated rendering should mostly assemble already prepared values into views.

When reviewing body cost, ask:

* What work happens every time this view updates?
* Is that work proportional to collection size?
* Does it allocate temporary arrays, strings, formatters, images, or wrappers?
* Does it run on the main actor during a user interaction?
* Could the same result be prepared when the input changes instead of when the UI renders?

Do not claim a numeric cost unless a trace, benchmark, signpost, log, or user-provided measurement supports it.

## Red flags in `body`

Avoid doing these directly in `body`, row bodies, view initializers, or modifiers that run as part of repeated rendering:

* sorting, filtering, grouping, or chunking;
* date parsing or date formatting;
* currency, number, percentage, or relative time formatting;
* formatter creation;
* attributed string construction;
* image decoding, resizing, or downsampling;
* JSON, HTML, Markdown, URL, or regex parsing;
* database reads, synchronous file access, or network calls;
* large mapping pipelines;
* expensive computed properties;
* allocation-heavy temporary arrays or strings;
* business logic that is not UI composition.

A small transformation in a small static view is not automatically a bug. Treat it as a performance risk when it is repeated often, applied to many items, triggered during scrolling or typing, runs on the main actor, or appears in a measured hot path.

## Body cost vs dependency scope

Separate two different problems:

* A dependency-scope refactor reduces how often a view updates.
* A body-cost refactor reduces how much work happens when it updates.

Splitting a view helps body cost only when it removes repeated work from the updating part of the tree or puts expensive work behind a narrower dependency boundary.

Moving expensive work from `body` into a computed property does not solve the problem if `body` still reads that computed property on every update.

## Render-ready models

Use render-ready models when raw domain data needs display-specific transformation before rendering.

A render-ready model should contain values that are cheap for SwiftUI to display:

* stable identity;
* display strings;
* precomputed visual flags;
* formatted amounts, dates, counts, and labels;
* image identifiers or cache keys, not decoded image work;
* simple enum state for styling;
* lightweight values needed by the row.

A render-ready model should not contain:

* live database queries;
* network calls;
* expensive lazy computed display fields;
* unstable generated IDs;
* formatter instances created per row without a specific reason;
* non-visual action closures used only for event routing.

Avoid storing formatter instances per row. Keep formatters in a shared formatter, cache, builder, presenter, or formatting service when Foundation formatter creation or repeated formatting is visible in the hot path.

Render-ready models are most useful for feeds, financial rows, search results, dashboards, analytics screens, catalog grids, timelines, and repeated content where the raw model is not display-ready.

Do not introduce render models mechanically for every small view. Use them when transformation cost, repeated work, or separation of UI state from domain state matters.

## Risky pattern

```swift
struct StatementScreen: View {
    let entries: [StatementEntry]
    let selectedCategory: Category?

    var body: some View {
        List {
            ForEach(
                entries
                    .filter { selectedCategory == nil || $0.category == selectedCategory }
                    .sorted { $0.date > $1.date }
            ) { entry in
                VStack(alignment: .leading) {
                    Text(entry.merchantName)
                    Text(entry.amount.formatted(.currency(code: entry.currencyCode)))
                    Text(entry.date.formatted(date: .abbreviated, time: .omitted))
                }
            }
        }
    }
}
```

This is risky because filtering, sorting, and formatting can run during view updates. If the list is large or the screen updates often, the render path becomes a data pipeline.

The problem is not the existence of sorting or formatting. The problem is doing it repeatedly as part of rendering.

## Preferred pattern

```swift
struct StatementRowModel: Identifiable, Equatable {
    let id: StatementEntry.ID
    let merchantName: String
    let amountText: String
    let dateText: String

    init(entry: StatementEntry) {
        id = entry.id
        merchantName = entry.merchantName
        amountText = entry.amount.formatted(.currency(code: entry.currencyCode))
        dateText = entry.date.formatted(date: .abbreviated, time: .omitted)
    }
}

@MainActor
@Observable
final class StatementModel {
    private var entries: [StatementEntry] = []
    private var selectedCategory: Category?

    private(set) var rows: [StatementRowModel] = []

    func setEntries(_ newEntries: [StatementEntry]) {
        entries = newEntries
        rebuildRows()
    }

    func selectCategory(_ category: Category?) {
        selectedCategory = category
        rebuildRows()
    }

    private func rebuildRows() {
        rows = Self.makeRows(
            entries: entries,
            selectedCategory: selectedCategory
        )
    }

    nonisolated private static func makeRows(
        entries: [StatementEntry],
        selectedCategory: Category?
    ) -> [StatementRowModel] {
        entries
            .filter { selectedCategory == nil || $0.category == selectedCategory }
            .sorted { $0.date > $1.date }
            .map(StatementRowModel.init)
    }
}

struct StatementScreen: View {
    let model: StatementModel

    var body: some View {
        List(model.rows) { row in
            StatementRow(row: row)
        }
    }
}
```

This moves transformation to the points where inputs change. The view renders already prepared data.

The `@MainActor` annotation makes UI-facing state mutation explicit. The `nonisolated` helper is useful only when the transformation does not need main-actor state and its inputs and outputs are safe to use across concurrency boundaries.

For small or medium transformations, synchronous rebuilding on the main actor may be acceptable. For large transformations, do not necessarily run `rebuildRows()` synchronously on the main actor.

A larger transformation can be prepared asynchronously and committed with one compact main-actor update:

```swift
@MainActor
func setEntries(_ newEntries: [StatementEntry]) async {
    entries = newEntries

    let selectedCategory = selectedCategory

    let preparedRows = await Task.detached {
        Self.makeRows(
            entries: newEntries,
            selectedCategory: selectedCategory
        )
    }.value

    rows = preparedRows
}
```

Use this pattern only when the work is large enough to justify task coordination and the transferred data is safe to use outside the main actor. Extracting code into an `async` function does not automatically move CPU-heavy work off the main actor.

## Derived collections

Derived collections are arrays, sections, pages, dictionaries, or grouped models produced from raw data.

Prefer deriving collections when the input changes:

* after a network response;
* after a database fetch;
* when a filter, sort option, or debounced search text changes;
* when a page is appended;
* when locale, currency, calendar, or display settings change.

Avoid deriving collections directly inside `ForEach`:

```swift
ForEach(entries.sorted { $0.date > $1.date }) { entry in
    StatementRow(entry: entry)
}
```

Prefer storing or passing the derived collection:

```swift
ForEach(rows) { row in
    StatementRow(row: row)
}
```

For very small static arrays, a local transformation can be acceptable. For large, scrolling, frequently changing, or user-interactive collections, treat repeated derivation as a performance risk.

## Formatting and parsing

Formatting is easy to hide inside views because it looks like presentation work. In large or frequently updated views, it can still become repeated CPU and allocation work.

Review these patterns carefully:

```swift
Text(transaction.date.formatted(date: .abbreviated, time: .omitted))
Text(transaction.amount.formatted(.currency(code: transaction.currencyCode)))
Text(formatter.string(from: transaction.date))
Text(markdownParser.render(transaction.description))
```

Do not flag every `.formatted(...)` call mechanically. It is usually fine for small static views, single labels, or rarely updated screens. Treat it as a body-cost issue mainly in repeated rows, high-frequency updates, scrolling, typing, animations, or measured CPU/allocation hot paths.

Prefer:

* formatted strings in render models for large or frequently updated collections;
* cached formatter instances when using Foundation formatters and formatter creation is visible in the hot path;
* formatting when source data, locale, calendar, currency, or display settings change;
* isolated formatter benchmarks when formatting appears in Time Profiler;
* signposts around large formatting batches.

Do not cache formatted strings blindly if they depend on locale, calendar, currency, time zone, accessibility settings, or user preferences. Make invalidation rules explicit.

## Expensive computed properties

Computed properties used by views are still part of the render path when `body` reads them.

Risky:

```swift
struct PortfolioModel {
    var holdings: [Holding]

    var topMovers: [Holding] {
        holdings
            .filter { abs($0.changePercent) > 2 }
            .sorted { abs($0.changePercent) > abs($1.changePercent) }
    }
}

struct PortfolioScreen: View {
    let model: PortfolioModel

    var body: some View {
        ForEach(model.topMovers) { holding in
            HoldingRow(holding: holding)
        }
    }
}
```

Computed properties are fine when they are cheap, scalar, and clearly local. They are risky when they allocate, iterate, parse, format, sort, filter, group, or perform I/O.

## MainActor preparation

SwiftUI rendering and UI updates happen on the main actor. Heavy data preparation on the main actor during view updates can block interaction and frame delivery.

Prefer this flow for large pure transformations:

1. Read or receive raw data.
2. Transform raw data into render models off the main actor when safe.
3. Apply one compact final state update on the main actor.
4. Render prepared values.

Use async preparation only when the transformation is pure enough to move safely and large enough to justify coordination. For small transformations, task overhead and complexity may not be worth it.

Extracting transformation into an `async` function does not automatically move CPU work off the main actor. The helper must not be main-actor-isolated, must not call main-actor-only APIs, and the transferred input and output should be safe to use across concurrency boundaries.

Avoid starting unbounded work on every keystroke. Debounce or cancel outdated work when appropriate.

Good places to build render models:

* after decoding a network response;
* after receiving a database result;
* when appending a page;
* when filter, sort, or debounced search changes;
* in a model, store, reducer, presenter, or view model layer;
* in a pure helper that can be benchmarked independently.

Risky places to build render models:

* every `body` evaluation;
* every row body;
* inside a `ForEach` expression;
* inside a view initializer called during repeated rendering;
* inside an unguarded row `onAppear`;
* synchronously on the main actor during scrolling, typing, dragging, or animation.

## Validation

Use validation to distinguish a likely risk from a measured problem.

### Time Profiler

Use Time Profiler when CPU cost is suspected. Look for app-specific functions under SwiftUI update stacks or around the tested interaction:

* sorters, filters, mappers, parsers, regex functions;
* formatters and formatted string builders;
* render model builders;
* row initializers;
* image preparation;
* computed properties read by views.

Useful Call Tree settings:

* Separate by Thread, to distinguish main-thread render work from background preparation;
* Invert Call Tree, to bring leaf app functions closer to the top;
* Hide System Libraries, to make app code easier to find.

If a costly function no longer appears near the top after a refactor, stop optimizing that function and re-check the user-visible interaction.

### SwiftUI Instrument

Use the SwiftUI instrument when the question is body frequency, body duration, or update breadth.

Check whether a view body runs more often than expected, whether a body has measurable duration, whether rows update during unrelated state changes, and whether a refactor actually reduced view work.

### Allocations

Use Allocations when the symptom suggests memory churn or repeated construction.

Look for repeated formatter creation, temporary strings, rebuilt arrays of render models, grouped dictionaries, decoded or resized images, regex construction, type-erased wrappers, or copy-on-write churn in large arrays.

### Signposts and timestamps

Add signposts around user-visible update boundaries: filter changed, sort changed, search text applied, render models built, page appended, state assigned, first visible rows updated, animation started or completed.

For simple user-visible latency, timestamps can be enough to locate the gap before opening Instruments. Measure the interval the user feels, not only cumulative CPU time across all threads.

### XCTest performance tests

Use XCTest performance tests for isolated transformation pipelines: render model generation, sorting, filtering, grouping, formatting, cache lookup, parser replacement, or precomputed lookup structures.

Do not use a microbenchmark as the only proof of scroll smoothness or interaction responsiveness. Use it to isolate app code cost, then validate the UI scenario.

## Review checklist

When reviewing body cost and render models, check:

* Does `body` mostly assemble already prepared values?
* Are sorting, filtering, grouping, and mapping outside the render path?
* Are formatted display strings prepared before rendering when rows are numerous or updates are frequent?
* Are expensive computed properties avoided in `body`?
* Are render models rebuilt only when their inputs change?
* Are render models small, stable, and display-focused?
* Is heavy preparation kept off the main actor when safe and worthwhile?
* Is the final main-actor state update compact?
* Are cache invalidation rules clear for locale, currency, calendar, time zone, accessibility settings, and user preferences?
* Is the proposed refactor targeted at the suspected or measured bottleneck?
* Is there a validation plan using Time Profiler, SwiftUI Instrument, Allocations, signposts, timestamps, or XCTest?

## Common mistakes

* Do not move expensive work from `body` into a computed property and call it from `body`.
* Do not create render models on every body evaluation.
* Do not rebuild all render models for every small row change in large or frequently updated collections unless measurement shows the cost is irrelevant.
* Do not build page or section arrays inside `body`.
* Do not create formatters per row unless the cost is known to be irrelevant.
* Do not cache formatted values without considering locale, calendar, currency, time zone, accessibility settings, and user preferences.
* Do not move heavy work off the main actor if the UI immediately awaits it and still blocks the interaction.
* Do not assume an `async` helper automatically moves CPU-heavy work off the main actor.
* Do not introduce async complexity for tiny transformations.
* Do not keep optimizing a function after it leaves the measured hot path.
* Do not claim numeric savings without measurement.
