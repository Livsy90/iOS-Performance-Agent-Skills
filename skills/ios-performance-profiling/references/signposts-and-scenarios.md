# Signposts and Scenarios

Use this reference when the task needs a reproducible scenario, signpost instrumentation, signpost naming, custom trace regions, before/after comparison, or a profiling report template.

## Contents

- [Core model](#core-model)
- [Reproducible scenario](#reproducible-scenario)
- [What to signpost](#what-to-signpost)
- [Signpost naming](#signpost-naming)
- [Using `OSSignposter`](#using-ossignposter)
- [Using `os_signpost`](#using-os_signpost)
- [Custom trace regions](#custom-trace-regions)
- [Before/after comparison](#beforeafter-comparison)
- [Profiling report template](#profiling-report-template)
- [Common mistakes](#common-mistakes)
- [Validation checklist](#validation-checklist)

## Core model

Signposts make app-specific work visible inside system traces.

Use them when Instruments can show that time was spent somewhere, but the trace does not clearly explain which product operation, screen state, request, cache step, decode step, or rendering phase the user was experiencing.

Signposts do not replace profiling. They add semantic boundaries to profiling.

A good signpost helps answer:

- what operation started and ended;
- which screen, flow, item, or phase it belonged to;
- whether the same operation became faster or slower after a change.

## Reproducible scenario

Define the scenario before collecting traces. Without a stable scenario, before/after comparisons are easy to misread.

Capture:

- device model and OS version;
- app version, commit, scheme, and build configuration;
- environment: production, staging, local mock, or fixture data;
- network, cache, authentication, and app state;
- screen or flow;
- exact steps;
- data size;
- number of runs;

Prefer this format:

```text
Scenario: Open catalog and reach first visible content
Device: iPhone 15 Pro, iOS 18.x
Build: Release, commit <sha>
Data: 500 products, image cache cold, authenticated user
Network: Wi-Fi, staging API
Steps:
1. Clear app state.
2. Launch app.
3. Sign in with test account.
4. Open Catalog tab.
5. Stop when the first product row is visible and tappable.
Runs: 5 cold runs
Primary signal: Catalog.Load signpost duration + Time Profiler stacks
Secondary signal: network waterfall + first interaction timestamp
Success criterion: p50 and p95 improve without higher memory or duplicate requests
```

When the scenario depends on external services, prefer controlled fixtures or mocked responses for local regression tests. Use production metrics for real-world confirmation.

## What to signpost

Signpost meaningful product operations, not every function.

Good candidates:

- launch phases: app initialization, root view construction, first content, first interaction;
- screen loading phases: cache read, network request, decode, model mapping, diff creation, render-ready state;
- expensive user actions: search, filter, checkout, media processing, export, import;
- update boundaries that are hard to identify in SwiftUI or UIKit traces;
- repeated work that may accidentally duplicate.

Avoid signposting tiny helpers unless they are known hot spots. Too many signposts add noise.

## Signpost naming

Use stable names that group comparable operations across runs.

Prefer:

```text
Catalog.Load
Catalog.NetworkFetch
Catalog.BuildViewModels
Catalog.FirstContent
Image.DecodeThumbnail
Launch.RootSceneReady
```

Avoid:

```text
load
start
finish
thing happened
Catalog.Load.12345
Product 98423 loaded
```

Use metadata for dynamic values, not the signpost name.

Prefer:

```text
Name: Catalog.Load
Metadata: category=shoes count=500 cache=cold
```

Avoid:

```text
Name: Catalog.Load.shoes.500.cold
```

Keep metadata useful but small. Do not log personal data, tokens, full sensitive URLs, customer identifiers, or large payloads.

## Using `OSSignposter`

Prefer `OSSignposter` in modern Swift code when available. It gives a structured API for intervals.

```swift
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

Use nested intervals only when they answer a real question. Use events for milestones such as `Catalog.FirstContentVisible`.

## Using `os_signpost`

Use `os_signpost` when working with older code, Objective-C interoperability, or APIs that already use `OSLog` directly.

```swift
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

## Custom trace regions

Custom trace regions are useful when the system trace is too broad.

Use them when:

- launch time is high, but it is unclear which feature initialized;
- Time Profiler shows a shared mapper used by several screens;
- Animation Hitches shows a hitch, but the transition has several phases;
- a network waterfall shows many requests, but only some block visible content.

Example product boundary:

```text
Checkout.Start
Checkout.LoadCart
Checkout.ApplyPromotions
Checkout.RenderSummary
Checkout.FirstInteractionReady
```

Profile the same scenario again and inspect the signpost track alongside Time Profiler, Animation Hitches, Network, File Activity, or System Trace.

## Before/after comparison

Do not compare traces from different scenarios.

Keep stable:

- device and OS;
- build configuration;
- account, cache, and network state;
- data size;
- number of runs.

Compare:

- median and tail values, not only best run;
- signpost duration;
- main-thread time inside the operation;
- allocation growth or retained memory when relevant;
- request count or disk-write count when relevant;
- whether the first useful UI became available earlier.

Use this table in reports:

```md
| Signal | Before | After | Change | Notes |
|---|---:|---:|---:|---|
| Catalog.Load p50 | 820 ms | 510 ms | -310 ms | 5 runs, warm cache |
| Catalog.Load p95 | 1.4 s | 760 ms | -640 ms | Still network-sensitive |
| Main-thread time | 420 ms | 180 ms | -240 ms | Mapping moved off critical path |
```

If only one run exists, say that the result is a signal, not proof.

## Profiling report template

```md
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
- Runs and steps:

## Tools Used

- Primary tool:
- Secondary tools:
- Signposts:

## Evidence

| Signal | Observation | What it proves | What it does not prove |
|---|---|---|---|
| <signal> | <observation> | <proof> | <limits> |

## Hypotheses

1. <Most likely hypothesis> — evidence: <trace, signpost, stack, metric>
2. <Alternative hypothesis> — next check: <what to inspect next>

## Recommendation and Verification

- Change:
- Risk / trade-off:
- Re-run the same scenario:
- Compare these signals:
- Regression guard:
- Not proven yet:
```


## Common mistakes

- Starting Instruments before defining a reproducible scenario.
- Comparing a cold-cache before trace with a warm-cache after trace.
- Changing multiple optimizations at once and then being unable to attribute the improvement.
- Naming signposts with dynamic IDs, which fragments the data.
- Adding signposts around every method instead of meaningful product operations.
- Treating signpost duration as CPU time. It can include waiting, suspension, I/O, locks, actor hops, or network delay.
- Treating one improved run as proof.
- Reporting averages only when p95 or p99 is the real user problem.

## Validation checklist

Before finalizing a profiling answer, check:

- Is the scenario specific enough to reproduce?
- Are device, OS, build configuration, data set, cache state, and run count recorded?
- Are signpost names stable and comparable across runs?
- Does the report separate what is proven from what is only suspected?
- Does the before/after comparison use the same scenario?
- Is the suggested fix tied to a measured signal?
- Is there a re-measurement step?
