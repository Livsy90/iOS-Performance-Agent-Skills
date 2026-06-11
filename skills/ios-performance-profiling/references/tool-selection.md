# Tool Selection

Use this reference when the task needs a deeper mapping from symptoms to Instruments templates, MetricKit, XCTest metrics, signposts, or production diagnostics.

This file helps the agent choose the first measurement path. It does not replace deeper domain skills such as `ios-launch-performance`, `swiftui-performance`, `swift-concurrency-performance`, `ios-perceived-performance`, or `swift-runtime-performance`.

## Contents

- [Core rule](#core-rule)
- [Start from the symptom](#start-from-the-symptom)
- [Capability and evidence check](#capability-and-evidence-check)
- [Tool selection matrix](#tool-selection-matrix)
- [Instruments template routing](#instruments-template-routing)
- [XCTest metrics routing](#xctest-metrics-routing)
- [MetricKit and Organizer routing](#metrickit-and-organizer-routing)
- [Signpost routing](#signpost-routing)
- [Production diagnostics routing](#production-diagnostics-routing)
- [How to combine signals](#how-to-combine-signals)
- [Common wrong tool choices](#common-wrong-tool-choices)
- [Output guidance](#output-guidance)

## Core rule

Choose the tool from the user-visible symptom, not from the optimization idea.

Do not start with “use Time Profiler” or “add caching” by default. First identify what the user experiences:

- launch is slow;
- the UI freezes;
- scrolling hitches;
- a screen loads slowly;
- memory grows;
- the app uses too much battery;
- production metrics regressed;
- a code path became slower after a change.

Then pick the tool that can prove or reject the most likely category of cost.

## Start from the symptom

Before choosing a tool, classify the problem.

Ask:

1. Is the symptom local, reproducible, and visible in development?
2. Is it production-only or device/cohort-specific?
3. Is it tied to launch, interaction, scrolling, background work, or a specific operation?
4. Is the app slow because it is doing CPU work, waiting, blocking, allocating, loading data, writing to disk, or rendering too much?
5. Is the issue about diagnosis, verification, or regression protection?

Use local Instruments when the issue can be reproduced. Use MetricKit, Organizer, analytics, logs, or crash/performance telemetry when the issue is production-only or varies by device, OS, release, thermal state, network, or data size.

## Capability and evidence check

Before recommending a tool, check what evidence can actually be collected.

Minimum context:

- device type and OS version;
- Simulator or real device;
- Debug, Release, or release-like build;
- app version or commit;
- reproducible scenario;
- data size and account state;
- number of runs;
- available artifacts: trace, screenshot, logs, MetricKit payload, Organizer screenshot, XCTest result, CI history.

If the user only has code, give a profiling plan. If the user has a trace, interpret the strongest signal and avoid generic advice.

## Tool selection matrix

| Symptom | Primary path | Secondary path | Strong signal | Next step if confirmed |
|---|---|---|---|---|
| Cold launch is slow | App Launch, Time Profiler | XCTest launch metric, MetricKit, signposts | long pre-main, app init, root view construction, SDK setup | route to `ios-launch-performance` |
| First screen appears, but interaction is late | App Launch, Time Profiler, signposts | XCTest UI measurement, MetricKit | work after first frame blocks first input | separate first frame from first interaction |
| Main thread freezes | Hangs, Time Profiler | System Trace, signposts | main thread busy, blocked, waiting, or lock contention | inspect stack and dependency chain |
| Scrolling hitches | Animation Hitches, Core Animation | Time Profiler, SwiftUI Instrument | long frames, expensive layout, drawing, cell/view work | route to SwiftUI or rendering guidance |
| SwiftUI updates too often | SwiftUI Instrument | Time Profiler, signposts | broad invalidation, repeated body work, identity churn | route to `swiftui-performance` |
| CPU spike during an operation | Time Profiler | XCTest clock/CPU metrics, signposts | hot stack dominates sample time | optimize the measured hot path |
| Async code feels slow | Time Profiler, Swift Concurrency Instrument | signposts, Hangs | actor hopping, MainActor bottleneck, task explosion | route to `swift-concurrency-performance` |
| Memory grows over time | Allocations, VM Tracker | Memory Graph, Leaks, XCTest memory metric | retained generations or growing resident memory | inspect retention and caches |
| Suspected retain cycle | Memory Graph Debugger | Allocations generation analysis | ownership path keeps object alive | fix ownership chain |
| Too many allocations | Allocations | Time Profiler, XCTest memory metric | high allocation rate or churn in hot path | route to runtime or code-level fix |
| Disk writes are high | File Activity, System Trace | MetricKit disk writes | repeated writes, main-thread I/O, write amplification | batch, defer, or reduce writes |
| Screen waits on network | Network instrument, URLSession metrics | signposts, server timing | waterfall, duplicate requests, slow TTFB, cache miss | fix request dependency or caching |
| Battery drain or heat | Energy Log, Power Profiler | MetricKit, Organizer | wakeups, timers, polling, sensors, background work | reduce frequency or stop invisible work |
| Production regression | MetricKit, Organizer | local Instruments reproduction | cohort/release/device-specific metric shift | reproduce locally or add diagnostics |
| Need regression guard | XCTest metrics | CI history, MetricKit release trend | stable before/after metric | add test threshold or trend monitoring |

## Instruments template routing

Use Instruments for local cause analysis.

### App Launch

Use when the symptom is cold launch, warm launch, first frame, first content, or first interaction.

Inspect:

- pre-main work;
- dynamic library loading;
- static initializers;
- `main`, `AppDelegate`, `SceneDelegate`, SwiftUI `App` initialization;
- root view/controller construction;
- work before first frame;
- work between first frame and first interaction.

Route deeper launch reasoning to `ios-launch-performance`.

### Time Profiler

Use when CPU work is suspected or when another tool shows a time interval that needs stack-level explanation.

Inspect:

- hot stacks on the main thread;
- expensive parsing, decoding, formatting, mapping, layout, rendering, image work;
- repeated small functions that accumulate cost;
- background work that competes with visible work.

Do not use Time Profiler alone to prove network, disk, locks, or actor waiting. Wall-clock delay is not always CPU work.

### Hangs

Use when the UI becomes unresponsive or the main thread stalls.

Inspect:

- busy main thread;
- synchronous disk or network work;
- lock contention;
- long waits;
- dependency cycles;
- actor or queue hops that block visible progress.

A hang is not always “high CPU.” Treat blocked and waiting states separately.

### Animation Hitches and Core Animation

Use when frames are missed during scrolling, transitions, gestures, or animations.

Inspect:

- long frames;
- layout and drawing work;
- image decoding;
- view/layer creation;
- offscreen rendering;
- expensive row construction;
- main-thread work during the frame interval.

Use Time Profiler after identifying the hitch window.

### SwiftUI Instrument

Use when the symptom involves repeated view updates, broad invalidation, unstable identity, expensive body evaluation, or SwiftUI-driven hitches.

Inspect:

- view update frequency;
- dependency changes;
- repeated body work;
- identity churn;
- state reads that invalidate too much UI.

Route code-level fixes to `swiftui-performance`.

### Allocations, Leaks, VM Tracker, Memory Graph

Use Allocations for allocation rate, object lifetime, and generation growth.

Use VM Tracker for resident memory and virtual memory categories.

Use Leaks for unreachable leaked allocations.

Use Memory Graph Debugger for retain cycles and logical leaks where objects are still strongly referenced but should have been released.

Do not call memory growth a leak until ownership or generation evidence supports it.

### Network, File Activity, Energy, Power

Use Network tools for request timing, duplicate requests, caching, waterfalls, and payload size.

Use File Activity or System Trace for synchronous I/O, write frequency, database transactions, cache writes, and logging volume.

Use Energy Log or Power Profiler for timers, wakeups, polling, sensors, location, Bluetooth, background tasks, repeated network work, and offscreen animations.

## XCTest metrics routing

Use XCTest performance metrics for regression protection and repeatable local comparisons. They are usually not the first tool for deep diagnosis.

Good candidates:

- launch time with `XCTApplicationLaunchMetric`;
- clock time with `XCTClockMetric`;
- CPU with `XCTCPUMetric`;
- memory with `XCTMemoryMetric`;
- storage with `XCTStorageMetric`;
- signpost intervals with `XCTOSSignpostMetric`;
- model mapping, JSON decoding, image processing, database queries, diff generation, and critical screen setup with stable fixtures.

Prefer XCTest when:

- the operation has stable inputs;
- the scenario can run in CI;
- the team needs a guard against future regressions;
- before/after comparison matters more than root-cause discovery.

Avoid XCTest metrics when:

- the scenario depends on live network;
- inputs are unstable;
- device thermal state dominates;
- the test measures too much unrelated app behavior;
- the result will be treated as a complete diagnosis.

## MetricKit and Organizer routing

Use MetricKit and Xcode Organizer when the issue may only appear in production.

Use them to answer:

- Did this release regress?
- Which device or OS cohorts are affected?
- Is the problem rare or frequent?
- Is the issue visible in p95 or p99, not just average?
- Are hangs, launch, memory, disk writes, or battery worse in the field?
- Does the issue correlate with a specific app version?

MetricKit and Organizer identify production signals. Instruments explains local causes.

When production data points to a regression, do not stop at “MetricKit says it regressed.” Use it to choose a local reproduction path or to add targeted signposts and logging.

## Signpost routing

Add signposts when system tools show a broad interval but the app-specific operation is unclear.

Use signposts around:

- launch phases;
- auth and routing decisions;
- screen loading;
- data fetch and decode;
- database transactions;
- image processing;
- SwiftUI model preparation;
- diff generation;
- expensive async operations;
- cache reads/writes;
- user interaction latency.

Good signposts have stable names, useful categories, and clear begin/end boundaries.

Avoid signposting every small function. Instrument user-visible operations and suspected expensive regions.

## Production diagnostics routing

Use production diagnostics when local reproduction is missing or incomplete.

Useful sources:

- MetricKit payloads;
- Xcode Organizer metrics;
- app analytics timings;
- custom signpost-derived telemetry;
- server timing;
- URLSession task metrics;
- logs around feature flags, cache state, and data size;
- release, device, OS, locale, network, and account cohort metadata.

Production diagnostics should narrow the search space. They should not be treated as a substitute for a local trace when a local trace can be captured.

## How to combine signals

Use a primary tool to identify the category of cost, then use a secondary tool to explain it.

Examples:

- Animation Hitches finds the bad frame; Time Profiler explains what ran during that frame.
- MetricKit shows a launch regression in production; App Launch and signposts explain the local launch phase.
- Allocations shows retained generations; Memory Graph explains the retaining path.
- Network instrument shows a waterfall; signposts show which UI state waited for which request.
- XCTest catches a regression; Instruments explains the cause.

Do not combine tools randomly. Each additional signal should answer a specific question.

## Common wrong tool choices

- Using Time Profiler for every delay, even when the app is waiting on network, disk, locks, or actors.
- Using Debug build traces to make Release performance claims.
- Using Simulator traces to prove scrolling, launch, memory pressure, power, or thermal behavior.
- Using Leaks as the main tool for retain cycles that are still strongly referenced.
- Using XCTest performance tests as a root-cause tool instead of a regression guard.
- Using MetricKit averages while ignoring p95, p99, and affected cohorts.
- Reading a SwiftUI hitch only as a rendering issue without checking state invalidation and identity.
- Treating allocation spikes as leaks without checking whether memory returns to baseline.
- Adding signposts after the fact without defining the scenario and expected interval.

## Output guidance

When this reference is used, include the tool choice and why it matches the symptom.

Prefer this shape:

```text
## Tool choice

Primary:
Secondary:
Why this matches the symptom:

## Expected signal

If the hypothesis is correct, the trace should show...

## What this tool cannot prove

...

## Next step

...
```

Always state what is not proven yet when the available evidence is incomplete.
