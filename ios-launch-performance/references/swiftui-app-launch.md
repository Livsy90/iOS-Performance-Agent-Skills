# SwiftUI App Launch

Use this reference when a launch investigation involves SwiftUI lifecycle code: `@main App`, root `Scene`, `WindowGroup`, root view setup, root observable state, environment injection, `.task`, `.task(id:)`, `.onAppear`, `scenePhase`, or SwiftUI-to-UIKit delegate bridging.

Keep this file focused on **SwiftUI work that affects launch, first frame, or early responsiveness**. Do not use it as a general SwiftUI performance guide for scrolling, diffing, identity, layout, invalidation, or rendering unless the issue is on the launch path.

## Scope Boundary

This file covers:

* SwiftUI lifecycle apps using `@main App`;
* `App` stored properties and `App.init`;
* root `Scene` and `WindowGroup` construction;
* root view creation;
* root observable model creation;
* `@StateObject`, `@State`, `@Observable`, `ObservableObject`, and environment-owned launch state;
* `.environment`, `.environmentObject`, `modelContainer`, and root dependency injection;
* `.task`, `.task(id:)`, `.onAppear`, `onChange`, and `scenePhase` work that affects launch or resume;
* `@UIApplicationDelegateAdaptor` and hybrid SwiftUI/UIKit startup paths;
* duplicate startup work across SwiftUI lifecycle, delegate adaptors, root views, and lifecycle-bound modifiers;
* first-frame and early post-launch responsiveness in SwiftUI apps.

This file does not cover:

* dyld, pre-main, `+load`, constructor functions, or static initializer diagnosis;
* static, dynamic, or mergeable linking strategy;
* detailed UIKit lifecycle implementation;
* third-party SDK startup policy;
* generic launch dependency graph design;
* tool-specific launch measurement setup;
* general SwiftUI scrolling, layout, identity, or invalidation performance unrelated to launch.

Use this file to identify SwiftUI entry points and launch-path ownership. Route implementation details to the focused reference when the issue belongs elsewhere.

## Contents

