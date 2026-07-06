# Profiling and Validation

Use this reference when the task asks for SwiftUI profiling help or provides performance artifacts such as Instruments traces, `xctrace` output, signpost logs, XCTest benchmark results, MetricKit payloads, screen recordings, memory graphs, console logs, or screenshots.

This is a validation reference for `swiftui-performance`. Use it to confirm or reject a specific SwiftUI performance hypothesis. Do not use it as a replacement for code review, and do not turn every SwiftUI task into a profiling session.

## Contents

* [Core rule](#core-rule)
* [Evidence levels](#evidence-levels)
* [Capability check](#capability-check)
* [Define the scenario](#define-the-scenario)
* [Build and runtime rules](#build-and-runtime-rules)
* [Tool selection](#tool-selection)
* [SwiftUI instrument](#swiftui-instrument)
* [Time Profiler](#time-profiler)
* [Hangs, hitches, and Core Animation](#hangs-hitches-and-core-animation)
* [Allocations](#allocations)
* [Memory graphs and leaks](#memory-graphs-and-leaks)
* [Signposts](#signposts)
* [Temporary SwiftUI debug probes](#temporary-swiftui-debug-probes)
* [`xctrace` command-line workflow](#xctrace-command-line-workflow)
* [XCTest performance tests](#xctest-performance-tests)
* [MetricKit](#metrickit)
* [Production signals](#production-signals)
* [Screen recordings](#screen-recordings)
* [Interpreting user-provided artifacts](#interpreting-user-provided-artifacts)
* [Before/after validation](#beforeafter-validation)
* [Map findings back to SwiftUI refactors](#map-findings-back-to-swiftui-refactors)
* [Response formats](#response-formats)
* [Common mistakes](#common-mistakes)
* [Final principle](#final-principle)

## Core rule

Do not claim that a performance issue was measured unless the task includes actual evidence.

Evidence can come from:

* an Instruments trace;
* `xctrace` output;
* signpost intervals;
* XCTest performance output;
* MetricKit payloads;
* a memory graph;
* a screen recording;
* user-provided timing logs;
* a profiling command the agent actually ran.

If evidence is missing, phrase findings as risks or hypotheses.

Prefer:

```md
This is a likely hot path because the row formats values during `body`.
To confirm it, profile the scroll interaction with the SwiftUI instrument or Time Profiler
and look for long body updates or formatter samples on the main thread.
```

Avoid:

```md
This causes a 200 ms hitch.
```

unless the artifact, benchmark, signpost interval, or command output actually shows that number.

## Evidence levels

Keep evidence labels explicit:

* **Static risk** — code structure suggests a possible issue, but no runtime evidence exists.
* **Hypothesis** — a suspected cause tied to a concrete reproducible scenario.
* **Debug signal** — `_printChanges()`, logs, counters, local timestamps, or temporary probes suggest where to look.
* **Measured local evidence** — Instruments, `xctrace`, XCTest metrics, signposts, memory graphs, or allocation data from a reproducible local scenario.
* **Production evidence** — MetricKit, Xcode Organizer, crash or hang diagnostics, telemetry, monitoring dashboards, or user reports across devices.

Do not promote a static risk into a measured result.

Prefer:

```md
Static risk: this row creates formatted strings during `body`.
Hypothesis: repeated formatting contributes to the scroll hitch.
Validation: profile the scroll path with Time Profiler and check whether formatting appears on the main thread during the hitch window.
```

Avoid:

```md
This formatting is definitely the cause.
```

unless the evidence supports that conclusion.

## Capability check

Before attempting local profiling, check whether the task has:

* macOS host;
* full Xcode installation, not only Command Line Tools;
* selected Xcode path;
* buildable project;
* runnable app target;
* known scheme and configuration;
* available simulator or device;
* reproducible scenario;
* permission to run shell commands;
* permission to create trace or log artifacts.

Useful checks:

```bash
xcode-select -p
xcodebuild -version
xcrun simctl list devices available
xcrun xctrace list templates
xcrun xctrace list devices
```

If any required capability is missing, do not force profiling. Provide local steps for the user and keep findings labeled as hypotheses.

Do not pretend that GUI Instruments was used if no trace was opened or no artifact was provided.

## Define the scenario

Every profiling task needs a reproducible scenario.

Capture:

* target screen;
* start state;
* exact user action;
* expected symptom;
* device or simulator;
* OS version;
* build configuration;
* data size;
* network and cache state;
* number of repetitions.

Good scenarios:

```md
Open Portfolio, scroll the holdings list from top to bottom, tap Load More once,
then continue scrolling for 10 seconds.
```

```md
Type five characters into search with 2,000 local rows loaded and observe whether rows update on every keystroke.
```

```md
Open the detail screen, start the chart animation, then switch tabs twice while the animation is running.
```

Avoid measuring vague interactions such as “the app feels slow.” Convert them into a concrete action first.

## Build and runtime rules

Prefer release-like conditions for validation:

* Release configuration when possible;
* same device class for before/after comparisons;
* same OS version;
* same account or data set;
* same network and cache state;
* same app state before each run;
* multiple runs for noisy metrics.

Debug builds are useful for diagnosis, logging, and `_printChanges()`, but they are not reliable proof of production performance.

Simulator results are useful for quick iteration and code-path discovery. Prefer physical devices for final conclusions about scrolling smoothness, animation hitches, launch time, thermal behavior, and memory pressure.

Low Power Mode can be used as a manual stress signal, not as a formal benchmark.

## Tool selection

Choose the smallest tool that can answer the question.

Use:

* **SwiftUI instrument** — body/update frequency, update scope, update groups, and representable update cost.
* **Time Profiler** — main-thread CPU, expensive app symbols, formatting, sorting, mapping, parsing, image preparation, and synchronous work.
* **Animation Hitches** — skipped frames, scroll jank, visible animation stutter, delayed gestures, and transition hitches.
* **Core Animation instruments** — layer/compositing pressure, offscreen rendering candidates, masks, shadows, blurs, transparency, and large animated surfaces.
* **Allocations** — temporary arrays, strings, formatter creation, render model rebuilds, type erasure, wrapper churn, and repeated image allocations.
* **Memory Graph** — retained view models, retain cycles, retained tasks, closures, publishers, streams, coordinators, delegates, and caches.
* **Signposts** — app-level phase boundaries aligned with profiler timelines.
* **XCTest metrics** — repeatable benchmarks for isolated transformation pipelines, launch metrics, or CI regression guards.
* **MetricKit and production telemetry** — production trends, hangs, launch time, memory, energy, disk, CPU, and diagnostics.
* **Screen recordings** — user-visible symptom definition and scenario creation.

Do not assume every dropped frame is caused by SwiftUI diffing. Rendering, layout, image decoding, main-thread blocking, GPU work, UIKit/AppKit bridges, and background CPU contention can all be involved.

## SwiftUI instrument

Use the SwiftUI instrument when available in the installed Instruments/Xcode version.

It is useful for:

* long view body updates;
* unnecessary updates;
* frequent update groups;
* expensive representable updates;
* correlating state changes with SwiftUI work;
* checking whether dependency narrowing reduced update work.

Inspect:

* views with long `body` updates;
* views updating after unrelated state changes;
* update groups around the target interaction;
* repeated row updates during scrolling or pagination;
* representable updates if the screen embeds UIKit/AppKit components.

Use it with Time Profiler when possible. The SwiftUI instrument can identify the problematic update region; Time Profiler helps identify which app code consumed CPU during that region.

If the SwiftUI template is missing, unsupported, or empty, fall back to Time Profiler, Hangs/Hitches, signposts, and temporary debug probes.

## Time Profiler

Use Time Profiler to answer:

* Is the main thread blocked?
* Which interaction creates the CPU spike?
* Which app symbols dominate sampled time?
* Is `body` indirectly calling expensive app code?
* Are formatters, sorters, mappers, decoders, or computed properties visible in the hot path?
* Is pagination append rebuilding old render data?
* Are row builders, view initializers, or representable updates expensive?

Prioritize app-specific symbols over framework noise.

Common SwiftUI findings:

* sorting inside `body`;
* filtering inside repeated content;
* date or currency formatting per row;
* formatter allocation during view updates;
* image decoding or resizing on the main thread;
* building large render model arrays during rendering;
* broad computed properties read by views;
* excessive per-row action/menu construction;
* expensive UIKit/AppKit representable updates.

Useful Call Tree settings:

* **Separate by Thread** — distinguish main-thread render work from background preparation.
* **Invert Call Tree** — surface leaf functions and app code more quickly.
* **Hide System Libraries** — reduce framework noise when searching for app symbols.

Interpretation rule:

* A high-cost app function on the main thread during the slow interaction is actionable.
* A large amount of SwiftUI framework time is context, not automatically the root cause.
* Look for app code that triggers expensive updates or makes SwiftUI reconcile too much work.

## Hangs, hitches, and Core Animation

Use Hangs, Animation Hitches, Core Animation, or Time Profiler when the symptom is visible stutter, delayed gestures, paused animations, scroll jank, or missed frames.

Check whether the hitch aligns with:

* main-thread blocking work;
* too much layout work;
* expensive drawing;
* image decoding or conversion;
* deep or frequently rebuilt view/layer hierarchies;
* heavy shadows, masks, blurs, overlays, or visual effects;
* repeated state changes during animation;
* list updates during scrolling;
* large transaction or commit phases.

For commit-phase hitches, inspect layout, display, prepare, and commit-related work. Heavy view hierarchy mutations, redundant layout invalidation, image preparation, and deep hierarchies can all contribute.

Do not reduce hitch analysis to “SwiftUI redraws too much” unless the trace shows SwiftUI update work as the dominant cause.

## Allocations

Use Allocations when the symptom suggests memory churn, repeated construction, or allocation-heavy updates.

Look for:

* repeated formatter creation;
* repeated string creation in rows;
* rebuilding large arrays of render models;
* temporary collection churn from sorting, filtering, or grouping;
* image decoding or resizing;
* repeated `AnyView` or wrapper construction in hot paths;
* copy-on-write structures copied during rendering;
* per-row closure-heavy helper objects.

Correlate allocation spikes with signposts or user interactions. Allocation volume alone is not enough; tie it to hitching, CPU spikes, memory pressure, or a regression.

## Memory graphs and leaks

Use memory graphs when the issue is growth, retention, or suspected leaks.

For SwiftUI screens, inspect:

* view models retained after navigation away;
* tasks retaining models after cancellation should have happened;
* closures capturing models, services, or parent views unexpectedly;
* long-lived publishers, notifications, or async streams;
* caches without eviction;
* UIKit/AppKit representables retaining coordinators or delegates;
* image caches retaining decoded images too aggressively.

For structured concurrency, check whether child tasks are tied to the expected lifecycle. For unstructured `Task {}` or `Task.detached`, check whether the task handle is stored and canceled when needed, or whether captured references keep the model alive.

Do not call every retained object a leak. Distinguish expected lifetime, cache retention, delayed release, retain cycle, and unbounded growth.

If the user provides only a memory graph screenshot, state what it proves visually and what requires full graph inspection.

## Signposts

Use signposts to mark app-level phases and align them with profiler timelines.

Good boundaries:

* user action started;
* network response received;
* render models built;
* page appended to state;
* filter applied;
* sort applied;
* search text processed;
* animation started;
* expensive cache lookup started/finished;
* image preparation started/finished.

Modern option:

```swift
import os

private let signposter = OSSignposter(
    subsystem: "com.example.app",
    category: "PortfolioPerformance"
)

@MainActor
func appendNextPage(_ page: HoldingPage) {
    let state = signposter.beginInterval("AppendHoldingPage")
    pages.append(page)
    signposter.endInterval("AppendHoldingPage", state)
}
```

Classic option:

```swift
import os

private let performanceLog = OSLog(
    subsystem: "com.example.app",
    category: "PortfolioPerformance"
)

@MainActor
func appendNextPage(_ page: HoldingPage) {
    os_signpost(.begin, log: performanceLog, name: "AppendHoldingPage")
    pages.append(page)
    os_signpost(.end, log: performanceLog, name: "AppendHoldingPage")
}
```

A signpost around a state mutation marks the app-level phase. It does not measure the full SwiftUI reconciliation, layout, drawing, or rendering cost by itself. Use it to align app events with Instruments timelines.

For async or overlapping operations, prefer signpost IDs:

```swift
import os

private let log = OSLog(subsystem: "com.example.app", category: "Search")

func applySearch(_ query: String) async {
    let id = OSSignpostID(log: log)

    os_signpost(
        .begin,
        log: log,
        name: "ApplySearch",
        signpostID: id,
        "query_length=%d",
        query.count
    )

    defer {
        os_signpost(.end, log: log, name: "ApplySearch", signpostID: id)
    }

    await model.applySearch(query)
}
```

Avoid logging sensitive values in signpost messages.

## Temporary SwiftUI debug probes

Use debug probes only for local diagnosis.

Useful probe:

```swift
var body: some View {
    let _ = Self._printChanges()

    return content
}
```

`_printChanges()` is an underscored diagnostic helper. Use it only as a temporary local debugging probe, preferably under `#if DEBUG`. Do not treat it as production API.

Other temporary probes:

* count row body invocations;
* log render model rebuilds;
* log filtering/sorting duration;
* log duplicate `.onAppear` pagination triggers;
* log page append counts;
* log task cancellation/restart events.

Remove debug probes before production.

## `xctrace` command-line workflow

Use `xctrace` for repeatable command-line profiling when GUI Instruments automation is not necessary.

Start by listing templates and devices:

```bash
xcrun xctrace list templates
xcrun xctrace list devices
```

Record by attaching to a running process:

```bash
xcrun xctrace record \
  --template 'Time Profiler' \
  --time-limit 30s \
  --output /tmp/SwiftUIPerf.trace \
  --attach <pid-or-process-name>
```

Record on a selected device:

```bash
xcrun xctrace record \
  --device '<device-name-or-udid>' \
  --template 'Time Profiler' \
  --time-limit 30s \
  --output /tmp/SwiftUIPerf.trace \
  --attach <pid-or-process-name>
```

Attaching to an already running process is useful for screen-level interactions, scrolling, filtering, pagination, and animation scenarios. For launch or cold-start scenarios, prefer a controlled launch workflow, XCTest launch metrics, or a profiling setup that captures process start.

Open the trace manually if GUI inspection is needed:

```bash
open -a Instruments /tmp/SwiftUIPerf.trace
```

Inspect exportable trace contents first:

```bash
xcrun xctrace export \
  --input /tmp/SwiftUIPerf.trace \
  --toc
```

Then export specific tables using the schema or path shown in the table of contents. Do not hardcode export paths across Xcode versions; inspect the `--toc` output first.

Template names, options, and export schemas vary by Xcode version. Use:

```bash
xcrun xctrace help record
xcrun xctrace help export
```

when exact command syntax is uncertain.

## XCTest performance tests

Use XCTest performance tests for repeatable local benchmarks of isolated app code.

Good candidates:

* render model generation;
* filtering;
* sorting;
* grouping;
* formatting pipelines;
* pagination state updates;
* cache lookups;
* pure data transformations;
* launch metrics with `XCTApplicationLaunchMetric`;
* UI metrics when the project and toolchain support a stable UI automation scenario.

Example for pure work:

```swift
final class PortfolioRenderingTests: XCTestCase {
    func testRenderModelGenerationPerformance() {
        let positions = TestFixtures.largePositionSet(count: 5_000)

        measure(metrics: [XCTClockMetric(), XCTCPUMetric(), XCTMemoryMetric()]) {
            _ = PortfolioRenderModelBuilder.makeRows(from: positions)
        }
    }
}
```

Example for launch:

```swift
final class LaunchPerformanceTests: XCTestCase {
    func testLaunchPerformance() throws {
        measure(metrics: [XCTApplicationLaunchMetric(waitUntilResponsive: true)]) {
            XCUIApplication().launch()
        }
    }
}
```

Use XCTest metrics to guard regressions in CI, but do not treat microbenchmarks as complete proof of UI smoothness. They isolate code costs; Instruments and real interactions are still needed for frame-time and rendering issues.

Use XCTest UI metrics only when the project has a stable UI automation scenario. They are useful for regression guarding, but Instruments is still better for root-cause analysis.

## MetricKit

Use MetricKit for production-level performance signals.

MetricKit can help prioritize investigation by showing trends in areas such as:

* app launch time;
* hangs;
* responsiveness;
* memory;
* CPU;
* disk writes;
* energy;
* crash diagnostics.

MetricKit is not a replacement for local profiling. It usually tells the agent where to investigate, not exactly which SwiftUI line to change.

Important rules:

* Treat MetricKit payloads as production evidence, but usually aggregated and delayed evidence.
* Correlate MetricKit trends with app versions, device classes, OS versions, and feature rollouts.
* Use local profiling to reproduce and root-cause the issue.
* Do not infer a specific SwiftUI cause from MetricKit alone unless the payload includes enough diagnostic context.
* Use the MetricKit API shape supported by the project’s deployment target and Xcode version.

Older code may receive `MXMetricPayload` and `MXDiagnosticPayload`. Newer reporting flows may expose different report types. Keep the analysis focused on the metric meaning, app version, device/OS dimensions, and diagnostic context.

Minimal subscriber shape:

```swift
import MetricKit

final class MetricsSubscriber: NSObject, MXMetricManagerSubscriber {
    func start() {
        MXMetricManager.shared.add(self)
    }

    func stop() {
        MXMetricManager.shared.remove(self)
    }

    func didReceive(_ payloads: [MXMetricPayload]) {
        // Persist or upload aggregated metrics for analysis.
    }

    func didReceive(_ payloads: [MXDiagnosticPayload]) {
        // Persist or upload diagnostics such as hangs or crashes.
    }
}
```

Do not block launch or the main thread while processing payloads. Serialize and upload later if necessary.

## Production signals

Use Xcode Organizer or production monitoring dashboards to identify patterns before deep local profiling.

Check:

* app version where regression started;
* device families affected;
* OS versions affected;
* launch type or launch phase if available;
* hang rate trends;
* memory pressure trends;
* animation hitches or responsiveness trends if exposed by the production tools;
* disk, CPU, or energy trends if relevant;
* whether the issue is isolated to a feature rollout.

Use production signals to choose the local scenario to reproduce.

Do not use aggregate production metrics as the only proof that a specific code refactor fixed the issue. Confirm with before/after local measurements when possible, then watch production trends after release.

## Screen recordings

A screen recording is useful evidence for user-visible symptoms, but it rarely proves root cause.

From a screen recording, the agent may infer:

* where the symptom occurs;
* whether it is during scrolling, navigation, typing, loading, or animation;
* whether the issue looks like hitching, delayed input, blank content, repeated loading, or layout instability;
* which scenario to profile.

Do not infer exact frame time, CPU cost, memory pressure, or SwiftUI invalidation cause from a recording alone.

Use the recording to create a profiling scenario.

## Interpreting user-provided artifacts

When the user provides artifacts, analyze them directly and state what they prove.

For Instruments screenshots:

* identify selected time range;
* identify selected instrument or lane;
* read visible call tree entries;
* distinguish app code from framework code;
* ask for or suggest exporting the selected call tree if details are missing.

For `.trace` files:

* try to inspect or export with `xctrace` if available;
* if unavailable, provide local export steps;
* prefer focused exports over huge raw dumps.

For signpost logs:

* group intervals by operation;
* compare min, median, and max if multiple runs exist;
* align intervals with user actions;
* look for variance and outliers.

For XCTest output:

* identify metric type;
* compare baseline and current values;
* check standard deviation or variance if available;
* avoid overreacting to one noisy run.

For MetricKit JSON:

* identify payload type;
* inspect app version, device, and OS dimensions if present;
* distinguish metric payloads from diagnostic payloads;
* look for repeated patterns rather than single isolated events.

For memory graphs:

* identify retained roots and ownership paths;
* distinguish expected retention from cycles;
* check whether navigation away should have released the object.

## Before/after validation

A good validation loop:

1. Define one scenario.
2. Capture baseline.
3. Form one hypothesis.
4. Apply one targeted refactor.
5. Re-run the same scenario.
6. Compare the same metrics.
7. Keep or revert the change based on evidence.

Avoid changing many variables at once.

For before/after reports, include:

```md
## Scenario

## Baseline evidence

## Hypothesis

## Change made

## New evidence

## Interpretation

## Remaining risks
```

## Map findings back to SwiftUI refactors

Use profiling results to choose targeted fixes.

Map findings to refactors:

* **Long body updates or broad SwiftUI update groups** — inspect Observation dependency scope, identity, and body cost.
* **Formatter, sorter, mapper, parser, or render-model-builder samples on the main thread** — move preparation out of `body`, row initializers, and repeated rendering paths.
* **Pagination append spike** — inspect flat append behavior, page boundaries, row cost, duplicate triggers, and final main-actor state mutation.
* **Repeated row updates after unrelated state changes** — inspect broad parent reads, environment reads, `ObservableObject` object-level invalidation, and dependency islands.
* **Per-row geometry or layout work** — inspect `GeometryReader`, `PreferenceKey`, layout feedback, custom layouts, and row modifier chains.
* **Compositing or animation hitches** — inspect shadows, masks, blurs, materials, overlays, animation scope, transitions, and layout-affecting animations.
* **Allocation spikes during scrolling or filtering** — inspect temporary arrays, strings, formatter creation, type erasure, closure-heavy rows, and render model rebuilds.
* **Retained view models after navigation** — inspect tasks, closures, publishers, notifications, `AsyncSequence` loops, delegates, coordinators, and caches.
* **MetricKit hang or responsiveness trend** — reproduce locally with a concrete scenario, signposts, and Time Profiler or Hangs/Animation Hitches.
* **Screen recording shows visible jank but no trace exists** — convert the recording into a reproducible scenario and choose the smallest relevant profiling tool.

Do not apply a large architectural refactor when the evidence points to a local row, formatter, image, layout, or animation issue.

## Response formats

When the user asks for profiling help, answer with:

1. Reproducible scenario.
2. Hypothesis.
3. Tool choice.
4. Exact command or local steps when possible.
5. What to look for.
6. How to interpret findings.
7. Refactor candidates.
8. How to compare before and after.

When the user provides profiling artifacts, answer with:

1. What the artifact shows.
2. What is measured vs inferred.
3. Most likely bottleneck.
4. Supporting evidence from the artifact.
5. What remains uncertain.
6. Suggested refactor or next diagnostic step.
7. Validation plan.

## Common mistakes

Avoid these mistakes:

* profiling a vague scenario;
* comparing Debug before with Release after;
* using simulator-only results as final proof of device smoothness;
* claiming SwiftUI diffing is the cause without trace evidence;
* treating MetricKit as a local profiler;
* treating screen recordings as CPU evidence;
* treating `_printChanges()` output as production measurement;
* adding signposts but forgetting to align them with profiler timelines;
* reading only framework symbols and ignoring app code;
* optimizing a microbenchmark while the real issue is layout, rendering, or scheduling;
* changing architecture before confirming the hot path;
* claiming numeric savings without a measured before/after comparison.

## Final principle

Profiling should make a SwiftUI hypothesis falsifiable.

Start with a concrete symptom, collect the smallest useful evidence, make one targeted change, and compare the same scenario before and after.
