# XCTest, MetricKit, and Xcode Organizer

Use this reference when the task involves XCTest performance tests, `XCTApplicationLaunchMetric`, CI regression guards, MetricKit payloads, Xcode Organizer, production regressions, device cohorts, p95, or p99.

This file helps the agent separate controlled regression measurement from production performance signals. XCTest, MetricKit, and Xcode Organizer can identify regressions, but they do not replace root-cause profiling.

## Scope Boundary

This reference covers:

* XCTest performance metrics and CI regression guards;
* `XCTApplicationLaunchMetric` for controlled launch tests;
* MetricKit payload interpretation at a routing level;
* Xcode Organizer production performance trends;
* percentiles, cohorts, release comparisons, and production-to-local investigation flow;
* deciding when to use XCTest, MetricKit, Organizer, Instruments, or signposts together.

This reference does not cover:

* deep Instruments stack interpretation;
* code-level SwiftUI, launch, memory, network, disk, runtime, or concurrency fixes;
* full MetricKit ingestion/backend architecture;
* privacy/legal design for telemetry;
* replacing local traces with production dashboards;
* generic analytics architecture.

Use this file when the task is about regression detection, production signal interpretation, or controlled performance guards. Route to the narrower profiling reference after the signal identifies the suspected cause.

## Contents

