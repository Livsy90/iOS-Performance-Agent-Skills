# Validation and Testing

Use this reference when reviewing how to verify perceived-performance claims for iOS screens and user flows.

This reference is about validation discipline. It helps the agent separate static code reasoning, implemented instrumentation, local runtime evidence, automated tests, profiling evidence, production evidence, and manual UX judgment.

For staged real-content rendering, read `references/progressive-rendering.md`.
For loading states, read `references/loading-states.md`.
For optimistic UI, read `references/optimistic-updates.md`.
For high-stakes flows, read `references/high-stakes-actions.md`.

## Contents

* [Core rule](#core-rule)
* [Agent capability boundaries](#agent-capability-boundaries)
* [Evidence ladder](#evidence-ladder)
* [Before/after discipline](#beforeafter-discipline)
* [Choosing validation methods](#choosing-validation-methods)
* [Manual validation](#manual-validation)
* [Screen recordings](#screen-recordings)
* [Release-like builds](#release-like-builds)
* [Older devices](#older-devices)
* [Low Power Mode](#low-power-mode)
* [Scrolling and animation checks](#scrolling-and-animation-checks)
* [Automated tests](#automated-tests)
* [Instruments](#instruments)
* [Production signals](#production-signals)
* [Custom instrumentation](#custom-instrumentation)
* [Acceptance criteria](#acceptance-criteria)
* [Validation by pattern](#validation-by-pattern)
* [Review checklist](#review-checklist)
* [Review output guidance](#review-output-guidance)

## Core rule

Do not present a perceived-performance recommendation as verified unless there is evidence.

A code change can be plausible without being proven. A screen can have better state modeling without being objectively smoother. A progressive-rendering refactor can reduce blank-screen time without reducing total loading time. A loading state can make progress visible without improving backend latency. An optimistic update can make a tap feel instant while still requiring rollback and reconciliation.

Use precise language:

* “This should reduce blank-screen time.”
* “This gives the user earlier feedback.”
* “This needs validation on a release-like build.”
* “This should be checked with a screen recording, Instruments trace, or production signal.”
* “This is a code-review hypothesis until measured.”

Avoid unsupported claims:

* “This fixes performance.”
* “This is now smooth.”
* “This will feel faster to users.”
* “This works well on older devices.”
* “Low Power Mode proves old-device performance.”
* “This is validated” when no validation was performed.

## Agent capability boundaries

The agent can usually inspect and improve:

* missing loading, empty, failed, refreshing, pending, and confirmed states;
* all-or-nothing rendering;
* UI that waits for unrelated async work before updating;
* optimistic UI without rollback or reconciliation;
* high-stakes flows that show success before confirmation;
* duplicate-submission guards, retry states, and recovery states;
* missing instrumentation points;
* missing state-transition tests when the repository is available;
* missing before/after measurement plan.

The agent can analyze user-provided evidence such as:

* screen recordings;
* trace summaries;
* Instruments screenshots;
* signpost logs;
* timestamped logs;
* XCTest or UI test reports;
* CI performance reports;
* MetricKit summaries;
* Xcode Organizer hangs or hitches;
* before/after measurements;
* reproducible user reports.

The agent cannot directly prove actual device smoothness, older-device behavior, Low Power Mode behavior, thermal behavior, release-build responsiveness, hitch frequency, production rates, or actual user perception unless it can run the app in the environment or the user provides evidence.

Do not claim that manual, device, production, or profiling validation was performed unless the agent actually ran it in the available environment or the user provided the result.

## Evidence ladder

Use the strongest available evidence, but be honest about what each level can and cannot prove.

| Evidence level          | Examples                                                                 | What it can support                                                                 | What it cannot prove alone                                             |
| ----------------------- | ------------------------------------------------------------------------ | ----------------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| Static code review      | State model review, async dependency review, UI transition review        | A likely risk, missing state, unsafe assumption, or plausible improvement           | Actual smoothness, perceived speed, device behavior, production impact |
| Instrumentation added   | Stage logs, signposts, analytics events, timestamp points                | The app is ready to measure specific stages                                         | That performance improved                                              |
| Manual screen recording | Before/after recording, slow-network recording, rollback recording       | Visual timing, blank-screen duration, first feedback, layout jumps, visible flicker | CPU cause, production frequency, older-device behavior                 |
| Local automated UI test | Loading appears before content, retry path works, duplicate tap disabled | State transitions and regression coverage                                           | Real user perception or frame smoothness                               |
| Local performance test  | Measured repeated operation, scroll gesture timing, launch measure       | Repeatable regression signal for a controlled scenario                              | Full UX quality or production behavior                                 |
| Instruments trace       | Time Profiler, Animation Hitches, Core Animation, Allocations, Network   | Runtime cause, main-thread stalls, hitches, layout/rendering/allocation cost        | Product acceptance, production prevalence                              |
| Device matrix           | Modern device, older device, different OS versions, Low Power Mode       | Device headroom and risk across hardware classes                                    | Real-world rate unless production data exists                          |
| Production signals      | MetricKit, Xcode Organizer, custom telemetry, user reports               | Real-world frequency, trends, device cohorts, version regressions                   | Exact root cause without correlation or local reproduction             |

Prefer before/after comparisons when possible. For perceived performance, include at least one user-visible signal such as tap-to-first-feedback, blank-screen duration, first meaningful content, first actionable state, rollback visibility, or visual stability.

## Before/after discipline

A before/after comparison is useful only if the scenario is comparable.

Keep these stable when possible:

* same device;
* same OS version;
* same build configuration;
* same app version or branch except for the tested change;
* same account or dataset size;
* same screen entry path;
* same network condition or controlled network profile;
* same cache state when relevant;
* same feature flags;
* same user action sequence;
* same measurement points.

Run more than once when timing is noisy. Do not overinterpret a single run if the result is close to noise.

Good phrasing:

* “Run the same scenario three to five times and compare median or representative values.”
* “Use the same dataset and cache state before and after.”
* “Record both versions and compare tap-to-first-content and layout stability.”

Avoid:

* “One local run proves the change.”
* “A simulator run proves production performance.”
* “A faster backend response proves the UI change helped perceived performance.”

## Choosing validation methods

Choose validation based on the claim. Do not recommend every tool for every change.

| Claim or symptom                    | Recommended validation                                                    | Evidence to collect                                                                                  |
| ----------------------------------- | ------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| “The screen no longer feels blank”  | Screen recording, tap-to-first-feedback timing                            | Time from action to first visible feedback, blank-screen duration                                    |
| “Primary content appears earlier”   | Stage timing, screen recording, optional signposts                        | Tap-to-first-content, first meaningful content, fully loaded time                                    |
| “Refresh feels better”              | Screen recording, slow network, stale-content test                        | Existing content remains visible, refresh indicator appears, failure preserves old content when safe |
| “Skeleton prevents layout jumps”    | Screen recording, slow network, visual inspection                         | Placeholder-to-content transition, section height stability, scroll position                         |
| “Optimistic update feels immediate” | Failure simulation, repeated taps, screen recording                       | Tap-to-visual-change, pending state, rollback/reconciliation behavior                                |
| “High-stakes flow is trustworthy”   | Scenario tests for success, rejection, timeout, unknown outcome           | Submitting/pending/confirmed/failed/unknown states, duplicate submission behavior                    |
| “Scrolling is smooth”               | Device recording, Animation Hitches, Core Animation, Time Profiler        | Dropped frames, hitch intervals, main-thread work, expensive layout/rendering                        |
| “Animation is smooth”               | Animation Hitches, screen recording, Time Profiler                        | Hitch moments, main-thread stalls, layout invalidation, compositing cost                             |
| “Main thread is blocked”            | Time Profiler, signposts, logs                                            | Main-thread stack during delay, app symbols, stage boundaries                                        |
| “Older devices are acceptable”      | Real older-device test, release-like build, optional production breakdown | Same scenario on target devices, device-class metrics                                                |
| “Production users are affected”     | MetricKit, Xcode Organizer, custom telemetry, user reports                | Device/app-version trends, hang/hitch rates, stage timing distributions                              |

The validation method should match the claim. A screen recording may show that the UI gives feedback earlier, but it does not explain CPU cost. Instruments may explain CPU cost, but it does not prove that the copy or state transition feels clear to users.

## Manual validation

Manual validation is appropriate when perceived performance depends on what the user sees and feels.

Use it for:

* tap-to-first-feedback;
* time to first meaningful content;
* loading-state clarity;
* skeleton or placeholder behavior;
* layout stability;
* scroll smoothness;
* animation continuity;
* high-stakes confirmation clarity;
* optimistic update failure behavior;
* retry and recovery flows.

Suggested scenarios:

* first load;
* refresh with existing content;
* slow network;
* failed request;
* retry after failure;
* repeated taps;
* screen disappearance during work;
* app backgrounding and returning;
* navigation away and back;
* long scrolling sessions;
* animation-heavy transitions.

The agent can suggest these scenarios. It should not claim they passed unless evidence is provided.

## Screen recordings

Screen recordings are useful because perceived performance is visual and temporal.

Use recordings to inspect:

* tap-to-first-feedback;
* blank-screen duration;
* first meaningful content;
* layout jumps;
* flicker between states;
* stale content during refresh;
* loading indicator timing;
* optimistic update feedback;
* rollback or failure transition;
* duplicate-submission behavior;
* visible animation hitches.

Suggested review method:

1. Start recording before the user action.
2. Perform the action once under normal conditions.
3. Repeat under slow network or constrained conditions when relevant.
4. Compare before and after changes.
5. Note timestamps for first feedback, first meaningful content, first actionable state, and final state.
6. Inspect whether any skeleton, placeholder, error, or secondary section causes visible layout movement.

Correct phrasing:

* “The recording shows feedback appearing immediately after tap.”
* “The screen remains blank until all sections load.”
* “The layout shifts when the secondary section appears.”
* “The rollback state is visible after simulated failure.”

Avoid:

* “Users will definitely perceive this as faster.”
* “This proves all devices are smooth.”
* “This fixes performance globally.”

## Release-like builds

Prefer release or release-like builds for performance validation.

Debug builds are useful for development, but they are not reliable proof of user-facing performance. They may include different compiler optimization levels, additional assertions, debug logging, overlays, and non-production behavior.

Use release-like builds when validating:

* scrolling;
* animations;
* launch or screen transition timing;
* rendering-heavy screens;
* CPU-heavy transformations;
* memory pressure;
* perceived responsiveness;
* repeated navigation flows.

If the repository is available, the agent can check whether performance tests or CI jobs run in a release-like configuration.

Do not claim release-build performance from a Debug-only run.

## Older devices

Older-device testing is valuable because real devices differ in CPU, GPU, memory bandwidth, thermal behavior, refresh characteristics, OS versions, and storage behavior.

Use older-device testing for:

* scroll-heavy feeds;
* media-heavy screens;
* animation-heavy flows;
* large SwiftUI or UIKit hierarchies;
* image decoding or resizing;
* expensive layout or compositing;
* startup and first-screen rendering;
* low memory headroom;
* features known to be used on older devices.

The agent cannot simulate an older device from code inspection. It can recommend older-device validation and explain which flows should be tested.

Do not claim that simulator results or a modern device prove older-device performance.

## Low Power Mode

Low Power Mode can be suggested as a cheap manual stress signal for render-heavy screens.

It may make limited performance headroom easier to notice, but it is not an older-device simulator. It does not reproduce older GPU architecture, memory bandwidth, display behavior, OS version, storage behavior, thermal state, or device-specific bottlenecks.

Use Low Power Mode as a signal for:

* frame drops;
* animation hitches;
* delayed tap response;
* expensive layout;
* image decoding pressure;
* compositing and transparency cost;
* render-heavy lists or feeds.

Correct phrasing:

* “Try the screen with Low Power Mode enabled as a manual stress signal.”
* “If the screen starts hitching under Low Power Mode, inspect layout, rendering, decoding, or compositing cost.”
* “Low Power Mode does not replace older-device testing.”

Avoid:

* “Low Power Mode proves old-device performance.”
* “The screen passes performance testing because it works in Low Power Mode.”
* “Low Power Mode is equivalent to a low-end device.”

## Scrolling and animation checks

Scrolling and animation issues are often perceived as performance problems even when data loading is fast.

Validate:

* first scroll after content appears;
* long continuous scrolling;
* rapid scroll direction changes;
* pull-to-refresh;
* navigation push and pop;
* modal presentation and dismissal;
* expanding sections;
* animated state transitions;
* image-heavy cells;
* skeleton-to-content transitions;
* error-to-retry-to-loaded transitions.

Look for:

* dropped frames;
* visible hitching;
* delayed interaction;
* layout jumps;
* image pop-in;
* repeated re-layout;
* content offset jumps;
* animation stutter;
* sudden main-thread stalls.

The agent can suggest these checks and can analyze recordings or traces if provided. It should not claim scrolling or animation is smooth without runtime evidence.

## Automated tests

Automated tests can help catch regressions when the same flow can be repeated.

Good candidates include:

* launch to first screen;
* tap to first loading state;
* tap to first meaningful content;
* refresh flow;
* search result rendering;
* scroll through a long list;
* navigation into and out of a heavy screen;
* retry after failure;
* optimistic update state transition;
* duplicate tap prevention;
* high-stakes pending/confirmed/unknown state transitions.

A UI test can verify state transitions even when it does not measure performance:

```swift id="0o0x4i"
func testLoadingStateAppearsBeforeContent() {
    let app = XCUIApplication()
    app.launch()

    app.buttons["Load"].tap()

    XCTAssertTrue(app.staticTexts["Loading"].waitForExistence(timeout: 1))
    XCTAssertTrue(app.staticTexts["Content"].waitForExistence(timeout: 5))
}
```

A performance test can measure repeatable work when the project supports it:

```swift id="as8wj1"
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

Use performance tests for regression detection, not as the only proof of perceived quality.

State limitations explicitly:

* simulator results may not match device behavior;
* CI hardware may vary;
* debug builds can distort performance;
* tests may not cover real data size;
* tests may miss visual quality issues;
* UI tests can be flaky if they depend on uncontrolled network or timing.

Prefer controlled test data, mocks, local fixtures, or test servers when validating state transitions.

## Instruments

Use Instruments when there is a runtime symptom that code review cannot prove or localize.

Useful when investigating:

* UI hangs;
* delayed tap response;
* animation hitches;
* scroll jank;
* high main-thread work;
* layout or rendering cost;
* image decoding cost;
* excessive allocations;
* expensive compositing;
* long synchronous work;
* startup;
* screen transition delays.

The agent can suggest which trace to collect and what to inspect. It can analyze a provided trace summary, screenshot, or exported data. It cannot claim the trace is clean unless it has seen the trace.

Connect symptoms to hypotheses:

* delayed first feedback → main-thread work, no immediate state update, or blocking request path;
* blank screen → missing loading state, missing first feedback, or all-or-nothing rendering;
* layout jumps → unstable placeholder or late section size changes;
* scroll hitches → expensive cell layout, image decoding, rendering, or main-thread work;
* animation hitches → main-thread work, layout invalidation, rendering cost, or compositing;
* slow transition → synchronous work during navigation or initial rendering;
* task continues after screen disappears → missing cancellation or owner-scoped task cleanup.

Use signposts or timestamp logs to align user-visible stages with Instruments timelines when possible.

## Production signals

Use production signals when the issue appears only in the wild, depends on real data, or needs trend tracking across devices and app versions.

Useful production signals include:

* Xcode Organizer hangs and hitches;
* MetricKit payloads and diagnostics;
* custom signposts;
* custom screen-stage timings;
* backend timing correlated with client stages;
* app-version trends;
* device-class and OS-version breakdowns;
* feature-level metrics;
* user reports with reproduction details.

Recommended breakdowns:

* app version;
* device model or device class;
* OS version;
* screen or feature;
* cold vs warm start;
* first load vs refresh;
* network type when available;
* logged-in state or dataset size when relevant;
* feature flag or experiment cohort.

Production signals can show real-world frequency and trends, but they may not identify the root cause alone. Correlate production metrics with screen stages, signposts, logs, traces, or a local reproduction when possible.

The agent can suggest instrumentation and analyze provided production summaries. It cannot access production data unless the user provides it or the environment has a connected source.

## Custom instrumentation

Perceived-performance improvements are easier to validate when important stages are named.

Consider logging or signposting:

* user action received;
* loading state shown;
* first placeholder shown;
* first meaningful content shown;
* primary action enabled;
* secondary content loaded;
* refresh started;
* refresh completed;
* optimistic update applied;
* backend confirmation received;
* rollback shown;
* high-stakes submission started;
* pending state shown;
* final confirmation shown;
* unknown outcome detected.

Example stage names:

```swift id="v0m8gd"
enum ScreenStage: String {
    case actionReceived
    case loadingShown
    case firstPlaceholderShown
    case firstContentShown
    case primaryActionEnabled
    case fullyLoaded
}
```

Example signpost shape:

```swift id="6d9n83"
import os

private let signposter = OSSignposter(
    subsystem: "com.example.app",
    category: "PerceivedPerformance"
)

func loadScreen() async {
    let signpostID = signposter.makeSignpostID()
    let state = signposter.beginInterval(
        "HomeScreenLoad",
        id: signpostID
    )

    await showLoadingState()
    signposter.emitEvent("LoadingShown", id: signpostID)

    await loadPrimaryContent()
    signposter.emitEvent("FirstContentShown", id: signpostID)

    await loadSecondaryContent()
    signposter.emitEvent("FullyLoaded", id: signpostID)

    signposter.endInterval("HomeScreenLoad", state)
}
```

Use instrumentation to compare before and after changes. Keep names stable enough to compare across builds.

Do not log sensitive data. For financial, medical, identity, or security-sensitive flows, review privacy and compliance requirements before adding telemetry.

## Acceptance criteria

Acceptance criteria should come from the product, design, engineering, or performance baseline when possible.

The agent may suggest useful targets, but should not invent hard pass/fail thresholds as if they are product requirements.

Useful acceptance criteria may include:

* loading state appears within an agreed time after tap;
* first meaningful content appears earlier than before;
* existing content remains visible during refresh;
* no empty-state flash before loading begins;
* skeleton-to-content transition does not shift layout noticeably;
* duplicate taps do not submit duplicate work;
* rollback is visible and understandable after failure;
* high-stakes unknown outcome is represented and recoverable;
* older-device behavior is acceptable for the target device class;
* no new hitches are visible in the validated interaction.

Good phrasing:

* “Use the team’s existing baseline if available.”
* “If the team has no threshold, first collect baseline measurements, then define a target.”
* “Treat this as a suggested validation target, not a universal standard.”

Avoid:

* “This must be under 100 ms” without product or team baseline.
* “This passes because it looks fine once.”
* “This is acceptable on all devices because it passed locally.”

## Validation by pattern

Use different validation depending on the perceived-performance pattern.

### Loading states

Validate:

* loading appears before the operation completes;
* loading, empty, failed, refreshing, and loaded states are distinct;
* retry gives visible feedback;
* blank screen is avoided during meaningful waits;
* stale content is preserved during refresh when safe.

Suggested evidence:

* screen recording;
* slow-network test;
* UI test for state transitions;
* timestamp from user action to loading state;
* failure simulation.

### Progressive rendering

Validate:

* primary content appears before secondary content;
* blank-screen duration is reduced;
* first meaningful content timing improves;
* layout remains stable as sections appear;
* partial failure does not collapse useful content;
* stale responses do not overwrite newer state.

Suggested evidence:

* screen recording before and after;
* stage timing for first content and fully loaded;
* signposts around section loads;
* slow-network or delayed-secondary-section test;
* layout-jump inspection.

### Optimistic updates

Validate:

* tap produces immediate local feedback;
* pending state is represented when needed;
* repeated taps are handled intentionally;
* failure triggers rollback, queued state, retry, or reconciliation;
* out-of-order responses do not overwrite newer state;
* server canonical state is applied when available.

Suggested evidence:

* failure simulation;
* repeated-tap test;
* delayed-response test;
* screen recording;
* state-transition unit or UI tests;
* logs for mutation IDs or request ordering.

### High-stakes actions

Validate:

* final success is not shown before authoritative confirmation;
* submitting, pending, confirmed, failed, and unknown states are distinct when relevant;
* duplicate submission is guarded;
* timeout and lost-response scenarios are handled;
* retry is safe or blocked until status is checked;
* app interruption and relaunch do not lose unresolved operation identity.

Suggested evidence:

* scenario tests for success, rejection, timeout, unknown outcome;
* app backgrounding and termination tests;
* duplicate-tap tests;
* logs or signposts for request IDs;
* product, backend, security, legal, or compliance confirmation when needed.

### Scrolling and animation

Validate:

* first scroll after content appears;
* long scrolling sessions;
* rapid direction changes;
* animated state transitions;
* image-heavy cells;
* skeleton-to-content transition;
* refresh and pagination transitions.

Suggested evidence:

* screen recording on target device;
* Animation Hitches trace;
* Core Animation trace;
* Time Profiler trace;
* Allocations trace when memory churn is suspected;
* older-device run when device headroom matters.

### Refresh and stale content

Validate:

* refresh does not clear useful content unless required;
* stale content is labeled or represented when freshness matters;
* refresh failure preserves old content when safe;
* refresh indicator appears and disappears correctly;
* old refresh responses do not overwrite newer screen state.

Suggested evidence:

* slow-network refresh test;
* failure simulation;
* screen recording;
* request identity logs;
* UI test for refresh state.

### Duplicate submission and repeated actions

Validate:

* repeated taps do not submit duplicate unsafe operations;
* controls show pending state or disable correctly;
* idempotency or request identifiers are used when needed;
* retry does not duplicate an already-submitted operation;
* UI communicates whether the operation is pending, failed, confirmed, or unknown.

Suggested evidence:

* repeated-tap test;
* network logs;
* server logs when available;
* state-transition tests;
* high-stakes scenario tests.

### Production-only issues

Validate:

* the issue is present in production signals;
* the affected app versions, device classes, and OS versions are identified;
* the signal is correlated with a screen, feature, or stage;
* local reproduction or trace is attempted using the production pattern;
* after a fix, the same metric trends down in the affected cohort.

Suggested evidence:

* MetricKit summaries;
* Xcode Organizer reports;
* custom telemetry;
* user reports;
* feature-level stage timings;
* release-over-release comparison.

## Review checklist

Before finalizing a validation recommendation, check:

* [ ] Is the claim based on code inspection, local runtime evidence, automated tests, profiling, or production evidence?
* [ ] Did the answer avoid claiming unmeasured improvement?
* [ ] Is the validation method appropriate for the specific claim?
* [ ] Is before/after comparison recommended when possible?
* [ ] Is release-like build validation recommended for performance-sensitive claims?
* [ ] Are older devices suggested when device headroom matters?
* [ ] Is Low Power Mode described only as a manual stress signal?
* [ ] Are screen recordings suggested for perceived timing and visual stability?
* [ ] Are scrolling and animation flows validated when relevant?
* [ ] Are failure, retry, offline, and app-interruption cases included when relevant?
* [ ] Are high-stakes timeout, duplicate, and unknown-outcome paths tested?
* [ ] Are production signals suggested for issues seen in the wild?
* [ ] Is instrumentation suggested when before/after comparison would otherwise be vague?
* [ ] Are privacy and compliance concerns mentioned for telemetry in sensitive flows?
* [ ] Does the answer distinguish what the agent can do from what needs manual or runtime validation?
* [ ] Are acceptance criteria tied to baseline, product requirements, or team thresholds rather than invented universal numbers?

## Review output guidance

When using this reference, explain:

```markdown id="sb3k0m"
## Validation scope

State whether the recommendation is based on code inspection, local runtime evidence, profiling evidence, automated tests, production data, or user-provided artifacts.

## What the agent can do

List code changes, instrumentation, tests, or artifact analysis that can be performed from the available repository or provided evidence.

## What needs runtime validation

List device, release-build, Low Power Mode, screen recording, Instruments, older-device, or production checks that require execution or user-provided evidence.

## Suggested validation plan

Provide the smallest useful validation plan for the specific pattern: progressive rendering, loading states, optimistic updates, high-stakes actions, scrolling, refresh, duplicate submission, or production-only issue.

## Evidence to collect

Name the metrics, recordings, traces, logs, state-transition tests, or production signals that would support or disprove the claim.

## Careful conclusion

Use careful language. Prefer “validate,” “measure,” “inspect,” and “compare” over “prove” unless the evidence is actually available.
```

Use careful language. Prefer “validate,” “measure,” “inspect,” and “compare” over “prove” unless the evidence is actually available.
