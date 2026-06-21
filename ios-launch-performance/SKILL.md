---
name: ios-launch-performance
description: Use this skill when diagnosing iOS app launch performance, startup regressions, first-frame readiness, or early responsiveness. Covers pre-main/dyld work, AppDelegate/SceneDelegate, SwiftUI App startup, launch orchestration, SDK initialization, and launch measurement. Do not use for general performance unless the code runs on the launch path.
---

# iOS Launch Performance

Use this skill to review the path from an app launch request to first visible UI and early responsiveness.

Focus on work that happens:

* before `main`
* during UIKit or SwiftUI startup
* inside `UIApplicationDelegate`, `UISceneDelegate`, or SwiftUI `App`
* during root UI creation
* before the first visible frame
* before the first meaningful interaction
* during early post-launch work that still affects user-perceived readiness

This is not a general iOS performance skill. Use it only when the issue is launch-specific or when the code runs on the launch path.

## Core Model

Treat launch as a pipeline with separate phases:

1. Process creation and system preparation
2. dyld loading, binding, fixups, runtime registration, and static initialization
3. UIKit or SwiftUI runtime startup
4. App-level initialization in `UIApplicationDelegate`, `UISceneDelegate`, or SwiftUI `App`
5. Launch orchestration and dependency setup
6. Root UI construction, layout, drawing, and first frame commit
7. Early post-launch work that affects responsiveness
8. Later feature-specific or maintenance work

Do not optimize blindly. First identify which phase is expensive, then recommend changes that move, remove, lazy-load, parallelize, serialize, or measure that specific work.

## When to Use This Skill

Use this skill when the task involves:

* slow app launch
* startup regressions
* cold, warm, or prewarmed launch
* resume-vs-launch confusion
* first-frame readiness
* early responsiveness after launch
* pre-main or dyld work
* Objective-C `+load`, `+initialize`, constructor functions, or static initialization
* AppDelegate or SceneDelegate startup work
* SwiftUI `@main App`, `WindowGroup`, root view setup, or environment injection
* dependency container setup during launch
* launch orchestrators or ordered startup steps
* third-party SDK initialization during launch
* framework linking strategy when launch cost is suspected
* launch metrics from Instruments, XCTest, MetricKit, or Xcode Organizer

Do not use this skill for:

* general scrolling performance
* memory leaks unrelated to launch
* rendering optimization unrelated to first frame
* networking performance outside startup
* broad architecture review unless the architecture affects the launch path

## Launch Scenario Classification

Before giving advice, classify what is being measured:

* **Cold launch**: the app process is not resident and little launch-related state is already warm.
* **Warm launch**: the app still starts a new process, but parts of system state, caches, or pages may already be warm.
* **Prewarmed launch**: the system may have prepared part of the launch path before the user explicitly opens the app.
* **Resume / already-running return**: the app process already exists and returns from background or suspension.
* **First install / first run / update launch**: launch includes extra setup such as migrations, cache creation, permissions, account/bootstrap work, or version-specific setup.
* **Unknown**: the measurement setup does not clearly separate the above cases.

Do not compare cold launch, warm launch, prewarmed launch, first-run launch, update launch, and resume as one metric.

Resume is not a full launch investigation unless the task explicitly asks about foregrounding latency.

## Investigation Workflow

### 1. Locate the launch path

Identify code that executes before first frame or before first meaningful interaction.

Inspect:

* app delegate
* scene delegate
* SwiftUI `App`
* root scene construction
* root view/model creation
* dependency container setup
* global/static initialization
* launch orchestrators
* SDK startup
* linked framework initialization
* first-screen routing and state restoration

### 2. Classify the slow phase

Classify the likely phase before recommending fixes:

* dyld/pre-main
* Objective-C or Swift runtime/static initialization
* app delegate startup
* scene delegate startup
* SwiftUI app/root view initialization
* launch orchestration or dependency setup
* root UI construction and first-frame rendering
* early post-launch responsiveness
* measurement ambiguity

If the available evidence is not enough to classify the phase, say so and recommend the next measurement step.

### 3. Classify startup work by necessity

For each startup task, classify it as:

* required before first frame
* required before first interaction
* needed soon after launch
* needed only after authentication/session state is known
* needed only by a later feature
* background maintenance

Work that is not required before first frame or first interaction should not block launch unless there is a correctness reason.

### 4. Build a dependency view

If launch has ordered steps, SDK setup calls, service registrations, or a launch orchestrator, identify:

