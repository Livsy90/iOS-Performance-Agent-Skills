# Launch Orchestration and Dependency Graph

Use this reference when launch work has grown into many ordered startup tasks, fragile initialization sequences, unsafe parallelism, or unclear readiness rules.

This file is about organizing post-entry launch work. It should not explain launch taxonomy, dyld internals, framework linking strategy, SDK startup policy, tool setup, MetricKit APIs, XCTest configuration, or Instruments workflow. Use the other references for those topics.

## When to Use This Reference

Use this reference when reviewing:

* a long `AppDelegate`, `SceneDelegate`, SwiftUI `App`, or launch coordinator;
* a startup sequence with many services, stores, routers, feature gates, or SDKs;
* launch work protected by comments such as "do not reorder";
* an attempted parallel launch implementation;
* startup crashes caused by reordered initialization;
* launch work that blocks first frame or first interaction;
* a slow first screen caused by full app-wide setup;
* a launch system with unclear readiness, timeout, or failure rules;
* a proposal to move startup work to background queues, tasks, workers, or task groups.

Do not use this reference only because launch is slow. First decide whether the problem is orchestration, dyld/pre-main, linking, AppDelegate/SceneDelegate work, SwiftUI root setup, SDK policy, first-frame rendering, or measurement setup.

## Scope Boundary

This reference answers:

* how startup work should be organized;
* which steps are truly launch-critical;
* which dependencies are hidden by serial ordering;
* whether parallelism is safe;
* which chain determines the minimum possible launch time;
* whether the app can show useful UI before full readiness;
* how failure, timeout, cancellation, and partial readiness should be modeled.

This reference does not answer:

* cold launch vs warm launch terminology;
* dyld, pre-main, static initialization, or `+load`;
* static vs dynamic vs mergeable linking strategy;
* detailed SDK startup policy;
* UIKit lifecycle ownership;
* SwiftUI `App` lifecycle details;
* MetricKit, XCTest, Organizer, or Instruments setup.

Route to the corresponding reference when those details are the core issue.

## Contents

