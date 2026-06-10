# Metrics, Instruments, XCTest, MetricKit, and Production Monitoring

Use this reference when a launch investigation depends on measurement evidence, trace interpretation, XCTest launch metrics, MetricKit payloads, Xcode Organizer data, signposts, CI baselines, or production launch monitoring.

This file is about **measuring, interpreting, validating, and monitoring iOS launch performance**. It should not be used as the primary guide for fixing dyld work, linking strategy, lifecycle startup code, SwiftUI root setup, SDK startup policy, or launch orchestration.

## Scope Boundary

This reference covers:

- Instruments App Launch traces
- Time Profiler during startup
- signposts and app-owned launch phase markers
- XCTest launch metrics
- `xctrace` exports
- CI baselines and regression gates
- MetricKit launch metrics
- Xcode Organizer production reports
- custom app launch telemetry
- production monitoring strategy
- evidence interpretation and routing to focused implementation references

This reference does not cover:

- launch taxonomy and target selection in detail
- dyld internals, `+load`, constructors, or static initialization fixes
- static, dynamic, or mergeable linking strategy
- AppDelegate, SceneDelegate, or root UI restructuring
- SwiftUI `App` and root view initialization fixes
- third-party SDK startup policy
- launch dependency graph design

Use this file to decide **what the evidence proves, what it does not prove, and which reference should be used next**.

## Contents