* which steps are truly required for the first visible UI
* which steps are required before the first meaningful interaction
* which steps can run independently
* which steps must stay ordered
* which steps touch the main thread or shared mutable state
* which failures must block launch
* which dependency chain forms the longest critical path

Treat comments, fragile ordering, and institutional knowledge as risk. Prefer explicit dependencies over relying on call order.

### 5. Look for hidden eager work

Check for launch work hidden in:

* Objective-C `+load`
* Objective-C `+initialize`
* C/C++ constructor functions
* C++ global objects with constructors
* Swift globals or static properties
* eager singletons
* dependency graph construction
* SDK auto-registration
* dynamic framework startup
* synchronous file I/O
* database opening or migration
* keychain-heavy work
* networking or remote configuration
* large decoding/parsing
* blocking locks, semaphores, dispatch groups, or synchronous waits on the main thread

Treat hidden initialization as launch-critical until measurement proves otherwise.

### 6. Recommend targeted changes

Prefer changes that directly shorten or unblock the launch path:

* remove unnecessary launch work
* lazy-load feature-specific services
* defer noncritical work until after visible UI or first interaction
* split launch-critical state from secondary app state
* make startup dependencies explicit
* replace hidden static initialization with explicit or lazy initialization
* move blocking work off the main thread only when safe and useful
* use bounded parallelism only after auditing dependencies, shared state, isolation, and failure behavior
* review linking strategy only when evidence points to pre-main or dyld cost
* shrink the first usable surface instead of initializing the whole application graph before display

Do not recommend broad rewrites unless the launch path cannot be safely improved with smaller changes.

### 7. Require validation

Every recommendation should include a validation path.

Use:

* Instruments App Launch for phase-level diagnosis
* Time Profiler for CPU-heavy startup paths
* dyld-related tools or logs when pre-main work is suspected
* `os_signpost` or equivalent markers for app-specific startup phases
* XCTest launch metrics for repeatable local or CI regression checks
* MetricKit or Xcode Organizer for production distributions

Prefer release-like builds, real devices, stable test data, repeated runs, and older supported hardware.

## High-Value Decision Rules

* Treat roughly 400 ms to first visible frame as an aggressive user-experience target, not as a watchdog threshold and not as a pre-main-only budget.
* Do not blame dyld by default. Slow launch can come from pre-main work, app initialization, launch orchestration, root UI creation, first-frame rendering, synchronous I/O, or early post-launch blocking.
* Treat `+load`, constructor functions, and static initialization with side effects as launch-critical until measurement proves otherwise.
* Prefer explicit setup, lazy initialization, or scoped one-time initialization over work hidden in load-time hooks.
* Do not present `+initialize` as a universal fix. It can help in some legacy Objective-C code, but explicit or lazy initialization is usually clearer in modern code.
* Do not recommend converting all modules to static linking. Consider launch cost, build time, binary size, duplicate symbols, resource packaging, SDK distribution, debugging, and mergeable libraries.
* Do not use arbitrary framework-count limits. More dynamic frameworks can increase launch work, but the real cost must be measured.
* Do not parallelize launch steps until dependencies, shared state, actor isolation, main-thread requirements, and failure behavior are explicit.
* Do not block the main thread while waiting for parallel startup work unless the code is proven safe, bounded, and required before launch can continue.
* Treat unsafe parallelism as a correctness risk even when it improves a local benchmark.
* Optimize the longest required dependency chain, not only the largest individual startup step.
* Do not assume that queueing work asynchronously on the main queue guarantees it runs after the first frame.
* Do not assume SwiftUI `.task` is always post-render or harmless. It is lifecycle-bound async work and can still affect early responsiveness.
* Do not rely on Debug builds, simulator-only runs, or a single modern device when judging launch performance.
* Do not use production launch histograms alone to identify the local bottleneck. Use them to prioritize and verify trends.

## Code Review Checklist

When reviewing launch-related code, check these areas first.

### Pre-main and runtime initialization

Look for `+load`, constructor functions, C++ global constructors, expensive Swift globals/statics, eager runtime hooks, Objective-C category-heavy modules, and dynamic frameworks with startup-time initializers.

Read `references/pre-main-dyld-and-static-initializers.md` when this area is relevant.

### Launch orchestration and dependency graph

Look for long ordered startup sequences, implicit dependencies, startup comments that encode required ordering, dependency containers built synchronously, unsafe parallelism, blocking waits, unclear failure behavior, and first-screen code that assumes the whole app graph is ready.

Read `references/launch-orchestration-and-dependency-graph.md` when this area is relevant.