* [Purpose](#purpose)
* [Core Model](#core-model)
* [Review Procedure](#review-procedure)
* [Critical Path vs Secondary Path](#critical-path-vs-secondary-path)
* [Dependency Graph](#dependency-graph)
* [Dependency Types](#dependency-types)
* [Scheduling Policy](#scheduling-policy)
* [Bounded Parallelism](#bounded-parallelism)
* [Main-Thread Blocking Risks](#main-thread-blocking-risks)
* [Race Conditions and Ordering Bugs](#race-conditions-and-ordering-bugs)
* [Cycle Detection](#cycle-detection)
* [Longest Dependency Chain](#longest-dependency-chain)
* [Partial Readiness](#partial-readiness)
* [Failure, Cancellation, and Timeout Policy](#failure-cancellation-and-timeout-policy)
* [Observability](#observability)
* [What the Agent Can Inspect](#what-the-agent-can-inspect)
* [Safe Patch Heuristics](#safe-patch-heuristics)
* [Review Checklist](#review-checklist)
* [Agent Guidance](#agent-guidance)
* [Boundary With Other References](#boundary-with-other-references)

## Purpose

Large iOS apps often accumulate launch work over time. What starts as a few setup calls can become a long sequence of services, SDKs, stores, routers, feature gates, session checks, and first-screen dependencies.

When that sequence is stored only as ordered code, comments, or team memory, it becomes hard to optimize safely. Engineers may know that one step must run before another, but the dependency is not represented anywhere the app can validate.

The goal is to turn launch from an implicit list into an explicit readiness model:

```text
startup work
→ dependency graph
→ critical path
→ safe scheduling
→ measurable readiness
```

The key output is not a new abstraction. The key output is clarity: which work is truly launch-critical, which work is merely early, and which work is historical baggage.

## Core Model

Model launch work as a set of named steps. Each step should have a reason to exist on the launch path.

In this reference, “first frame” means the first app-rendered frame after the system launch screen. “First interaction” means the first moment when the visible UI can respond correctly to the user’s intended action. Optimizing one while damaging the other is not a successful launch optimization.

For each step, identify:

* what it prepares;
* whether it is required before first frame;
* whether it is required before first interaction;
* whether it can run after visible UI;
* whether it can be lazy on first feature use;
* which other steps it depends on;
* which steps depend on it;
* whether it must run on the main actor or main thread;
* whether it touches shared mutable state;
* how failure should affect app readiness;
* how long it takes in representative measurements.

Do not preserve a launch step only because it has always been there.

## Review Procedure

When using this reference:

1. Identify the current startup sequence.
2. Convert ordered calls into named launch steps.
3. Classify each step as first-frame critical, first-interaction, secondary, lazy, or maintenance.
4. Identify dependencies between steps.
5. Mark main-thread, shared-state, side-effect, failure, timeout, and cancellation constraints.
6. Find the longest required dependency chain.
7. Look for work that can move out of global launch.
8. Decide whether parallelism is safe, useful, and bounded.
9. Recommend the smallest safe change.
10. Define validation for both correctness and timing.

Avoid generic advice such as "run this in parallel" or "move this to a background queue" unless dependencies and readiness rules are explicit.

## Critical Path vs Secondary Path

Separate launch work by user-visible necessity.

### First-Frame Critical Path

A step belongs to the first-frame critical path only when the app cannot produce a valid first app-rendered frame without it.

Typical first-frame critical-path candidates:

* deciding the initial safe shell or route;
* loading minimal local session state;
* preparing safety or correctness checks required before any UI can appear;
* installing crash handling when required by product policy;
* preparing the smallest state needed by the first visible screen.

Do not mark a step first-frame critical only because it has always run during launch.

### First-Interaction Path

A step belongs to the first-interaction path when the app can draw a valid initial UI without it, but the first meaningful user action would be disabled, unsafe, or incorrect until it completes.

Examples:

* enabling the first screen’s primary action;
* validating local session state before showing sensitive data;
* loading cached data required by an immediately visible control;
* preparing a minimal command handler for the first intended route;
* completing lightweight local readiness needed for the first tap, scroll, or navigation action.

This work should not block the first app-rendered frame when a safe loading, locked, disabled, or placeholder state is possible.

If interaction depends on this work, the UI should expose that readiness honestly instead of appearing fully interactive while commands silently block or fail.

### Secondary Path

A step belongs to the secondary path when it is useful soon after launch but is not needed for the first frame or first interaction.

Common secondary-path candidates:

* nonessential analytics enrichment;
* cache cleanup;
* secondary feature preloading;
* deferred SDK modules;
* background synchronization;
* warmups for screens not visible at launch;
* optional personalization.

Secondary work should not compete with first-frame or first-interaction work for the main thread, CPU, I/O, locks, or memory pressure.

Do not move all secondary work to “right after first frame” as one burst. Stagger noncritical work by priority and user interaction risk.

### Lazy Path

A step belongs to the lazy path when only a later feature needs it.

Prefer lazy setup when:

* the first screen does not need the subsystem;
* the user may never visit the feature in this session;
* setup can be hidden behind the feature's own loading state;
* partial app readiness is acceptable.

Lazy setup is not free. It moves cost to a later interaction, so the feature must own loading, failure, cancellation, and retry behavior.

### Maintenance Path

A step belongs to the maintenance path when it supports long-term app health but does not affect launch readiness.

Common maintenance candidates:

* old cache cleanup;
* log trimming;
* optional prefetching;
* analytics upload;
* stale local data reconciliation;
* feature warmups not required by the initial route.

Maintenance work should not run at launch by default unless there is a clear product, compliance, or correctness reason.

## Dependency Graph

Represent startup dependencies explicitly.

A dependency graph answers:

* Which steps are ready to run now?
* Which steps are blocked?
* Which completed step unblocks other work?
* Which chain determines the minimum possible launch time?
* Which dependencies are real and which are accidental?

A launch step should not depend on another step because of ordering habit. It should depend on another step only when it consumes output, state, registration, configuration, authorization, or side effects produced by that step.

## Dependency Types

Use this taxonomy when converting an ordered startup list into an explicit launch graph.

### Data Dependency

One step needs data produced by another step.

Review questions:

* Can the consumer work with cached, partial, or stale data?
* Can the producer provide a lightweight initial value?
* Can the consumer request the data lazily later?

### Configuration Dependency

One step needs environment, flags, remote config, region, tenant, build channel, or experiment state.

Review questions:

* Which configuration is required for the first screen?
* Which configuration can be updated after launch?
* Can defaults unblock first frame safely?
* What visible behavior changes if defaults are later replaced?
* Is the default fail-open or fail-closed for safety-sensitive decisions?

### Registration Dependency

One step must register handlers, routes, observers, or integrations before another step can use them.

Review questions:

* Is registration required globally or only for a feature?
* Can registration move closer to feature activation?
* Is registration idempotent?
* What early event would be missed if registration moves later?

### Main-Thread Dependency

A step must run on the main thread or main actor because it touches UI, UIKit, SwiftUI state, or a main-thread-only API.

Review questions:

* Is the whole step main-thread-only, or only a small section?
* Can preparation run off the main thread and commit a small result on the main thread?
* Does this step block UI creation or first interaction?
* Is CPU-heavy work accidentally running on the main actor?

A step being `async` does not mean it leaves the main actor. If the step is called from a `@MainActor` launch coordinator, CPU-heavy work may still run on the main actor unless the work is isolated elsewhere and transferred safely.

### Side-Effect Dependency

One step relies on another step's side effects rather than returned data.

Review questions:

* Can the side effect become explicit state?
* Can the dependency be expressed as a named readiness condition?
* Is the side effect safe to repeat, cancel, or retry?

### Failure Dependency

One step changes how launch should continue if another step fails.

Review questions:

* Should failure block launch?
* Should failure degrade the first screen?
* Should failure trigger retry in the background?
* Should failure be surfaced to the user?
* Which failure mode should be measured or logged?
* Is the fallback fail-open or fail-closed?

## Scheduling Policy

Do not introduce parallelism until dependencies are explicit.

A safe scheduler should respect:

* dependency completion;
* priority;
* main-thread requirements;
* resource limits;
* cancellation;
* timeout;
* failure policy;
* duplicate prevention;
* deterministic readiness gates.

Parallel launch work is useful only when independent steps exist. If the graph is mostly one long chain, adding workers will not help much. In that case, optimize the chain itself or reduce what the first screen requires.

Watch for priority inversion: a high-priority launch-critical step can be blocked by a low-priority secondary task holding a shared lock, database connection, cache, file handle, or executor.

## Bounded Parallelism

Avoid unbounded startup work. Launch is already CPU, I/O, and memory sensitive.

Bounded parallelism means:

* do not start every pending step at once;
* keep the main thread available for UI work;
* avoid saturating storage with many competing reads;
* avoid many services contending for the same locks;
* avoid heavy network, disk, and CPU warmups competing at the same time;
* prefer small, measurable batches over uncontrolled fan-out.

The optimal worker count is not a fixed rule. It depends on device class, work type, thread affinity, I/O pressure, thermal state, and what the first frame needs.

## Main-Thread Blocking Risks

Be very cautious with designs where the main thread waits for parallel startup work to complete.

This pattern can improve a benchmark in some cases but carries several risks:

* the main thread is unavailable for UI creation;
* one launch step may need the main thread and deadlock;
* old devices have fewer cores to compensate for the blocked main thread;
* a slow background step can still delay the whole launch;
* error handling often becomes harder to reason about;
* cancellation and timeout behavior can become unclear.

If the app must wait before showing UI, keep the waiting boundary minimal and explicit. Prefer showing a small valid UI while secondary readiness continues when product behavior allows it.

## Race Conditions and Ordering Bugs

Unsafe parallelism often exposes dependencies that were previously hidden by serial ordering.

Common symptoms:

* startup crashes only on some devices;
* first launch fails but second launch succeeds;
* flags or configuration sometimes appear unavailable;
* database or storage setup races with consumers;
* analytics, routing, or push handlers miss early events;
* state restoration depends on services that are not ready;
* errors disappear when logging or breakpoints are added.

When these symptoms appear, do not solve them by returning to a giant serial list by default. First identify the missing dependency and encode it explicitly.

## Cycle Detection

A launch dependency graph must not contain cycles.

A cycle means that step A waits for step B, while step B directly or indirectly waits for step A. Cycles usually reveal one of these design problems:

* two services initialize each other;
* configuration and dependency injection are interleaved too tightly;
* a feature module requires app-wide setup too early;
* a shared singleton hides a dependency that should be explicit;
* a first-screen dependency actually belongs to a later feature.

When a cycle appears, break it by extracting a smaller readiness condition, using a lightweight placeholder state, or moving a feature-specific dependency out of the launch-critical path.

## Longest Dependency Chain

After the graph is explicit, optimize the longest chain rather than only the largest individual step.

A single expensive step is easy to notice, but a sequence of medium-cost dependent steps can define the minimum possible launch time.

For the longest chain, ask:

* Can any dependency be weakened from required to optional?
* Can a producer emit a minimal initial result earlier?
* Can a consumer accept partial state?
* Can the first screen avoid this chain entirely?
* Can the chain be split into first-frame and post-interaction readiness?
* Can a later feature own the cost instead of global launch?

If the longest chain cannot be parallelized, reduce the first screen's dependency on that chain.

## Partial Readiness

A mature app does not always need full readiness before showing the first screen.

Useful readiness levels:

* app can draw the initial shell;
* app can show cached or placeholder first-screen content;
* app can respond to basic navigation;
* app can complete the first intended interaction;
* all launch-critical services are ready;
* all feature modules are ready;
* background maintenance is complete.

Partial readiness requires explicit product behavior. The UI must know what is available, what is still loading, what is disabled temporarily, and how failures are handled.

Partial readiness should be represented in state the UI can observe. Avoid hidden global readiness flags that make the UI appear available while commands silently block, spin, or fail.

Do not hide incomplete readiness behind a UI that appears interactive but blocks or fails on first touch.

## Failure, Cancellation, and Timeout Policy

Launch work should not assume unlimited time or perfect success.

Classify failures as:

* fatal for app launch;
* fatal for authenticated area but not for logged-out UI;
* blocking for the first screen only;
* degradable with cached/default state;
* retryable in the background;
* feature-local and not launch-blocking.

Review whether launch steps need:

* cancellation when the app moves to background;
* timeout when waiting for network, disk, or another subsystem;
* retry with backoff;
* fallback state;
* telemetry for timeout and degraded readiness;
* a user-visible degraded state.

For safety, privacy, compliance, security, and kill-switch decisions, define whether failure is fail-open or fail-closed. Do not let timeout behavior accidentally choose a permissive state.

Avoid making first frame depend on open-ended network or server behavior.

A launch orchestrator without failure policy often becomes either too fragile or too permissive. It may crash for recoverable issues or silently continue after a critical invariant is broken.

## Observability

A dependency graph is useful only if the app can explain what happened.

For each significant step or group, collect enough information to answer:

* when it became eligible to run;
* when it started;
* when it finished;
* how much time it spent running;
* how much time it spent waiting;
* whether it ran on the main thread, main actor, or background executor;
* whether it waited on dependencies, locks, I/O, worker capacity, or main actor availability;
* whether it failed, timed out, or degraded;
* which readiness level it unlocked.

Separate run time from wait time. A step may look slow because it waited for dependencies, a shared lock, main actor availability, I/O contention, or a saturated worker pool.

Record the critical path duration separately from total startup work duration.

Prefer signposts or structured startup events over ad hoc print statements. The goal is to reconstruct the launch timeline and dependency chain during local profiling and production regression analysis.

Keep this section focused on what orchestration should expose. For how to collect, visualize, or compare these events, use `metrics-instruments-xctest-metrickit.md`.

## What the Agent Can Inspect

When repository access is available, inspect concrete launch orchestration code instead of giving generic advice.

Search for launch coordinators and bootstrap systems:

```sh
rg "LaunchCoordinator|Startup|Bootstrap|Orchestrator|Readiness|AppReady|initialize|configure|start|register|resolve|prepare" .
```

Search for ordered startup comments and implicit dependencies:

```sh
rg "do not reorder|must run before|must be called before|depends on|after .* initialized|ready|readiness|bootstrap complete|startup complete" .
```

Search for parallelism and blocking waits:

```sh
rg "TaskGroup|async let|DispatchGroup|OperationQueue|Task\.detached|semaphore|wait\(|sync\(|DispatchQueue\.main\.sync|performAndWait" .
```

Search for main-actor or main-thread launch work:

```sh
rg "@MainActor|MainActor\.run|DispatchQueue\.main|OperationQueue\.main" .
```

Search for broad dependency graph construction:

```sh
rg "registerAll|resolveAll|buildContainer|assemble|container\.register|container\.resolve|make.*Graph|create.*Graph|dependency graph" .
```

Search for readiness flags and global gates:

```sh
rg "ready|isReady|appReady|startupFinished|bootstrapFinished|initialized|isInitialized|didInitialize|canInteract" .
```

Search for first-screen coupling to app-wide readiness:

```sh
rg "isReady|appReady|startupFinished|bootstrapFinished|showMain|initialRoute|rootViewController|WindowGroup|UIHostingController" .
```

Use search results as leads, not proof. Confirm whether matched code actually runs before first frame or first interaction.

The agent can:

* convert a serial startup list into named steps;
* classify each step by readiness requirement;
* identify hidden dependencies implied by ordering;
* detect unsafe or unbounded parallelism;
* suggest explicit readiness states;
* propose lazy or feature-owned setup for noncritical work;
* recommend instrumentation around launch steps;
* propose small local patches when correctness is clear.

The agent cannot reliably:

* prove performance improvement without measurement;
* decide product readiness rules without context;
* defer auth, security, privacy, crash reporting, routing, or compliance work without checking correctness;
* parallelize startup safely without dependency and failure analysis;
* treat a passing local run as proof that ordering is safe.

## Safe Patch Heuristics

When the agent is allowed to edit code, prefer small, reversible changes.

Good patch candidates:

* introduce named steps and explicit dependencies while preserving the current execution order;
* split a large serial startup method into named phases;
* extract explicit readiness states;
* replace comment-based ordering with named dependencies;
* add idempotency guards around startup steps;
* move clearly feature-specific setup behind feature activation;
* defer noncritical secondary work behind an explicit post-first-frame or post-first-interaction trigger;
* replace eager dependency construction with factories when call sites already tolerate laziness;
* add bounded task groups only after dependencies are explicit;
* add signposts or structured startup events around named steps;
* add fallback or degraded states for partial readiness.

Risky patch candidates requiring extra care:

* changing authentication, privacy, security, crash reporting, payment, or compliance startup order;
* converting a serial startup sequence into parallel work without dependency audit;
* changing deep-link, push, shortcut, or restoration readiness;
* changing database migration or keychain ordering;
* making launch show UI before required privacy/session state is known;
* introducing lazy setup when the first feature access has no loading or failure state;
* changing startup behavior across multiple scenes or windows;
* hiding work behind background queues without reducing contention or improving readiness.

If correctness is uncertain, recommend instrumentation or decomposition first, then behavior-changing optimization after evidence is available.

## Review Checklist

When reviewing a launch orchestration system, check:

* [ ] Is the startup sequence represented as named steps rather than one anonymous list?
* [ ] Does each step declare why it belongs on the launch path?
* [ ] Is each step classified as first-frame critical, first-interaction, secondary, lazy, or maintenance?
* [ ] Are dependencies explicit and minimal?
* [ ] Are main-thread requirements explicit?
* [ ] Are side effects documented as readiness conditions?
* [ ] Are failure, timeout, retry, and cancellation rules defined?
* [ ] Are fail-open/fail-closed defaults explicit for safety-sensitive steps?
* [ ] Can independent work run without blocking first frame?
* [ ] Is parallelism bounded?
* [ ] Are cycles impossible or detected?
* [ ] Is the longest dependency chain visible in measurement?
* [ ] Are wait time and run time measured separately for important steps?
* [ ] Can the first screen render with partial readiness?
* [ ] Can later features own their own setup instead of forcing global launch work?
* [ ] Are deferred tasks attached to explicit triggers and fallback states?
* [ ] Does deferred secondary work avoid starting as one large burst immediately after first frame?
* [ ] Is the recommendation connected to correctness validation and timing validation?

## Agent Guidance

When applying this reference, produce a graph-oriented review:

```markdown
### Launch graph assessment

Describe whether startup is serial, partially parallel, dependency-driven, or unclear.

### Critical path

Name the work that appears required before first frame or first interaction.

### Hidden dependencies

List dependencies currently implied by ordering, shared state, side effects, or comments.

### Parallelism safety

Explain what can run independently and what must remain ordered.

### Longest chain

Identify the chain most likely to determine launch duration.

### Partial readiness

Explain whether the app can show a useful first screen before full readiness.

### Correctness risks

Call out auth, routing, privacy, security, crash reporting, state restoration, multi-window, database, keychain, or compliance concerns.

### Recommended changes

Suggest changes that reduce launch-critical work, make dependencies explicit, or move feature-specific work out of global startup.

### Validation

Explain how to verify correctness and performance after changing orchestration.
```

Keep recommendations tied to evidence. If the code path or measurement boundary is unclear, label the finding as a hypothesis.

## Boundary With Other References

Use this reference for launch orchestration and dependency graph design.

Read `references/launch-taxonomy-and-targets.md` when the issue involves:

* cold, warm, prewarmed, resume, first install, or update launch terminology;
* launch target selection;
* measurement scenario classification.

Read `references/pre-main-dyld-and-static-initializers.md` when the issue involves:

* dyld;
* pre-main work;
* `+load`;
* `+initialize`;
* constructor functions;
* Objective-C categories;
* runtime registration;
* static initialization.

Read `references/linking-strategy.md` when the issue involves:

* dynamic frameworks;
* static libraries;
* mergeable libraries;
* modularization and launch-time linking trade-offs;
* binary layout;
* order-file considerations.

Read `references/appdelegate-scenedelegate-and-first-frame.md` when the issue involves:

* `UIApplicationDelegate`;
* `UISceneDelegate`;
* window setup;
* root view controller creation;
* first-frame readiness;
* main-thread lifecycle work.

Read `references/swiftui-app-launch.md` when the issue involves:

* SwiftUI `App`;
* `WindowGroup`;
* root view setup;
* observable state;
* `.task`;
* `.onAppear`;
* `scenePhase`;
* `@UIApplicationDelegateAdaptor`;
* environment initialization.

Read `references/third-party-sdks-at-launch.md` when the issue involves:

* analytics;
* crash reporting;
* ads;
* attribution;
* remote config;
* push;
* feature flags;
* security SDKs;
* vendor initialization strategy.

Read `references/metrics-instruments-xctest-metrickit.md` when the issue involves:

* Instruments;
* Time Profiler;
* signposts;
* XCTest launch metrics;
* MetricKit;
* Xcode Organizer;
* CI baselines;
* production monitoring.

Do not read all references by default.
