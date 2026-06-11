---
name: ios-perceived-performance
description: Use this skill for product-level iOS responsiveness and loading/feedback flows, including perceived latency, time to first feedback, progressive rendering, loading states, skeletons, placeholders, optimistic updates, rollback behavior, high-stakes actions, UI continuity, and responsiveness validation. Do not use it for low-level CPU, memory, rendering, Swift Concurrency, or launch-performance diagnostics unless the task is about user-perceived speed.
---

# iOS Perceived Performance

## Purpose

Use this skill to review whether an iOS screen or flow feels responsive to users, even when raw execution time does not change.

This skill focuses on user-visible feedback, loading behavior, staged content, continuity, optimistic UI, high-stakes actions, and validation of perceived responsiveness.

## When to use this skill

Use this skill when the task involves:

- a screen that feels slow, stuck, blank, jumpy, or unresponsive;
- no visible feedback after a tap or gesture;
- delayed first meaningful content;
- loading states, placeholders, skeletons, empty states, retry states, or refresh states;
- progressive rendering, partial content, section-level loading, or stale-while-refreshing UI;
- optimistic updates, pending state, rollback, retries, or local/server reconciliation;
- high-stakes, irreversible, destructive, financial, legal, medical, identity, or security-sensitive actions;
- duplicate submissions or repeated taps during async work;
- perceived latency trade-offs where the product behavior matters as much as the raw duration;
- validation using recordings, UI tests, Instruments, MetricKit, production logs, or user-visible signals.

## When not to use this skill

Do not use this skill for:

- low-level CPU, allocation, ARC, memory, runtime, or compiler-performance investigations;
- SwiftUI invalidation, identity, layout, or scrolling problems unless the question is about perceived responsiveness;
- Swift Concurrency internals unless task behavior affects visible UI feedback or staged loading;
- app launch performance unless the question is about first useful screen or first interaction from the user's point of view;
- visual redesign requests with no responsiveness, loading, or feedback concern;
- claims that require running the app when no runtime evidence is available.

Use `ios-performance-profiling`, `swiftui-performance`, `swift-concurrency-performance`, `swift-runtime-performance`, or `ios-launch-performance` when those domains are the primary problem.

## Core model

Users do not experience CPU samples, network traces, database timings, or async task graphs directly.

They experience:

- whether the app reacts immediately;
- when useful content first appears;
- whether progress is clear;
- whether the screen stays stable while work continues;
- whether failures and retries are understandable;
- whether pending work feels trustworthy;
- whether actions can be repeated accidentally;
- whether the UI tells the truth about the server-authoritative outcome.

Start with the user-visible delay or uncertainty before proposing lower-level optimization.

## Agent capability boundaries

The agent can usually inspect and improve:

- state models for loading, loaded, empty, error, refreshing, pending, syncing, and failed states;
- whether UI changes happen only after async work completes;
- whether user actions receive immediate visual feedback;
- whether refresh clears useful existing content unnecessarily;
- whether a screen can render critical content before secondary content;
- whether optimistic updates have pending, rollback, retry, and conflict behavior;
- whether high-stakes actions wait for confirmation before showing final success;
- whether duplicate submissions are prevented.

The agent can suggest, but cannot prove without runtime evidence:

- whether a screen feels fast enough;
- whether a skeleton improves perception;
- whether a deliberate delay is appropriate;
- whether Low Power Mode exposes low performance headroom;
- whether older devices perform acceptably;
- whether a product flow feels trustworthy to users.

Do not claim that manual or device-based validation was performed unless the user supplied evidence such as a recording, trace, test result, profiling output, or production signal.

## Review workflow

1. Identify the user-visible symptom: no feedback, blank screen, all-or-nothing loading, jumpy update, unclear progress, unsafe optimism, repeated submission, or distrust.
2. Locate the state transition that creates the symptom.
3. Decide whether the first useful improvement is product/UI feedback or low-level performance work.
4. Check whether the UI acknowledges user actions immediately.
5. Check whether critical content can appear before secondary content.
6. Check whether refresh can preserve useful existing content.
7. Check whether loading, empty, error, retry, refreshing, pending, synced, and failed states are distinct enough.
8. Decide whether optimistic UI is safe, reversible, and honest.
9. For high-stakes actions, prefer explicit progress and server-confirmed success over optimistic final states.
10. Propose the smallest change that improves user-visible responsiveness.
11. State what can be determined from code and what needs runtime validation.
12. Recommend validation using evidence that reflects user experience.

