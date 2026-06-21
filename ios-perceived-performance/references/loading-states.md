# Loading States

Use this reference when reviewing what the app shows while content is unavailable, loading, refreshing, loading more, transitioning, empty, or failed.

This reference is about state communication. It covers placeholders, skeleton views, progress indicators, cached content, stale content, pagination loaders, loading copy, empty states, error states, retry behavior, and transitions between loading, loaded, empty, and failed states.

For staged real-content rendering, read `references/progressive-rendering.md`.
For optimistic UI and rollback, read `references/optimistic-updates.md`.
For high-stakes operations that should not appear complete before confirmation, read `references/high-stakes-actions.md`.
For runtime validation, read `references/validation-and-testing.md`.

## Contents

* [Core idea](#core-idea)
* [What the agent can and cannot prove](#what-the-agent-can-and-cannot-prove)
* [State model first](#state-model-first)
* [Display-safe errors](#display-safe-errors)
* [Initial loading](#initial-loading)
* [Progress indicators](#progress-indicators)
* [Placeholders and skeleton views](#placeholders-and-skeleton-views)
* [Cached, stale, and partial content](#cached-stale-and-partial-content)
* [Empty states](#empty-states)
* [Error states](#error-states)
* [Refreshing existing content](#refreshing-existing-content)
* [Pagination and loading more](#pagination-and-loading-more)
* [Transitions between states](#transitions-between-states)
* [Anti-flicker policy](#anti-flicker-policy)
* [Cancellation and stale responses](#cancellation-and-stale-responses)
* [Controls during pending work](#controls-during-pending-work)
* [Loading copy](#loading-copy)
* [Accessibility](#accessibility)
* [Implementation checklist](#implementation-checklist)
* [Validation](#validation)
* [Review output guidance](#review-output-guidance)

## Core idea

A loading state should tell the user that the app is working, what kind of content is expected, and what they can do next.

A blank or static screen can look broken even when the app is loading correctly. A good loading state reduces uncertainty without pretending that work completed earlier than it did.

Loading states do not usually reduce actual execution time. They improve perceived responsiveness by making progress, structure, available content, and state transitions visible.

The goal is not to show a spinner everywhere. The goal is to communicate the current state honestly and keep the user oriented.

## What the agent can and cannot prove

The agent can inspect and improve:

* missing loading states;
* screens that show a blank view while loading;
* state models that collapse loading, empty, failed, refreshing, and loading-more into one case;
* refresh flows that hide existing content unnecessarily;
* pagination flows that replace the whole screen when only the next page is loading;
* cached content that is not used even when it could reduce blank time;
* stale content that is shown without clear labeling;
* missing retry paths;
* missing disabled or pending state for controls;
* placeholders that cause layout jumps;
* error states that do not explain what happened;
* empty states that provide no next action;
* loading copy that is vague, misleading, or prematurely certain;
* async race conditions where an older response can overwrite a newer state.

The agent cannot prove that a loading state feels better without runtime evidence.

Do not claim:

* “This makes the app faster.”
* “This guarantees better perceived performance.”
* “The loading experience is now correct.”
* “Users will definitely perceive this as faster.”

Prefer:

* “This makes loading visible.”
* “This separates loading from empty and error states.”
* “This should reduce uncertainty during the wait.”
* “Validate with a screen recording or state-transition test.”

## State model first

Avoid representing too many states with one boolean.

Risky:

```swift
@MainActor
final class InboxModel {
    private(set) var isLoading = false
    private(set) var messages: [Message] = []

    func load() async {
        isLoading = true
        messages = (try? await service.loadMessages()) ?? []
        isLoading = false
    }
}
```

This cannot distinguish:

* loading;
* loaded with content;
* loaded but empty;
* failed;
* refreshing existing content;
* failed refresh with stale content still visible;
* loading next page;
* failed next-page load.

Prefer explicit states:

```swift
enum LoadState<Value> {
    case idle
    case loading
    case loaded(Value)
    case empty
    case failed(DisplayError)
}
```

For refresh flows, keep existing content when safe:

```swift
enum InboxState {
    case loading
    case loaded(messages: [Message], isRefreshing: Bool)
    case empty
    case failed(DisplayError)
}
```

For screens that can show cached content before fresh content arrives:

```swift
enum FeedState {
    case loading
    case showingCached(items: [FeedItem], isRefreshing: Bool)
    case loaded(items: [FeedItem])
    case empty
    case failed(DisplayError)
}
```

The exact shape depends on the screen. The important part is that loading, empty, loaded, failed, refreshing, cached, and loading-more are not accidentally collapsed into the same UI.

`idle` should usually be transient. Do not leave a user-visible screen in `idle` with `EmptyView()` unless the parent starts loading immediately or the screen genuinely has nothing to show yet.

## Display-safe errors

Do not store or display raw technical errors directly when the user needs understandable recovery.

Risky:

```swift
enum LoadState<Value> {
    case loading
    case loaded(Value)
    case failed(Error)
}
```

This is acceptable internally, but raw `Error` values are often not ideal for UI because they may be:

* not `Equatable`;
* too technical;
* not localized;
* unsafe to display directly;
* missing recovery guidance;
* inconsistent across service layers.

Prefer mapping errors to display-safe values at the UI boundary:

```swift
struct DisplayError: Equatable {
    let title: String
    let message: String
    let retryTitle: String
}
```

Example:

```swift
enum LoadState<Value> {
    case idle
    case loading
    case loaded(Value)
    case empty
    case failed(DisplayError)
}
```

Keep the underlying technical error available for logging, diagnostics, or analytics when appropriate, but do not make user-facing copy depend on raw implementation details.

## Initial loading

Initial loading is the first time the screen has no useful content to show.

Use one of these depending on context:

* progress indicator for generic waiting;
* skeleton or placeholder when the content structure is predictable;
* cached content when safe and available;
* empty-state shell when the screen needs orientation before data arrives;
* full-screen loading only when the entire screen is unavailable;
* section-level loading when parts of the screen can appear independently.

Risky:

```swift
var body: some View {
    if model.items.isEmpty {
        EmptyView()
    } else {
        ItemList(items: model.items)
    }
}
```

This makes loading and empty look the same.

Prefer:

```swift
var body: some View {
    switch model.state {
    case .loading:
        ItemListPlaceholder()

    case .loaded(let items):
        ItemList(items: items)

    case .empty:
        ContentUnavailableView(
            "No items yet",
            systemImage: "tray",
            description: Text("New items will appear here when they are available.")
        )

    case .failed(let error):
        RetryView(
            title: error.title,
            message: error.message,
            actionTitle: error.retryTitle
        ) {
            Task { await model.reload() }
        }

    case .idle:
        EmptyView()
    }
}
```

Use system components when they fit. For example, SwiftUI `ContentUnavailableView` can provide standard empty-state presentation on supported OS versions. If the app supports earlier iOS versions, use a custom empty-state view or an availability-gated fallback.

## Progress indicators

Use progress indicators when the app is doing work and the user needs to know it has not stalled.

Use determinate progress when progress is measurable:

```swift
struct ImportState {
    var completedFiles: Int
    var totalFiles: Int

    var progress: Double {
        guard totalFiles > 0 else { return 0 }
        return Double(completedFiles) / Double(totalFiles)
    }
}
```

Use indeterminate progress when duration or progress cannot be estimated.

Do not fake precision. If the app does not know how much work remains, avoid a determinate progress bar that moves arbitrarily.

Good candidates for determinate progress:

* file import;
* upload with known byte count;
* export with known steps;
* batch operation with known item count;
* download with known content length.

Good candidates for indeterminate progress:

* waiting for an unknown network response;
* short server operation;
* authentication check;
* initial request with unknown duration.

Avoid showing a spinner as the only UI for long waits when more informative progress, cached content, partial content, or section-level loading is possible.

## Placeholders and skeleton views

Placeholders and skeleton views are useful when no real content is available yet but the structure is predictable.

Use them to:

* show the expected layout;
* reserve stable space;
* reduce the sense of a blank screen;
* prevent layout jumps when real content arrives;
* communicate that content is loading.

Skeletons should reserve approximately the same size as the expected content. A skeleton that changes height significantly when real content arrives can still cause layout jumps.

Do not use skeletons when:

* real cached content can be shown instead;
* partial real content can be shown instead;
* the content shape is not predictable;
* the skeleton would be misleading;
* the operation is high-stakes and needs explicit progress;
* the skeleton hides an error;
* the skeleton remains indefinitely without explanation.

Risky:

```swift
if let profile {
    ProfileHeader(profile)
}
```

If the profile header appears late, the layout may jump.

Prefer a stable placeholder when the header is expected:

```swift
switch state.profile {
case .loading:
    ProfileHeaderPlaceholder()

case .loaded(let profile):
    ProfileHeader(profile)

case .empty:
    ProfileUnavailableView()

case .failed:
    ProfileRetryView()

case .idle:
    ProfileHeaderPlaceholder()
}
```

Skeletons should visually read as temporary. They should not look like real content, and they should not be interactive.

For accessibility, skeleton-only decorative elements should usually be hidden from assistive technologies.

## Cached, stale, and partial content

Cached or partial content can often improve perceived performance more than a full-screen loader.

Prefer showing available real content when it is safe and useful:

* cached profile while refreshing fresh profile;
* previously loaded feed while loading new posts;
* dashboard sections independently as they arrive;
* local drafts before remote sync completes;
* last known settings while checking server state.

Represent cached or stale content explicitly when freshness matters:

```swift
enum AccountSummaryState {
    case loading
    case showingCached(summary: AccountSummary, isRefreshing: Bool)
    case loaded(summary: AccountSummary)
    case failed(DisplayError)
}
```

Do not show stale content silently when it could mislead the user.

Be careful with:

* account balances;
* trading data;
* medical data;
* legal status;
* security settings;
* identity verification;
* permissions;
* server-authoritative business state.

When stale content is shown, consider communicating it:

```text
Showing last updated data.
```

or:

```text
Refreshing…
```

Only label content as stale when that distinction is meaningful to the user. Do not add noisy freshness labels to low-risk screens where they do not help.

For high-stakes or server-authoritative actions, read `references/high-stakes-actions.md`.

## Empty states

An empty state means loading completed but there is no content to show.

Do not show an empty state while data is still loading. Do not show a loading state after the app already knows the result is empty.

A useful empty state usually explains:

* what is missing;
* why it may be missing;
* whether the user can do anything about it;
* what action is available next.

Good empty states:

* “No saved articles yet” with an action to browse articles;
* “No transactions in this period” with a date filter action;
* “No search results” with a suggestion to change the query;
* “No downloads” with a call to start one.

Risky:

```swift
Text("Nothing here")
```

Prefer context and recovery:

```swift
ContentUnavailableView(
    "No saved articles",
    systemImage: "bookmark",
    description: Text("Articles you save will appear here.")
)
```

If the empty state depends on filters, mention that context:

```swift
ContentUnavailableView(
    "No results for this filter",
    systemImage: "line.3.horizontal.decrease.circle",
    description: Text("Try changing the date range or clearing filters.")
)
```

If `ContentUnavailableView` is not available for the app’s deployment target, use a custom empty-state view with equivalent title, description, image, and optional action.

## Error states

An error state should explain that loading failed and provide a realistic recovery path.

Avoid replacing all errors with generic copy.

Risky:

```swift
Text("Something went wrong")
```

Prefer actionable copy:

```swift
RetryView(
    title: "Could not load messages",
    message: "Check your connection and try again.",
    actionTitle: "Retry"
) {
    Task { await model.reload() }
}
```

For section-level content, prefer section-level errors when the rest of the screen remains useful:

```swift
switch state.recommendations {
case .loading:
    RecommendationPlaceholder()

case .loaded(let recommendations):
    RecommendationList(recommendations)

case .empty:
    EmptyRecommendationsView()

case .failed:
    InlineRetryView("Could not load recommendations") {
        Task { await model.reloadRecommendations() }
    }

case .idle:
    EmptyView()
}
```

Use a full-screen error only when the screen has no useful content without the failed data.

If stale content is still useful, keep it visible and show a smaller error or toast:

```text
Could not refresh. Showing the latest available content.
```

Do not expose raw backend messages, internal error codes, or sensitive implementation details directly to users.

## Refreshing existing content

Refreshing is different from initial loading.

Initial loading may need a placeholder, skeleton, cached content, or full-screen loading state. Refresh often should preserve existing content and show a smaller indicator.

Risky:

```swift
func refresh() async {
    state = .loading

    do {
        let items = try await service.loadItems()
        state = items.isEmpty ? .empty : .loaded(items)
    } catch {
        state = .failed(mapError(error))
    }
}
```

This clears useful content during refresh.

Prefer preserving existing content when safe:

```swift
enum FeedState {
    case loading
    case loaded(items: [FeedItem], isRefreshing: Bool)
    case empty
    case failed(DisplayError)
}

@MainActor
func refresh() async {
    guard case .loaded(let currentItems, _) = state else {
        await loadInitial()
        return
    }

    state = .loaded(items: currentItems, isRefreshing: true)

    do {
        let newItems = try await service.loadItems()
        state = newItems.isEmpty
            ? .empty
            : .loaded(items: newItems, isRefreshing: false)
    } catch {
        state = .loaded(items: currentItems, isRefreshing: false)
        errorPresenter.show("Could not refresh. Showing the latest available content.")
    }
}
```

Use stale content carefully when old content could mislead the user. For financial, legal, medical, security-sensitive, or server-authoritative data, read `references/high-stakes-actions.md`.

## Pagination and loading more

Loading the next page should not usually replace the whole screen with an initial loading state.

Risky:

```swift
func loadNextPage() async {
    state = .loading

    do {
        let page = try await service.loadNextPage()
        items.append(contentsOf: page.items)
        state = .loaded(items)
    } catch {
        state = .failed(mapError(error))
    }
}
```

This hides already loaded content even though only the next page is unavailable.

Prefer modeling pagination separately:

```swift
enum PageLoadState: Equatable {
    case idle
    case loadingNextPage
    case failedNextPage(DisplayError)
    case reachedEnd
}
```

Example:

```swift
struct FeedViewState: Equatable {
    var items: [FeedItem]
    var pageState: PageLoadState
    var isRefreshing: Bool
}
```

Render the next-page state near the end of the list:

```swift
List {
    ForEach(state.items) { item in
        FeedRow(item: item)
    }

    switch state.pageState {
    case .idle:
        EmptyView()

    case .loadingNextPage:
        ProgressView()
            .frame(maxWidth: .infinity)

    case .failedNextPage(let error):
        InlineRetryView(error.title) {
            Task { await model.loadNextPage() }
        }

    case .reachedEnd:
        EndOfListView()
    }
}
```

Review pagination for:

* loading-more footer;
* failed-page retry;
* reached-end state;
* duplicate next-page requests;
* old content disappearing during page load;
* multiple `onAppear` triggers from the last row;
* stale page responses arriving after filters or search query changed.

Do not use a full-screen loader for page append unless the existing content is invalid or unavailable.

## Transitions between states

Review transitions, not only final states.

Important transitions:

* idle → loading;
* loading → loaded;
* loading → empty;
* loading → failed;
* loaded → refreshing;
* refreshing → loaded;
* refreshing → failed while preserving old content;
* loaded → loading next page;
* loading next page → loaded with appended content;
* loading next page → failed next page;
* failed → retrying;
* empty → loading after user action;
* loaded → empty after filters change;
* cached → loaded fresh content;
* cached → failed refresh.

Common problems:

* flicker between loading and loaded;
* empty state flashes before loading begins;
* old content disappears during refresh;
* error replaces useful stale content;
* retry button starts work but gives no feedback;
* controls remain enabled during submission;
* multiple loading indicators compete for attention;
* layout jumps when placeholders are replaced;
* older async response overwrites newer state.

Prefer transitions that keep the user oriented.

For very short operations, avoid UI flicker. It can be better to keep the current state visible or delay showing a transient loading indicator briefly, but do this carefully and consistently. Do not add artificial delays to hide real performance problems.

## Anti-flicker policy

A loading indicator that appears and disappears too quickly can feel like a visual glitch.

For short operations, consider one of these approaches:

* keep the previous content visible while work completes;
* show no loader unless the operation exceeds a short threshold;
* use a small inline indicator instead of a full-screen loader;
* use a stable placeholder only when the screen would otherwise look broken.

Do not delay the real operation just to make a loader visible.

Acceptable:

```swift
Task {
    let spinnerTask = Task {
        try? await Task.sleep(for: .milliseconds(150))
        guard !Task.isCancelled else { return }
        await MainActor.run {
            state = .loading
        }
    }

    do {
        let result = try await service.load()
        spinnerTask.cancel()
        state = result.isEmpty ? .empty : .loaded(result)
    } catch {
        spinnerTask.cancel()
        state = .failed(mapError(error))
    }
}
```

This delays showing the spinner, not the data.

Use this carefully. A delayed spinner can be useful for tiny requests, but it should not hide meaningful waits, suppress needed feedback, or create inconsistent behavior across similar screens.

## Cancellation and stale responses

Loading states should represent the current request, not an older request that completed late.

Common cases:

* user changes search query quickly;
* user changes filters while a request is in flight;
* user leaves and re-enters the screen;
* retry starts while an older request is still running;
* pagination request completes after refresh replaced the feed;
* `.onAppear` starts duplicate work;
* `.task` restarts when its `id` changes.

Use cancellation or request identity to prevent stale responses from overwriting newer state.

Example with request identity:

```swift
@MainActor
final class SearchModel {
    private var currentRequestID = UUID()
    private(set) var state: LoadState<[SearchResult]> = .idle

    func search(query: String) async {
        let requestID = UUID()
        currentRequestID = requestID
        state = .loading

        do {
            let results = try await service.search(query: query)

            guard currentRequestID == requestID else { return }

            state = results.isEmpty
                ? .empty
                : .loaded(results)
        } catch {
            guard currentRequestID == requestID else { return }
            state = .failed(mapError(error))
        }
    }
}
```

When using SwiftUI `.task(id:)`, make the `id` represent the input that should restart the work:

```swift
SearchResultsView(query: query)
    .task(id: query) {
        await model.search(query: query)
    }
```

Do not let an old response show results for a query, filter, account, or screen that is no longer current.

Cancellation of local loading does not necessarily cancel a server-side operation. For high-stakes operations, read `references/high-stakes-actions.md`.

## Controls during pending work

Controls that trigger work should communicate pending state when repeated interaction would be confusing, expensive, or harmful.

Review:

* submit buttons;
* retry buttons;
* save buttons;
* search apply buttons;
* filter apply buttons;
* pagination triggers;
* destructive action buttons;
* upload or import controls.

Prefer replacing or disabling controls while work is in progress:

```swift
Button {
    Task { await model.save() }
} label: {
    if model.isSaving {
        ProgressView()
    } else {
        Text("Save")
    }
}
.disabled(model.isSaving)
```

Client-side disabling prevents accidental repeated taps, but it is not a complete safety mechanism for high-stakes or duplicate-sensitive operations.

For high-stakes actions, use explicit state, idempotency, and authoritative confirmation. Read `references/high-stakes-actions.md`.

## Loading copy

Loading copy should be clear, brief, and honest.

Prefer specific copy when the action is meaningful:

* “Loading messages…”
* “Uploading 3 files…”
* “Checking availability…”
* “Preparing preview…”
* “Refreshing feed…”
* “Loading more…”

Avoid vague or misleading copy:

* “Please wait…”
* “Almost done…” when progress is unknown;
* “Finishing up…” when the app has no estimate;
* “Success” before the authoritative system confirms success.

Do not over-explain routine loading. For short, obvious operations, a simple indicator may be enough.

For high-stakes actions, copy should make the status explicit:

* “Submitting transfer…”
* “Waiting for confirmation…”
* “Checking status…”

Only tell users not to close the screen if the product and backend flow actually require the screen to remain active. In many flows, the operation can continue on the server and the app should provide a safe way to recover or check status later.

Use high-stakes language only when the product actually requires it.

## Accessibility

Loading and state changes should be understandable with assistive technologies.

Check:

* Does the loading indicator have an accessible label when needed?
* Does the empty state explain itself through accessible text?
* Does a retry control have a clear accessible label?
* Does focus jump unexpectedly when content appears?
* Are brief state changes announced when the UI change is not otherwise obvious?
* Are skeletons hidden from accessibility if they do not represent real content?
* Does preserved stale content communicate that refresh is in progress?
* Does a disabled button communicate why it is unavailable when the reason is not obvious?
* Does a pagination footer announce loading or failure clearly?

For brief or non-obvious updates, accessibility announcements can be appropriate. Do not overuse announcements for every minor loading update; too many announcements can make the experience noisy.

## Implementation checklist

When reviewing loading states, check:

* [ ] Are loading, loaded, empty, failed, refreshing, cached, and loading-more states distinct when relevant?
* [ ] Is initial loading different from refreshing existing content?
* [ ] Does the UI avoid blank or static screens during meaningful waits?
* [ ] Is cached or partial content shown when safe and useful?
* [ ] Is stale content labeled or handled carefully when freshness matters?
* [ ] Is a progress indicator shown when the app might otherwise look stalled?
* [ ] Is determinate progress used only when progress is actually measurable?
* [ ] Are placeholders or skeletons used only when structure is predictable?
* [ ] Do placeholders reserve stable space and avoid layout jumps?
* [ ] Are skeletons clearly temporary and non-interactive?
* [ ] Does the empty state explain why content is unavailable?
* [ ] Does the empty state provide a useful next action when possible?
* [ ] Does the error state include a realistic recovery path?
* [ ] Are section-level errors used when the rest of the screen remains useful?
* [ ] Does refresh preserve existing content when safe?
* [ ] Does pagination show a next-page loading or retry state without replacing the whole screen?
* [ ] Are duplicate refresh, retry, submit, or pagination requests guarded?
* [ ] Are stale async responses prevented from overwriting newer state?
* [ ] Are high-stakes states explicit and authoritative-confirmed?
* [ ] Does loading copy avoid false precision or premature success?
* [ ] Are accessibility labels and announcements considered?
* [ ] Is the perceived improvement validated with runtime evidence when claimed?

## Validation

Validate loading states with evidence that reflects what the user sees.

Useful validation:

* screen recording from action to visible feedback;
* time to first visible loading state;
* time to first meaningful content;
* retry flow test;
* slow network testing;
* offline or failure simulation;
* repeated refresh attempts;
* pagination loading and failed-page retry;
* rapid query or filter changes;
* stale response test;
* app backgrounding or navigation away during loading;
* accessibility review with VoiceOver when state changes are dynamic;
* UI tests that verify state transitions.

Do not claim that loading states improved performance unless there is evidence.

Correct phrasing:

* “This separates loading, empty, and error states.”
* “This should make the wait visible instead of looking stalled.”
* “This preserves existing content during refresh.”
* “This prevents an old response from replacing a newer state.”
* “Validate by recording the transition from tap to first feedback.”

Avoid:

* “This makes loading faster.”
* “This fixes the performance problem.”
* “Users will definitely perceive this as faster.”
* “A skeleton always improves perceived performance.”
* “A spinner is enough for every loading state.”

For older-device testing, Low Power Mode, release builds, Instruments, production signals, and broader responsiveness validation, read `references/validation-and-testing.md`.

## Review output guidance

When using this reference, explain:

```markdown
## Finding

The screen does not clearly communicate loading, empty, failed, refreshing, cached, or pagination states.

## User impact

The user may see a blank screen, unclear wait, misleading empty state, disappearing content, stale data without context, or a missing recovery path.

## Recommended change

Introduce explicit loading, loaded, empty, failed, refreshing, cached, and loading-more states where relevant. Use placeholders, cached content, progress indicators, empty states, inline pagination loaders, or retry UI based on what the user needs to understand.

## Safety checks

Call out whether stale content is safe, whether duplicate requests are guarded, whether older async responses can overwrite newer state, and whether high-stakes flows require authoritative confirmation.

## Validation

Record the state transition and verify that the user receives visible feedback before the operation completes. Test slow network, failure, retry, refresh, pagination, and stale-response scenarios.
```