* [Core Model](#core-model)
* [Review Procedure](#review-procedure)
* [SwiftUI Launch Path](#swiftui-launch-path)
* [App Type and App.init](#app-type-and-appinit)
* [Scene and WindowGroup Construction](#scene-and-windowgroup-construction)
* [Root View and Root Model Setup](#root-view-and-root-model-setup)
* [Environment Injection](#environment-injection)
* [Lifecycle-Bound Work](#lifecycle-bound-work)
* [.task, .onAppear, and scenePhase](#task-onappear-and-scenephase)
* [Delegate Bridging in SwiftUI Lifecycle Apps](#delegate-bridging-in-swiftui-lifecycle-apps)
* [First-Frame and Early Responsiveness](#first-frame-and-early-responsiveness)
* [Duplicate Startup Paths](#duplicate-startup-paths)
* [What the Agent Can Inspect](#what-the-agent-can-inspect)
* [Safe Patch Heuristics](#safe-patch-heuristics)
* [Review Checklist](#review-checklist)
* [Agent Guidance](#agent-guidance)
* [Boundary With Other References](#boundary-with-other-references)

## Core Model

SwiftUI launch work is often distributed.

In a SwiftUI lifecycle app, startup work can appear in:

* stored properties of the `App` type;
* `App.init`;
* `Scene` construction;
* `WindowGroup`;
* root view initialization;
* root observable state creation;
* dependency containers injected through the environment;
* `modelContainer` setup;
* `@UIApplicationDelegateAdaptor`;
* `.task`;
* `.task(id:)`;
* `.onAppear`;
* `scenePhase` handlers;
* first-screen view models.

The danger is that launch code can look declarative and harmless while still building a large object graph, resolving dependencies, starting tasks, touching storage, or triggering main-actor work before the first screen is useful.

The goal is not to make the SwiftUI root empty. The goal is to keep only the work required for a valid first frame or first meaningful interaction on the launch path.

Do not treat every root `@StateObject`, `@State`, `@Observable`, or `ObservableObject` value as a launch problem. The risk is expensive initialization, broad dependency construction, early side effects, or heavy first access during launch.

Preserve SwiftUI ownership semantics. Moving a model out of `@StateObject`, `@State`, root-owned storage, or environment ownership can change lifetime, identity, cancellation, scene behavior, and state restoration.

## Review Procedure

When using this reference:

1. Identify whether the app uses SwiftUI lifecycle, UIKit lifecycle, or a hybrid setup.
2. Trace the launch path from `@main App` to the first visible view.
3. Inspect stored properties and `App.init`.
4. Inspect `Scene` and `WindowGroup` construction.
5. Inspect root view and root model initialization.
6. Inspect environment injection, dependency containers, and persistence setup.
7. Inspect `.task`, `.task(id:)`, `.onAppear`, `onChange`, and `scenePhase` work.
8. Check `@UIApplicationDelegateAdaptor` for hidden or duplicated startup work.
9. Classify each task as first-frame, first-interaction, soon-after-launch, resume-only, feature-lazy, or background maintenance.
10. Recommend the smallest safe change and define validation.

Do not treat SwiftUI lifecycle modifiers as automatically safe post-launch work. Confirm when they run and what resources they compete for.

## SwiftUI Launch Path

A typical SwiftUI lifecycle launch path can involve:

```text id="4k0o0q"
process and runtime startup
→ SwiftUI App value creation
→ App stored property initialization
→ App.init
→ body evaluation for scenes
→ WindowGroup / Scene construction
→ root view creation
→ root state and environment setup
→ first layout and rendering
→ lifecycle-bound work starts
→ early responsiveness
```

This is a simplified model. The exact path depends on app structure, OS behavior, scene lifecycle, prewarming, restoration, and entry point.

Do not assume that work moved out of `App.init` is automatically outside launch. It may still run during root scene construction, root view initialization, `.task`, `.onAppear`, or first-screen model setup.

## App Type and App.init

Treat the SwiftUI `App` type as a launch boundary.

High-risk patterns in `App` stored properties and `App.init`:

* dependency container construction;
* service locator setup;
* database opening or migration;
* keychain-heavy work;
* file reads;
* JSON or plist decoding;
* remote configuration;
* feature flag fetching;
* analytics or logging startup;
* SDK initialization;
* session restoration;
* root route resolution that requires I/O;
* large object graph construction;
* synchronous waits;
* main-actor blocking work.

`App.init` should usually be boring. It can establish minimal process-level state, but it should not become a replacement for the old “put everything in AppDelegate” pattern.

Prefer:

* cheap configuration;
* minimal local state;
* lightweight dependency descriptors;
* factories instead of fully built graphs;
* explicit startup phases;
* lazy feature-owned setup;
* a first screen that can render with partial readiness.

If `App.init` contains SDK startup, route SDK policy to `third-party-sdks-at-launch.md`.

If `App.init` builds a large ordered startup graph, route scheduling and dependency decisions to `launch-orchestration-and-dependency-graph.md`.

## Scene and WindowGroup Construction

`Scene` and `WindowGroup` construction can still be part of launch.

Review:

* whether scene construction creates heavy root views;
* whether `WindowGroup` captures large models or containers;
* whether route selection happens synchronously;
* whether scene creation duplicates process-wide setup;
* whether multiple scenes can recreate global work;
* whether scene restoration triggers first-launch-only work;
* whether the app builds more UI than the first visible scene needs.

Capturing a reference in `WindowGroup` is not automatically expensive. The risk is constructing the captured value eagerly at the root, resolving broad dependencies, or sharing process-wide mutable state across scenes unintentionally.

Prefer scene construction that installs a minimal valid root UI and moves secondary work behind explicit readiness or feature activation.

Do not assume a single global scene. SwiftUI apps can have multiple windows or scene instances depending on platform and configuration.

Decide whether a root model is process-wide or scene-owned. A process-wide model should not accidentally be recreated for every scene. A scene-owned model should not accidentally store scene-specific routing, navigation, restoration, or presentation state globally.

## Root View and Root Model Setup

Root views are often cheap as values, but their initialization and model ownership can still trigger launch work.

Review root-level use of:

* `@StateObject`;
* `@State`;
* `@Observable`;
* `ObservableObject`;
* `@EnvironmentObject`;
* `@Environment`;
* `@Bindable`;
* `modelContainer`;
* dependency containers;
* service locators;
* composition roots;
* routers and coordinators;
* first-screen view models.

High-risk patterns:

* root model creates the entire app graph;
* root model opens database or keychain synchronously;
* root model starts network or remote config immediately;
* root model creates view models for many tabs or flows;
* root view initializes feature modules not needed for first frame;
* root state depends on session, routing, SDKs, or persistence that could be partial;
* root environment contains many eager objects because it is convenient;
* unstructured tasks are started from root model initialization without clear lifetime or cancellation.

Prefer a small launch state model that can decide or display the first valid UI, then let features own their own setup.

Do not split root state blindly. Preserve ownership, lifetime, identity, cancellation, scene behavior, and restoration semantics.

Persistence containers can be launch-critical if they open stores, run migrations, validate schema, or touch disk before first frame. Check whether the first screen truly needs the full container immediately, whether migration should be measured separately, and whether the UI has a safe loading, locked, or degraded state.

Do not blindly defer persistence setup. Database or schema migration may be required before safe data access.

## Environment Injection

Environment injection can hide broad launch work.

Review:

* `.environment(...)`;
* `.environmentObject(...)`;
* custom environment keys;
* `@Environment`;
* `@EnvironmentObject`;
* `@Bindable`;
* persistence containers;
* model containers;
* resolver/container injection;
* global app state injection;
* feature service injection at the root.

High-risk patterns:

* building all environment values before the first visible UI;
* injecting many feature-specific objects globally;
* resolving dependencies while constructing environment values;
* using environment as a service locator for the entire app graph;
* making the first screen depend on all environment objects even when it only needs one small state;
* updating broad root environment state immediately after launch.

Prefer:

* lightweight environment values;
* factories or descriptors for feature-specific services;
* feature-local injection;
* lazy service creation at feature boundaries;
* small root state that only supports initial routing and first-frame UI.

For Observation-based models injected with `.environment(model)`, the launch risk is usually model construction, first access, and broad root ownership. For `ObservableObject` injected with `.environmentObject`, also watch for broad object-level invalidation after early startup updates.

If environment setup represents a large ordered dependency graph, use this file to identify the SwiftUI entry point, then route graph design to `launch-orchestration-and-dependency-graph.md`.

## Lifecycle-Bound Work

SwiftUI lifecycle-bound work can start early enough to affect launch.

Review:

* `.task`;
* `.task(id:)`;
* `.onAppear`;
* `.onChange`;
* `scenePhase` handlers;
* root view model `init`;
* root view model async methods started from modifiers;
* unstructured `Task {}` started from root model or view initialization;
* first-screen `.task` or `.onAppear`;
* tab or navigation root lifecycle modifiers.

Classify lifecycle-bound work by necessity:

* required before first frame;
* required before first interaction;
* useful soon after launch;
* resume-only;
* feature-specific and lazy;
* background maintenance.

Work that is not required before first frame or first interaction should not compete with launch-critical work unless there is a correctness reason.

## .task, .onAppear, and scenePhase

Do not assume `.task` is harmless because it is async.

`.task` can start while the root scene is becoming visible, compete for CPU, I/O, storage, locks, or main-actor time, trigger observable updates, and affect the first meaningful interaction.

A `.task` launched from a `@MainActor` view model or root model can still execute CPU-heavy work on the main actor. Moving code into an async function does not automatically move parsing, mapping, sorting, or dependency construction off the main actor.

Review `.task` and `.task(id:)` for:

* network calls;
* database access;
* keychain access;
* JSON or plist decoding;
* image decoding;
* SDK startup;
* dependency resolution;
* feature warmups;
* task fan-out;
* missing cancellation;
* repeated execution because the identity changes;
* main-actor work after an async call;
* updates to broad observable state.

For `.task(id:)`, inspect whether the id is stable. Changing the id cancels the previous task and starts a new one. This is useful for real input changes, but risky when the id changes because of unstable route, session, scene, or model identity during launch.

Review `.onAppear` for:

* repeated execution;
* duplicated work already started in `.task`;
* work triggered by container/root views instead of feature views;
* synchronous work before the UI becomes responsive;
* observer registration with side effects.

Do not use `.onAppear` as a one-time launch hook. It can run more than once because of navigation, scene changes, view identity changes, conditional roots, tab switches, restoration, or parent hierarchy changes.

Review `scenePhase` for:

* treating foreground resume as cold launch;
* refreshing too much data immediately on `.active`;
* duplicating launch work when returning from background;
* restarting tasks already owned by root views;
* running heavy work while the first scene is trying to become interactive;
* missing idempotency and cancellation.

Prefer explicit readiness and idempotency:

* a task should know why it runs;
* a task should know whether it is launch, resume, or feature work;
* a task should have cancellation behavior;
* repeated lifecycle events should not restart expensive work unnecessarily;
* early tasks should not update broad root state if a smaller model can own the result.

## Delegate Bridging in SwiftUI Lifecycle Apps

SwiftUI apps may still use UIKit launch hooks through:

```swift id="r82wip"
@UIApplicationDelegateAdaptor(AppDelegate.self) var appDelegate
```

Treat the adapted delegate as part of the launch path when it performs startup work.

Review:

* `application(_:didFinishLaunchingWithOptions:)`;
* `application(_:configurationForConnecting:options:)`;
* notification/deep-link handling;
* SDK initialization;
* push registration;
* background task registration;
* app-wide dependency setup;
* duplicate work also performed in SwiftUI `App.init`, root `.task`, `.onAppear`, or `scenePhase`.

Use this file to identify delegate bridging from SwiftUI.

Use `appdelegate-scenedelegate-and-first-frame.md` to review the delegate implementation itself.

Do not assume SwiftUI lifecycle removes AppDelegate launch cost when an adaptor is present.

## First-Frame and Early Responsiveness

A SwiftUI app can draw quickly and still feel slow if early tasks block the main actor, storage, CPU, or first interaction.

Review:

* whether the root UI can render with minimal state;
* whether first-screen state can be cached, placeholder-based, or partial;
* whether cached or last-known content is safe for the current auth, privacy, lock, and account state;
* whether broad observable updates immediately invalidate the root hierarchy;
* whether the first screen appears but then blocks on a required task;
* whether post-first-frame tasks compete with initial interaction;
* whether the app shows an interactive-looking UI before it is actually usable.

Cached or last-known content is acceptable only when it is safe for the current auth, privacy, lock, account, and session state. Do not show sensitive cached UI before the app knows whether it is allowed to display it.

Prefer:

* minimal first-frame state;
* explicit loading or disabled states;
* cached or placeholder first-screen content when safe;
* feature-owned loading;
* bounded post-first-frame work;
* signposts around root setup and first interaction readiness.

Do not hide important launch latency by moving everything after first draw if the app immediately blocks the user.

## Duplicate Startup Paths

SwiftUI launch code often duplicates work across multiple lifecycle surfaces.

Check for duplicate or competing startup work across:

* `App.init`;
* `WindowGroup`;
* root view initialization;
* root model initialization;
* `@UIApplicationDelegateAdaptor`;
* `.task`;
* `.onAppear`;
* `scenePhase`;
* dependency containers;
* SDK facades;
* first-screen view models.

Common symptoms:

* first launch and resume both trigger the same heavy setup;
* `.task` and `.onAppear` start the same request;
* delegate adaptor initializes an SDK and root view model initializes it again;
* root model builds services already built by a composition root;
* scenePhase `.active` refreshes data already loading from root `.task`;
* multiple scenes repeat app-wide bootstrap work.

Prefer explicit ownership. Each startup responsibility should have one owner, one trigger, idempotency, cancellation, and failure behavior.

## What the Agent Can Inspect

When repository access is available, inspect concrete SwiftUI launch-path code instead of giving generic advice.

Search for SwiftUI lifecycle entry points:

```sh id="jbdqbe"
rg "@main|struct .*: App|WindowGroup|Scene" .
```

Search for `App.init` and delegate bridging:

```sh id="u49w1v"
rg "struct .*: App|@main|^\s*init\(|@UIApplicationDelegateAdaptor|UIApplicationDelegateAdaptor|UIApplicationDelegate" .
```

This search is approximate. In many projects, `struct MyApp: App` and `init()` appear on different lines or in extensions.

Search for eager root state and environment injection:

```sh id="o9k1zb"
rg "@Environment\(|@EnvironmentObject|@Bindable|@StateObject|@State|@Observable|ObservableObject|\.environment\(|\.environmentObject\(" .
```

Search for persistence and dependency containers at the root:

```sh id="8ydru9"
rg "modelContainer|container|resolver|dependencies|graph|ServiceLocator|Assembler|CompositionRoot|make.*Container|build.*Graph" .
```

Search for lifecycle-bound startup work:

```sh id="5nznmw"
rg "\.task\s*\{|\.task\s*\(|\.onAppear\s*\{|scenePhase|onChange\s*\(of:.*scenePhase" .
```

Search for unstructured async work started near launch surfaces:

```sh id="bw8tjb"
rg "Task\s*\{|Task\(|Task\.detached|async let|withTaskGroup|await .*load|await .*start|await .*initialize|await .*bootstrap" .
```

Search for expensive work near SwiftUI launch surfaces:

```sh id="wyxpqy"
rg "Data\(|contentsOf:|FileManager|Keychain|SecItem|JSONDecoder|PropertyListDecoder|URLSession|migrate|openDatabase|resolveAll|registerAll|wait\(|semaphore|sync\(" .
```

Use search results as leads, not proof. Confirm whether matched code runs before first frame, before first interaction, on resume, per scene, or only after a later feature appears.

The agent can:

* trace SwiftUI launch path from `@main App` to first visible view;
* identify heavy app stored properties and `App.init` work;
* identify heavy root model construction;
* identify eager environment and dependency setup;
* classify `.task`, `.onAppear`, `onChange`, and `scenePhase` work by launch necessity;
* detect duplicate startup paths between SwiftUI lifecycle code and delegate adaptors;
* propose smaller launch state or feature-owned setup when safe;
* recommend signposts around SwiftUI root setup and first-screen readiness.

The agent cannot reliably:

* treat every `.task` as post-launch;
* treat every SwiftUI state object as launch cost;
* change root state ownership safely without understanding lifetime and identity;
* defer auth, privacy, security, routing, or compliance work without product context;
* prove first-frame timing without measurement;
* decide SDK startup policy without SDK-specific constraints.

## Safe Patch Heuristics

When the agent is allowed to edit code, prefer small, reversible changes.

Good patch candidates:

* add phase markers around `App.init`, root scene construction, root model creation, first `.task`, and first interaction readiness before moving behavior;
* move clearly noncritical work out of `App.init`;
* split root model into minimal launch state and feature-owned state;
* replace eager root environment object construction with lightweight factories when call sites tolerate laziness;
* move noncritical `.task` work behind explicit post-first-frame, post-first-interaction, or feature-specific triggers;
* add idempotency guards to `.task`, `.onAppear`, `scenePhase`, and delegate-adaptor work;
* prevent duplicate work across `.task`, `.onAppear`, `scenePhase`, and delegate callbacks;
* make the first screen show cached, placeholder, loading, or disabled state while secondary work continues;
* move feature-specific environment injection closer to the feature boundary;
* add signposts around SwiftUI root setup and first-screen readiness.

Risky patch candidates requiring extra care:

* moving authentication, privacy lock, security, payment, fraud, or compliance routing out of launch;
* changing `@UIApplicationDelegateAdaptor` behavior without checking delegate responsibilities;
* changing root state ownership from `@StateObject`, `@State`, `@Observable`, or environment without understanding lifetime;
* removing environment objects used by deep links, notifications, routing, or restoration;
* making `.task` work lazy when visible UI has no loading, failure, or retry state;
* backgrounding work that immediately updates main actor state or competes with first interaction;
* moving persistence or model container setup without checking first-screen data needs;
* changing multi-scene behavior without idempotency checks.

If correctness is uncertain, recommend instrumentation or decomposition first, then behavior-changing optimization after evidence is available.

## Review Checklist

Use this checklist when reviewing SwiftUI launch code.

* [ ] Is this truly launch or early responsiveness rather than general SwiftUI performance?
* [ ] Does the app use SwiftUI lifecycle, UIKit lifecycle, or a hybrid setup?
* [ ] Are `App` stored properties cheap?
* [ ] Is `App.init` free of broad synchronous setup?
* [ ] Does `WindowGroup` construct only the UI needed for the first visible scene?
* [ ] Are root models minimal and cheap to create?
* [ ] Is root model ownership correct for process-wide vs scene-owned state?
* [ ] Are environment values lightweight?
* [ ] Are feature-specific services injected closer to feature boundaries?
* [ ] Is persistence/model-container setup required before first frame or first interaction?
* [ ] Does `.task` work have a launch, resume, or feature-specific reason?
* [ ] Are `.task(id:)` identifiers stable and intentional?
* [ ] Are `.task`, `.onAppear`, and `scenePhase` handlers idempotent?
* [ ] Is duplicate startup work avoided across delegate adaptor and SwiftUI lifecycle code?
* [ ] Can the first screen render with partial, cached, or placeholder state safely?
* [ ] Does early async work avoid blocking first interaction?
* [ ] Are auth, privacy, security, routing, restoration, and compliance requirements preserved?
* [ ] Is the recommendation connected to a validation plan?

## Agent Guidance

When applying this reference, produce a SwiftUI launch-oriented review:

```markdown id="6193sl"
### SwiftUI launch path

Describe the path from `@main App` to the first visible view.

### Launch-critical SwiftUI work

Name work in `App.init`, `Scene`, `WindowGroup`, root view/model setup, environment injection, delegate adaptor, `.task`, `.onAppear`, or `scenePhase` that appears to affect first frame or first interaction.

### Necessity classification

Classify work as first-frame, first-interaction, soon-after-launch, resume-only, feature-lazy, or background maintenance.

### Duplicate startup paths

Identify work repeated across SwiftUI lifecycle, delegate adaptor, root models, `.task`, `.onAppear`, or `scenePhase`.

### Recommended changes

Suggest the smallest safe changes: reduce eager root setup, split launch state, move feature work later, add idempotency, or add explicit readiness.

### Correctness risks

Call out auth, routing, privacy, security, state restoration, multi-scene, persistence, SDK, or compliance concerns.

### Unknowns / evidence needed

What is not proven yet: whether the work runs before first frame, before first interaction, only on resume, per scene, or only after feature navigation.

### Validation

Explain how to verify first-frame and early responsiveness with trace, signposts, XCTest launch metrics, manual interaction checks, or production data.
```

Keep recommendations tied to evidence. If the timing boundary is unclear, label the finding as a hypothesis and recommend measurement.

## Boundary With Other References

Use this reference for SwiftUI lifecycle work that affects launch, first frame, or early responsiveness.

Read `references/launch-taxonomy-and-targets.md` when the issue involves:

* cold, warm, prewarmed, resume, first install, or update launch terminology;
* launch target selection;
* measurement scenario classification;
* whether two numbers are comparable.

Read `references/pre-main-dyld-and-static-initializers.md` when the issue involves:

* dyld;
* pre-main work;
* `+load`;
* `+initialize`;
* constructor functions;
* Objective-C categories;
* runtime registration;
* static initialization before SwiftUI lifecycle begins.

Read `references/linking-strategy.md` when the issue involves:

* dynamic frameworks;
* static libraries;
* mergeable libraries;
* modularization and launch-time linking trade-offs;
* binary layout;
* order-file considerations.

Read `references/launch-orchestration-and-dependency-graph.md` when the issue involves:

* root SwiftUI setup resolving a large ordered startup graph;
* critical path analysis;
* startup step dependencies;
* hidden ordering;
* safe parallelism;
* failure policy;
* dependency-chain optimization.

Read `references/appdelegate-scenedelegate-and-first-frame.md` when the issue involves:

* implementation details inside `UIApplicationDelegate`;
* implementation details inside `UISceneDelegate`;
* window setup;
* root view controller creation;
* first-frame readiness in UIKit lifecycle code;
* main-thread lifecycle work.

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
