# Progressive Rendering

Use this reference when reviewing iOS screens or flows that wait for all data before showing useful UI, delay primary content behind secondary content, clear existing content during refresh, or update the UI only after every dependency finishes.

This reference is about revealing **real content** in stages. It is not about skeletons alone and not about pretending that data is ready earlier than it is.

For placeholders, skeletons, progress indicators, loading copy, empty states, loading/error transitions, and pagination loaders, read `references/loading-states.md`.
For optimistic UI and rollback, read `references/optimistic-updates.md`.
For high-stakes, irreversible, financial, legal, security-sensitive, identity-related, or trust-sensitive flows, read `references/high-stakes-actions.md`.
For runtime validation, read `references/validation-and-testing.md`.

## Contents

* [Scope boundary](#scope-boundary)
* [Core model](#core-model)
* [What the agent can and cannot prove](#what-the-agent-can-and-cannot-prove)
* [When progressive rendering helps](#when-progressive-rendering-helps)
* [When progressive rendering is risky](#when-progressive-rendering-is-risky)
* [Review procedure](#review-procedure)
* [State model requirements](#state-model-requirements)
* [Display-safe errors](#display-safe-errors)
* [Consistency groups](#consistency-groups)
* [Common patterns](#common-patterns)

  * [Avoid all-or-nothing loading](#avoid-all-or-nothing-loading)
  * [Render critical content first](#render-critical-content-first)
  * [Use section-level failure boundaries](#use-section-level-failure-boundaries)
  * [Preserve stale content during refresh](#preserve-stale-content-during-refresh)
  * [Handle pagination as staged content](#handle-pagination-as-staged-content)
  * [Keep layout stable during staged updates](#keep-layout-stable-during-staged-updates)
  * [Avoid stale responses, duplicate loads, and abandoned work](#avoid-stale-responses-duplicate-loads-and-abandoned-work)
  * [Keep main-actor updates compact](#keep-main-actor-updates-compact)
* [Decision rules](#decision-rules)
* [Implementation checklist](#implementation-checklist)
* [Validation](#validation)
* [Review output guidance](#review-output-guidance)

## Scope boundary

Progressive rendering is a perceived-performance and product-state modeling technique.

Use it to reduce the time until the user sees meaningful structure, primary content, cached content, or an actionable part of the screen.

Do not present it as a low-level optimization. Progressive rendering may reduce blank-screen time without reducing total execution time, CPU work, memory use, network latency, or backend latency.

Progressive rendering changes **when useful real content becomes visible**. It does not automatically make rendering cheap, remove main-thread work, fix slow queries, or replace profiling.

Keep detailed loading-copy, skeleton, placeholder, empty-state, error-copy, and pagination-loader guidance in `references/loading-states.md`. Keep detailed structured-concurrency guidance in the concurrency skill or related references.

## Core model

Progressive rendering means presenting useful UI in stages instead of waiting for the entire screen to be ready.

The core question is:

> Can the user understand or use the screen before every dependency has finished?

A good progressive-rendering refactor usually changes one or more of these:

* one full-screen state becomes section-level state;
* primary content renders before secondary content;
* cached or stale content appears while fresh content loads;
* refresh preserves useful existing content instead of clearing the screen;
* partial failures are shown inline instead of collapsing the whole flow;
* layout reserves stable regions so late sections do not cause jumps;
* independent sections can load, fail, retry, or refresh without blocking the whole screen.

The goal is not to show random pieces earlier. The goal is to show the **right** pieces earlier.

## What the agent can and cannot prove

The agent can inspect and improve:

* all-or-nothing loading;
* unrelated async dependencies that block UI updates;
* slow secondary content blocking primary content;
* refresh flows that clear useful content;
* state models that cannot represent partial content;
* staged updates that may cause layout jumps;
* duplicate loads caused by independent section loaders;
* stale responses that can overwrite newer screen, filter, account, or query state;
* section-level failures that unnecessarily collapse the whole screen;
* heavy data preparation performed during each main-actor section update.

The agent cannot prove that the screen feels faster without runtime evidence.

Do not claim:

* “This fixes performance.”
* “This is now smooth.”
* “This will definitely feel faster.”
* “This reduces total loading time.”
* “This reduces CPU cost.”

Prefer:

* “This should reduce blank-screen time.”
* “This lets primary content appear before secondary content.”
* “This changes perceived latency, not necessarily total load time.”
* “Validate with tap-to-first-content timing, screen recording, or a trace.”

## When progressive rendering helps

Progressive rendering is useful when:

* the screen has independent sections;
* primary content is more important than secondary content;
* some data is available earlier than the rest;
* cached or stale content is useful during refresh;
* first-page content can appear before next-page content;
* the user can read or act before all sections are complete;
* slow secondary content currently blocks the whole screen;
* partial failure can be shown inline without collapsing the flow.

Good candidates:

* profile header before recommendations;
* account summary before transaction history when consistency rules allow it;
* article body before comments;
* search results before suggestions;
* cached feed while refreshing;
* first page before next page;
* product details before related products;
* local draft content before sync status;
* dashboard primary card before promotions, badges, or remote decorations.

## When progressive rendering is risky

Do not recommend progressive rendering blindly.

It may be wrong when:

* partial content would mislead the user;
* all data must be consistent at the same point in time;
* the product requires an atomic result;
* staged rendering would cause visible layout jumps;
* partial errors would be harder to explain than one clear error;
* ordering is essential for comprehension or correctness;
* a secondary section actually determines whether primary content is valid;
* stale content would create trust, safety, legal, financial, medical, or security risk;
* the result must be authoritative-confirmed before it is meaningful.

For financial, legal, destructive, irreversible, security-sensitive, identity-related, or trust-sensitive flows, route to `references/high-stakes-actions.md` before suggesting staged rendering.

## Review procedure

Before proposing progressive rendering, answer:

1. What is the primary content?
2. What can safely appear later?
3. Which sections are truly independent?
4. Which sections must stay consistent with each other?
5. Which dependencies are unrelated to first useful content?
6. Which failures should block the whole screen?
7. Which failures can be shown inline?
8. Can existing content remain visible during refresh?
9. Is stale content safe enough to show?
10. Would staged updates cause layout jumps?
11. Does the state model support partial content?
12. Can old responses overwrite newer state?
13. Can duplicate section loads happen?
14. Does data preparation happen off-main when safe?
15. How will the improvement be validated?

If the code does not answer a product or design question, say so explicitly. Do not invent safety rules for the product.

## State model requirements

A progressive screen usually needs state that can represent important sections independently.

Prefer section-level state when sections can load, fail, retry, or refresh separately:

```swift
struct HomeScreenState {
    var header: Loadable<Header>
    var shortcuts: Loadable<[Shortcut]>
    var recommendations: Loadable<[Recommendation]>
}

enum Loadable<Value> {
    case idle
    case loading
    case loaded(Value)
    case empty
    case failed(DisplayError)
}
```

This model is useful only if product rules allow sections to be independent. Avoid it when the screen must be internally consistent as one atomic snapshot.

For refreshable sections, include enough state to preserve useful content:

```swift
enum SectionState<Value> {
    case idle
    case loading
    case loaded(Value, isRefreshing: Bool)
    case empty
    case failed(DisplayError)
}
```

For cached content, represent it explicitly when freshness matters:

```swift
enum SectionState<Value> {
    case loading
    case showingCached(Value, isRefreshing: Bool)
    case loaded(Value)
    case empty
    case failed(DisplayError)
}
```

The exact model should match product semantics. Do not add complex state when the screen is small, local, and does not need staged behavior.

## Display-safe errors

Do not expose raw technical errors directly through user-facing section state.

Risky:

```swift
enum Loadable<Value> {
    case idle
    case loading
    case loaded(Value)
    case empty
    case failed(Error)
}
```

Raw `Error` values may be too technical, not localized, not `Equatable`, unsafe to display, or missing recovery guidance.

Prefer a display-safe error model at the UI boundary:

```swift
struct DisplayError: Equatable {
    let title: String
    let message: String
    let retryTitle: String
}
```

Keep technical errors available for logging, analytics, or diagnostics when appropriate, but do not make visible UI copy depend on raw implementation details.

## Consistency groups

Not all sections are independent.

Sometimes a group of sections must be loaded or updated from the same logical snapshot, while other sections can render later.

Examples:

* account balance and available credit may need the same snapshot;
* cart items and total price may need the same pricing response;
* eligibility status and allowed actions may need the same authorization result;
* medical or legal summary sections may need atomic consistency;
* security settings and effective permissions may need the same authoritative state.

Group dependent sections into the same load boundary:

```swift
struct AccountSnapshot {
    let balance: Balance
    let availableCredit: Money
    let allowedActions: [AccountAction]
}

struct AccountScreenState {
    var accountSnapshot: Loadable<AccountSnapshot>
    var recommendations: Loadable<[Recommendation]>
    var promotions: Loadable<[Promotion]>
}
```

Do not split sections independently if doing so can show contradictory, misleading, or unsafe information.

Use progressive rendering around consistency boundaries, not through them.

## Common patterns

### Avoid all-or-nothing loading

Smell:

```swift
async let header = service.loadHeader()
async let activity = service.loadActivity()
async let recommendations = service.loadRecommendations()

state = try await .loaded(header, activity, recommendations)
```

This may be technically concurrent, but the UI still waits for every section before showing anything useful.

Prefer section-level updates when sections are independent:

```swift
@MainActor
func load() async {
    state.header = .loading
    state.activity = .loading
    state.recommendations = .loading

    async let header: Void = loadHeader()
    async let activity: Void = loadActivity()
    async let recommendations: Void = loadRecommendations()

    _ = await (header, activity, recommendations)
}

@MainActor
private func loadHeader() async {
    do {
        let header = try await service.loadHeader()
        state.header = .loaded(header)
    } catch {
        state.header = .failed(mapError(error))
    }
}
```

The important change is not merely using `async let`. The important change is allowing UI state to update by section.

`async let` is scoped. The parent async scope will wait for child tasks before it exits. That is fine when the screen or model owns the entire load operation, but staged rendering still requires section loaders to apply state as each section finishes.

For dynamic section counts, use a task group or another explicit loading strategy. For screen-owned UI loading, avoid `Task.detached` by default because it loses structured context and actor inheritance.

### Render critical content first

Identify the minimum content that makes the screen meaningful.

Ask:

* What tells the user they are in the right place?
* What content can they read first?
* Which action can become available first?
* Which sections are secondary?
* Which secondary failures should be inline?
* Which data must be present before the primary action is safe?

Do not block first useful content behind comments, recommendations, promotions, badges, remote decorations, analytics enrichment, or personalization unless the product requires them.

### Use section-level failure boundaries

Use full-screen failure when primary content cannot load or when the screen has no useful partial state.

For secondary content, prefer inline failure when product rules allow it:

```swift
@MainActor
private func loadRecommendations() async {
    do {
        let value = try await service.loadRecommendations()
        state.recommendations = value.isEmpty ? .empty : .loaded(value)
    } catch {
        state.recommendations = .failed(mapError(error))
    }
}
```

A secondary failure should not collapse the whole screen unless that section is required for correctness, safety, or trust.

If stale or cached content is still useful, prefer preserving it and showing a smaller inline error.

### Preserve stale content during refresh

Refreshing should not always clear existing content.

Smell:

```swift
func refresh() async {
    state = .loading
    state = await loadFreshState()
}
```

If the user already had useful content, this can create a blank or unstable experience.

Prefer preserving existing content when it is safe:

```swift
@MainActor
func refresh() async {
    guard case .loaded(let oldItems, _) = state.feed else {
        await loadInitial()
        return
    }

    state.feed = .loaded(oldItems, isRefreshing: true)

    do {
        let freshItems = try await service.loadItems()
        state.feed = freshItems.isEmpty
            ? .empty
            : .loaded(freshItems, isRefreshing: false)
    } catch {
        state.feed = .loaded(oldItems, isRefreshing: false)
        errorPresenter.showRefreshFailed()
    }
}
```

Use stale content carefully when old data may be misleading, sensitive, unsafe, or must be visually marked as stale.

For high-stakes or server-authoritative data, read `references/high-stakes-actions.md`.

### Handle pagination as staged content

Paginated screens are a common progressive-rendering case.

The first page often represents primary content. Next-page loading should usually be modeled separately from initial loading.

Risky:

```swift
func loadNextPage() async {
    state = .loading
    state = await loadAllPagesAgain()
}
```

This can hide already useful content.

Prefer a separate pagination state:

```swift
struct FeedState {
    var items: [FeedItem]
    var pageState: PageState
}

enum PageState {
    case idle
    case loadingNextPage
    case failedNextPage(DisplayError)
    case reachedEnd
}
```

This allows the first page to remain visible while the next page loads, fails, retries, or reaches the end.

For detailed loading-more UI, retry footer, and pagination loaders, read `references/loading-states.md`.

### Keep layout stable during staged updates

Progressive rendering can feel worse if late sections change size and push content around.

Check whether:

* expected sections reserve stable space;
* placeholders roughly match final size;
* late banners or cards can push primary content unexpectedly;
* refresh preserves scroll position when possible;
* section insertion does not interrupt reading;
* skeletons do not change height significantly when real content appears.

Do not trade a blank screen for a jumpy screen.

For skeletons, placeholders, loading copy, and empty-state presentation, read `references/loading-states.md`.

### Avoid stale responses, duplicate loads, and abandoned work

Progressive rendering often creates more independent loading paths. Make sure this does not create stale UI, duplicate work, or abandoned tasks.

Check:

* Can the same section load twice?
* Can an older response overwrite a newer screen state?
* What happens when the user changes account, filter, query, or tab?
* What happens when refresh starts during initial loading?
* What happens when the screen disappears?
* Are owner-scoped tasks cancelled or allowed to finish intentionally?
* Are loaders idempotent?
* Is there a request identifier or version check?

Example with request identity:

```swift
@MainActor
final class HomeModel {
    private var currentRequestID = UUID()
    private(set) var state = HomeScreenState(
        header: .idle,
        shortcuts: .idle,
        recommendations: .idle
    )

    func load(accountID: Account.ID) async {
        let requestID = UUID()
        currentRequestID = requestID

        state.header = .loading
        state.shortcuts = .loading
        state.recommendations = .loading

        async let header: Void = loadHeader(accountID: accountID, requestID: requestID)
        async let shortcuts: Void = loadShortcuts(accountID: accountID, requestID: requestID)
        async let recommendations: Void = loadRecommendations(accountID: accountID, requestID: requestID)

        _ = await (header, shortcuts, recommendations)
    }

    private func isCurrent(_ requestID: UUID) -> Bool {
        currentRequestID == requestID
    }

    private func loadHeader(accountID: Account.ID, requestID: UUID) async {
        do {
            let header = try await service.loadHeader(accountID: accountID)
            guard isCurrent(requestID) else { return }
            state.header = .loaded(header)
        } catch {
            guard isCurrent(requestID) else { return }
            state.header = .failed(mapError(error))
        }
    }
}
```

Do not let a response for an old query, account, filter, tab, or screen instance replace the current state.

### Keep main-actor updates compact

Progressive rendering can increase the number of UI state updates. That is usually fine, but each update should stay cheap.

Avoid doing heavy data preparation on the main actor during each section completion.

Risky:

```swift
@MainActor
private func loadRecommendations() async {
    let response = try await service.loadRecommendations()

    state.recommendations = .loaded(
        response.items
            .sorted(by: expensiveSort)
            .map(RecommendationViewModel.init)
    )
}
```

Prefer preparing render-ready values off-main when safe, then applying compact state changes on the main actor:

```swift
private func loadRecommendations() async {
    do {
        let response = try await service.loadRecommendations()
        let viewModels = await prepareRecommendationViewModels(response.items)

        await MainActor.run {
            state.recommendations = viewModels.isEmpty
                ? .empty
                : .loaded(viewModels)
        }
    } catch {
        await MainActor.run {
            state.recommendations = .failed(mapError(error))
        }
    }
}
```

Only move pure, thread-safe transformations off-main. UI state updates must still happen on the main actor.

Do not claim progressive rendering improved performance if staged updates introduce hitches or main-thread work.

## Decision rules

* Prefer progressive rendering when it reduces blank-screen time without misleading the user.
* Prefer section-level state only when sections can load, fail, retry, or refresh independently.
* Keep consistency groups atomic when sections must agree with each other.
* Preserve stale content during refresh only when stale content is safe and clearly represented.
* Use full-screen failure for primary-content failure or atomic flows.
* Use inline failure for secondary sections when partial content is still useful.
* Keep staged layout stable; do not trade a blank screen for a jumpy screen.
* Prevent old responses from overwriting newer screen state.
* Keep main-actor state updates compact.
* Do not recommend staged rendering for high-stakes flows until safety and confirmation requirements are clear.
* Do not claim success without validation evidence.

## Implementation checklist

Before finalizing a recommendation, check:

* [ ] The primary content is identified.
* [ ] Secondary content is separated from first useful content.
* [ ] The state model can represent section-level loading.
* [ ] Display-safe errors are used at the UI boundary.
* [ ] Sections that must stay consistent are grouped together.
* [ ] Partial failures do not collapse useful content unnecessarily.
* [ ] Refresh does not clear existing content unless required.
* [ ] Stale content is safe or visually marked.
* [ ] First-page loading and next-page loading are modeled separately when relevant.
* [ ] Layout remains stable as sections update.
* [ ] Duplicate loads are avoided.
* [ ] Stale responses are ignored or reconciled.
* [ ] Task lifetime and cancellation are considered.
* [ ] `Task.detached` is avoided for screen-owned UI loading unless there is a clear reason.
* [ ] Heavy mapping, formatting, sorting, or filtering is not moved into repeated main-actor updates.
* [ ] The recommendation includes a realistic validation step.

## Validation

Validate progressive rendering with evidence that reflects user perception.

Useful validation:

* screen recording from tap to first meaningful content;
* tap-to-first-content timing;
* time until the primary action becomes available;
* slow-network testing;
* repeated refresh attempts;
* filter/query/account switching during load;
* stale response test;
* inspection for layout jumps during staged updates;
* scroll-position check during refresh;
* Instruments when staged rendering still hitches or blocks the main thread;
* production signals when available.

Use precise phrasing:

* “This should reduce blank-screen time.”
* “This lets primary content appear before secondary content.”
* “This keeps the first page visible while the next page loads.”
* “Validate by measuring tap-to-first-content and checking for layout jumps.”

Avoid:

* “This makes the screen faster.”
* “This fixes performance.”
* “This guarantees better UX.”
* “Section-level loading always improves perceived performance.”
* “Using `async let` automatically gives progressive rendering.”

For older-device testing, Low Power Mode, release builds, Instruments, production signals, and broader responsiveness validation, read `references/validation-and-testing.md`.

## Review output guidance

When using this reference, explain:

```markdown
## Finding

The screen waits for too much work before showing useful UI.

## User impact

The user sees a blank or static state even though primary content, cached content, or an actionable part of the screen could appear earlier.

## Recommended change

Introduce section-level state where product rules allow it. Render critical content first, preserve stale content during refresh when safe, keep consistency groups atomic, and show inline loading or error states for secondary sections.

## Trade-offs

Call out consistency, stale-data, layout-stability, cancellation, stale-response, main-thread-work, and product-safety risks.

## Validation

Measure tap-to-first-content and inspect a screen recording for layout jumps or confusing transitions. Test slow network, refresh, failure, stale responses, and filter/query/account changes during load.
```