* [Core Model](#core-model)
* [When to Use This Reference](#when-to-use-this-reference)
* [Local Tests vs Production Signals](#local-tests-vs-production-signals)
* [XCTest Performance Tests](#xctest-performance-tests)
* [Launch Performance Tests](#launch-performance-tests)
* [CI Regression Guards](#ci-regression-guards)
* [MetricKit Payloads](#metrickit-payloads)
* [Xcode Organizer](#xcode-organizer)
* [Percentiles and Cohorts](#percentiles-and-cohorts)
* [From Production Signal to Local Trace](#from-production-signal-to-local-trace)
* [What the Agent Can Inspect](#what-the-agent-can-inspect)
* [Decision Rules](#decision-rules)
* [Common Mistakes](#common-mistakes)
* [Validation Checklist](#validation-checklist)
* [Output Notes](#output-notes)

## Core Model

XCTest, MetricKit, and Xcode Organizer answer different parts of the same performance question.

XCTest performance tests are best for repeatable local or CI regression protection. They work well when the app has a controlled scenario, stable inputs, and a measurable operation that should not get slower over time.

MetricKit and Xcode Organizer are production signals. They help identify whether users are seeing regressions in the wild, which releases or devices are affected, and whether the problem appears only in tail latency.

Use them together:

1. XCTest guards known critical paths.
2. MetricKit and Organizer reveal production behavior.
3. Instruments explains local causes after a production or CI signal points to a suspicious area.

Do not treat any one of these as complete evidence by itself.

An XCTest performance test measures one controlled scenario under test conditions. It should not be generalized to all devices, production data sizes, network states, thermal states, or app states unless those conditions are represented.

A production dashboard can show that a release regressed. It usually does not identify the exact code path. Use production data to choose a reproduction path, add targeted signposts, or prioritize local profiling.

## When to Use This Reference

Use this reference when the task mentions:

* XCTest performance tests, `measure {}`, or `XCTMetric`;
* `XCTClockMetric`, `XCTCPUMetric`, `XCTMemoryMetric`, `XCTStorageMetric`, or similar metrics;
* `XCTOSSignpostMetric`;
* `XCTApplicationLaunchMetric`;
* launch performance tests in UI tests;
* CI thresholds, baselines, variance, or regression guards;
* MetricKit payloads, `MXMetricPayload`, `MXDiagnosticPayload`, reports, or `MXMetricManager`;
* Xcode Organizer performance reports;
* production regressions after a release;
* affected devices, OS versions, app versions, builds, or cohorts;
* p50, p75, p95, p99, long-tail latency, or outliers.

Do not use this reference as the primary guide for deep stack interpretation, SwiftUI invalidation, memory ownership paths, network waterfalls, disk I/O, hangs, animation hitches, or launch architecture. Route to the narrower reference after the signal identifies the suspected cause.

## Local Tests vs Production Signals

Choose the workflow from the question.

| Need                                     | Prefer                              | Why                                               |
| ---------------------------------------- | ----------------------------------- | ------------------------------------------------- |
| Guard a known operation                  | XCTest performance test             | Repeatable and CI-friendly                        |
| Guard controlled app launch              | `XCTApplicationLaunchMetric`        | Stable launch metric for a defined scenario       |
| Diagnose why local work is slow          | Instruments                         | Stacks, timelines, waits, allocations             |
| Detect release regression                | MetricKit or Organizer              | Real users and real devices                       |
| Compare affected hardware or OS versions | Organizer or grouped MetricKit data | Cohort-level signal                               |
| Investigate tail latency                 | MetricKit, Organizer, percentiles   | p95/p99 expose bad experiences hidden by averages |

A clean XCTest run does not prove production is healthy. A production regression does not identify the exact code path by itself. Connect both sides with reproducible scenarios, signposts, and local traces.

## XCTest Performance Tests

Use XCTest performance tests for controlled, repeatable operations.

Good candidates:

* model mapping;
* JSON decoding;
* database queries with stable fixtures;
* diff generation;
* image processing with fixed images;
* critical screen setup with mocked data;
* expensive deterministic functions;
* signposted app-specific operations with stable boundaries.

Weak candidates:

* real network requests;
* server-dependent timing;
* live feature flags;
* broad end-to-end flows with many uncontrolled dependencies;
* scenarios that require manual external state changes;
* tests whose measured block includes unrelated setup by accident.

### Basic Pattern

```swift id="nbys0f"
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

If the first iteration includes one-time initialization, decide whether that cost is part of the scenario. Otherwise warm up explicitly or separate first-run cost from steady-state cost.

Record:

* device;
* OS version;
* Xcode version when relevant;
* simulator/runtime when relevant;
* build configuration;
* fixture size;
* run count;
* metric type;
* baseline;
* variance;
* threshold policy.

Without that context, the number is hard to compare.

## Launch Performance Tests

Use `XCTApplicationLaunchMetric` when the task is to guard launch time against regressions.

```swift id="7y2pmr"
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

Use `waitUntilResponsive: true` when the guarded boundary should include launch-to-responsiveness, not only process/app startup.

```swift id="bta5zm"
import XCTest

final class LaunchPerformanceTests: XCTestCase {
    func testLaunchUntilResponsivePerformance() {
        measure(metrics: [
            XCTApplicationLaunchMetric(waitUntilResponsive: true)
        ]) {
            let app = XCUIApplication()
            app.launchArguments = ["--performance-test"]
            app.launch()
        }
    }
}
```

Keep the measured boundary consistent across before/after comparisons.

Make the launch scenario stable:

* use explicit launch arguments for performance mode;
* seed or reset app state consistently;
* avoid real network dependency before first screen;
* clear caches only when testing cold-like behavior intentionally;
* separate logged-out launch, logged-in launch, deep-link launch, notification launch, and post-update launch when they have different startup paths;
* record whether the scenario is cold launch, warm launch, resume, first run, post-update, or cache-controlled.

A launch performance test can show that a controlled launch scenario regressed. It does not explain why. Use App Launch, Time Profiler, signposts, and the `ios-launch-performance` skill to inspect pre-main work, app initialization, root scene construction, first frame, and first interaction.

Do not treat `XCTApplicationLaunchMetric`, custom first-content timestamps, MetricKit launch metrics, and first-interaction readiness as interchangeable. They measure different boundaries.

## CI Regression Guards

Use CI performance tests to catch known regressions, not to discover every unknown performance issue.

A useful CI guard has:

* a deterministic scenario;
* stable fixtures;
* a repeatable environment;
* a meaningful metric;
* a clear failure policy;
* enough history to avoid chasing noise.

Avoid failing CI on tiny differences from one run. Performance data is noisy, especially across shared machines, simulator runs, thermal state, background load, Xcode versions, and OS runtimes.

Establish variance before enforcing a threshold. Store baseline history with device, OS, Xcode, simulator/runtime, build configuration, fixture size, and run count.

Prefer policies such as:

* compare against a rolling baseline;
* require a meaningful percentage regression before failing;
* require repeated failure before blocking merge;
* alert first, then enforce after the signal is stable;
* run critical performance tests on dedicated hardware when possible;
* track trends separately from hard merge blocking.

If tests run on shared CI hardware or Simulator, prefer alerting or trend monitoring before hard blocking.

When reviewing a proposed performance test, ask whether:

* the operation is performance-sensitive;
* fixtures are stable;
* setup is outside the measured block;
* one-time initialization is intentionally included or excluded;
* Release-like configuration is available;
* the failure message points to a useful local profiling path.

## MetricKit Payloads

Use MetricKit when the issue may only appear in production or needs release-level visibility.

MetricKit is useful for release-level questions:

* did this version regress;
* which devices or OS versions are affected;
* whether the issue is frequent or rare;
* whether the regression is visible only in p95 or p99;
* whether diagnostics point to hangs, crashes, disk writes, CPU, memory, launch, or responsiveness issues.

A typical app registers a MetricKit subscriber and receives payloads from the system.

```swift id="0p7qtu"
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

MetricKit API shapes vary by OS and Xcode version. Some projects receive `MXMetricPayload` / `MXDiagnosticPayload`; newer code may use report-based APIs where available. Keep the reference focused on metric meaning, app version, device/OS cohort, sample count, and percentile behavior rather than one API shape.

MetricKit processing should be lightweight, nonblocking, privacy-safe, and preferably off the user-visible path. Avoid heavy parsing, synchronous disk writes, network uploads, or large payload transformation inside the callback.

When reviewing MetricKit data, separate:

* app version;
* build number;
* OS version;
* device model or class;
* sample count;
* median behavior;
* p95 behavior;
* p99 behavior;
* diagnostics that point to specific failure modes;
* whether the issue affects all users or a specific cohort.

Do not jump from a production metric directly to a code fix. Use it to choose a local reproduction and profiling plan.

## Xcode Organizer

Use Xcode Organizer when the task involves production performance trends available from Apple-collected reports.

Organizer is useful when the user asks:

* which release regressed;
* whether hangs, launches, memory, disk writes, or related signals changed;
* which devices or OS versions are affected;
* whether the issue is broad enough to prioritize;
* whether production behavior differs from local testing.

Treat Organizer as a production dashboard, not a debugger. It can point to the release, cohort, and metric. It usually does not replace a local trace.

Organizer data availability depends on Apple-collected reports, user/device conditions, release adoption, and sample volume. Treat small samples and missing cohorts cautiously.

## Percentiles and Cohorts

Averages can hide the real user experience.

For launch, hangs, screen loading, and responsiveness, inspect tail behavior:

* **p50** — typical user experience;
* **p75** — moderately slow users;
* **p95** — users having noticeably bad experiences;
* **p99** — severe tail behavior and rare but painful cases.

Compare cohorts before proposing causes:

* app version;
* build number;
* device model or class;
* OS version;
* region when relevant;
* logged-in state;
* feature flag or experiment group;
* cold versus warm launch when available;
* data-size cohort when relevant.

Include data-size cohorts when they affect performance:

* number of accounts;
* feed size;
* cached items;
* database size;
* image count;
* enabled feature set;
* offline data volume.

A regression isolated to older devices suggests a different investigation path than a regression across all devices. A p99-only regression may indicate rare blocking, cache misses, migrations, lock contention, networking dependency, background contention, or large per-user data.

Do not interpret p95 or p99 without sample count. A scary p99 from a tiny cohort may be less actionable than a moderate p95 regression across a large cohort.

## From Production Signal to Local Trace

Use this workflow when MetricKit or Organizer shows a regression:

1. Identify the affected metric.
2. Identify app versions, builds, OS versions, and device cohorts.
3. Check sample count and confidence.
4. Check whether the regression appears in median, p95, p99, or all of them.
5. Identify whether data-size, account state, feature flag, experiment, region, or launch mode may define the affected cohort.
6. Recreate the closest local scenario.
7. Add signposts if the production metric is too broad.
8. Capture the matching Instruments trace.
9. Form one focused hypothesis.
10. Make the smallest fix that addresses the measured cause.
11. Re-run the local scenario.
12. Watch the next production release for confirmation.

Do not assume that the top production metric and the easiest local hotspot are the same problem.

If local reproduction fails, add targeted signposts, lightweight diagnostics, or cohort metadata for the next release instead of guessing from the dashboard.

## What the Agent Can Inspect

When repository access is available, inspect existing tests, metrics, and production hooks before giving generic advice.

Search for XCTest performance tests:

```sh id="b71dot"
rg "measure\(|XCTMetric|XCTApplicationLaunchMetric|XCTClockMetric|XCTCPUMetric|XCTMemoryMetric|XCTStorageMetric|XCTOSSignpostMetric" .
```

Search for launch test setup:

```sh id="e6emoj"
rg "XCUIApplication\(|launchArguments|launchEnvironment|--performance-test|resetState|clearCache|seedData" .
```

Search for MetricKit integration:

```sh id="l7jhe9"
rg "MetricKit|MXMetricManager|MXMetricManagerSubscriber|MXMetricPayload|MXDiagnosticPayload|MXHangDiagnostic|MXDiskWriteExceptionDiagnostic|MXCrashDiagnostic" .
```

Search for custom production timings:

```sh id="6ynqt2"
rg "firstContent|firstInteraction|readyToUse|appReady|duration|latency|p95|p99|percentile|cohort|release" .
```

Search for CI performance configuration:

```sh id="8gzj4m"
rg "performance|baseline|threshold|xcodebuild|test-without-building|resultBundlePath|xcresult|CI" .
```

Use matches as leads, not proof. Confirm the measured boundary, sample count, device/OS cohort, and metric meaning.

The agent can:

* identify whether the question is local, CI, or production;
* recommend an XCTest metric for a controlled scenario;
* suggest a launch performance test boundary;
* review CI guard stability;
* interpret MetricKit or Organizer signals at a routing level;
* identify affected cohorts and missing context;
* propose a production-to-local investigation plan.

The agent cannot reliably:

* infer root cause from MetricKit or Organizer alone;
* treat one XCTest run as proof of production behavior;
* compare metrics with different boundaries;
* use a CI performance test as a replacement for Instruments;
* claim a regression without sample count, baseline, or comparable scenario;
* decide telemetry privacy requirements without product/legal context.

## Decision Rules

* Use XCTest when the operation can be controlled and repeated.
* Use `XCTApplicationLaunchMetric` when launch is the scenario being guarded.
* Use `waitUntilResponsive: true` when the guarded boundary should include responsiveness.
* Use MetricKit or Organizer when the problem is production-only or release-specific.
* Use Instruments when the question is why the metric regressed.
* Use signposts when the system metric is too broad to map to app-specific operations.
* Use p95 and p99 when user pain matters more than average behavior.
* Use sample count and cohorts before prioritizing a tail regression.
* Use cohorts before blaming code that affects only one path.
* Use CI thresholds only after the test is stable enough to avoid constant noise.
* Use targeted diagnostics in the next release when local reproduction is not possible.

## Common Mistakes

* Treating XCTest performance tests as deep profiling tools.
* Measuring setup accidentally inside `measure {}`.
* Treating one-time initialization as steady-state cost without deciding that boundary explicitly.
* Running performance tests with unstable network, remote config, or live backend data.
* Comparing Debug results to Release expectations.
* Treating Simulator results as production truth.
* Failing CI on tiny noisy differences.
* Adding strict thresholds before establishing baseline variance.
* Looking only at averages when the problem is tail latency.
* Interpreting p95 or p99 without sample count.
* Ignoring device cohorts, OS version differences, app version, build, and data-size cohorts.
* Assuming Organizer or MetricKit identifies the exact code path.
* Adding heavy processing to MetricKit callbacks.
* Comparing XCTest launch metrics, custom first-content timestamps, and MetricKit launch metrics as if they shared the same boundary.
* Claiming a launch optimization worked without checking the same scenario before and after.

## Validation Checklist

Before finalizing an XCTest, MetricKit, or Organizer answer, check:

* Did you identify whether the signal is local, CI, or production?
* Did you separate regression detection from root-cause diagnosis?
* Did you name the metric being measured?
* Did you define the measured boundary?
* Did you include device, OS, build configuration, and data-set assumptions?
* Did you consider p95 or p99 when user pain may be hidden by averages?
* Did you include sample count when interpreting tail behavior?
* Did you mention cohorts when production data is involved?
* Did you avoid claiming certainty from one run or one dashboard?
* Did you recommend Instruments or signposts when root cause is still unknown?
* Did you propose a repeatable before/after comparison?
* Did you suggest a regression guard only for stable, important scenarios?
* Did you note what the metric does not prove?

## Output Notes

For XCTest tasks, return:

1. Scenario.
2. Metric.
3. Measured block boundary.
4. Input stability rules.
5. Baseline, variance, and threshold policy.
6. CI usage.
7. What this does not prove.

For launch performance test tasks, return:

1. Launch scenario.
2. Whether `waitUntilResponsive` should be used.
3. App state, cache state, and launch arguments.
4. Metric boundary.
5. What the launch test can detect.
6. What still needs Instruments or signposts.

For MetricKit or Organizer tasks, return:

1. Production signal.
2. Affected app versions and builds.
3. Affected device and OS cohorts.
4. p50, p95, p99, or available distribution.
5. Sample count and confidence.
6. Local reproduction plan.
7. Instruments or signposts needed.
8. What is not proven yet.

When evidence is incomplete, say what is known, what is not proven yet, and what should be measured next.
