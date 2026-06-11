# Progressive Rendering

Use this reference when reviewing iOS screens or flows that wait for all data before showing useful UI, delay primary content behind secondary content, clear existing content during refresh, or update the UI only after every dependency finishes.

This reference is about revealing **real content** in stages. For placeholders, skeletons, progress indicators, loading copy, and loading/error transitions, read `references/loading-states.md`. For optimistic UI and rollback, read `references/optimistic-updates.md`. For high-stakes, irreversible, financial, legal, security-sensitive, or trust-sensitive flows, read `references/high-stakes-actions.md`. For runtime validation, read `references/validation-and-testing.md`.

## Contents

- [Scope boundary](#scope-boundary)
- [Core model](#core-model)
- [What the agent can and cannot prove](#what-the-agent-can-and-cannot-prove)
- [When progressive rendering helps](#when-progressive-rendering-helps)
- [When progressive rendering is risky](#when-progressive-rendering-is-risky)
- [Review procedure](#review-procedure)
- [State model requirements](#state-model-requirements)
- [Common patterns](#common-patterns)
  - [Avoid all-or-nothing loading](#avoid-all-or-nothing-loading)
  - [Render critical content first](#render-critical-content-first)
  - [Use section-level failure boundaries](#use-section-level-failure-boundaries)
  - [Preserve stale content during refresh](#preserve-stale-content-during-refresh)
  - [Keep layout stable during staged updates](#keep-layout-stable-during-staged-updates)
  - [Avoid duplicate loads and abandoned work](#avoid-duplicate-loads-and-abandoned-work)
- [Decision rules](#decision-rules)
- [Implementation checklist](#implementation-checklist)
- [Validation](#validation)
- [Review output guidance](#review-output-guidance)

## Scope boundary

Progressive rendering is a perceived-performance and product-state modeling technique.

Use it to reduce the time until the user sees meaningful structure, primary content, or an actionable part of the screen.

Do not present it as a low-level optimization. It may reduce blank-screen time without reducing total execution time, CPU work, memory use, network latency, or backend latency.

Keep detailed loading-copy, skeleton, placeholder, empty-state, and error-copy guidance in `references/loading-states.md`. Keep detailed structured-concurrency guidance in the concurrency skill or related references.

## Core model

Progressive rendering means presenting useful UI in stages instead of waiting for the entire screen to be ready.

The core question is:

> Can the user understand or use the screen before every dependency has finished?

A good progressive-rendering refactor usually changes one or more of these:

- one full-screen state becomes section-level state;
- primary content renders before secondary content;
- refresh preserves useful existing content instead of clearing the screen;
- partial failures are shown inline instead of collapsing the whole flow;
- layout reserves stable regions so late sections do not cause jumps.

The goal is not to show random pieces earlier. The goal is to show the **right** pieces earlier.

## What the agent can and cannot prove

The agent can inspect all-or-nothing loading, unrelated async dependencies that block UI updates, refresh flows that clear useful content, state models that cannot represent partial content, staged updates that may cause layout jumps, and duplicate loads caused by independent section loaders.

The agent cannot prove that the screen feels faster without runtime evidence.

Prefer careful language:

- “This should reduce blank-screen time.”
- “This lets primary content appear before secondary content.”
- “This changes perceived latency, not necessarily total load time.”
- “Validate with tap-to-first-content timing, screen recording, or a trace.”

Avoid unsupported language:

- “This fixes performance.”
- “This is now smooth.”
- “This will definitely feel faster.”

## When progressive rendering helps

Progressive rendering is useful when:

- the screen has independent sections;
- primary content is more important than secondary content;
- some data is available earlier than the rest;
- cached or stale content is useful during refresh;
- the user can read or act before all sections are complete;
- slow secondary content currently blocks the whole screen;
- partial failure can be shown inline without collapsing the flow.

Good candidates: profile header before recommendations, account summary before transaction history, article body before comments, search results before suggestions, cached feed while refreshing, product details before related products, or local draft content before sync status.

## When progressive rendering is risky

Do not recommend progressive rendering blindly.

It may be wrong when:

- partial content would mislead the user;
- all data must be consistent at the same point in time;
- the product requires an atomic result;
- staged rendering would cause visible layout jumps;
- partial errors would be harder to explain than one clear error;
- ordering is essential for comprehension or correctness;
- the result must be server-confirmed before it is meaningful.

For financial, legal, destructive, irreversible, security-sensitive, or trust-sensitive flows, route to `references/high-stakes-actions.md` before suggesting staged rendering.

## Review procedure

Before proposing progressive rendering, answer:

1. What is the primary content?
2. What can safely appear later?
3. Which sections are independent?
4. Which dependencies are unrelated to first useful content?
5. Which failures should block the whole screen?
6. Which failures can be shown inline?
7. Can existing content remain visible during refresh?
8. Is stale content safe enough to show?
9. Would staged updates cause layout jumps?
10. Does the state model support partial content?
11. Can duplicate section loads happen?
12. How will the improvement be validated?

If the code does not answer a product or design question, say so explicitly. Do not invent safety rules for the product.

## State model requirements

A progressive screen usually needs state that can represent important sections independently.

Prefer section-level state when sections can load, fail, or refresh separately:

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
    case failed(Error)
}
```

This model is useful only if product rules allow sections to be independent. Avoid it when the screen must be internally consistent as one atomic snapshot.

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
async let header: Void = loadHeader()
async let activity: Void = loadActivity()
async let recommendations: Void = loadRecommendations()

_ = await (header, activity, recommendations)

private func loadHeader() async {
    do {
        state.header = .loaded(try await service.loadHeader())
    } catch {
        state.header = .failed(error)
    }
}
```

The important change is not merely using `async let`. The important change is allowing UI state to update by section.

### Render critical content first

Identify the minimum content that makes the screen meaningful.

Ask what tells the user they are in the right place, what content they can read first, which action can become available first, which sections are secondary, and which secondary failures should be inline.

Do not block first useful content behind comments, recommendations, promotions, badges, remote decorations, or analytics enrichment unless the product requires them.

### Use section-level failure boundaries

Use full-screen failure when primary content cannot load or when the screen has no useful partial state.

For secondary content, prefer inline failure when product rules allow it:

```swift
private func loadRecommendations() async {
    do {
        let value = try await service.loadRecommendations()
        state.recommendations = value.isEmpty ? .empty : .loaded(value)
    } catch {
        state.recommendations = .failed(error)
    }
}
```

A secondary failure should not collapse the whole screen unless that section is required for correctness, safety, or trust.

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
state = .loaded(items: oldItems, isRefreshing: true)

do {
    state = .loaded(items: try await service.loadItems(), isRefreshing: false)
} catch {
    state = .loaded(items: oldItems, isRefreshing: false)
    errorPresenter.showRefreshFailed()
}
```

Use stale content carefully when old data may be misleading, sensitive, unsafe, or must be visually marked as stale.

### Keep layout stable during staged updates

Progressive rendering can feel worse if late sections change size and push content around.

Check whether expected sections reserve stable space, late banners or cards can push primary content unexpectedly, placeholders roughly match final size, and refresh preserves scroll position when possible.

For skeletons, placeholders, loading copy, and empty-state presentation, read `references/loading-states.md`.

### Avoid duplicate loads and abandoned work

Progressive rendering often creates more independent loading paths. Make sure this does not create duplicate work or abandoned tasks.

Check whether the same section can load twice, what happens when the screen disappears, what happens when refresh starts during initial loading, whether owner-scoped tasks are cancelled, and whether loaders are idempotent.

Use `async let` when the set of section loads is small, fixed, and belongs to one async scope. Use a task group when the number of sections is dynamic. Use stored `Task {}` only when work is intentionally owned by the screen or model lifetime and must be cancelled externally.

## Decision rules

- Prefer progressive rendering when it reduces blank-screen time without misleading the user.
- Prefer section-level state only when sections can load, fail, or refresh independently.
- Preserve stale content during refresh only when stale content is safe and clearly represented.
- Use full-screen failure for primary-content failure or atomic flows.
- Use inline failure for secondary sections when partial content is still useful.
- Keep staged layout stable; do not trade a blank screen for a jumpy screen.
- Do not recommend staged rendering for high-stakes flows until safety and confirmation requirements are clear.
- Do not claim success without validation evidence.

## Implementation checklist

Before finalizing a recommendation, check:

- [ ] The primary content is identified.
- [ ] Secondary content is separated from first useful content.
- [ ] The state model can represent section-level loading.
- [ ] Partial failures do not collapse useful content unnecessarily.
- [ ] Refresh does not clear existing content unless required.
- [ ] Stale content is safe or visually marked.
- [ ] Layout remains stable as sections update.
- [ ] Duplicate loads are avoided.
- [ ] Task lifetime and cancellation are considered.
- [ ] The recommendation includes a realistic validation step.

## Validation

Validate progressive rendering with evidence that reflects user perception:

- screen recording from tap to first meaningful content;
- tap-to-first-content timing;
- time until the primary action becomes available;
- slow-network testing;
- repeated refresh attempts;
- inspection for layout jumps during staged updates;
- Instruments when staged rendering still hitches or blocks the main thread;
- production signals when available.

Use precise phrasing: “This should reduce blank-screen time”, “This lets primary content appear before secondary content”, or “Validate by measuring tap-to-first-content and checking for layout jumps.”

Avoid: “This makes the screen faster”, “This fixes performance”, or “This guarantees better UX.”

For older-device testing, Low Power Mode, release builds, Instruments, production signals, and broader responsiveness validation, read `references/validation-and-testing.md`.

## Review output guidance

When using this reference, explain:

```markdown
## Finding
The screen waits for too much work before showing useful UI.

## User impact
The user sees a blank or static state even though primary content could appear earlier.

## Recommended change
Introduce section-level state, render critical content first, preserve stale content during refresh when safe, and show inline loading/error states for secondary sections.

## Trade-offs
Call out consistency, stale-data, layout-stability, cancellation, and product-safety risks.

## Validation
Measure tap-to-first-content and inspect a screen recording for layout jumps or confusing transitions.
```