- [Core Rule](#core-rule)
- [Review Procedure](#review-procedure)
- [Evidence Types](#evidence-types)
- [Tool Selection](#tool-selection)
- [Instruments App Launch](#instruments-app-launch)
- [Time Profiler During Launch](#time-profiler-during-launch)
- [XCTest Launch Metrics](#xctest-launch-metrics)
- [MetricKit Launch Metrics](#metrickit-launch-metrics)
- [Xcode Organizer](#xcode-organizer)
- [Signposts and Launch Phase Markers](#signposts-and-launch-phase-markers)
- [Custom App Telemetry](#custom-app-telemetry)
- [CI Baselines and Regression Gates](#ci-baselines-and-regression-gates)
- [What the Agent Can Inspect](#what-the-agent-can-inspect)
- [Common Interpretation Mistakes](#common-interpretation-mistakes)
- [Agent Guidance](#agent-guidance)
- [Boundary With Other References](#boundary-with-other-references)

## Core Rule

Do not optimize launch from a single unexplained number.

First identify:

- launch scenario
- entry point
- measurement boundary
- evidence source
- build configuration
- device class
- run count and variance
- app data state
- whether the metric represents first frame, extended launch, resume, or responsiveness

Measurement should answer one of four questions:

1. 1**What happened in this launch?**
2. 2**Did a known launch path regress?**
3. 3**Is this affecting users in production?**
4. 4**Did a proposed fix improve the intended phase without breaking readiness?**

If the evidence cannot answer the question being asked, recommend the missing measurement instead of guessing.

## Review Procedure

When using this reference:

1. 1Identify what evidence the user provided.
2. 2Classify the launch scenario and measurement boundary.
3. 3Decide whether the evidence is local, automated, production, custom, or manual.
4. 4Identify which question the evidence can answer: root cause, regression, trend, prioritization, or validation.
5. 5Check build type, device, OS, run count, variance, app data state, entry point, and network/session conditions.
6. 6Interpret the metric without mixing first frame, extended launch, resume, background launch, and responsiveness.
7. 7Look for phase evidence: pre-main, app lifecycle, root UI, first frame, post-first-frame responsiveness, SDK startup, or orchestration.
8. 8Route to the implementation reference only after the likely phase or cause is clear.
9. 9Recommend missing measurement when evidence is insufficient.
10. 10Tie every recommendation to the metric, trace area, or production distribution that should improve.

## Evidence Types

Classify the evidence before interpreting it.

### Local Trace Evidence

Examples:

- Instruments App Launch trace
- Time Profiler trace during launch
- `xctrace` recording or export
- signpost timeline
- local release-like device profiling

Best for:

- root-cause investigation
- phase-level diagnosis
- identifying main-thread blocking
- finding expensive call stacks
- validating a specific local fix

Not enough for:

- production impact by itself
- long-term trend
- device and audience distribution
- CI regression gating unless converted into repeatable tests

### Automated Test Evidence

Examples:

- XCTest launch metrics
- performance test output
- CI launch metric trends
- baseline comparisons

Best for:

- detecting regressions in a known path
- validating a release candidate
- comparing a change against a stable baseline
- catching repeated launch-path drift

Not enough for:

- full root cause without traces
- production audience impact
- mixed entry points or real-world data states

### Production Evidence

Examples:

- MetricKit launch metrics
- Xcode Organizer performance data
- production telemetry dashboards
- release-to-release launch distributions

Best for:

- prioritizing user impact
- seeing device, OS, audience, and release-level trends
- detecting regressions that local testing missed
- tracking p90/p95/p99 behavior

Not enough for:

- exact local root cause
- call-stack attribution
- safe code change selection without follow-up investigation

### Custom App Telemetry

Examples:

- app-defined startup milestones
- app-owned launch phase markers
- first-screen readiness events
- home data loaded events
- route resolved events
- signpost-derived intervals

Best for:

- app-specific readiness interpretation
- understanding custom milestones
- correlating user-visible readiness with app phases

Not enough for:

- full system launch duration
- pre-main and system-side work
- comparison with Apple metrics unless boundaries match

## Tool Selection

Use the tool that answers the actual question.


Do not compare metrics with different boundaries unless the difference is explicit and intentional.

## Instruments App Launch

Use Instruments App Launch when the task needs local phase-level evidence.

It is useful for:

- identifying whether time is spent before app code, in app lifecycle code, in first-frame rendering, or in early responsiveness
- checking whether launch work is on the main thread
- correlating app-owned signposts with system launch phases
- validating whether a proposed fix improves the expected phase

When interpreting an App Launch trace, ask:

- Is this a cold, warm, prewarmed, first-run, post-update, resume, or unknown scenario?
- Is the build release-like?
- Is the trace from a physical device?
- Which device and OS version were used?
- Does the trace include app-owned signposts?
- Is the expensive work before `main`, in lifecycle callbacks, during root UI construction, or after first draw?
- Does the trace show first-frame improvement but continued main-thread blocking after first frame?

Do not use a single local trace as proof of production impact. Use it to find the mechanism.

## Time Profiler During Launch

Use Time Profiler when the task needs call-stack evidence for CPU-heavy launch work.

It is useful for:

- identifying expensive functions during startup
- finding synchronous parsing, decoding, database, file, or keychain work
- detecting expensive dependency graph construction
- confirming whether work is on the main thread
- checking whether a “small” helper hides a large call chain

When using Time Profiler during launch:

- profile a release-like build
- use a physical device when possible
- repeat measurements
- keep the launch scenario stable
- focus on call stacks within the launch interval
- correlate with signposts or launch phase markers

Time Profiler is not enough when the issue is mostly dyld image loading, production distribution, or metric boundary confusion. Route accordingly.

## XCTest Launch Metrics

Use XCTest launch metrics when the question is: **did this known launch path regress?**

The primary XCTest launch metric is:

```swift
XCTApplicationLaunchMetric()
```

Use it for repeated, automated launch measurement of a known path.

Use:

```swift
XCTApplicationLaunchMetric(waitUntilResponsive: true)
```

when the boundary should include early responsiveness rather than only launch-to-first-frame behavior.

When reviewing XCTest launch tests, check:

- whether the launch scenario is stable
- whether launch arguments create a deterministic state
- whether the app data container is reset or preserved intentionally
- whether authentication/session state is controlled
- whether network behavior is mocked, disabled, or intentionally included
- whether the test uses a release-like configuration when possible
- whether the test runs enough iterations
- whether baselines account for variance
- whether the metric boundary matches the product question
- whether the test path represents app-icon launch, deep link, notification, or another entry point

XCTest launch metrics are good regression tests. They do not automatically explain the root cause. Use traces and signposts when a regression needs diagnosis.

## MetricKit Launch Metrics

Use MetricKit when the task needs production launch monitoring or build-to-build regression detection.

Relevant MetricKit concepts include:

- `MXMetricManager`
- `MXMetricPayload`
- `MXMetricPayload.applicationLaunchMetrics`
- `MXAppLaunchMetric`
- histogrammed launch values such as time to first draw

MetricKit is useful for:

- seeing launch distributions across real devices
- tracking release-to-release regressions
- prioritizing user impact
- watching p90/p95/p99 launch behavior
- detecting issues missed by local profiling or CI

When interpreting MetricKit launch data, check:

- app version
- OS version
- device class
- sample size
- foreground launch vs resume semantics
- percentile or histogram bucket
- audience and rollout differences
- whether first-run or post-update launches are mixed into the distribution
- whether a single average hides tail latency

MetricKit does not provide detailed call stacks for root cause. Use it to prioritize and validate trends, then investigate with local traces, signposts, and focused code review.

## Xcode Organizer

Use Xcode Organizer when the task needs an Apple-provided production view of launch performance and release trends.

Organizer is useful for:

- identifying whether a release regressed
- comparing app versions
- seeing device and OS distribution patterns
- prioritizing real-user launch issues
- communicating launch performance to the team

Organizer is not enough for:

- exact root cause
- app-owned phase attribution
- call-stack diagnosis
- proving a specific code path caused a regression

Use Organizer to decide **where to look**, not as the final explanation of **why it happened**.

## Signposts and Launch Phase Markers

Use signposts to make app-owned startup phases visible in local traces and, when appropriate, custom telemetry.

Good launch phase markers include:

- app delegate start/end
- scene connection start/end
- SwiftUI root construction boundaries
- dependency container setup
- initial route resolution
- first-screen view model creation
- database open or migration
- SDK minimal startup
- root UI installed
- first data request started/finished
- first interaction readiness

Keep signposts stable and meaningful. Do not create a marker for every small function.

Use stable, important intervals when considering MetricKit signpost metrics or production dashboards.

A good signpost answers:

- what phase started
- what phase ended
- whether it blocked first frame or first interaction
- whether it ran on the main thread or background executor
- whether it succeeded, failed, timed out, or degraded

Signposts should support investigation. They should not become a replacement for removing unnecessary launch work.

## Custom App Telemetry

Custom launch telemetry can complement Apple tools, but it must be labeled carefully.

Recommended context fields:

- app version
- build number
- device model
- OS version
- launch scenario if known
- entry point
- authenticated vs unauthenticated state
- first install, update, or returning-user state
- app data size bucket
- network state if relevant
- Low Power Mode if available
- thermal state if relevant
- milestone name
- milestone boundary
- duration
- success, failure, timeout, or degraded state

Useful app-owned milestones:

- `processStarted` only if the app can define it accurately
- `appDelegateStarted`
- `sceneConnectionStarted`
- `rootUIInstalled`
- `initialRouteResolved`
- `firstScreenModelReady`
- `firstFrameVisible` only if the app can measure it correctly
- `firstInteractionReady`
- `homeDataLoaded`

Do not name a custom metric `launchTime` unless its boundary is explicit.

Do not compare custom app timestamps with MetricKit, Organizer, XCTest, or Instruments values unless the boundaries match.

App-level timestamps may miss system-side work, prewarming, and pre-main cost.

## CI Baselines and Regression Gates

Use CI launch metrics to detect regressions in stable, repeatable launch paths.

Good CI practice:

- use a fixed launch scenario
- use a known entry point
- control app data state
- control authentication/session state
- avoid uncontrolled network dependency
- use release-like build settings when possible
- run enough iterations
- track median and tail metrics
- store historical trends
- use thresholds that account for variance
- validate on older supported devices when possible
- keep local and CI measurement boundaries consistent

Avoid:

- failing builds on a single noisy run
- comparing simulator Debug runs to physical-device Release baselines
- mixing first-run and returning-user launches
- mixing app-icon and deep-link paths
- relying only on averages
- setting thresholds before variance is understood

CI should catch regressions. It should not be the only tool used for root-cause analysis.

## What the Agent Can Inspect

When repository access is available, inspect measurement code before drawing conclusions.

Search for XCTest launch metrics and test configuration:

```sh
rg "XCTApplicationLaunchMetric|measure\(|launchArguments|launchEnvironment|waitUntilResponsive" .
```

Search for signposts and app-owned phase markers:

```sh
rg "os_signpost|OSSignposter|OSLog|signpost|beginInterval|endInterval|firstFrame|firstDraw|readyToUse" .
```

Search for MetricKit integration:

```sh
rg "MXMetricManager|MXMetricPayload|MXAppLaunchMetric|applicationLaunchMetrics|didReceive.*payload" .
```

Search for custom launch telemetry and business milestones:

```sh
rg "launchMetric|startupMetric|appLaunch|startup|homeLoaded|startupComplete|appReady|initialRouteResolved|firstInteraction" .
```

Search for CI baselines and performance thresholds:

```sh
rg "baseline|threshold|regression|performance test|xctrace|XCTMetric|xcresult|launch performance|startup performance" .
```

Search results identify measurement code, not the bottleneck. Use them to understand what is being measured and what evidence is missing.

The agent can:

- ask for an Instruments trace, `xctrace` export, XCTest output, MetricKit payload, Organizer screenshot/export, CI trend, or production dashboard summary
- inspect launch tests, signposts, MetricKit subscribers, logging, metric upload, and test launch arguments
- propose signpost placement around app-owned launch phases
- identify mismatched metric boundaries
- recommend missing measurement before suggesting code changes

The agent cannot reliably:

- prove root cause from a production histogram alone
- prove production impact from one local trace
- compare custom milestones to Apple metrics without boundary matching
- interpret a single number without scenario, device, build, and variance context
- declare a fix successful without measuring the same scenario and boundary before and after

## Common Interpretation Mistakes

- Treating resume as launch.
- Treating background launch as first-frame launch.
- Treating launch screen display as the first app-rendered frame.
- Treating custom “home loaded” as first-frame launch time.
- Comparing `main`-to-first-screen logging with app icon to first draw.
- Comparing Debug simulator runs to Release device runs.
- Comparing cold launch to warm launch.
- Comparing app-icon launch to deep-link, notification, widget, or document launch.
- Ignoring prewarming when interpreting app-level timestamps.
- Reporting averages without p90/p95/p99, variance, sample size, or device class.
- Treating MetricKit or Organizer as root-cause tools.
- Treating XCTest launch metrics as proof of what function is slow.
- Optimizing first frame while leaving the app unresponsive after first draw.
- Adding too many signposts without stable names or clear boundaries.
- Failing CI on noisy launch measurements before baseline variance is understood.
- Calling a measurement “launch” without documenting the boundary.

## Agent Guidance

When applying this reference, produce a measurement-oriented review:

```markdown
### Evidence provided
Instruments / Time Profiler / signposts / XCTest / MetricKit / Organizer / CI / custom telemetry / manual timing / unknown.

### Launch classification
Cold / warm / prewarmed / first-run / post-update / resume / background launch / unknown.

### Measurement boundary
What the number appears to measure: app icon to first draw, process start to first draw, `main` to first screen, first draw to responsiveness, extended launch, resume, custom milestone, or unknown.

### Evidence quality
Device, OS, build type, run count, variance, app data state, entry point, audience, and whether the evidence is comparable.

### What this evidence can prove
Root cause / regression / production impact / trend / validation / only a symptom.

### What it cannot prove
Call out missing context, missing trace, missing baseline, boundary mismatch, or production/local mismatch.

### Recommended next measurement
The smallest measurement step needed before making or validating code changes.

### Routing
Which focused reference should be used next if the likely phase is clear.
```

Keep measurement interpretation separate from implementation recommendations. If the likely phase is clear, route to the focused reference and keep the measurement context attached.

## Boundary With Other References

Use this reference for launch measurement, trace interpretation, regression detection, and production monitoring.

Read `references/launch-taxonomy-and-targets.md` when the issue involves:

- cold, warm, prewarmed, resume, first install, or update launch terminology
- launch target selection
- measurement scenario classification
- whether two numbers are comparable

Read `references/pre-main-dyld-and-static-initializers.md` when the evidence points to:

- dyld
- pre-main work
- `+load`
- `+initialize`
- constructor functions
- Objective-C categories
- runtime registration
- static initialization

Read `references/linking-strategy.md` when the evidence points to:

- dynamic frameworks
- static libraries
- mergeable libraries
- modularization and launch-time linking trade-offs
- binary layout
- order-file considerations

Read `references/launch-orchestration-and-dependency-graph.md` when the evidence points to:

- critical path analysis
- startup step dependencies
- hidden ordering
- safe parallelism
- failure policy
- dependency-chain optimization

Read `references/appdelegate-scenedelegate-and-first-frame.md` when the evidence points to:

- `UIApplicationDelegate`
- `UISceneDelegate`
- window setup
- root view controller creation
- first-frame readiness
- main-thread lifecycle work

Read `references/swiftui-app-launch.md` when the evidence points to:

- SwiftUI `App`
- `WindowGroup`
- root view setup
- observable state
- `.task`
- `.onAppear`
- `scenePhase`
- `@UIApplicationDelegateAdaptor`
- environment initialization

Read `references/third-party-sdks-at-launch.md` when the evidence points to:

- analytics
- crash reporting
- ads
- attribution
- remote config
- push
- feature flags
- security SDKs
- vendor initialization strategy

Do not read all references by default.