### AppDelegate, SceneDelegate, and first frame

Look for synchronous dependency setup, unconditional SDK initialization, database opening or migration, keychain access, synchronous networking, heavy root view model construction, duplicate app/scene setup, and expensive first-screen rendering.

Read `references/appdelegate-scenedelegate-and-first-frame.md` when this area is relevant.

### SwiftUI App startup

Look for heavy work in `App.init`, root `Scene` construction, root view initialization, eager observable model creation, environment injection, `.task`, `.onAppear`, `scenePhase`, and `@UIApplicationDelegateAdaptor`.

Read `references/swiftui-app-launch.md` when this area is relevant.

### Third-party SDKs

Every SDK that starts during launch must justify why it needs to run before first frame or first interaction.

Classify SDKs as launch-critical, first-interaction required, post-first-frame acceptable, feature-specific and lazy, or background-only.

Be careful with blanket deferral. Crash reporting, security, deep linking, attribution, push routing, remote config, and feature flags can have correctness requirements.

Read `references/third-party-sdks-at-launch.md` when this area is relevant.

### Measurement

When measurements are noisy, clarify:

* device model and OS version
* Debug vs Release/Profile configuration
* simulator vs physical device
* cold/warm/prewarmed/resume classification
* first install, update launch, or returning-user launch
* network and account state
* app data size
* authenticated vs unauthenticated launch
* test iteration count and variance

Read `references/metrics-instruments-xctest-metrickit.md` when measurement setup or interpretation is relevant.

## Reference Routing

Use reference files only when the task needs extra detail.

* Read `references/launch-taxonomy-and-targets.md` when the task involves cold/warm/prewarmed/resume terminology, first-frame targets, first install/update launch classification, or measurement setup.
* Read `references/pre-main-dyld-and-static-initializers.md` when the task involves dyld, pre-main, `+load`, `+initialize`, constructor functions, Objective-C categories, runtime registration, or static initialization.
* Read `references/linking-strategy.md` when the task involves dynamic frameworks, static libraries, mergeable libraries, modularization, launch-time linking trade-offs, binary layout, or order-file considerations.
* Read `references/launch-orchestration-and-dependency-graph.md` when the task involves launch steps, critical path modeling, explicit dependencies, safe parallelism, race/deadlock risks, failure handling, or longest-chain optimization.
* Read `references/appdelegate-scenedelegate-and-first-frame.md` when the task involves `UIApplicationDelegate`, `UISceneDelegate`, dependency setup, root UI creation, first-frame readiness, or main-thread startup work.
* Read `references/swiftui-app-launch.md` when the task involves SwiftUI `App`, `WindowGroup`, root view setup, observable state, `.task`, `.onAppear`, `scenePhase`, `@UIApplicationDelegateAdaptor`, or environment initialization.
* Read `references/third-party-sdks-at-launch.md` when the task involves analytics, crash reporting, ads, attribution, remote config, push, feature flags, security SDKs, or vendor initialization strategy.
* Read `references/metrics-instruments-xctest-metrickit.md` when the task involves Instruments, Time Profiler, signposts, XCTest launch metrics, MetricKit, Xcode Organizer, CI baselines, or production monitoring.

Do not read all references by default.

## Output Format

When reviewing code or diagnosing a launch report, return the following sections.

### Launch classification

State whether the issue appears to involve cold launch, warm launch, prewarmed launch, first install/update launch, resume, or unknown.

Mention measurement ambiguity if present.

### Suspected phase

Identify the most likely phase:

* pre-main
* static initialization
* app delegate
* scene setup
* SwiftUI root setup
* launch orchestration
* first-frame rendering
* early post-launch responsiveness
* measurement/setup issue

### Critical path

Identify the required startup chain if enough information is available.

Mention hidden ordering, implicit dependencies, or unknown dependencies when they block safe optimization.

### Findings

List concrete risks found in the code, architecture, metrics, or trace.

Avoid generic advice not connected to evidence.

### Recommended changes

Group recommendations by priority:

1. High-confidence launch-path fixes
2. Dependency/orchestration fixes
3. Measurement or instrumentation needed
4. Optional architectural cleanup

### Validation

Explain how to verify the change locally and in production.

Include the expected metric, trace area, or startup phase that should improve.

## Non-Goals

Do not use this skill for:

* general scrolling performance
* memory leaks
* rendering optimization unrelated to app startup
* networking performance outside launch
* broad app architecture review unrelated to startup
* generic SwiftUI review unless the code affects launch, first frame, or early responsiveness
