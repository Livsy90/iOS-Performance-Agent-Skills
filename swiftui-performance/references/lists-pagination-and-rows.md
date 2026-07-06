# Lists, Pagination, and Rows

Use this reference when the task involves SwiftUI `List`, `ScrollView`, `LazyVStack`, feeds, paginated data, row complexity, scroll hitches, duplicate pagination loads, expensive append/update paths, or list performance regressions.

## Contents

* [Core model](#core-model)
* [Start from the symptom](#start-from-the-symptom)
* [Choosing the container](#choosing-the-container)
* [Stable identity](#stable-identity)
* [Row cost](#row-cost)
* [Pagination and update locality](#pagination-and-update-locality)
* [Do not derive pages inside `body`](#do-not-derive-pages-inside-body)
* [Filtering, sorting, and formatting](#filtering-sorting-and-formatting)
* [Row interactions](#row-interactions)
* [Layout and geometry](#layout-and-geometry)
* [Pagination triggers](#pagination-triggers)
* [Background work and UI headroom](#background-work-and-ui-headroom)
* [Validation](#validation)
* [Common mistakes](#common-mistakes)

## Core model

Large, growing, scrolling, or frequently updating collections are SwiftUI hot paths.

Optimize lists for stable identity, cheap row construction, predictable structure, narrow state dependencies, explicit pagination boundaries, and local updates that make the changed unit obvious.

A performant list makes it clear what changed: one row, one page, one loading footer, one filter result, or one selection state. If every append, filter, sort, or small state change forces SwiftUI to reconcile a large unstable structure, scrolling and pagination become fragile.

## Start from the symptom

Do not optimize a list only because the code looks complex.

First identify the user-facing symptom: scroll hitches, a frozen "Load more" tap, a main-thread spike during append, unrelated row updates, duplicate page loads, memory growth while scrolling, or hitches during search/filter/sort.

Then connect the symptom to a likely hot path: collection identity, row body cost, repeated data preparation, pagination append structure, per-row layout work, row interactions, or main-actor work during list updates.

Avoid replacing the list container, introducing sections, or rewriting rows until the suspected bottleneck matches the symptom.

## Choosing the container

Do not treat `List`, `LazyVStack`, and `VStack` as interchangeable.

Prefer `List` when the UI needs platform-native long-list behavior, swipe actions, editing, selection, accessibility behavior, native row styling, or large dynamic datasets where system behavior matters.

Prefer `ScrollView` with `LazyVStack` or `LazyHStack` when the UI needs custom spacing, card layouts, mixed static and dynamic regions, pinned content, or more layout control than `List` provides.

Use `VStack` only for small, fixed-size content.

Avoid eager stacks for long dynamic collections:

```swift
ScrollView {
    VStack {
        ForEach(feedRows) { row in FeedCard(row: row) }
    }
}
```

Prefer a lazy container for dynamic content:

```swift
ScrollView {
    LazyVStack(spacing: 12) {
        ForEach(feedRows) { row in FeedCard(row: row) }
    }
}
```

Do not recommend replacing `List` with `LazyVStack` as a default performance fix. Explain reuse, memory behavior, OS/version behavior, accessibility, editing, swipe actions, and row complexity trade-offs. `LazyVStack` is lazy construction, not UIKit-style cell reuse.

## Stable identity

Large collections need stable, cheap identifiers.

Avoid index-based identity when insertion, deletion, sorting, filtering, or pagination can change positions:

```swift
ForEach(Array(rows.enumerated()), id: \.offset) { _, row in
    TransactionRow(row: row)
}
```

Prefer identity from the data:

```swift
ForEach(rows) { row in
    TransactionRow(row: row)
}
```

Avoid computed IDs that create a new value on every access:

```swift
struct RowModel: Identifiable {
    var id: UUID { UUID() }
}
```

Prefer stored identity:

```swift
struct RowModel: Identifiable, Equatable {
    let id: Transaction.ID
    let title: String
    let amountText: String
}
```

Use `.id(...)` only for a deliberate identity boundary, such as resetting local state or defining scroll targets. Do not use `.id(UUID())` as a refresh workaround.

## Row cost

Rows in large lists should mostly render already prepared values.

Watch for repeated work in row `body`, row initializers, row modifiers, and computed properties read by rows:

* currency/date formatting;
* localized string construction;
* image decoding/resizing;
* icon lookup;
* sorting or filtering child data;
* expensive computed properties;
* database reads;
* synchronous file access;
* long modifier chains;
* overlays, masks, shadows, and blurs.

Risky:

```swift
Text(transaction.amount.formatted(.currency(code: transaction.currencyCode)))
Image(categoryIconName(for: transaction.category))
```

Prefer render-ready row models for non-trivial rows:

```swift
struct TransactionRowModel: Identifiable, Equatable {
    let id: Transaction.ID
    let title: String
    let amountText: String
    let categoryIconName: String
}
```

The row should mostly render `TransactionRowModel` values instead of preparing display data. Render-ready models are especially useful in long lists, paginated feeds, search results, and frequently refreshed screens.

Do not flag every small formatting call mechanically. Treat row work as a performance risk when it is repeated across many rows, happens during scrolling or typing, or appears in a measured hot path.

## Pagination and update locality

For paginated data with non-trivial rows, avoid modeling the UI as one endlessly growing flat array when appending a page causes broad reconciliation or visible hitches.

A flat array append is often fine for simple rows with stable identity. Treat it as a risk when append causes measured hitches, old rows are rebuilt expensively, or pagination updates coincide with heavy row work.

Risky when rows are expensive and append is visible in profiling:

```swift
List {
    ForEach(model.transactions) { transaction in
        TransactionRow(row: transaction)
    }
}
```

Appending to one large flat array can make the append path harder to reason about when old and new rows share one large collection boundary. Even if only new elements were appended, a frequently updating parent, expensive row body, broad dependencies, or repeated transformation work can make this update path visible as a hitch.

Prefer stable page or section models when pages are appended incrementally, existing rows should remain unchanged, rows include formatting/icons/menus/swipe actions/bindings/heavy modifiers, or profiling shows a main-thread spike during page append.

Do not introduce page models mechanically. Use them when they preserve a real backend or product pagination boundary, reduce repeated work, or make measured append behavior easier to validate.

Example:

```swift
struct TransactionPage: Identifiable, Equatable {
    let id: Int
    let rows: [TransactionRowModel]
}

List {
    ForEach(model.pages) { page in
        Section {
            ForEach(page.rows) { row in TransactionRow(row: row) }
        }
    }
}
```

Append a new page instead of rebuilding previous pages:

```swift
@MainActor
func appendPage(_ response: TransactionPageResponse) {
    pages.append(TransactionPage(
        id: response.pageNumber,
        rows: response.items.map(TransactionRowModel.init)
    ))
}
```

The goal is not to use `Section` for its own sake. The goal is to make the changed unit explicit: one new page, one new section, one loading footer, or one changed row.

Stable page boundaries can reduce repeated reconciliation and row construction work during pagination, especially when old pages do not change. This is a pattern to test, not a guarantee.

Using `Section` can affect list styling, accessibility grouping, separators, headers, footers, and platform behavior. If those semantics are undesirable, keep page boundaries in the model but render them without visible section styling where appropriate.

## Do not derive pages inside `body`

Do not create page sections by chunking a flat array inside `body`.

Risky:

```swift
List {
    ForEach(model.transactions.chunked(into: 20)) { page in
        Section {
            ForEach(page) { transaction in TransactionRow(transaction: transaction) }
        }
    }
}
```

This can allocate new page values during rendering, create unstable page identity, and hide the real pagination boundary.

Prefer storing pages as stable state:

```swift
@MainActor
@Observable
final class TransactionsModel {
    var pages: [TransactionPage] = []
}
```

If the backend returns pages, preserve that boundary in the UI model. If the backend returns a flat result, create stable page models during data ingestion, not during view rendering.

For UI-facing list state, prefer explicit main-actor isolation. Build rows off the main actor when safe, then append pages with one compact main-actor update.

## Filtering, sorting, and formatting

Avoid transforming the collection inside repeated rendering paths.

Risky:

```swift
ForEach(model.transactions.sorted { $0.date > $1.date }) { transaction in ... }
ForEach(model.transactions) { if $0.matches(filter) { TransactionRow(transaction: $0) } }
```

Prefer preparing visible rows before rendering:

```swift
List(model.visibleRows) { row in
    TransactionRow(row: row)
}
```

Prepare render models when data is loaded, a page is appended, search text changes, sort order changes, or filter state changes. Then apply one compact state update on the main actor.

## Row interactions

Treat row-level interactions as part of row complexity.

Rows become more expensive when every row builds `swipeActions`, menus, context menus, gestures, buttons, custom bindings, or closure-heavy action surfaces.

Stored closure properties and closure-heavy row APIs can make row values harder to compare and reason about because SwiftUI cannot meaningfully compare closures. This is a review signal, not proof of a performance issue.

Prefer keeping stored row input visual and small. When closures are necessary, use explicit capture lists and capture stable IDs instead of whole parent values:

```swift
ForEach(page.rows) { row in
    TransactionRow(row: row)
        .onTapGesture { [model, id = row.id] in
            model.openTransaction(id)
        }
        .swipeActions {
            Button("Archive") { [model, id = row.id] in
                model.archiveTransaction(id)
            }
        }
}
```

This does not make closures automatically diffable. It reduces accidental captures, keeps row inputs cleaner, and makes dependencies easier to reason about.

## Layout and geometry

Avoid per-row layout probes in large collections.

Risky:

```swift
List(cards) { card in
    GeometryReader { proxy in
        LoyaltyCardView(card: card, width: proxy.size.width)
    }
}
```

Inside rows, `GeometryReader` can also change sizing behavior because it takes the proposed space from its parent. If it must be used in a row, make the expected height and alignment explicit.

Prefer reading geometry at a stable container boundary or using layout APIs that do not need explicit geometry:

```swift
LoyaltyCardView(card: card)
    .frame(maxWidth: .infinity, alignment: .leading)
```

Review rows carefully when they contain `GeometryReader`, preference keys, nested scroll views, repeated overlays, masks, shadows, blurs, animated layout changes, or custom layouts.

## Pagination triggers

Be careful with `.onAppear` inside rows. It can fire many times during scrolling, navigation, cell reuse-like behavior, and view reconstruction.

Risky:

```swift
.onAppear {
    if row.id == rows.last?.id {
        Task { await model.loadNextPage() }
    }
}
```

Guard pagination explicitly:

```swift
.task(id: row.id) {
    guard model.shouldLoadNextPage(afterAppearing: row.id) else { return }
    await model.loadNextPageIfNeeded(trigger: row.id)
}
```

A pagination trigger should check that the row is close enough to the end, a next page exists, a page load is not already running, the same request has not already been fired, and the trigger still matches the current query/filter/sort state.

For large lists, prefer a sentinel or footer prefetch view over attaching lifecycle work to every row. Use row-level triggers only when the guard is cheap and duplicate attempts are fully handled by the model.

Example sentinel:

```swift
List {
    ForEach(model.pages) { page in
        Section {
            ForEach(page.rows) { row in
                TransactionRow(row: row)
            }
        }
    }

    if model.pagination.hasNextPage {
        ProgressView()
            .task(id: model.pagination.nextPageKey) {
                await model.loadNextPageIfNeeded(trigger: model.pagination.nextPageKey)
            }
    }
}
```

For button-based pagination, keep the button state explicit:

```swift
Button("Load more") {
    Task { await model.loadNextPageIfNeeded(trigger: model.pagination.nextPageKey) }
}
.disabled(model.pagination.isLoading || !model.pagination.hasNextPage)
```

The model method should be idempotent. It should guard loading state, next-page availability, requested page keys, current filter/sort/search context, and cancellation before committing visible state.

## Background work and UI headroom

Moving data preparation off the main actor can help, but it does not make the work free.

Good candidates for background preparation include mapping network responses to render models, formatting large row batches, sorting, grouping, filtering, image resizing or decoding when safe, and cache lookups that do not require main-actor state.

Apply compact final state updates on the main actor.

Extracting row preparation into an `async` function does not automatically move CPU work off the main actor. The builder must not be main-actor-isolated, and transferred inputs and outputs should be safe across concurrency boundaries.

Move image preparation off the main actor only with APIs and data types that are safe for background use. Avoid touching UIKit view, view controller, layer, or SwiftUI view state from background work.

If the UI immediately awaits background preparation before it can update, the interaction can still feel blocked. Prefer cancellation, debounce, incremental updates, or a loading state when the work is user-visible.

Do not saturate all CPU cores during scrolling. Aggressive background work can still hurt responsiveness if it competes with UI rendering.

## Validation

Do not call a list optimization successful without a before/after validation path.

For scroll hitches, reproduce the same scroll path and use Animation Hitches, Core Animation, Time Profiler, or the SwiftUI instrument. Check whether the main thread is blocked and whether the cost is row body work, layout, drawing, image work, or repeated collection transforms.

For pagination append hitches:

1. Start with a known number of existing rows.
2. Trigger pagination by button tap or near-end scroll.
3. Signpost: pagination triggered, request started, response received, render models built, page appended to state, first new row visible.
4. In Time Profiler, inspect the main thread during the append window.
5. Compare flat append with stable page/section append using the same scenario.

Useful Time Profiler call tree settings:

* Separate by Thread;
* Invert Call Tree;
* Hide System Libraries.

Look for app-specific work first: row initializers, row `body` work, formatter creation, sorting/filtering/grouping, render model rebuilding, image preparation, closure/action builders, and custom layout code.

Use the SwiftUI instrument when available to inspect body invocation count, body invocation duration, views updating more often than expected, list rows recomputed during append/filter/sort, and broad invalidation after unrelated state changes.

Use Allocations when the symptom suggests repeated temporary arrays, repeated strings, formatter creation, rebuilding large render-model arrays, per-row type erasure, or large copy-on-write structures copied during rendering.

Use MetricKit or production telemetry to detect trends in hangs, animation responsiveness, memory growth, or slow interactions, then reproduce locally with Instruments when possible.

Temporary probes such as `Self._printChanges()`, row body counters, pagination trigger logs, and render model duration logs can help during investigation. Remove debug probes before shipping.

## Common mistakes

* Replacing `List` with `LazyVStack` without identifying the actual bottleneck.
* Using `VStack` for long dynamic content.
* Using index-based identity for rows that can move, be inserted, or be deleted.
* Using `UUID()` or computed UUIDs as row identity.
* Sorting, filtering, grouping, formatting, or chunking inside `body`.
* Treating flat arrays as automatically bad even when rows are simple, identity is stable, and no append hitch is measured.
* Appending pages to one flat array when profiling shows broad update work and expensive rows.
* Creating page sections by chunking a flat array during rendering.
* Treating `Section` as a guaranteed performance fix instead of a stable pagination boundary to validate.
* Building heavy swipe actions, menus, custom bindings, or closures for every row without checking row cost.
* Placing `GeometryReader` or preference-key feedback inside every row.
* Starting unguarded pagination loads from `.onAppear`.
* Creating unstructured pagination tasks without idempotency, cancellation, or duplicate-load guards.
* Assuming an `async` helper automatically moves row preparation off the main actor.
* Saturating the CPU with background processing while the user is scrolling.
* Claiming a frame-rate or millisecond improvement without a trace, log, benchmark, or user-provided measurement.
