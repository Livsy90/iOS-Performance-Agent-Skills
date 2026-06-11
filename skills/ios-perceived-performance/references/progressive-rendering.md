# Progressive Rendering

Use this reference when reviewing screens or flows that wait for all data before showing anything, delay primary content behind secondary content, clear existing content during refresh, or update the UI in one large final step.

This reference is about revealing real content in stages. For placeholders, skeletons, progress indicators, loading copy, and loading/error transitions, read `references/loading-states.md`. For optimistic UI, read `references/optimistic-updates.md`. For high-stakes or irreversible flows, read `references/high-stakes-actions.md`. For broader validation, read `references/validation-and-testing.md`.

## Contents

- Core model
- Agent capability boundary
- When it helps
- When it is risky
- Review workflow
- State model requirements
- Critical content first
- Section-level errors
- Stale while refreshing
- Partial UI updates
- Stable layout
- Duplicate loads and cancellation
- Validation
- Review output guidance

## Core model

Progressive rendering means presenting useful UI in stages instead of waiting for the entire screen to be ready.

It can improve perceived performance because users see structure, feedback, or meaningful content earlier. It does not necessarily reduce total execution time. Treat it as a product and state-modeling technique, not as a low-level optimization.

The goal is to reduce the time until the user can understand the screen or start interacting with the most important content.

Good progressive rendering separates primary content from secondary content, first load from refresh, section-level loading from full-screen loading, section-level failure from full-screen failure, and useful stale content from dangerous stale content.

## Agent capability boundary

The agent can inspect and improve:

- all-or-nothing loading states;
- screens where primary content could appear before secondary content;
- code that waits for unrelated async operations before updating UI;
- refresh flows that clear useful existing content;
- state models that cannot represent partial content;
- missing section-level loading, empty, or error states;
- UI updates that happen only after every dependency completes.

The agent cannot prove that the screen feels faster without runtime evidence.

Avoid unsupported claims:

- “This will definitely feel faster.”
- “This fixes the performance issue.”
- “The screen is now responsive.”

Prefer precise language:

- “This should reduce blank-screen time.”
- “This lets primary content appear before secondary content.”
- “Validate with a screen recording or tap-to-first-content timing.”
- “This changes perceived latency, not necessarily total load time.”

## When it helps

Progressive rendering is useful when:

- the screen has independent sections;
- primary content is more important than secondary content;
- some data is available earlier than the rest;
- cached or stale content is useful during refresh;
- the user can start reading or interacting before all sections are complete;
- the current implementation shows a blank screen for too long;
- slow secondary content blocks the whole screen;
- partial failure can be shown inline without collapsing the whole flow.

Common examples: profile header before recommendations, account summary before transaction history, article body before comments, search results before filters, cached feed during refresh, product details before related products.

## When it is risky

Do not recommend progressive rendering blindly.

It may be wrong when:

- partial content would mislead the user;
- all data must be consistent at the same point in time;
- the product requires an atomic result;
- the screen is small and already loads quickly;
- staged rendering would cause layout jumps;
- partial errors would be harder to explain than a single error state;
- ordering is essential;
- the result must be server-confirmed before it is meaningful.

For financial, legal, destructive, irreversible, security-sensitive, or trust-sensitive flows, read `references/high-stakes-actions.md`.

## Review workflow

Before proposing progressive rendering, answer:

1. What is the primary content?
2. What can safely appear later?
3. Which sections are independent?
4. Which failures should block the whole screen?
5. Which failures can be shown inline?
6. Can existing content remain visible during refresh?
7. Is stale content safe enough to show?
8. Would staged updates cause layout jumps?
9. Does the state model support partial content?
10. Can the improvement be validated with tap-to-first-content timing or screen recording?

If these questions cannot be answered from code, mark the missing product or design decision explicitly.

## State model requirements

Progressive rendering usually requires a state model that can represent section-level progress.

Avoid a single loaded state that waits for every dependency:

```swift
enum ScreenState {
    case loading
    case loaded(Content)
    case failed(Error)
}
```

Prefer section-level state when sections can load, fail, or refresh independently:

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

This lets the UI render the screen shell and update sections independently.

Do not introduce section-level state just to make the model look more sophisticated. Use it when it changes what the user can see or do earlier.

For deeper loading-state modeling, empty states, placeholders, and loading copy, read `references/loading-states.md`.

## Critical content first

Identify what makes the screen meaningful.

Ask:

- What can the user read or act on first?
- Which content defines the screen?
- Which sections are secondary or decorative?
- Which data can arrive later without confusing the user?
- Which failures should block the whole screen?
- Which failures can be shown inline?