## Decision rules

- If the user sees no response after an action, add immediate feedback before optimizing the operation itself.
- If the screen is blank while work is happening, consider cached content, placeholders, skeletons, or progressive rendering.
- If the screen waits for unrelated data before showing anything, consider section-level loading or staged rendering.
- If refresh removes useful content, preserve stale content when it is safe and mark it as refreshing.
- If loading, empty, error, and retry states collapse into one state, separate them before tuning performance.
- If an action is low-risk and reversible, optimistic UI may improve perceived latency.
- If an action is financial, destructive, legal, medical, identity-related, security-sensitive, irreversible, or server-authoritative, do not show final success before confirmation.
- If duplicate submissions are possible, add pending state, disabled controls, idempotency, or request tracking.
- If a recommendation depends on product trust or user perception, describe it as a product hypothesis unless evidence exists.
- If a performance claim cannot be proven from code, include a validation plan instead of claiming success.

## Gotchas

- Do not treat skeletons or placeholders as actual execution-time improvements.
- Do not hide errors behind indefinite loading states.
- Do not recommend optimistic success for high-stakes or irreversible actions.
- Do not suggest artificial delays as a default performance fix.
- Do not clear content during refresh unless stale content is unsafe or misleading.
- Do not split every screen into independent loading sections if partial content would confuse the user.
- Do not claim Low Power Mode simulates an older device. Use it only as a rough manual stress signal.
- Do not say “this feels faster” without evidence from a recording, trace, metric, test, or user-visible behavior.
- Do not make broad low-level optimization recommendations when the immediate problem is missing feedback or unclear state.

## Evidence and validation

Prefer evidence that reflects what the user experiences:

- screen recording from action to first feedback and first meaningful content;
- tap-to-first-feedback timing;
- time to first meaningful content;
- time until primary interaction becomes available;
- repeated refresh attempts;
- slow-network testing;
- UI tests that capture state transitions;
- Instruments traces for hangs, hitches, layout, rendering, and main-thread work;
- MetricKit or Xcode Organizer responsiveness data;
- production logs around loading stages;
- user reports describing stuck, blank, jumpy, or misleading UI.

Use careful language when evidence is incomplete:

- “This should reduce blank-screen time.”
- “This gives the user earlier feedback.”
- “This needs validation on a release build.”
- “Validate with a screen recording, Instruments trace, UI test, or production signal.”

Avoid unsupported claims:

- “This fixes performance.”
- “This is now smooth.”
- “This will feel faster to users.”
- “This works well on older devices.”

## Reference routing

Read these only when relevant:

- `references/progressive-rendering.md` — read when a screen waits for all data before showing useful UI, can show critical content first, needs section-level loading, or clears existing content during refresh.
- `references/loading-states.md` — read when the task involves loading, empty, error, retry, refreshing, skeleton, placeholder, or progress states.
- `references/optimistic-updates.md` — read when the task involves optimistic UI, pending state, rollback, retries, conflict handling, local reconciliation, or server sync.
- `references/high-stakes-actions.md` — read when the action is financial, legal, destructive, irreversible, medical, identity-related, security-sensitive, or server-authoritative.
- `references/validation-and-testing.md` — read when the task asks how to validate perceived responsiveness, older-device behavior, Low Power Mode signals, release-build behavior, screen recordings, Instruments traces, MetricKit, or production evidence.

## Output expectations

When reviewing a screen or flow, respond with:

```markdown
## Finding

Describe the perceived performance issue.

## User impact

Explain what the user sees or feels: no feedback, blank screen, jumpy updates, delayed interaction, unclear progress, unsafe optimism, duplicate submission risk, or premature success.

## Evidence

Point to code, state model, flow behavior, trace, recording, metric, user report, or missing UI state.

## Recommended change

Suggest the smallest useful product/UI change first: immediate feedback, distinct loading state, progressive rendering, stale-while-refreshing content, optimistic update with rollback, explicit confirmation, duplicate-submission prevention, or validation.

## Implementation notes

Explain what can be changed in code and what depends on product, design, backend, or runtime evidence.

## Validation

State how to verify the improvement. Do not claim manual, device, or production validation was performed unless evidence is provided.
```

Prefer user-visible improvements over low-level optimization when the bottleneck is missing feedback, all-or-nothing rendering, unclear loading, unsafe optimistic behavior, or poor continuity.
