# Animation Hitches and SwiftUI

Use this reference when the task involves animation hitches, scrolling hitches, dropped frames, Core Animation, SwiftUI Instrument, repeated view updates, frame budget, or UI responsiveness traces.

This file helps the agent choose and interpret UI responsiveness profiling evidence. It does not replace the deeper `swiftui-performance` skill for code-level SwiftUI invalidation, identity, layout, and state-scope fixes.

## Contents

- [Core model](#core-model)
- [When to use this reference](#when-to-use-this-reference)
- [Start from the visible symptom](#start-from-the-visible-symptom)
- [Capability and evidence check](#capability-and-evidence-check)
- [Tool routing](#tool-routing)
- [Animation Hitches workflow](#animation-hitches-workflow)
- [Core Animation workflow](#core-animation-workflow)
- [SwiftUI Instrument workflow](#swiftui-instrument-workflow)
- [Time Profiler handoff](#time-profiler-handoff)
- [Signposts for UI responsiveness](#signposts-for-ui-responsiveness)
- [Common hitch patterns](#common-hitch-patterns)
- [Decision rules](#decision-rules)
- [Gotchas](#gotchas)
- [Fix directions](#fix-directions)
- [Validation](#validation)
- [Output guidance](#output-guidance)

## Core model

A UI hitch happens when the app misses the time budget for producing a smooth visual update.

Do not diagnose a hitch as “SwiftUI is slow” or “Core Animation is slow” without evidence. A missed frame can come from main-thread CPU work, expensive layout, drawing, image decoding, view/layer creation, broad SwiftUI invalidation, unstable identity, blocked I/O, lock waits, or async work resuming on the main actor at the wrong time.

The profiling goal is to locate the missed-frame window, identify what work overlapped it, and connect the fix to that evidence.

## When to use this reference

Use this reference when the task mentions animation hitches, scrolling hitches, dropped frames, frame budget, Core Animation, Animation Hitches, SwiftUI Instrument, repeated SwiftUI updates, broad invalidation, body work in a trace, slow gestures, transitions, interactive animations, or UI responsiveness traces.

Do not use this reference as the main source for general SwiftUI code review without profiling context, launch-only performance unless first screen or first interaction hitches, pure memory/network/disk issues unless they overlap visible UI stalls, or generic animation design.

## Start from the visible symptom

First classify what the user sees.

Ask:

1. Does it happen during scroll, transition, gesture, first render, repeated updates, or a specific interaction?
2. Is it reproducible on a real device?
3. Is it tied to a screen, data size, device class, OS version, or build configuration?
4. Is it a single long pause, repeated small stutters, or consistently low frame rate?
5. Does it happen before content appears, while content updates, or after the user interacts?

A scroll hitch in a feed, a slow navigation transition, and repeated SwiftUI updates may all show dropped frames, but they usually need different inspection paths.

## Capability and evidence check

Before interpreting UI performance, check:

- real device or Simulator;
- device model, refresh rate, and iOS version;
- Debug, Release, or release-like build;
- data size, account state, and reproduction steps;
- Low Power Mode, thermal pressure, or recording overhead;
- whether the trace includes the exact hitch interval;
- whether screenshots, Instruments trace, signpost log, or code are available.

Prefer real-device Release or release-like profiling for UI responsiveness. Simulator and Debug builds are useful for early investigation, but not enough to claim a production UI fix.

## Tool routing

| Symptom | Primary tool | Secondary tool | Strong signal |
|---|---|---|---|
| Dropped frames during animation or transition | Animation Hitches | Core Animation, Time Profiler | long hitch window with overlapping app/render work |
| Scrolling stutters in feed/list | Animation Hitches, Core Animation | Time Profiler, SwiftUI Instrument | long frames, expensive row work, layout, drawing, image decode |
| SwiftUI views update too often | SwiftUI Instrument | Time Profiler, signposts | repeated body work, broad invalidation, identity churn |
| Gesture feels delayed | Animation Hitches, Time Profiler | Hangs, signposts | work blocks input handling or main-thread progress |
| First screen appears but first interaction is late | App Launch, Animation Hitches, Time Profiler | signposts | work after first frame blocks interaction |
| Smooth locally, bad in production | MetricKit, Organizer | local reproduction with Animation Hitches | device/OS cohort regression or high p95/p99 |

Use Time Profiler after identifying a bad interval. It explains stack-level cost inside the hitch window; it should not be the only evidence for a visual hitch.

## Animation Hitches workflow

Use Animation Hitches when the visible problem is missed frames during scrolling, gestures, transitions, or animations.

Workflow:

1. Record the exact scenario where the hitch is visible.
2. Locate the worst hitch window, not just average frame rate.
3. Check whether the hitch overlaps main-thread work, rendering work, layout, drawing, image work, or app-specific signposts.
4. Use Time Profiler or System Trace to explain suspicious intervals.
5. Route to SwiftUI, runtime, concurrency, disk, or image-pipeline guidance based on the strongest signal.
6. Re-record the same scenario after the fix.

Inspect long frame intervals, spikes near user input, long main-thread stretches, layout/rendering work, image decoding, view/layer creation, animation callbacks, repeated row construction, and signposts for data updates or model preparation.

Avoid diagnosing from a single screenshot. The strongest evidence is usually the combination of a visible hitch interval and the work that overlaps it.

## Core Animation workflow

Use Core Animation when the problem may be in layer rendering, compositing, layout, drawing, or view/layer behavior.

Inspect layout during animation, view/layer creation, expensive drawing, offscreen rendering risk, blending/transparency, shadows, blur, masks, corner radius, rasterization, large layer effects, layer tree commits, image decode or texture upload timing, and animations running for invisible or offscreen content.

Core Animation evidence is especially useful when Time Profiler shows little obvious app CPU cost but frames still miss their budget.

Do not assume every missed frame is caused by compositing. If the main thread is busy preparing models, decoding images, formatting text, or running SwiftUI updates, Core Animation may only show the consequence.

## SwiftUI Instrument workflow

Use the SwiftUI Instrument when the trace suggests repeated SwiftUI updates, excessive body work, broad invalidation, or identity churn.

Inspect:

- views updating more often than expected;
- state changes that invalidate a large subtree;
- repeated `body` evaluation in rows or expensive containers;
- identity churn in lists, grids, and conditional branches;
- frequent creation of short-lived view models or derived values;
- expensive computed properties used by `body`;
- `.task`, `.onAppear`, or lifecycle work firing repeatedly;
- bindings or environment values that cause wide update propagation.

A SwiftUI trace should lead to a narrower code question: which state read invalidates this subtree, which identity changes cause reconstruction, which repeated work happens during `body`, and which lifecycle callback is coupled to scrolling or animation?

Route code-level reasoning to `swiftui-performance` after the profiling evidence identifies the likely update path.

## Time Profiler handoff

Use Time Profiler when the hitch window needs stack-level explanation.

Inspect the time range around the hitch, not the whole trace. Look for hot main-thread stacks, repeated small functions that accumulate, expensive formatting/parsing/sorting/diffing/mapping, text measurement, image decoding/resizing, synchronous disk reads, lock waits, dispatch waits, MainActor work during animation, and app code inside layout, drawing, row construction, or update callbacks.

Do not optimize a globally hot function if it is not active during the hitch window. The fix must target the user-visible frame miss.

## Signposts for UI responsiveness

Add signposts when the system trace shows a broad hitch window but the app-specific operation is unclear.

Useful signpost regions include screen load, first content render, first interaction readiness, list data update, diff generation, row model creation, image request/decode/display, database fetch and mapping, search/filter/sort, animation-triggering state change, and expensive async operations that resume on the main actor.

Example:

```swift
import os

private let logger = Logger(subsystem: "com.example.app", category: "FeedPerformance")
private let signposter = OSSignposter(logger: logger)

func rebuildVisibleFeedModels() {
    let state = signposter.beginInterval("Feed model rebuild")
    defer { signposter.endInterval("Feed model rebuild", state) }

    // Expensive mapping, filtering, sorting, or diff preparation.
}
```

Do not signpost every small helper. Instrument user-visible operations and suspected expensive regions.

## Common hitch patterns

### Expensive row construction

Signals: scrolling hitches as new rows appear; Time Profiler points to mapping, formatting, image work, layout, or text measurement; SwiftUI Instrument shows repeated row updates.

Fix direction: precompute stable row models outside the frame-critical path, move image decode/resize out of row construction, avoid expensive computed properties in `body`, reduce row dependency scope, and stabilize identity.

### Broad SwiftUI invalidation

Signals: many unrelated views update after a small state change; body work repeats during animation or scrolling; SwiftUI Instrument points to wide dependency propagation.

Fix direction: move state reads to the smallest view that needs them, split observable models by ownership and update frequency, avoid parent views reading state only needed by children, and route code-level guidance to `swiftui-performance`.

### Unstable identity

Signals: rows reconstruct instead of update; animations look inconsistent; state resets during list changes; SwiftUI Instrument shows identity churn.

Fix direction: use stable model identifiers, avoid `UUID()` or random IDs during render, avoid changing `.id(...)` as a refresh mechanism, and keep conditional view identity predictable.

### Image decode on the frame path

Signals: hitch appears when images enter the viewport; Time Profiler shows image decoding, resizing, or decompression; memory and CPU spike during scroll.

Fix direction: decode and resize before display when possible, cache appropriately, avoid oversized images, prioritize visible images, and cancel work for invisible cells/views.

### Layout or drawing during animation

Signals: long frame intervals during transitions; Core Animation or Time Profiler points to layout, text measurement, drawing, masks, shadows, blur, or layer updates.

Fix direction: reduce layout complexity on the animated path, avoid repeated text measurement, simplify expensive effects, and avoid animating properties that cause heavy layout or rendering work.

## Decision rules

- If frames drop but CPU is not high, inspect rendering, layout, blocking, image upload, memory pressure, and synchronization.
- If Time Profiler shows hot code outside the hitch interval, do not treat it as the cause.
- If SwiftUI updates repeat, identify the state dependency before proposing a refactor.
- If rows hitch during scroll, inspect row construction, image work, identity, layout, and lifecycle callbacks.
- If the issue appears only on older devices, profile on the lowest supported device class before claiming a fix.
- If the trace is from Debug or Simulator, label conclusions as tentative.
- If a fix changes user-visible behavior, mention the trade-off.

## Gotchas

- A dropped frame is a symptom, not a root cause.
- Average FPS can hide rare but painful hitches. Inspect the worst intervals.
- The main thread can be blocked without high CPU.
- SwiftUI `body` evaluation is not automatically the problem; repeated expensive work inside or triggered by updates is the problem.
- `LazyVStack` is not equivalent to cell reuse. Do not recommend replacing `List` blindly.
- `drawingGroup()` and rasterization-style fixes can move cost or increase memory. Do not suggest them as generic fixes.
- `EquatableView` helps only when equality is cheap and prevents meaningful repeated work.
- Caching can reduce hitches but can also increase memory pressure and later stalls.
- Disabling animations can hide the symptom without fixing the underlying responsiveness issue.
- Do not call a UI fix successful without re-running the same interaction trace.

## Fix directions

Connect the fix to the measured cause.

| Evidence | Likely fix direction |
|---|---|
| Hot main-thread stack during hitch | remove, defer, cache, or move measured work off the frame-critical path |
| Repeated SwiftUI updates | narrow state reads, stabilize identity, reduce broad observable dependencies |
| Expensive row construction | precompute row models, simplify rows, avoid expensive `body` work |
| Image decode during scroll | resize/decode earlier, cache, prioritize visible images, cancel invisible work |
| Layout/drawing dominates | simplify layout/effects, reduce measurement, avoid expensive animated properties |
| Blocked main thread | remove synchronous I/O, lock waits, dispatch waits, or dependency cycles |
| Async resumes flood main actor | coalesce updates, batch results, add cancellation, reduce MainActor work |
| Production-only hitches | compare cohorts, add signposts, reproduce on affected device/OS/data size |

Prefer one focused fix at a time. Broad rewrites make before/after evidence harder to trust.

## Validation

Validate with the same scenario that exposed the hitch.

Minimum validation: same device class or lowest supported device, same build configuration, same screen/data/interaction, before/after trace comparison, worst hitch interval instead of average FPS only, and confirmation that the suspected work is reduced or moved out of the frame-critical path.

Useful signals include fewer or shorter hitch intervals, shorter main-thread work during the frame window, reduced repeated SwiftUI updates, lower row construction cost, image decode no longer overlapping scroll frames, fewer layout/drawing spikes, and improved production p95/p99 after release.

If validation is unavailable, say what is still unproven and what trace should be captured next.

## Output guidance

When analyzing an animation or scrolling hitch, respond with:

```text
## Symptom
## Best profiling path
Primary:
Secondary:
Why:
## Strongest signal to look for
## Likely causes
1. ...
2. ...
## Code or trace areas to inspect
## Fix direction
## Validation
```

When interpreting an existing trace or screenshot, respond with:

```text
## What the trace shows
## Strongest signal
## What is not proven yet
## Next inspection step
## Suggested fix direction
## Re-measurement plan
```
