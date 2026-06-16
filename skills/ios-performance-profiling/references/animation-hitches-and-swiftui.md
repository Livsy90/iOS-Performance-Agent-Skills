# Animation Hitches and SwiftUI

Use this reference when the task involves animation hitches, scrolling hitches, dropped frames, Core Animation, SwiftUI Instrument, repeated view updates, frame budget, or UI responsiveness traces.

This file helps the agent choose and interpret UI responsiveness profiling evidence. It does not replace the deeper `swiftui-performance` skill for code-level SwiftUI invalidation, identity, layout, and state-scope fixes.

## Scope Boundary

This reference covers:

* animation and scrolling hitches;
* dropped frames and long frame intervals;
* Core Animation, SwiftUI Instrument, Animation Hitches, and Time Profiler handoff;
* repeated SwiftUI updates when they overlap visible UI stalls;
* signposts for UI responsiveness;
* evidence-driven routing to SwiftUI, rendering, image, concurrency, disk, runtime, or launch guidance.

This reference does not cover:

* general SwiftUI code review without profiling context;
* launch performance unless first screen or first interaction hitches;
* memory, network, disk, or battery issues unless they overlap visible UI stalls;
* detailed SwiftUI invalidation, identity, layout, or state-scope fixes;
* generic animation design advice;
* visual design critique unrelated to measured responsiveness.

Use this file to choose and interpret profiling evidence. Route code-level SwiftUI fixes to `swiftui-performance` after the likely update path, state scope, identity issue, or layout/rendering cost is identified.

## Contents

