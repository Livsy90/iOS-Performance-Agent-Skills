# Signposts and Scenarios

Use this reference when the task needs a reproducible scenario, signpost instrumentation, signpost naming, custom trace regions, before/after comparison, or a profiling report template.

This file helps the agent make profiling evidence repeatable and app-specific. Signposts do not replace profiling tools; they add semantic boundaries to traces so the agent can connect system-level cost to product operations.

## Scope Boundary

This reference covers:

* reproducible profiling scenarios;
* scenario documentation for before/after comparison;
* signpost instrumentation strategy;
* signpost naming and metadata;
* custom trace regions;
* app-defined milestones;
* before/after profiling reports;
* profiling report structure and validation.

This reference does not cover:

* tool-specific Instruments setup in depth;
* code-level SwiftUI, runtime, memory, network, disk, or launch fixes;
* production monitoring strategy except as validation evidence;
* generic logging architecture;
* analytics/event tracking design;
* replacing Instruments, MetricKit, XCTest metrics, or production telemetry with manual timing.

Use this file when profiling evidence needs clearer scenario boundaries, app-specific trace regions, or a report format. Route root-cause analysis and fixes to the focused performance reference after the measured cost is identified.

## Contents

* [Core Model](#core-model)
* [Reproducible Scenario](#reproducible-scenario)
* [What to Signpost](#what-to-signpost)
* [Signpost Naming](#signpost-naming)
* [Using `OSSignposter`](#using-ossignposter)
* [Using `os_signpost`](#using-os_signpost)
* [Custom Trace Regions](#custom-trace-regions)
* [Custom Timestamps](#custom-timestamps)
* [Before/After Comparison](#beforeafter-comparison)
* [What the Agent Can Inspect](#what-the-agent-can-inspect)
* [Profiling Report Template](#profiling-report-template)
* [Common Mistakes](#common-mistakes)
* [Validation Checklist](#validation-checklist)

## Core Model

Signposts make app-specific work visible inside system traces.

Use them when Instruments can show that time was spent somewhere, but the trace does not clearly explain which product operation, screen state, request, cache step, decode step, persistence step, rendering phase, or readiness milestone the user was experiencing.

Signposts do not replace profiling. They add semantic boundaries to profiling.

A good signpost helps answer:

* what operation started and ended;
* which screen, flow, item class, or phase it belonged to;
* whether the same operation became faster or slower after a change;
* whether the measured interval overlaps CPU work, I/O, network delay, rendering, actor hops, waits, or user-visible readiness.

A signpost interval measures elapsed wall-clock time between begin and end. It may include CPU work, suspension, actor hops, lock waits, disk I/O, network delay, rendering work, or waiting for another subsystem. Use Time Profiler, System Trace, Network, File Activity, Animation Hitches, Allocations, or other tools to explain what happened inside the interval.

Do not treat signpost duration as CPU time.

## Reproducible Scenario

Define the scenario before collecting traces. Without a stable scenario, before/after comparisons are easy to misread.

Capture:

* device model and OS version;
* app version, commit, scheme, and build configuration;
* environment: production, staging, local mock, or fixture data;
* network, cache, authentication, and app state;
* whether the scenario is cold launch, warm launch, first run, post-update, resume, cold cache, warm cache, offline, or fixture-backed;
* screen or flow;
* exact steps;
* data size;
* number of runs;
* primary and secondary signals;
* success criterion.

Prefer this format:

```text id="9y6gkt"
Scenario: Open catalog and reach first visible content
Device: iPhone 15 Pro, iOS 18.x
Build: Release, commit <sha>
Data: 500 products, image cache cold, authenticated user
Network: Wi-Fi, staging API
State: cold app launch, clean app data, cold image cache
Steps:
1. Clear app state.
2. Launch app.
3. Sign in with test account.
4. Open Catalog tab.
5. Stop when the first product row is visible and tappable.
Runs: 5 cold runs
Primary signal: Catalog.Load signpost duration + Time Profiler stacks
Secondary signal: network waterfall + first interaction timestamp
Success criterion: p50 and p95 improve without higher memory, duplicate requests, or worse first interaction readiness
```

When the scenario depends on external services, prefer controlled fixtures or mocked responses for local regression tests. Use production metrics for real-world confirmation.

Do not compare a cold-cache before trace with a warm-cache after trace unless that difference is the thing being tested.

## What to Signpost

Signpost meaningful product operations, not every function.

Good interval candidates:

* launch phases: app initialization, root view construction, first content, first interaction;
* screen loading phases: cache read, network request, decode, model mapping, diff creation, persistence, render-ready state;
* expensive user actions: search, filter, checkout, media processing, export, import;
* update boundaries that are hard to identify in SwiftUI or UIKit traces;
* repeated work that may accidentally duplicate;
* background sync or maintenance work that may compete with foreground responsiveness.

Good event candidates:

* `FirstContentVisible`;
* `FirstInteractionReady`;
* `CacheHit`;
* `CacheMiss`;
* `FallbackShown`;
* `RouteResolved`;
* `UserInputReceived`;
* `VisibleContentUpdated`.

Use events for milestones that do not have a meaningful duration. Use intervals for work with a meaningful start and end.

Avoid signposting tiny helpers unless they are known hot spots. Too many signposts add noise.

Signposts are lightweight when used well, but high-frequency signposts in per-frame, per-row, or tight-loop code can add noise and overhead. Use them around meaningful operations, not every call.

## Signpost Naming

Use stable names that group comparable operations across runs.

Prefer:

```text id="m9h72x"
Catalog.Load
Catalog.NetworkFetch
Catalog.BuildViewModels
Catalog.FirstContent
Image.DecodeThumbnail
Launch.RootSceneReady
```

Avoid:

```text id="ldaooh"
load
start
finish
thing happened
Catalog.Load.12345
Product 98423 loaded
```

Use metadata for dynamic values, not the signpost name.

Prefer:

```text id="tlm2qs"
Name: Catalog.Load
Metadata: category=shoes count=500 cache=cold
```

Avoid:

```text id="vplqup"
Name: Catalog.Load.shoes.500.cold
```

Keep metadata useful, small, and low-cardinality. High-cardinality values such as item IDs, full URLs, query text, user identifiers, customer identifiers, or account identifiers make traces noisy and can expose sensitive data.

Do not log personal data, tokens, full sensitive URLs, customer identifiers, auth headers, request bodies, or large payloads in signpost names or metadata.

Use stable subsystem and category values. Prefer categories such as `Launch`, `Catalog`, `Search`, `Images`, `Persistence`, `Networking`, or `Checkout` over dynamically generated categories.

Use distinct signpost IDs when the same interval name can overlap, such as multiple image decodes, parallel requests, concurrent searches, or repeated row operations. Reusing stable names is good; confusing overlapping intervals is not.

## Using `OSSignposter`

Prefer `OSSignposter` in modern Swift code when available. It gives a structured API for intervals.

```swift id="5q110t"
import OSLog

private let logger = Logger(
    subsystem: "com.example.app",
    category: "Catalog"
)

private let signposter = OSSignposter(logger: logger)

func loadCatalog() async throws -> [ProductViewModel] {
    let id = signposter.makeSignpostID()
    let state = signposter.beginInterval("Catalog.Load", id: id)
    defer { signposter.endInterval("Catalog.Load", state) }

    let products = try await api.fetchProducts()
    return products.map(ProductViewModel.init)
}
```

Use nested intervals only when they answer a real question.

Example nested regions:

```text id="t84cma"
Catalog.Load
  Catalog.CacheRead
  Catalog.NetworkFetch
  Catalog.Decode
  Catalog.BuildViewModels
```

Use events for milestones such as `Catalog.FirstContentVisible` or `Catalog.FirstInteractionReady`.

For async code, make sure the interval covers the awaited operation. A signpost around a function that only starts a `Task {}` or `Task.detached` will measure task creation, not the work performed by that task.

Do not use dynamic names to distinguish overlapping work. Use stable names plus distinct signpost IDs and safe metadata.

## Using `os_signpost`

Use `os_signpost` when working with older code, Objective-C interoperability, C APIs, or code that already uses `OSLog` directly.

```swift id="ulwuo4"
import os.signpost

private let log = OSLog(subsystem: "com.example.app", category: "Search")

func performSearch(query: String) async throws -> [SearchResult] {
    let id = OSSignpostID(log: log)
    os_signpost(.begin, log: log, name: "Search.Query", signpostID: id)
    defer { os_signpost(.end, log: log, name: "Search.Query", signpostID: id) }

    return try await searchService.search(query)
}
```

Add small, non-sensitive metadata only when it helps compare runs. Use `%{public}` only for values that are safe to expose in logs and traces.

Use `%{private}` for sensitive values if they must be logged at all, but prefer avoiding sensitive data in profiling markers.

## Custom Trace Regions

Custom trace regions are useful when the system trace is too broad.

Use them when:

* launch time is high, but it is unclear which feature initialized;
* Time Profiler shows a shared mapper used by several screens;
* Animation Hitches shows a hitch, but the transition has several phases;
* Network shows many requests, but only some block visible content;
* File Activity shows disk work, but the responsible product phase is unclear;
* SwiftUI updates repeat, but the triggering product operation is unclear.

Example product boundary:

```text id="z2sxdw"
Checkout.Start
Checkout.LoadCart
Checkout.ApplyPromotions
Checkout.RenderSummary
Checkout.FirstInteractionReady
```

Profile the same scenario again and inspect the signpost track alongside Time Profiler, Animation Hitches, Network, File Activity, Allocations, VM Tracker, or System Trace.

A custom trace region should help narrow the question. It should not become a second logging system.

## Custom Timestamps

Custom timestamps can be useful when signposts are unavailable or when the app already records product milestones.

Use custom timestamps for:

* app-defined launch milestones;
* first content visibility;
* first interaction readiness;
* product-level loading phases;
* production telemetry where Instruments traces are unavailable.

Caveats:

* custom timestamps are app-defined intervals;
* they may not share the same boundary as Instruments, XCTest metrics, MetricKit, or Organizer;
* they can be affected by where the app records the timestamp;
* they can miss work before or after the app-defined boundary;
* they should not be compared directly with tool-provided metrics unless the boundaries match.

When reporting custom timestamps, name the exact start and end boundary.

Example:

```text id="zm3h2k"
Catalog.FirstContentTime
Start: user tapped Catalog tab
End: first product row visible and tappable
Boundary owner: CatalogViewModel + CatalogView on first visible content callback
```

Do not report a custom timestamp as “launch time”, “render time”, or “first frame” unless its boundary actually matches that metric.

## Before/After Comparison

Do not compare traces from different scenarios.

Keep stable:

* device and OS;
* build configuration;
* account, cache, and network state;
* data size;
* number of runs;
* entry point;
* app state;
* measurement boundary.

Compare:

* median and tail values, not only best run;
* signpost duration;
* main-thread time inside the operation;
* allocation growth or retained memory when relevant;
* request count or disk-write count when relevant;
* wakeups or background duration when relevant;
* whether the first useful UI became available earlier;
* whether first interaction readiness improved or regressed.

Change one meaningful variable at a time when possible. If several changes are shipped together, report attribution confidence as lower.

Use this table in reports:

```md id="pes4au"
| Signal | Before | After | Change | Notes |
|---|---:|---:|---:|---|
| Catalog.Load p50 | 820 ms | 510 ms | -310 ms | 5 runs, warm cache |
| Catalog.Load p95 | 1.4 s | 760 ms | -640 ms | Still network-sensitive |
| Main-thread time | 420 ms | 180 ms | -240 ms | Mapping moved off critical path |
```

If only one run exists, say that the result is a signal, not proof.

## What the Agent Can Inspect

When repository access is available, inspect existing profiling markers and scenario helpers instead of inventing generic instrumentation.

Search for signposts and logging infrastructure:

```sh id="jh8f1e"
rg "OSSignposter|os_signpost|beginInterval|endInterval|emitEvent|Signpost|OSLog|Logger" .
```

Search for profiling markers and timing code:

```sh id="g36mbi"
rg "CFAbsoluteTimeGetCurrent|CACurrentMediaTime|Date\(|measure|metric|trace|firstContent|readyToUse|firstInteraction|firstFrame" .
```

Search for scenario, fixture, or test helpers:

```sh id="rsf8pc"
rg "launchArguments|launchEnvironment|fixture|mock|stub|resetState|clearCache|UITest|XCTest|seedData|testAccount" .
```

Search for possible dynamic signpost names or fragmented timing labels:

```sh id="83f6y7"
rg "beginInterval\(.*\\+|os_signpost\(.*\\+|Logger\(.*\\+|OSLog\(.*\\+" .
```

Search for launch and readiness milestones:

```sh id="v3j1u5"
rg "firstContent|firstInteraction|readyToUse|appReady|rootReady|contentVisible|routeResolved|launchComplete" .
```

Use matches as leads, not proof. Confirm whether names are stable, intervals are correctly paired, metadata is safe, and measurements cover the intended work.

The agent can:

* propose a reproducible scenario;
* identify missing scenario variables;
* suggest meaningful signpost intervals and events;
* improve signpost naming and metadata;
* define before/after comparison signals;
* create a profiling report template;
* call out what remains unproven.

The agent cannot reliably:

* infer CPU cost from signpost duration alone;
* prove improvement from one run;
* compare traces with different cache, network, data, build, or device state;
* decide root cause without the appropriate profiling tool;
* treat manual timestamps as equivalent to tool metrics unless boundaries match.

## Profiling Report Template

```md id="2mtzib"
# Profiling Report: <problem or flow>

## Summary

- Symptom:
- Strongest signal:
- Likely cause:
- Recommended next step:
- Confidence: low / medium / high

## Scenario

- Device / OS:
- App version / commit:
- Build configuration:
- Environment / data set:
- Cache / network state:
- App state / entry point:
- Runs and steps:

## Tools Used

- Primary tool:
- Secondary tools:
- Signposts:
- Custom timestamps:

## Evidence

| Signal | Observation | What it proves | What it does not prove |
|---|---|---|---|
| <signal> | <observation> | <proof> | <limits> |

## Hypotheses

1. <Most likely hypothesis> — evidence: <trace, signpost, stack, metric>
2. <Alternative hypothesis> — next check: <what to inspect next>

## Unknowns / Evidence Needed

- <missing trace, device, scenario, run count, production signal, code path, or tool evidence>

## Recommendation and Verification

- Change:
- Risk / trade-off:
- Re-run the same scenario:
- Compare these signals:
- Regression guard:
- Not proven yet:
```

## Common Mistakes

* Starting Instruments before defining a reproducible scenario.
* Comparing a cold-cache before trace with a warm-cache after trace.
* Comparing different accounts, data sizes, devices, OS versions, or build configurations.
* Changing multiple optimizations at once and then being unable to attribute the improvement.
* Naming signposts with dynamic IDs, which fragments the data.
* Adding signposts around every method instead of meaningful product operations.
* Putting personal data, full URLs, tokens, customer IDs, request bodies, or large payloads into signpost names or metadata.
* Treating signpost duration as CPU time. It can include waiting, suspension, I/O, locks, actor hops, rendering, or network delay.
* Adding a signpost around code that only starts unstructured async work, then assuming it measured the async work.
* Treating custom timestamps as equivalent to Instruments, XCTest, MetricKit, or Organizer metrics without matching boundaries.
* Treating one improved run as proof.
* Reporting averages only when p95 or p99 is the real user problem.
* Adding high-frequency signposts in per-frame, per-row, or tight-loop code without a specific reason.

## Validation Checklist

Before finalizing a profiling answer, check:

* Is the scenario specific enough to reproduce?
* Are device, OS, build configuration, data set, cache state, app state, and run count recorded?
* Are signpost names stable and comparable across runs?
* Are dynamic values stored as small, safe metadata instead of names?
* Are intervals correctly paired with begin and end?
* Do async signposts cover the awaited work rather than only task creation?
* Does the report separate what is proven from what is only suspected?
* Does the before/after comparison use the same scenario?
* Is the suggested fix tied to a measured signal?
* Is there a re-measurement step?
* Did the fix preserve correctness, loading states, cancellation, cache freshness, memory, request count, disk writes, energy behavior, and accessibility responsiveness?
* Are unknowns or missing evidence clearly listed?
