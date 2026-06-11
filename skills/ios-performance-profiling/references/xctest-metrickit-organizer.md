# XCTest, MetricKit, and Xcode Organizer

Use this reference when the task involves XCTest performance tests, `XCTApplicationLaunchMetric`, CI regression guards, MetricKit payloads, Xcode Organizer, production regressions, device cohorts, p95, or p99.

## Contents

- [Core model](#core-model)
- [When to use this reference](#when-to-use-this-reference)
- [Local tests vs production signals](#local-tests-vs-production-signals)
- [XCTest performance tests](#xctest-performance-tests)
- [Launch performance tests](#launch-performance-tests)
- [CI regression guards](#ci-regression-guards)
- [MetricKit payloads](#metrickit-payloads)
- [Xcode Organizer](#xcode-organizer)
- [Percentiles and cohorts](#percentiles-and-cohorts)
- [From production signal to local trace](#from-production-signal-to-local-trace)
- [Decision rules](#decision-rules)
- [Common mistakes](#common-mistakes)
- [Validation checklist](#validation-checklist)
- [Output notes](#output-notes)

## Core model

XCTest, MetricKit, and Xcode Organizer answer different parts of the same performance question.

XCTest performance tests are best for repeatable local or CI regression protection. They work well when the app has a controlled scenario, stable inputs, and a measurable operation that should not get slower over time.

MetricKit and Xcode Organizer are production signals. They help identify whether users are seeing regressions in the wild, which releases or devices are affected, and whether the problem appears only in tail latency.

Use them together:

1. XCTest guards known critical paths.
2. MetricKit and Organizer reveal production behavior.
3. Instruments explains local causes after a production or CI signal points to a suspicious area.

Do not treat any one of these as complete evidence by itself.

## When to use this reference

Use this reference when the task mentions:

- XCTest performance tests, `measure {}`, or `XCTMetric`;
- `XCTClockMetric`, `XCTCPUMetric`, `XCTMemoryMetric`, or similar metrics;
- `XCTApplicationLaunchMetric`;
- launch performance tests in UI tests;
- CI thresholds, baselines, or regression guards;
- MetricKit payloads, `MXMetricPayload`, or `MXMetricManager`;
- Xcode Organizer performance reports;
- production regressions after a release;
- affected devices, OS versions, app versions, or build cohorts;
- p95, p99, long-tail latency, or outliers.

Do not use this reference as the primary guide for deep stack interpretation, SwiftUI invalidation, memory ownership paths, network waterfalls, or launch architecture. Route to the narrower reference after the signal identifies the suspected cause.

## Local tests vs production signals

Choose the workflow from the question.

| Need | Prefer | Why |
|---|---|---|
| Guard a known operation | XCTest performance test | repeatable and CI-friendly |
| Guard controlled app launch | `XCTApplicationLaunchMetric` | stable launch metric |
| Diagnose why local work is slow | Instruments | stacks, timelines, waits, allocations |
| Detect release regression | MetricKit or Organizer | real users and real devices |
| Compare affected hardware or OS versions | Organizer or grouped MetricKit data | cohort-level signal |
| Investigate tail latency | MetricKit, Organizer, percentiles | p95/p99 expose bad experiences hidden by averages |

A clean XCTest run does not prove production is healthy. A production regression does not identify the exact code path by itself. Connect both sides with reproducible scenarios and local traces.

## XCTest performance tests

Use XCTest performance tests for controlled, repeatable operations.

Good candidates:

- model mapping;
- JSON decoding;
- database queries with stable fixtures;
- diff generation;
- image processing with fixed images;
- critical screen setup with mocked data;
- expensive deterministic functions.

Weak candidates:

- real network requests;
- server-dependent timing;
- live feature flags;
- broad end-to-end flows with many uncontrolled dependencies;
- scenarios that require manual external state changes.

### Basic pattern

```swift
import XCTest

final class CatalogPerformanceTests: XCTestCase {
    func testCatalogRowModelCreation() {
        let products = ProductFixtures.makeProducts(count: 2_000)

        measure(metrics: [
            XCTClockMetric(),
            XCTCPUMetric(),
            XCTMemoryMetric()
        ]) {
            _ = products.map(CatalogRowModel.init)
        }
    }
}
```

Keep the measured block focused. Put setup outside the measured block unless setup is the thing being measured.

Record device, OS version, build configuration, fixture size, run count, metric type, baseline, and threshold policy. Without that context, the number is hard to compare.

## Launch performance tests

Use `XCTApplicationLaunchMetric` when the task is to guard launch time against regressions.

```swift
import XCTest

final class LaunchPerformanceTests: XCTestCase {
    func testColdLaunchPerformance() {
        measure(metrics: [XCTApplicationLaunchMetric()]) {
            let app = XCUIApplication()
            app.launchArguments = ["--performance-test"]
            app.launch()
        }
    }
}
```

Make the launch scenario stable:

- use explicit launch arguments for performance mode;
- seed or reset app state consistently;
- avoid real network dependency before first screen;
- clear caches only when testing cold-like behavior intentionally;
- separate logged-out launch, logged-in launch, and deep-link launch when they have different startup paths.

A launch performance test can show that a controlled launch scenario regressed. It does not explain why. Use App Launch, Time Profiler, signposts, and the `ios-launch-performance` skill to inspect pre-main work, app initialization, root scene construction, first frame, and first interaction.

## CI regression guards

Use CI performance tests to catch known regressions, not to discover every unknown performance issue.

A useful CI guard has:

- a deterministic scenario;
- stable fixtures;
- a repeatable environment;
- a meaningful metric;
- a clear failure policy;
- enough history to avoid chasing noise.

Avoid failing CI on tiny differences from one run. Performance data is noisy, especially across shared machines, simulator runs, thermal state, and background load.

Prefer policies such as:

- compare against a rolling baseline;
- require a meaningful percentage regression before failing;
- require repeated failure before blocking merge;
- alert first, then enforce after the signal is stable;
- run critical performance tests on dedicated hardware when possible.

When reviewing a proposed performance test, ask whether the operation is performance-sensitive, fixtures are stable, setup is outside the measured block, Release-like configuration is available, and the failure message points to a useful local profiling path.

## MetricKit payloads

Use MetricKit when the issue may only appear in production or needs release-level visibility.

MetricKit is useful for release-level questions: did this version regress, which devices or OS versions are affected, is the issue frequent or rare, and is the regression visible only in p95 or p99?

A typical app registers a MetricKit subscriber and receives payloads from the system.

```swift
import MetricKit

final class MetricSubscriber: NSObject, MXMetricManagerSubscriber {
    func start() {
        MXMetricManager.shared.add(self)
    }

    func didReceive(_ payloads: [MXMetricPayload]) {
        for payload in payloads {
            process(payload)
        }
    }
}
```

Keep this code lightweight. Metric collection should not become a performance problem itself.

When reviewing MetricKit data, separate app version, build number, OS version, device model or class, sample count, median behavior, p95 behavior, p99 behavior, and diagnostics that point to specific failure modes.

Do not jump from a production metric directly to a code fix. Use it to choose a local reproduction and profiling plan.

## Xcode Organizer

Use Xcode Organizer when the task involves production performance trends available from Apple-collected reports.

Organizer is useful when the user asks which release regressed, whether hangs, launches, memory, disk writes, or related signals changed, which devices or OS versions are affected, and whether the issue is broad enough to prioritize.

Treat Organizer as a production dashboard, not a debugger. It can point to the release, cohort, and metric. It usually does not replace a local trace.

## Percentiles and cohorts

Averages can hide the real user experience.

For launch, hangs, screen loading, and responsiveness, inspect tail behavior:

- p50: typical user experience;
- p75: moderately slow users;
- p95: users having noticeably bad experiences;
- p99: severe tail behavior and rare but painful cases.

Compare cohorts before proposing causes: app version, build number, device model, OS version, region when relevant, logged-in state, feature flag or experiment group, and cold versus warm launch when available.

A regression isolated to older devices suggests a different investigation path than a regression across all devices. A p99-only regression may indicate rare blocking, cache misses, migrations, lock contention, networking dependency, or background contention.

## From production signal to local trace

Use this workflow when MetricKit or Organizer shows a regression:

1. Identify the affected metric.
2. Identify app versions, OS versions, and device cohorts.
3. Check whether the regression appears in median, p95, p99, or all of them.
4. Recreate the closest local scenario.
5. Add signposts if the production metric is too broad.
6. Capture the matching Instruments trace.
7. Form one focused hypothesis.
8. Make the smallest fix that addresses the measured cause.
9. Re-run the local scenario.
10. Watch the next production release for confirmation.

Do not assume that the top production metric and the easiest local hotspot are the same problem.

## Decision rules

- Use XCTest when the operation can be controlled and repeated.
- Use `XCTApplicationLaunchMetric` when launch is the scenario being guarded.
- Use MetricKit or Organizer when the problem is production-only or release-specific.
- Use Instruments when the question is why the metric regressed.
- Use signposts when the system metric is too broad to map to app-specific operations.
- Use p95 and p99 when user pain matters more than average behavior.
- Use cohorts before blaming code that affects only one path.
- Use CI thresholds only after the test is stable enough to avoid constant noise.

## Common mistakes

- Treating XCTest performance tests as deep profiling tools.
- Measuring setup accidentally inside `measure {}`.
- Running performance tests with unstable network, remote config, or live backend data.
- Comparing Debug results to Release expectations.
- Treating Simulator results as production truth.
- Failing CI on tiny noisy differences.
- Looking only at averages when the problem is tail latency.
- Ignoring device cohorts and OS version differences.
- Assuming Organizer or MetricKit identifies the exact code path.
- Adding heavy processing to MetricKit callbacks.
- Claiming a launch optimization worked without checking the same scenario before and after.

## Validation checklist

Before finalizing an XCTest, MetricKit, or Organizer answer, check:

- Did you identify whether the signal is local, CI, or production?
- Did you separate regression detection from root-cause diagnosis?
- Did you name the metric being measured?
- Did you include device, OS, build configuration, and data-set assumptions?
- Did you consider p95 or p99 when user pain may be hidden by averages?
- Did you mention cohorts when production data is involved?
- Did you avoid claiming certainty from one run or one dashboard?
- Did you recommend Instruments or signposts when root cause is still unknown?
- Did you propose a repeatable before/after comparison?
- Did you suggest a regression guard only for stable, important scenarios?

## Output notes

For XCTest tasks, return the scenario, metrics, measured-block boundary, input stability rules, and CI usage.

For MetricKit or Organizer tasks, return the production signal, affected cohorts, median/p95/p99 behavior, local reproduction scenario, and the Instruments trace or signposts needed to confirm the cause.

When evidence is incomplete, say what is known, what is not proven yet, and what should be measured next.