Prefer loading primary content before secondary content when the sections are independent.

Example decisions:

- Product details should not wait for related products.
- Article body should not wait for comments.
- Account balance may need stronger freshness and consistency rules than marketing banners.

Use this pattern only when primary and secondary sections are independent enough to load and fail separately.

## Section-level errors

A progressive screen needs section-level failure boundaries.

A risky pattern is loading everything into one `loaded` state and converting any error into one full-screen failure. That lets a secondary failure collapse the whole screen.

Use full-screen failure when the primary content cannot load or the screen has no useful partial state.

Use inline failure when:

- the failed section is secondary;
- the rest of the screen is still useful;
- retry can be scoped to that section;
- the failure can be explained without confusing the user.

Read `references/loading-states.md` for error-state presentation, empty states, retry copy, and loading/error transitions.

## Stale while refreshing

Refreshing should not always clear existing content.

If the user already has useful content, replacing it with a blank loading state can make the screen feel unstable or slower.

Prefer preserving existing content during refresh when it is safe. For example, keep loaded items visible and represent refresh separately, such as `loaded(items: [ListItem], isRefreshing: Bool)`.

This keeps the screen useful while communicating that refresh is happening.

Use stale content carefully when:

- old content may be dangerous or misleading;
- data is financial, medical, legal, or security-sensitive;
- the user needs server-confirmed freshness;
- stale content should be visually marked;
- actions on stale content could cause mistakes.

For broader loading, refreshing, and error-state transitions, read `references/loading-states.md`.

## Partial UI updates

Avoid delaying the whole screen behind slow secondary work.

The problem can exist even with parallel requests if the UI waits for all results before updating.

Prefer section-level updates when product rules allow partial content:

- mark expected sections as loading;
- update primary sections as soon as their data is ready;
- keep secondary sections loading independently;
- show empty or failed state per section when appropriate;
- avoid collapsing the whole screen because of secondary failure.

Use `async let` when the set of section loads is small, fixed, and belongs to one async scope. Use a task group when the number of section loads is dynamic.

Keep detailed task-lifetime, actor, and structured-concurrency guidance in the concurrency skill or related references.

## Stable layout

Progressive rendering can feel worse if every section changes size after loading.

Review whether the UI reserves stable space for expected sections.

Risky pattern:

```swift
if let banner {
    BannerView(banner)
}
```

If the banner appears late and pushes content down, the screen may jump.

Prefer a stable region when the section is expected. The exact UI depends on design. The principle is stable layout during staged updates.

For skeletons, placeholders, loading copy, and empty-state presentation, read `references/loading-states.md`.

## Duplicate loads and cancellation

Progressive rendering often creates more independent loading paths. Make sure this does not create duplicate work or abandoned tasks.

Check:

- Can the same section load be started twice?
- What happens when the screen disappears?
- What happens when refresh starts while initial loading is still running?
- Are section loaders idempotent or protected from duplicate requests?
- Is cancellation handled at the owner boundary?
- Does the UI ignore results from outdated requests?

Do not use unstructured `Task {}` merely to make section loading parallel. Prefer structured concurrency when all child work belongs to one async scope.

Keep this reference focused on how task lifetime affects staged UI updates. For deeper task-lifetime and structured-concurrency guidance, use the concurrency skill.

## Validation

Validate progressive rendering with evidence that reflects user perception:

- screen recording from tap to first meaningful content;
- time to first meaningful content;
- time until primary action becomes available;
- repeated refresh attempts;
- slow-network testing;
- inspection for layout jumps during staged updates.

Do not claim that progressive rendering improved performance unless there is evidence.

Correct phrasing:

- “This should reduce blank-screen time.”
- “This lets primary content appear before secondary content.”
- “Validate by measuring tap-to-first-content and checking for layout jumps.”

Avoid:

- “This makes the screen faster.”
- “This fixes performance.”
- “This guarantees better UX.”

For older-device testing, Low Power Mode, release builds, Instruments, production signals, and broader responsiveness validation, read `references/validation-and-testing.md`.

## Review output guidance

When using this reference, explain:

- the screen or flow that waits for too much work before showing useful UI;
- what the user sees: blank screen, static state, delayed primary content, or layout jump;
- the smallest useful change: section-level state, critical content first, stale-while-refreshing, or inline section errors;
- the validation path: tap-to-first-content timing, screen recording, repeated refresh attempts, slow-network testing, or layout-jump inspection.
