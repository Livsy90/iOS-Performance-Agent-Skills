# Validation and Testing

Use this reference when reviewing how to verify perceived-performance claims for iOS screens and user flows.

This reference is about validation discipline. It helps the agent separate code-level reasoning, implemented instrumentation, local runtime evidence, automated tests, profiling evidence, production evidence, and manual UX judgment.

For staged real-content rendering, read `references/progressive-rendering.md`. For loading states, read `references/loading-states.md`. For optimistic UI, read `references/optimistic-updates.md`. For high-stakes flows, read `references/high-stakes-actions.md`.

## Contents

- [Core rule](#core-rule)
- [Agent capability boundaries](#agent-capability-boundaries)
- [Evidence ladder](#evidence-ladder)
- [Choosing validation methods](#choosing-validation-methods)
- [Manual validation](#manual-validation)
- [Screen recordings](#screen-recordings)
- [Release-like builds](#release-like-builds)
- [Older devices](#older-devices)
- [Low Power Mode](#low-power-mode)
- [Scrolling and animation checks](#scrolling-and-animation-checks)
- [Automated tests](#automated-tests)
- [Instruments](#instruments)
- [Production signals](#production-signals)
- [Custom instrumentation](#custom-instrumentation)
- [Validation by pattern](#validation-by-pattern)
- [Review checklist](#review-checklist)
- [Review output guidance](#review-output-guidance)

## Core rule

Do not present a perceived-performance recommendation as verified unless there is evidence.

A code change can be plausible without being proven. A screen can have better state modeling without being objectively smoother. A progressive-rendering refactor can reduce blank-screen time without reducing total loading time.

Use precise language:

- “This should reduce blank-screen time.”
- “This gives the user earlier feedback.”
- “This needs validation on a release build.”
- “This should be checked with a screen recording, Instruments trace, or production signal.”

Avoid unsupported claims:

- “This fixes performance.”
- “This is now smooth.”
- “This will feel faster to users.”
- “This works well on older devices.”
- “Low Power Mode proves old-device performance.”

## Agent capability boundaries

The agent can usually inspect and improve:

- missing loading, empty, failed, refreshing, pending, and confirmed states;
- all-or-nothing rendering;
- UI that waits for unrelated async work before updating;
- optimistic UI without rollback;
- high-stakes flows that show success before confirmation;
- duplicate-submission guards, retry states, and recovery states;
- missing instrumentation points;
- missing tests when the repository is available.

The agent can analyze user-provided evidence such as screen recordings, trace summaries, logs with timestamps, XCTest/UI test reports, CI performance reports, MetricKit summaries, Xcode Organizer hangs or hitches, before/after measurements, and reproducible user reports.

The agent cannot directly prove actual device smoothness, older-device behavior, Low Power Mode behavior, thermal behavior, release-build responsiveness, hitch frequency, production rates, or actual user perception unless it can run the app in the environment or the user provides evidence.

Do not claim that manual, device, production, or profiling validation was performed unless the agent actually ran it in the available environment or the user provided the result.

## Evidence ladder

Use the strongest available evidence, but be honest about what each level can prove.


Prefer before/after comparisons when possible. For perceived performance, include at least one user-visible signal such as tap-to-first-feedback, blank-screen duration, first meaningful content, or visual stability.

## Choosing validation methods

Choose validation based on the claim. Do not recommend every tool for every change.


## Manual validation

Manual validation is appropriate when perceived performance depends on what the user sees and feels.

Use it for tap-to-first-feedback, time to first meaningful content, loading-state clarity, skeleton or placeholder behavior, layout stability, scroll smoothness, animation continuity, high-stakes confirmation clarity, and optimistic update failure behavior.

Suggested scenarios:

- first load, refresh with existing content, slow network, failed request, and retry after failure;
- repeated taps, screen disappearance during work, app backgrounding, and returning after navigation;
- long scrolling sessions and animation-heavy transitions.

The agent can suggest these scenarios. It should not claim they passed unless evidence is provided.

## Screen recordings

Screen recordings are useful because perceived performance is visual and temporal.

Use recordings to inspect tap-to-first-feedback, blank-screen duration, first meaningful content, layout jumps, flicker between states, stale content during refresh, loading indicator timing, optimistic update feedback, rollback or failure transition, duplicate-submission behavior, and animation hitches visible to the eye.

Suggested review method:

1. 1Start recording before the user action.
2. 2Perform the action once under normal conditions.
3. 3Repeat under slow network or constrained conditions when relevant.
4. 4Compare before and after changes.
5. 5Note timestamps for first feedback, first meaningful content, and final state.

Correct phrasing:

- “The recording shows feedback appearing immediately after tap.”
- “The screen remains blank until all sections load.”
- “The layout shifts when the secondary section appears.”
- “The rollback state is visible after simulated failure.”

Avoid:

- “Users will definitely perceive this as faster.”
- “This proves all devices are smooth.”
- “This fixes performance globally.”

## Release-like builds

Prefer release or release-like builds for performance validation.

Debug builds are useful for development, but they are not reliable proof of user-facing performance. They may include different compiler optimization levels, additional assertions, debug logging, overlays, and non-production behavior.

Use release-like builds when validating scrolling, animations, launch or screen transition timing, rendering-heavy screens, CPU-heavy transformations, memory pressure, perceived responsiveness, and repeated navigation flows.

If the repository is available, the agent can check whether performance tests or CI jobs run in a release-like configuration.

## Older devices

Older-device testing is valuable because real devices differ in CPU, GPU, memory bandwidth, thermal behavior, refresh characteristics, OS versions, and storage behavior.

Use older-device testing for scroll-heavy feeds, media-heavy screens, animation-heavy flows, large SwiftUI or UIKit hierarchies, image decoding or resizing, expensive layout or compositing, startup and first-screen rendering, and low memory headroom.

The agent cannot simulate an older device from code inspection. It can recommend older-device validation and explain which flows should be tested.

Do not claim that simulator results or a modern device prove older-device performance.

## Low Power Mode

Low Power Mode can be suggested as a cheap manual stress signal for render-heavy screens.

It may make limited performance headroom easier to notice, but it is not an older-device simulator. It does not reproduce older GPU architecture, memory bandwidth, display behavior, OS version, storage behavior, thermal state, or device-specific bottlenecks.

Use Low Power Mode as a signal for frame drops, animation hitches, delayed tap response, expensive layout, image decoding pressure, compositing and transparency cost, and render-heavy lists or feeds.

Correct phrasing:

- “Try the screen with Low Power Mode enabled as a manual stress signal.”
- “If the screen starts hitching under Low Power Mode, inspect layout, rendering, decoding, or compositing cost.”
- “Low Power Mode does not replace older-device testing.”

Avoid:

- “Low Power Mode proves old-device performance.”
- “The screen passes performance testing because it works in Low Power Mode.”
- “Low Power Mode is equivalent to a low-end device.”

## Scrolling and animation checks

Scrolling and animation issues are often perceived as performance problems even when data loading is fast.

Validate first scroll after content appears, long continuous scrolling, rapid scroll direction changes, pull-to-refresh, navigation push/pop, modal presentation/dismissal, expanding sections, animated state transitions, image-heavy cells, skeleton-to-content transitions, and error-to-retry-to-loaded transitions.

Look for dropped frames, visible hitching, delayed interaction, layout jumps, image pop-in, repeated re-layout, content offset jumps, animation stutter, or sudden main-thread stalls.

The agent can suggest these checks and can analyze recordings or traces if provided. It should not claim scrolling or animation is smooth without runtime evidence.

## Automated tests

Automated tests can help catch regressions when the same flow can be repeated.

Good candidates include launch to first screen, tap to first loading state, tap to first meaningful content, refresh flow, search result rendering, scroll through a long list, navigation into and out of a heavy screen, retry after failure, optimistic update state transition, and duplicate tap prevention.

A UI test can verify state transitions even when it does not measure performance:

```swift
func testLoadingStateAppearsBeforeContent() {
    let app = XCUIApplication()
    app.launch()

    app.buttons["Load"].tap()

    XCTAssertTrue(app.staticTexts["Loading"].waitForExistence(timeout: 1))
    XCTAssertTrue(app.staticTexts["Content"].waitForExistence(timeout: 5))
}
```

A performance test can measure repeatable work when the project supports it:

```swift
func testFeedScrollPerformance() {
    let app = XCUIApplication()
    app.launch()

    let feed = app.collectionViews.firstMatch
    XCTAssertTrue(feed.waitForExistence(timeout: 5))

    measure {
        feed.swipeUp()
        feed.swipeDown()
    }
}
```

Use performance tests for regression detection, not as the only proof of perceived quality. State limitations explicitly: simulator results may not match device behavior, CI hardware may vary, debug builds can distort performance, tests may not cover real data size, and tests may miss visual quality issues.

## Instruments

Use Instruments when there is a runtime symptom that code review cannot prove or localize.

Useful when investigating UI hangs, delayed tap response, animation hitches, scroll jank, high main-thread work, layout or rendering cost, image decoding cost, excessive allocations, expensive compositing, long synchronous work, startup, or screen transition delays.

The agent can suggest which trace to collect and what to inspect. It can analyze a provided trace summary, screenshot, or exported data. It cannot claim the trace is clean unless it has seen the trace.

Connect symptoms to hypotheses:

- delayed first feedback → main-thread work, no immediate state update, or blocking request path;
- blank screen → missing loading state or all-or-nothing rendering;
- layout jumps → unstable placeholder or late section size changes;
- scroll hitches → expensive cell layout, image decoding, rendering, or main-thread work;
- animation hitches → main-thread work, layout invalidation, rendering cost, or compositing;
- slow transition → synchronous work during navigation or initial rendering;
- task continues after screen disappears → missing cancellation or owner-scoped task cleanup.

## Production signals

Use production signals when the issue appears only in the wild, depends on real data, or needs trend tracking across devices and app versions.

Useful production signals include Xcode Organizer hangs and hitches, MetricKit payloads and hang diagnostics, custom signposts, custom screen-stage timings, backend timing correlated with client stages, app-version trends, device-class and OS-version breakdowns, feature-level metrics, and user reports with reproduction details.

Recommended breakdowns: app version, device model or device class, OS version, screen or feature, cold vs warm start, first load vs refresh, network type when available, and logged-in state or dataset size when relevant.

The agent can suggest instrumentation and analyze provided production summaries. It cannot access production data unless the user provides it or the environment has a connected source.

## Custom instrumentation

Perceived-performance improvements are easier to validate when important stages are named.

Consider logging or signposting: user action received, loading state shown, first placeholder shown, first meaningful content shown, primary action enabled, secondary content loaded, refresh started/completed, optimistic update applied, backend confirmation received, rollback shown, high-stakes submission started, final confirmation shown, and unknown outcome detected.

Example stage names:

```swift
enum ScreenStage: String {
    case actionReceived
    case loadingShown
    case firstContentShown
    case primaryActionEnabled
    case fullyLoaded
}
```

Use instrumentation to compare before and after changes. Do not log sensitive data. For financial, medical, identity, or security-sensitive flows, review privacy and compliance requirements before adding telemetry.

## Validation by pattern

Use different validation depending on the perceived-performance pattern.


## Review checklist

Before finalizing a validation recommendation, check:

- [ ] Is the claim based on code inspection, local runtime evidence, automated tests, profiling, or production evidence?
- [ ] Did the answer avoid claiming unmeasured improvement?
- [ ] Is the validation method appropriate for the practice?
- [ ] Is release-like build validation recommended for performance-sensitive claims?
- [ ] Are older devices suggested when device headroom matters?
- [ ] Is Low Power Mode described only as a manual stress signal?
- [ ] Are screen recordings suggested for perceived timing and visual stability?
- [ ] Are scrolling and animation flows validated when relevant?
- [ ] Are failure, retry, offline, and app-interruption cases included when relevant?
- [ ] Are high-stakes unknown outcomes tested?
- [ ] Are production signals suggested for issues seen in the wild?
- [ ] Is instrumentation suggested when before/after comparison would otherwise be vague?
- [ ] Are privacy and compliance concerns mentioned for telemetry in sensitive flows?
- [ ] Does the answer distinguish what the agent can do from what needs manual or runtime validation?

## Review output guidance

When using this reference, explain:

```markdown
## Validation scope

State whether the recommendation is based on code inspection, local runtime evidence, profiling evidence, automated tests, or production data.

## What the agent can do

List code changes, instrumentation, tests, or analysis that can be performed from the available repository or artifacts.

## What needs runtime validation

List device, release-build, Low Power Mode, recording, Instruments, or production checks that require execution or user-provided evidence.

## Suggested validation plan

Provide the smallest useful validation plan for the specific pattern: progressive rendering, loading states, optimistic updates, or high-stakes actions.

## Evidence to collect

Name the metrics, recordings, traces, logs, or production signals that would prove or disprove the claim.
```

Use careful language. Prefer “validate,” “measure,” “inspect,” and “compare” over “prove” unless the evidence is actually available.