* [Core Model](#core-model)
* [When to Use This Reference](#when-to-use-this-reference)
* [Start From the Visible Symptom](#start-from-the-visible-symptom)
* [Capability and Evidence Check](#capability-and-evidence-check)
* [Tool Routing](#tool-routing)
* [Animation Hitches Workflow](#animation-hitches-workflow)
* [Core Animation Workflow](#core-animation-workflow)
* [SwiftUI Instrument Workflow](#swiftui-instrument-workflow)
* [Time Profiler Handoff](#time-profiler-handoff)
* [Signposts for UI Responsiveness](#signposts-for-ui-responsiveness)
* [What the Agent Can Inspect](#what-the-agent-can-inspect)
* [Common Hitch Patterns](#common-hitch-patterns)
* [Decision Rules](#decision-rules)
* [Gotchas](#gotchas)
* [Fix Directions](#fix-directions)
* [Validation](#validation)
* [Output Guidance](#output-guidance)

## Core Model

A UI hitch happens when the app misses the time budget for producing a smooth visual update.

The frame budget depends on display refresh rate. A 60 Hz device gives roughly 16.7 ms per frame, while 120 Hz devices have a smaller per-frame budget. Do not hard-code one budget without knowing the device, refresh rate, and scenario.

Do not diagnose a hitch as “SwiftUI is slow” or “Core Animation is slow” without evidence. A missed frame can come from main-thread CPU work, expensive layout, drawing, image decoding, view/layer creation, broad SwiftUI invalidation, unstable identity, blocked I/O, lock waits, memory pressure, or async work resuming on the main actor at the wrong time.

Do not assume async work is off the frame path. A task may parse, map, sort, diff, or publish results on the `MainActor` during an animation or scroll. Check where the expensive work actually executes and where observable updates are delivered.

The profiling goal is to locate the missed-frame window, identify what work overlapped it, and connect the fix to that evidence.

## When to Use This Reference

Use this reference when the task mentions:

* animation hitches;
* scrolling hitches;
* dropped frames;
* frame budget;
* Core Animation;
* Animation Hitches;
* SwiftUI Instrument;
* repeated SwiftUI updates;
* broad invalidation;
* body work in a trace;
* slow gestures;
* transitions;
* interactive animations;
* UI responsiveness traces.

Do not use this reference as the main source for:

* general SwiftUI code review without profiling context;
* launch-only performance unless first screen or first interaction hitches;
* pure memory, network, disk, or battery issues unless they overlap visible UI stalls;
* generic animation design.

## Start From the Visible Symptom

First classify what the user sees.

Ask:

1. Does it happen during scroll, transition, gesture, first render, repeated updates, or a specific interaction?
2. Is it reproducible on a real device?
3. Is it tied to a screen, data size, device class, OS version, or build configuration?
4. Is it a single long pause, repeated small stutters, or consistently low frame rate?
5. Does it happen before content appears, while content updates, or after the user interacts?

A scroll hitch in a feed, a slow navigation transition, and repeated SwiftUI updates may all show dropped frames, but they usually need different inspection paths.

## Capability and Evidence Check

Before interpreting UI performance, check:

* real device or Simulator;
* device model, refresh rate, and iOS version;
* Debug, Release, or release-like build;
* data size, account state, and reproduction steps;
* Low Power Mode, thermal pressure, or recording overhead;
* whether the trace includes the exact hitch interval;
* whether screenshots, Instruments trace, signpost log, or code are available.

Prefer real-device Release or release-like profiling for UI responsiveness. Simulator and Debug builds are useful for early investigation, but not enough to claim a production UI fix.

A trace captured on a 120 Hz device may expose hitches that are less visible at 60 Hz, while older 60 Hz devices may expose CPU, memory, storage, and thermal limits that a newer ProMotion device hides.

## Tool Routing

| Symptom                                            | Primary tool                                 | Secondary tool                            | Strong signal                                                  |
| -------------------------------------------------- | -------------------------------------------- | ----------------------------------------- | -------------------------------------------------------------- |
| Dropped frames during animation or transition      | Animation Hitches                            | Core Animation, Time Profiler             | Long hitch window with overlapping app/render work             |
| Scrolling stutters in feed/list                    | Animation Hitches, Core Animation            | Time Profiler, SwiftUI Instrument         | Long frames, expensive row work, layout, drawing, image decode |
| SwiftUI views update too often                     | SwiftUI Instrument                           | Time Profiler, signposts                  | Repeated body work, broad invalidation, identity churn         |
| Gesture feels delayed                              | Animation Hitches, Time Profiler             | Hangs, signposts                          | Work blocks input handling or main-thread progress             |
| First screen appears but first interaction is late | App Launch, Animation Hitches, Time Profiler | Signposts                                 | Work after first frame blocks interaction                      |
| Smooth locally, bad in production                  | MetricKit, Organizer                         | Local reproduction with Animation Hitches | Device/OS cohort regression or high p95/p99                    |

Use Time Profiler after identifying a bad interval. It explains stack-level cost inside the hitch window; it should not be the only evidence for a visual hitch.

## Animation Hitches Workflow

Use Animation Hitches when the visible problem is missed frames during scrolling, gestures, transitions, or animations.

Workflow:

1. Record the exact scenario where the hitch is visible.
2. Locate the worst hitch window, not just average frame rate.
3. Check whether the hitch overlaps main-thread work, rendering work, layout, drawing, image work, blocking waits, or app-specific signposts.
4. Use Time Profiler or System Trace to explain suspicious intervals.
5. Route to SwiftUI, runtime, concurrency, disk, image-pipeline, or rendering guidance based on the strongest signal.
6. Re-record the same scenario after the fix.

Inspect long frame intervals, spikes near user input, long main-thread stretches, layout/rendering work, image decoding, view/layer creation, animation callbacks, repeated row construction, and signposts for data updates or model preparation.

If CPU is not high but the UI stalls, inspect waits, locks, `dispatch_sync`, semaphores, main-actor backpressure, run loop stalls, I/O waits, memory pressure, and resource contention.

Avoid diagnosing from a single screenshot. The strongest evidence is usually the combination of a visible hitch interval and the work that overlaps it.

## Core Animation Workflow

Use Core Animation when the problem may be in layer rendering, compositing, layout, drawing, or view/layer behavior.

Inspect:

* layout during animation;
* view/layer creation;
* expensive drawing;
* offscreen rendering risk;
* blending/transparency;
* shadows;
* blur;
* masks;
* corner radius;
* rasterization;
* large layer effects;
* layer tree commits;
* image decode or texture upload timing;
* animations running for invisible or offscreen content.

Core Animation evidence is especially useful when Time Profiler shows little obvious app CPU cost but frames still miss their budget.

Core Animation debug overlays such as blended layers or offscreen rendering can provide leads, but they are not root-cause proof by themselves. Validate them against the hitch interval.

Do not assume every missed frame is caused by compositing. If the main thread is busy preparing models, decoding images, formatting text, or running SwiftUI updates, Core Animation may only show the consequence.

## SwiftUI Instrument Workflow

Use the SwiftUI Instrument when the trace suggests repeated SwiftUI updates, excessive body work, broad invalidation, or identity churn.

Inspect:

* views updating more often than expected;
* state changes that invalidate a large subtree;
* repeated `body` evaluation in rows or expensive containers;
* identity churn in lists, grids, and conditional branches;
* frequent creation of short-lived view models or derived values;
* expensive computed properties used by `body`;
* `.task`, `.onAppear`, or lifecycle work firing repeatedly;
* bindings or environment values that cause wide update propagation.

For Observation-based state, inspect which properties are read by the view that updates. For `ObservableObject`, inspect whether broad object-level publishing invalidates more UI than needed.

A SwiftUI trace should lead to a narrower code question: which state read invalidates this subtree, which identity changes cause reconstruction, which repeated work happens during `body`, and which lifecycle callback is coupled to scrolling or animation?

Route code-level reasoning to `swiftui-performance` after the profiling evidence identifies the likely update path.

## Time Profiler Handoff

Use Time Profiler when the hitch window needs stack-level explanation.

Inspect the time range around the hitch, not the whole trace.

Look for:

* hot main-thread stacks;
* repeated small functions that accumulate;
* expensive formatting, parsing, sorting, diffing, or mapping;
* text measurement;
* image decoding or resizing;
* synchronous disk reads;
* lock waits;
* dispatch waits;
* `MainActor` work during animation;
* app code inside layout, drawing, row construction, update callbacks, or lifecycle callbacks.

Do not optimize a globally hot function if it is not active during the hitch window. The fix must target the user-visible frame miss.

## Signposts for UI Responsiveness

Add signposts when the system trace shows a broad hitch window but the app-specific operation is unclear.

Useful signpost regions include:

* screen load;
* first content render;
* first interaction readiness;
* list data update;
* diff generation;
* row model creation;
* image request/decode/display;
* database fetch and mapping;
* search/filter/sort;
* animation-triggering state change;
* expensive async operations that resume on the main actor.

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

For async work, make sure the signpost interval covers the awaited operation you care about. A signpost around a function that only starts a detached task will not measure the detached work.

Do not signpost every small helper. Instrument user-visible operations and suspected expensive regions.

## What the Agent Can Inspect

When repository access is available, inspect concrete UI responsiveness risks instead of giving generic advice.

Search for SwiftUI lifecycle work near rows and screens:

```sh
rg "\.task\s*\{|\.task\s*\(|\.onAppear\s*\{|onChange\s*\(" .
```

Search for identity churn:

```sh
rg "\.id\(|UUID\(\)|Identifiable|ForEach" .
```

Search for expensive body-adjacent work:

```sh
rg "DateFormatter|NumberFormatter|JSONDecoder|sorted\(|map\(|filter\(|reduce\(|NSAttributedString|Text\(" .
```

Search for image work:

```sh
rg "UIImage|CGImage|Image\(|AsyncImage|decode|resize|thumbnail|Nuke|Kingfisher|SDWebImage" .
```

Search for blocking work:

```sh
rg "Data\(|contentsOf:|FileManager|Keychain|SecItem|DispatchQueue\.main\.sync|semaphore|wait\(|NSLock|os_unfair_lock" .
```

Search for animation and layout-heavy code:

```sh
rg "GeometryReader|PreferenceKey|matchedGeometryEffect|drawingGroup|shadow|blur|mask|clipShape|cornerRadius|animation\(" .
```

Search for broad state and environment updates:

```sh
rg "@EnvironmentObject|@Environment\(|@Observable|ObservableObject|@Published|objectWillChange|\.environment\(|\.environmentObject\(" .
```

Search for main-actor async work that may publish during animation or scrolling:

```sh
rg "@MainActor|MainActor\.run|Task\s*\{|Task\(|Task\.detached|async let|withTaskGroup|await .*map|await .*sort|await .*load|await .*update" .
```

Use matches as leads, not proof. Confirm that the matched code runs during the hitch interval.

## Common Hitch Patterns

### Expensive Row Construction

Signals: scrolling hitches as new rows appear; Time Profiler points to mapping, formatting, image work, layout, or text measurement; SwiftUI Instrument shows repeated row updates.

Fix direction: precompute stable row models outside the frame-critical path, move image decode/resize out of row construction, avoid expensive computed properties in `body`, reduce row dependency scope, and stabilize identity.

### Broad SwiftUI Invalidation

Signals: many unrelated views update after a small state change; body work repeats during animation or scrolling; SwiftUI Instrument points to wide dependency propagation.

Fix direction: move state reads to the smallest view that needs them, split observable models by ownership and update frequency, avoid parent views reading state only needed by children, and route code-level guidance to `swiftui-performance`.

### Unstable Identity

Signals: rows reconstruct instead of update; animations look inconsistent; state resets during list changes; SwiftUI Instrument shows identity churn.

Fix direction: use stable model identifiers, avoid `UUID()` or random IDs during render, avoid changing `.id(...)` as a refresh mechanism, and keep conditional view identity predictable.

### Image Decode on the Frame Path

Signals: hitch appears when images enter the viewport; Time Profiler shows image decoding, resizing, or decompression; memory and CPU spike during scroll.

Fix direction: decode and resize before display when possible, cache appropriately, avoid oversized images, prioritize visible images, and cancel work for invisible cells/views.

Image fixes must balance CPU, memory, disk cache, network, and cancellation. Pre-decoding everything can move the hitch into memory pressure or startup work.

### Layout or Drawing During Animation

Signals: long frame intervals during transitions; Core Animation or Time Profiler points to layout, text measurement, drawing, masks, shadows, blur, or layer updates.

Fix direction: reduce layout complexity on the animated path, avoid repeated text measurement, simplify expensive effects, and avoid animating properties that cause heavy layout or rendering work.

### Lifecycle Callback Work During Scrolling or Transitions

Signals: `.task`, `.onAppear`, `onChange`, or row lifecycle work fires repeatedly during scroll or navigation; Time Profiler shows network setup, decoding, model creation, state publishing, or dependency work near visible hitches.

Fix direction: move work to stable owners, add idempotency and cancellation, prefetch outside the frame-critical path, avoid starting feature work from every row appearance, and coalesce state updates.

### Async Results Flood the Main Actor

Signals: background work appears to finish during animation or scroll; Time Profiler or Swift concurrency traces show many resumes on the main actor; SwiftUI Instrument shows broad updates after async results are published.

Fix direction: batch results, coalesce updates, reduce `MainActor` work, narrow the state that receives updates, apply cancellation, and avoid publishing many small updates into broad root state.

## Decision Rules

* If frames drop but CPU is not high, inspect rendering, layout, blocking, image upload, memory pressure, synchronization, and waits.
* If Time Profiler shows hot code outside the hitch interval, do not treat it as the cause.
* If SwiftUI updates repeat, identify the state dependency before proposing a refactor.
* If rows hitch during scroll, inspect row construction, image work, identity, layout, and lifecycle callbacks.
* If the issue appears only on older devices, profile on the lowest supported device class before claiming a fix.
* If the trace is from Debug or Simulator, label conclusions as tentative.
* If a fix changes user-visible behavior, mention the trade-off.
* If post-background work publishes many results to broad SwiftUI state, treat that as part of the visible hitch investigation when it overlaps scrolling or animation.

## Gotchas

* A dropped frame is a symptom, not a root cause.
* Average FPS can hide rare but painful hitches. Inspect the worst intervals.
* The main thread can be blocked without high CPU.
* SwiftUI `body` evaluation is not automatically the problem; repeated expensive work inside or triggered by updates is the problem.
* `LazyVStack` is not equivalent to cell reuse. Do not recommend replacing `List` blindly.
* For large feeds, compare `List` and `LazyVStack` under the same data and interaction. The right container depends on OS version, row complexity, update pattern, identity stability, and reuse behavior.
* `drawingGroup()` and rasterization-style fixes can move cost or increase memory. Do not suggest them as generic fixes.
* `EquatableView` helps only when equality is cheap and prevents meaningful repeated work.
* Caching can reduce hitches but can also increase memory pressure and later stalls.
* Moving work off the main thread is not sufficient if results are published back to broad SwiftUI state too frequently. Batch, coalesce, or narrow the update scope.
* Disabling animations can hide the symptom without fixing the underlying responsiveness issue.
* Do not call a UI fix successful without re-running the same interaction trace.

## Fix Directions

Connect the fix to the measured cause.

| Evidence                           | Likely fix direction                                                             |
| ---------------------------------- | -------------------------------------------------------------------------------- |
| Hot main-thread stack during hitch | Remove, defer, cache, or move measured work off the frame-critical path          |
| Repeated SwiftUI updates           | Narrow state reads, stabilize identity, reduce broad observable dependencies     |
| Expensive row construction         | Precompute row models, simplify rows, avoid expensive `body` work                |
| Image decode during scroll         | Resize/decode earlier, cache, prioritize visible images, cancel invisible work   |
| Layout/drawing dominates           | Simplify layout/effects, reduce measurement, avoid expensive animated properties |
| Blocked main thread                | Remove synchronous I/O, lock waits, dispatch waits, or dependency cycles         |
| Async resumes flood main actor     | Coalesce updates, batch results, add cancellation, reduce `MainActor` work       |
| Production-only hitches            | Compare cohorts, add signposts, reproduce on affected device/OS/data size        |

Prefer one focused fix at a time. Broad rewrites make before/after evidence harder to trust.

## Validation

Validate with the same scenario that exposed the hitch.

Minimum validation:

* same device class or lowest supported device;
* same build configuration;
* same screen, data, and interaction;
* before/after trace comparison;
* worst hitch interval instead of average FPS only;
* confirmation that the suspected work is reduced or moved out of the frame-critical path.

Useful signals include:

* fewer or shorter hitch intervals;
* shorter main-thread work during the frame window;
* reduced repeated SwiftUI updates;
* lower row construction cost;
* image decode no longer overlapping scroll frames;
* fewer layout/drawing spikes;
* fewer main-actor result bursts during animation or scroll;
* improved production p95/p99 after release.

Also validate that the fix did not change scrolling behavior, animation timing, image quality, loading states, cancellation behavior, memory pressure, accessibility responsiveness, or correctness.

If validation is unavailable, say what is still unproven and what trace should be captured next.

## Output Guidance

When analyzing an animation or scrolling hitch, respond with:

```text
## Symptom
## Interaction window
Scroll / transition / gesture / first render / repeated update / first interaction after launch
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
## Interaction window
## Strongest signal
## What is not proven yet
## Next inspection step
## Suggested fix direction
## Re-measurement plan
```
