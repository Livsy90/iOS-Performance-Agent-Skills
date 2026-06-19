# Tool Selection

Use this reference when the task needs a deeper mapping from symptoms to Instruments templates, MetricKit, XCTest metrics, signposts, or production diagnostics.

This file helps the agent choose the first measurement path. It does not replace deeper domain skills such as `ios-launch-performance`, `swiftui-performance`, `swift-concurrency-performance`, `ios-perceived-performance`, or `swift-runtime-performance`.

## Scope Boundary

This reference covers:

* choosing the first measurement path from a user-visible symptom;
* routing between Instruments templates, XCTest metrics, MetricKit, Organizer, signposts, and production diagnostics;
* deciding which signal would confirm or reject a hypothesis;
* combining profiling signals without over-interpreting them;
* routing to deeper profiling references after the first tool choice is clear.

This reference does not cover:

* deep interpretation of each Instruments template;
* code-level SwiftUI, launch, concurrency, runtime, memory, network, disk, or power fixes;
* full profiling report templates;
* production telemetry architecture;
* tool installation or Xcode setup in depth;
* replacing domain-specific performance skills.

Use this file as the profiling router. Once the cost category is identified, move to the focused reference or skill for deeper reasoning.

## Contents

* [Core Rule](#core-rule)
* [Start From the Symptom](#start-from-the-symptom)
* [Capability and Evidence Check](#capability-and-evidence-check)
* [Tool Selection Matrix](#tool-selection-matrix)
* [Instruments Template Routing](#instruments-template-routing)
* [XCTest Metrics Routing](#xctest-metrics-routing)
* [MetricKit and Organizer Routing](#metrickit-and-organizer-routing)
* [Signpost Routing](#signpost-routing)
* [Production Diagnostics Routing](#production-diagnostics-routing)
* [How to Combine Signals](#how-to-combine-signals)
* [What the Agent Can Inspect](#what-the-agent-can-inspect)
* [Common Wrong Tool Choices](#common-wrong-tool-choices)
* [Boundary With Other References](#boundary-with-other-references)
* [Output Guidance](#output-guidance)

## Core Rule

Choose the tool from the user-visible symptom, not from the optimization idea.

Do not start with “use Time Profiler” or “add caching” by default. First identify what the user experiences:

* launch is slow;
* the UI freezes;
* scrolling hitches;
* a screen loads slowly;
* memory grows;
* the app uses too much battery;
* production metrics regressed;
* a code path became slower after a change.

Then pick the tool that can prove or reject the most likely category of cost.

A good first tool choice answers:

* what category of cost is most likely;
* what signal would confirm or reject the hypothesis;
* what the tool cannot prove;
* what deeper reference or skill should be used next.

## Start From the Symptom

Before choosing a tool, classify the problem.

Ask:

1. Is the symptom local, reproducible, and visible in development?
2. Is it production-only or device/cohort-specific?
3. Is it tied to launch, interaction, scrolling, background work, or a specific operation?
4. Is the app slow because it is doing CPU work, waiting, blocking, allocating, loading data, writing to disk, or rendering too much?
5. Is the issue about diagnosis, verification, or regression protection?
6. Is the user asking to interpret an existing artifact or to design a measurement plan?

Use local Instruments when the issue can be reproduced. Use MetricKit, Organizer, analytics, logs, or crash/performance telemetry when the issue is production-only or varies by device, OS, release, thermal state, network, or data size.

If the user already has an artifact, prioritize interpreting the strongest signal in that artifact over giving a generic profiling plan.

## Capability and Evidence Check

Before recommending a tool, check what evidence can actually be collected.

Minimum context:

* device type and OS version;
* Simulator or real device;
* Debug, Release, or release-like build;
* app version or commit;
* reproducible scenario;
* data size and account state;
* number of runs;
* available artifacts: trace, screenshot, logs, MetricKit payload, Organizer screenshot, XCTest result, CI history.

Do not recommend a tool as immediately runnable unless the environment supports it. If the agent cannot run Instruments, `xctrace`, XCTest, or collect MetricKit payloads directly, provide local steps or ask for the artifact.

If the user only has code, give a profiling plan. If the user has a trace, interpret the strongest signal and avoid generic advice.

## Tool Selection Matrix

| Symptom                                          | Primary path                                | Secondary path                             | Strong signal                                                                           | Next step if confirmed                            |
| ------------------------------------------------ | ------------------------------------------- | ------------------------------------------ | --------------------------------------------------------------------------------------- | ------------------------------------------------- |
| Cold launch is slow                              | App Launch, Time Profiler                   | XCTest launch metric, MetricKit, signposts | long pre-main, app init, root view construction, SDK setup                              | route to `ios-launch-performance`                 |
| First screen appears, but interaction is late    | App Launch, Time Profiler, signposts        | XCTest UI measurement, MetricKit           | work after first frame blocks first input                                               | separate first frame from first interaction       |
| Main thread freezes                              | Hangs, Time Profiler                        | System Trace, signposts                    | main thread busy, blocked, waiting, or lock contention                                  | inspect stack and dependency chain                |
| Scrolling hitches                                | Animation Hitches, Core Animation           | Time Profiler, SwiftUI Instrument          | long frames, expensive layout, drawing, cell/view work                                  | route to SwiftUI or rendering guidance            |
| SwiftUI updates too often                        | SwiftUI Instrument                          | Time Profiler, signposts                   | broad invalidation, repeated body work, identity churn                                  | route to `swiftui-performance`                    |
| CPU spike during an operation                    | Time Profiler                               | XCTest clock/CPU metrics, signposts        | hot stack dominates sample time                                                         | optimize the measured hot path                    |
| Async code consumes CPU or creates task overhead | Time Profiler, Swift Concurrency Instrument | signposts                                  | actor hopping, task fan-out, executor overhead, MainActor bottleneck                    | route to `swift-concurrency-performance`          |
| Async code freezes UI or waits indefinitely      | Hangs, Time Profiler                        | Swift Concurrency Instrument, signposts    | MainActor blocked, actor dependency, semaphore bridge, circular wait                    | inspect wait chain and cancellation/lifetime      |
| Memory grows over time                           | Allocations, VM Tracker                     | Memory Graph, Leaks, XCTest memory metric  | retained generations or growing resident memory                                         | inspect retention and caches                      |
| Suspected retain cycle                           | Memory Graph Debugger                       | Allocations generation analysis            | ownership path keeps object alive                                                       | fix ownership chain                               |
| Too many allocations                             | Allocations                                 | Time Profiler, XCTest memory metric        | high allocation rate or churn in hot path                                               | route to runtime or code-level fix                |
| Disk writes are high                             | File Activity, System Trace                 | MetricKit disk writes                      | repeated writes, main-thread I/O, write amplification                                   | batch, defer, or reduce writes                    |
| Screen waits on network                          | Network instrument, URLSessionTaskMetrics   | signposts, server timing                   | waterfall, duplicate requests, slow TTFB, cache miss                                    | fix request dependency or caching                 |
| Network completes but UI still appears late      | Time Profiler, File Activity, signposts     | Network instrument                         | decode, persistence, image processing, model mapping, main-thread update after response | inspect post-response work                        |
| Battery drain or heat                            | Energy Log, Power Profiler                  | MetricKit, Organizer                       | wakeups, timers, polling, sensors, background work                                      | reduce frequency or stop invisible work           |
| Production regression                            | MetricKit, Organizer                        | local Instruments reproduction             | cohort/release/device-specific metric shift                                             | reproduce locally or add diagnostics              |
| Production hangs or responsiveness regression    | MetricKit, Organizer                        | Hangs, Time Profiler, signposts            | release/cohort hang increase                                                            | reproduce selected flow or add targeted signposts |
| Need regression guard                            | XCTest metrics                              | CI history, MetricKit release trend        | stable before/after metric                                                              | add test threshold or trend monitoring            |

## Instruments Template Routing

Use Instruments for local cause analysis.

### App Launch

Use when the symptom is cold launch, warm launch, first frame, first content, or first interaction.

Inspect:

* pre-main work;
* dynamic library loading;
* static initializers;
* `main`, `AppDelegate`, `SceneDelegate`, SwiftUI `App` initialization;
* root view/controller construction;
* work before first frame;
* work between first frame and first interaction.

Route deeper launch reasoning to `ios-launch-performance`.

### Time Profiler

Use when CPU work is suspected or when another tool shows a time interval that needs stack-level explanation.

Inspect:

* hot stacks on the main thread;
* expensive parsing, decoding, formatting, mapping, layout, rendering, image work;
* repeated small functions that accumulate cost;
* background work that competes with visible work;
* post-network or post-disk work that delays UI after the external wait is done.

Do not use Time Profiler alone to prove network, disk, locks, or actor waiting. Wall-clock delay is not always CPU work.

### Hangs

Use when the UI becomes unresponsive or the main thread stalls.

Inspect:

* busy main thread;
* synchronous disk or network work;
* lock contention;
* long waits;
* dependency cycles;
* actor or queue hops that block visible progress;
* main actor waiting on another actor, task, lock, callback, or queue.

A hang is not always “high CPU.” Treat blocked and waiting states separately.

### System Trace

Use when the issue involves scheduling, thread states, blocking, wakeups, I/O, lock contention, or interactions between multiple threads, queues, actors, and system services.

Inspect:

* thread states;
* waits and wakeups;
* queue handoffs;
* lock contention;
* I/O blocking;
* work that competes with the main thread;
* scheduling behavior that Time Profiler alone does not explain.

Use System Trace when Time Profiler shows symptoms but not enough scheduling or blocking context.

### Animation Hitches and Core Animation

Use when frames are missed during scrolling, transitions, gestures, or animations.

Inspect:

* long frames;
* layout and drawing work;
* image decoding;
* view/layer creation;
* offscreen rendering;
* expensive row construction;
* main-thread work during the frame interval.

Use Time Profiler after identifying the hitch window.

### SwiftUI Instrument

Use when the symptom involves repeated view updates, broad invalidation, unstable identity, expensive body evaluation, or SwiftUI-driven hitches.

Inspect:

* view update frequency;
* dependency changes;
* repeated body work;
* identity churn;
* state reads that invalidate too much UI.

Route code-level fixes to `swiftui-performance`.

### Allocations, Leaks, VM Tracker, Memory Graph

Use Allocations for allocation rate, object lifetime, and generation growth.

Use VM Tracker for resident memory, dirty memory, and virtual memory categories.

Use Leaks for unreachable leaked allocations.

Use Memory Graph Debugger for retain cycles and logical leaks where objects are still strongly referenced but should have been released.

Do not call memory growth a leak until ownership or generation evidence supports it.

### Network, File Activity, Energy, Power

Use Network tools and `URLSessionTaskMetrics` for request timing, duplicate requests, caching, waterfalls, redirects, connection reuse, and payload size.

Use File Activity or System Trace for synchronous I/O, write frequency, database transactions, cache writes, file coordination, and logging volume.

Use Energy Log or Power Profiler for timers, wakeups, polling, sensors, location, Bluetooth, background tasks, repeated network work, and offscreen animations.

`URLSessionTaskMetrics` can help separate DNS, TCP, TLS, request, response, transfer, redirect, cache, and connection reuse behavior.

## XCTest Metrics Routing

Use XCTest performance metrics for regression protection and repeatable local comparisons. They are usually not the first tool for deep diagnosis.

Good candidates:

* launch time with `XCTApplicationLaunchMetric`;
* clock time with `XCTClockMetric`;
* CPU with `XCTCPUMetric`;
* memory with `XCTMemoryMetric`;
* storage with `XCTStorageMetric`;
* signpost intervals with `XCTOSSignpostMetric`;
* model mapping, JSON decoding, image processing, database queries, diff generation, and critical screen setup with stable fixtures.

Prefer XCTest when:

* the operation has stable inputs;
* the scenario can run in CI;
* the team needs a guard against future regressions;
* before/after comparison matters more than root-cause discovery.

Avoid XCTest metrics when:

* the scenario depends on live network;
* inputs are unstable;
* device thermal state dominates;
* the test measures too much unrelated app behavior;
* the result will be treated as a complete diagnosis.

Do not add strict thresholds before establishing variance and baseline behavior. Prefer trend-based checks or measured baselines when CI/device noise is significant.

## MetricKit and Organizer Routing

Use MetricKit and Xcode Organizer when the issue may only appear in production.

Use them to answer:

* Did this release regress?
* Which device or OS cohorts are affected?
* Is the problem rare or frequent?
* Is the issue visible in p95 or p99, not just average?
* Are hangs, launch, memory, disk writes, or battery worse in the field?
* Does the issue correlate with a specific app version?
* Does the regression affect all users or only a device, OS, network, thermal, account, or data-size cohort?

MetricKit and Organizer identify production signals. Instruments explains local causes.

Prefer distributions, percentiles, affected cohorts, and release deltas over averages alone.

When production data points to a regression, do not stop at “MetricKit says it regressed.” Use it to choose a local reproduction path or to add targeted signposts and logging.

## Signpost Routing

Add signposts when system tools show a broad interval but the app-specific operation is unclear.

Use signposts around:

* launch phases;
* auth and routing decisions;
* screen loading;
* data fetch and decode;
* database transactions;
* image processing;
* SwiftUI model preparation;
* diff generation;
* expensive async operations;
* cache reads/writes;
* user interaction latency;
* first content and first interaction readiness.

Good signposts have stable names, useful categories, and clear begin/end boundaries.

A signpost interval is wall-clock time, not CPU time. Use it to define the app-specific region, then use the relevant tool to explain what happened inside it.

Avoid signposting every small function. Instrument user-visible operations and suspected expensive regions.

## Production Diagnostics Routing

Use production diagnostics when local reproduction is missing or incomplete.

Useful sources:

* MetricKit payloads;
* Xcode Organizer metrics;
* app analytics timings;
* custom signpost-derived telemetry;
* server timing;
* `URLSessionTaskMetrics`;
* logs around feature flags, cache state, and data size;
* release, device, OS, locale, network, and account cohort metadata.

Production diagnostics should narrow the search space. They should not be treated as a substitute for a local trace when a local trace can be captured.

Keep production diagnostics privacy-safe and low-cardinality. Avoid logging personal data, tokens, full URLs, request bodies, auth headers, customer identifiers, or high-cardinality raw values.

## How to Combine Signals

Use a primary tool to identify the category of cost, then use a secondary tool to explain it.

Examples:

* Animation Hitches finds the bad frame; Time Profiler explains what ran during that frame.
* MetricKit shows a launch regression in production; App Launch and signposts explain the local launch phase.
* Allocations shows retained generations; Memory Graph explains the retaining path.
* Network instrument shows a waterfall; signposts show which UI state waited for which request.
* Network completes early; Time Profiler and File Activity explain post-response decode, persistence, image, or UI work.
* Hangs shows the main thread waiting; System Trace explains the thread, lock, queue, actor, or I/O dependency.
* XCTest catches a regression; Instruments explains the cause.

Do not combine tools randomly. Each additional signal should answer a specific question.

## What the Agent Can Inspect

When repository access is available, inspect existing measurement hooks and likely profiling targets instead of giving only generic tool advice.

Search for existing profiling markers:

```sh id="n2qeg7"
rg "OSSignposter|os_signpost|beginInterval|endInterval|XCTOSSignpostMetric|Logger|OSLog" .
```

Search for XCTest performance tests:

```sh id="dqksbv"
rg "measure\(|XCTMetric|XCTApplicationLaunchMetric|XCTClockMetric|XCTCPUMetric|XCTMemoryMetric|XCTStorageMetric|XCTOSSignpostMetric" .
```

Search for MetricKit integration:

```sh id="4zhyf9"
rg "MXMetricManager|MXMetricPayload|MXDiagnosticPayload|MetricKit|MXHangDiagnostic|MXDiskWriteExceptionDiagnostic" .
```

Search for network timing hooks:

```sh id="0hdz1h"
rg "URLSessionTaskMetrics|urlSession\(_:task:didFinishCollecting:\)|taskInterval|transactionMetrics|URLSessionTaskTransactionMetrics" .
```

Search for common local profiling targets:

```sh id="nujhr1"
rg "FileManager|Data\(|contentsOf:|DispatchQueue\.main\.sync|Task\.detached|@MainActor|NSCache|UIImage|GeometryReader|\.task\s*\{|\.onAppear" .
```

Search for background and power risks:

```sh id="bgr0z9"
rg "Timer|CADisplayLink|poll|retry|BGTaskScheduler|beginBackgroundTask|CLLocationManager|CMMotionManager|CBCentralManager|AVCaptureSession|URLSessionConfiguration.background" .
```

Use matches as leads, not proof. Confirm the tool choice from the user-visible symptom.

The agent can:

* choose a first measurement path;
* explain why that tool matches the symptom;
* state the signal expected from the tool;
* state what the tool cannot prove;
* route to the next focused reference;
* propose a local profiling plan when no artifact exists;
* interpret available evidence instead of repeating generic tool advice.

The agent cannot reliably:

* prove root cause without the appropriate artifact;
* claim a measurement without a trace, metric, log, benchmark, or user-provided evidence;
* use MetricKit or Organizer as a complete root-cause explanation;
* use XCTest metrics as a full replacement for Instruments;
* infer production impact from one local Debug or Simulator trace;
* treat any single tool as sufficient for all performance symptoms.

## Common Wrong Tool Choices

* Using Time Profiler for every delay, even when the app is waiting on network, disk, locks, or actors.
* Using Debug build traces to make Release performance claims.
* Using Simulator traces to prove scrolling, launch, memory pressure, power, or thermal behavior.
* Using Leaks as the main tool for retain cycles that are still strongly referenced.
* Using XCTest performance tests as a root-cause tool instead of a regression guard.
* Adding brittle XCTest thresholds before establishing baseline variance.
* Using MetricKit averages while ignoring p95, p99, release deltas, and affected cohorts.
* Reading a SwiftUI hitch only as a rendering issue without checking state invalidation and identity.
* Treating allocation spikes as leaks without checking whether memory returns to baseline.
* Treating signpost duration as CPU time.
* Adding signposts after the fact without defining the scenario and expected interval.
* Treating production telemetry as safe by default without checking privacy, cardinality, and data minimization.

## Boundary With Other References

Use this file to choose the first measurement path.

Read `references/time-profiler-and-hangs.md` when the task needs:

* CPU stack interpretation;
* Hangs interpretation;
* main-thread freezes;
* blocked threads;
* synchronous I/O;
* lock contention;
* stack interpretation.

Read `references/animation-hitches-and-swiftui.md` when the symptom is:

* dropped frames;
* scrolling hitches;
* Core Animation;
* SwiftUI Instrument;
* repeated UI updates;
* animation or gesture jank.

Read `references/memory-leaks-and-allocations.md` when the symptom is:

* retained growth;
* leaks;
* retain cycles;
* allocations;
* Memory Graph;
* VM Tracker;
* caches;
* decoded images;
* allocation churn.

Read `references/network-disk-and-power-profiling.md` when the symptom is:

* request waterfall;
* duplicated requests;
* cache misses;
* disk I/O;
* write amplification;
* wakeups;
* battery drain;
* thermal pressure;
* background work.

Read `references/signposts-and-scenarios.md` when the task needs:

* reproducible scenarios;
* signpost naming;
* custom trace regions;
* before/after comparison;
* profiling report structure.

Do not read all references by default. Route from the symptom to the smallest useful next reference.

## Output Guidance

When this reference is used, include the tool choice and why it matches the symptom.

Prefer this shape:

```text id="y8rhq3"
## Tool choice

Primary:
Secondary:
Why this matches the symptom:

## Expected signal

If the hypothesis is correct, the trace should show...

## What this tool cannot prove

...

## Route next

If confirmed, read/use:

## Next step

...
```

Always state what is not proven yet when the available evidence is incomplete.

If the user has no artifact, give a profiling plan. If the user has an artifact, interpret the artifact first.
