# SwiftUI App Launch

Use this reference when a launch investigation involves SwiftUI lifecycle code: `@main App`, root `Scene` construction, `WindowGroup`, root view setup, root observable state, environment injection, `.task`, `.task(id:)`, `.onAppear`, `scenePhase`, or SwiftUI-to-UIKit delegate bridging.

Keep this file focused on SwiftUI-specific launch behavior. Do not use it as the primary guide for dyld/pre-main work, framework linking, UIKit lifecycle implementation, third-party SDK policy, launch orchestration, or launch measurement tooling.

## Scope Boundary

This file covers:

- SwiftUI `@main App` startup code
- stored properties and `init` in the app type
- root `Scene` and `WindowGroup` construction
- root view initialization
- root model ownership with `@StateObject`, `@State`, `@Observable`, `ObservableObject`, and environment injection
- eager dependency setup caused by root view/model construction
- `.task`, `.task(id:)`, `.onAppear`, and `scenePhase` work that starts during or immediately after launch
- `@UIApplicationDelegateAdaptor` in SwiftUI lifecycle apps
- SwiftUI-specific first-screen readiness
- post-visible work that can still affect early responsiveness

This file does not cover:

- dyld, pre-main, Objective-C `+load`, constructor functions, or static initializer internals; use `pre-main-dyld-and-static-initializers.md`
- static vs dynamic linking, mergeable libraries, or framework dependency strategy; use `linking-strategy.md`
- UIKit `UIApplicationDelegate` / `UISceneDelegate` implementation, root `UIWindow`, or `UIViewController` first-frame setup; use `appdelegate-scenedelegate-and-first-frame.md`
- launch-step dependency graphs, critical-path scheduling, and safe launch parallelism; use `launch-orchestration-and-dependency-graph.md`
- SDK-specific startup policy for analytics, ads, crash reporting, attribution, push, remote config, feature flags, or security vendors; use `third-party-sdks-at-launch.md`
- Instruments, Time Profiler, XCTest, MetricKit, Organizer, signpost design, or CI baselines; use `metrics-instruments-xctest-metrickit.md`
- general SwiftUI scrolling, diffing, identity, animation, layout, or memory performance unless the code is on the launch path

## Core Model

In a SwiftUI lifecycle app, SwiftUI creates the app value, evaluates its scenes, creates the active scene's root view hierarchy, manages observable state ownership, and starts lifecycle-bound work as views and scenes enter the hierarchy.

For launch review, model the SwiftUI path as:

```text
@main App instance
→ app stored properties
→ App.init
→ Scene declaration
→ WindowGroup root content
→ root view and root model creation
→ environment setup
→ first root body evaluation / layout / draw
→ lifecycle-bound startup work
→ early responsiveness
```

The goal is not to avoid all startup work in SwiftUI. The goal is to keep the first visible shell cheap, deterministic, and correct while moving secondary work behind explicit readiness points.

Do not assume SwiftUI lifecycle callbacks are single-use launch hooks. Root views can be recreated, tasks can be cancelled and restarted, scene phase changes also happen during resume, and observable models can trigger broad updates immediately after launch.

## What the Agent Can Inspect

When repository access is available, inspect SwiftUI launch entry points and root dependencies before giving generic advice.

Search for SwiftUI app entry points:

```sh
rg "@main|: App|WindowGroup|DocumentGroup|Settings\\s*\\{" .
```

Search for UIKit delegate bridging in SwiftUI lifecycle apps:

```sh
rg "UIApplicationDelegateAdaptor" .
```

Search for root lifecycle-triggered work:

```sh
rg "\\.task\\s*\\{|\\.task\\s*\\(|\\.onAppear\\s*\\{|scenePhase|onChange\\s*\\(of:.*scenePhase" .
```

Search for eager root state and dependency injection:

```sh
rg "@StateObject|@Observable|ObservableObject|@EnvironmentObject|\\.environment\\(|\\.environmentObject\\(|modelContainer|container|resolver|dependencies|graph" .
```

Search for expensive work that may be reached from root initialization or root lifecycle callbacks:

```sh
rg "Data\\(|contentsOf:|FileManager|Keychain|SecItem|JSONDecoder|PropertyListDecoder|migrate|migration|open|load|fetch|refresh|sync|wait\\(|semaphore|DispatchQueue\\.main\\.sync|Task\\.detached" .
```

Use search results as leads only. Confirm whether the code is actually reached during launch, whether it blocks the main actor, and whether the first visible UI or first interaction depends on it.

The agent can:

- trace the SwiftUI launch path from `@main App` to the first visible view
- identify heavy app stored properties, `App.init` work, root model construction, and environment setup
- classify root `.task`, `.onAppear`, and `scenePhase` work by launch necessity
- detect duplicate startup paths between SwiftUI lifecycle code and delegate adaptors
- suggest splitting launch-critical state from full app state
- suggest lazy creation of feature-specific services and root models
- propose idempotency for lifecycle callbacks that may run more than once
- hand off UIKit, SDK, orchestration, or measurement details to the appropriate reference
- propose small behavior-preserving patches when repository context is sufficient

The agent cannot reliably:

- prove first-frame timing without measurement or a trace
- infer whether `.task` starts after the first frame is displayed
- assume `.onAppear` runs only once
- assume `scenePhase == .active` means a fresh app launch
- move UI-bound work away from the main actor without checking API and isolation requirements
- defer authentication, routing, security, crash reporting, privacy, or compliance work without product context
- know whether a root model initializer is expensive without inspecting its call graph
- assume Observation automatically solves launch cost; model construction and first reads can still be expensive

## SwiftUI Launch Review Areas

### App Stored Properties and `App.init`

Review the `@main App` type first. Stored properties and `init` often look harmless while hiding work that must finish before the first scene can render.

Look for:

- dependency containers created eagerly
- global coordinators that start subsystems in their initializer
- persistence containers or model containers created before the first scene appears
- keychain or session reads that expand into slow call chains
- remote configuration, token refresh, or network checks started from the app type
- analytics, logging, push, attribution, feature flags, or SDK startup in the app type
- work hidden behind names such as `shared`, `live`, `default`, `make`, `bootstrap`, `configure`, `resolve`, or `start`
- root models that subscribe to many notifications, publishers, async streams, or observers during construction

Prefer:

- a small app type that creates only the first visible shell
- cheap local state used to choose the initial route
- explicit bootstrap phases instead of hidden work in stored properties
- lazy services for features that are not visible at launch
- a narrow launch model instead of a full application graph
- fast local decisions for routing, security, and session state when correctness requires them before UI

Avoid:

- starting every subsystem because `App.init` is a convenient central location
- doing blocking I/O before any scene can render
- making the root dependency container resolve all services eagerly
- hiding expensive work behind property wrappers, singletons, or static factories
- starting unowned background tasks from `App.init` without cancellation, priority, and failure handling

Review direction:

- Keep `App.init` minimal.
- If state is required before the first view, make the state small and locally available.
- Move secondary work to explicit startup methods, post-visible phases, or feature-owned lazy initialization.
- Route SDK-specific decisions to `third-party-sdks-at-launch.md`.
- Route multi-step dependency scheduling to `launch-orchestration-and-dependency-graph.md`.

### Root `Scene` and `WindowGroup`

Review the closure that creates the first SwiftUI hierarchy.

Look for:

- root views that construct many feature modules immediately
- branches that compute expensive state before choosing the initial route
- expensive environment values computed inline
- root modifiers that start persistence, networking, or SDK setup
- model containers attached at the root without checking whether the first screen needs them immediately
- synchronous restoration of large navigation or tab state
- eager setup for tabs, flows, or features that are not visible on the first screen

Prefer:

- a lightweight shell that can render with minimal local state
- route decisions based on cheap already-local information
- factories or handles instead of fully initialized feature graphs
- first-screen dependencies separated from later-tab dependencies
- progressive loading for first-screen content
- safe defaults when remote configuration is not yet available and product rules allow it

Avoid treating `WindowGroup` as an application bootstrap method. It declares scenes and root content; it should not become a hidden service startup hub.

### Root View Initialization

SwiftUI views are values and may be recreated. The launch risk is not the existence of a view initializer; the risk is expensive, side-effectful, or blocking work while constructing the root hierarchy.

Look for root view initializers that:

- synchronously load files or decode large data
- open databases or perform migrations
- touch keychain or secure storage repeatedly
- resolve many services from a container
- create image caches, formatters, search indexes, or stores eagerly
- start tasks, timers, observers, or notifications as side effects
- compute first-screen content that could be cached, placeholder-based, or asynchronous

Prefer initializers that only assign already-prepared dependencies or cheap value state.

If setup is required, move it to a model or coordinator with an explicit method and clear lifecycle. Then decide whether that method must run before first frame, before first interaction, after the shell is visible, or only when a feature is opened.

### Root Observable State

Root observable objects and models are frequent launch bottlenecks because their initializers often become application startup code.

Check:

- `@StateObject` initializers in root views
- `@State` owners for `@Observable` models
- `ObservableObject` instances injected as environment objects
- global stores or app state objects passed through root environment
- model initializers that resolve services, fetch data, subscribe broadly, or start refreshes
- immediate mutations after construction that invalidate a large part of the root hierarchy

Prefer:

- cheap model construction
- explicit async loading after construction
- a small launch-specific state object separate from full app state
- lazy child models for screens not visible at launch
- cached state for the first visible UI
- narrow observable dependencies for the first shell

Avoid:

- making one root app state object own every subsystem immediately
- starting network refreshes, data sync, migrations, or cache cleanup in model `init`
- using `@StateObject` or `@State` ownership as a place to hide expensive startup work
- injecting a massive environment object that constructs the whole app before the first screen

When suggesting changes, preserve SwiftUI ownership rules. Do not replace `@StateObject` with `@ObservedObject` just to change launch timing. Fix the initializer, ownership boundary, or loading phase instead.

Observation can reduce unnecessary view updates compared with broader observable patterns, but it does not make expensive model creation free. Treat model initialization, first property reads, and initial async loading as separate launch concerns.

## Lifecycle-Triggered Work

### `.task`

Treat `.task` as asynchronous work whose lifetime is tied to the modified view. SwiftUI can cancel it when the view disappears or its identity changes. With `.task(id:)`, changing the id can cancel and restart the task.

This makes `.task` useful for cancellable view-bound work, but it is not a guarantee that the work starts only after the first frame is displayed.

Review root `.task` closures for:

- long main-actor sections
- synchronous work before the first suspension point
- immediate network or persistence work needed only by later features
- repeated execution because the view is recreated or reinserted
- `.task(id:)` restarts caused by unstable identifiers
- cancellation and failure behavior
- priority that competes with early interaction
- duplicate startup also triggered from `App.init`, delegate adaptors, `.onAppear`, or `scenePhase`

Prefer:

- short setup before the first suspension point
- cancellable async work
- stable ids when restart behavior is intentional
- explicit guards for one-time startup when one-time semantics are required
- feature-specific work started by feature entry points, not the root shell
- clear separation between first-frame work and early post-visible work

Do not describe `.task` as automatically post-render. Say that it is lifecycle-bound async work, then require measurement if the launch impact matters.

### `.onAppear`

Treat `.onAppear` as a visibility callback, not as a one-time launch hook.

Review `.onAppear` closures for:

- repeated startup work when navigation, scene changes, identity changes, or conditional root branches reinsert the view
- synchronous main-thread work
- unstructured `Task` creation without cancellation ownership
- duplicate calls also triggered by `.task` or `scenePhase`
- state mutations that cause immediate expensive root re-rendering

Prefer:

- `.task` for cancellable async work tied to a view lifetime
- an explicit model or coordinator method for one-time app/session setup
- idempotency when repeated execution would be incorrect or expensive
- small appearance work that updates visible UI rather than global app state

### `scenePhase`

Use `scenePhase` for scene lifecycle reactions. Do not use it as a blanket cold-launch detector.

Review `scenePhase` handlers for:

- running launch setup on every `.active` transition
- treating foreground resume as cold launch
- refreshing too much data immediately when returning from background
- duplicating work already started from `.task`, `.onAppear`, `App.init`, or delegate callbacks
- doing heavy work while the first scene is trying to become interactive
- work that should happen on background transition but waits until the next active transition

Prefer:

- separate handling for first launch and foreground resume
- freshness-based refresh policies
- cheap foreground checks followed by lazy or cancellable refresh
- explicit state that records whether initial setup already completed
- minimal background cleanup that runs promptly when entering background

Use `launch-taxonomy-and-targets.md` when the distinction between cold launch, warm launch, prewarmed launch, and resume matters.

## Delegate Bridging in SwiftUI Lifecycle Apps

SwiftUI lifecycle apps can still bridge to UIKit delegate callbacks with `@UIApplicationDelegateAdaptor`. Treat the adapted delegate as part of the launch path when it starts work during launch.

Review:

- `@UIApplicationDelegateAdaptor` declarations
- adapted delegate initializers
- `application(_:didFinishLaunchingWithOptions:)` in the adapted delegate
- push notification, deep link, security, crash reporting, or SDK setup performed through the delegate
- duplicated initialization between the app type, root tasks, scene phase handlers, and delegate callbacks

Do not move all delegate logic into SwiftUI views. Use the app/scene delegate reference for UIKit lifecycle responsibilities and the SDK reference for vendor startup policy. In this SwiftUI reference, focus on whether bridging creates hidden eager work or duplicate startup paths.

## First-Screen Readiness in SwiftUI

For launch review, distinguish between rendering a valid first shell and completing all app data loading.

A valid first SwiftUI shell may contain:

- a local session decision
- a locked, login, onboarding, loading, or main-shell route
- cached first-screen data
- placeholder or skeleton content
- disabled controls with clear loading state
- minimal navigation chrome
- a small state machine that represents startup progress

It usually does not need to contain:

- refreshed remote configuration when safe defaults exist
- fully synchronized user data
- secondary tab data
- personalized recommendations
- warmed caches for later screens
- feature modules not visible on the first screen
- cleanup, compaction, analytics upload, or maintenance work

Be careful with authentication, privacy, and routing. If the first screen must not be shown until a local security decision is made, keep that decision launch-critical but make it as small and local as possible. Do not replace required security or compliance gates with a visual shortcut.

If deep links, push notifications, shortcuts, widgets, or universal links can determine the first route, classify the minimum routing state as launch-critical. Defer only the work that is not required to choose and display the correct initial destination.

## Review Questions

When reviewing SwiftUI launch code, ask:

1. What is the first SwiftUI view that must become visible?
2. Which state is strictly required to choose that view?
3. Which app stored properties and initializers run before that view can be created?
4. Which root models are created eagerly, and what do their initializers do?
5. Which environment values are computed eagerly?
6. Which services are resolved before the first visible shell?
7. Which `.task`, `.onAppear`, or `scenePhase` callbacks start during launch or immediately after the first scene appears?
8. Can this work run more than once because of SwiftUI lifecycle behavior?
9. Is startup work split into launch-critical, first-interaction, post-visible, and feature-lazy phases?
10. Is any UIKit delegate bridge duplicating work already started from SwiftUI?
11. Does the first screen require full app readiness, or only a narrow subset of state?
12. What measurement will prove that the change improves first frame or early responsiveness?

## Common Findings and Recommended Direction

### Heavy App Initialization

Finding:

- The app type constructs a full graph or starts services before any scene can render.

Recommended direction:

- Keep the app type minimal.
- Create only launch-critical state.
- Move secondary startup to explicit async phases, post-visible phases, or feature-owned lazy initialization.
- Use the SDK and orchestration references when the work involves vendor startup or many ordered launch steps.

### Heavy Root Observable Model

Finding:

- The root model performs I/O, data fetches, migrations, broad subscription setup, or service registration in `init`.

Recommended direction:

- Make construction cheap.
- Move work to explicit methods.
- Start only the subset required for first-screen correctness.
- Use cached, placeholder, or loading state when product rules allow it.
- Keep root observable dependencies narrow.

### Root `.task` Does Too Much

Finding:

- A root `.task` starts many operations immediately, including work not needed for the first screen.

Recommended direction:

- Split the task into smaller phases.
- Keep the first phase short and cancellable.
- Move feature-specific operations to feature entry points.
- Avoid long main-actor sections.
- Validate early responsiveness, not only time to first frame.

### `.onAppear` Used as a Launch Hook

Finding:

- `.onAppear` starts one-time app setup and can run more than once.

Recommended direction:

- Move one-time session setup to an explicit app/session model with idempotency.
- Use `.task` for cancellable view-bound async work.
- Guard repeated work when repetition would be incorrect or expensive.

### `scenePhase` Refresh Causes Resume Jank

Finding:

- The app refreshes large state on every `.active` transition.

Recommended direction:

- Separate launch from resume.
- Check freshness before refresh.
- Make resume work cancellable and priority-aware.
- Do not block early interaction on noncritical refresh.

### Environment Injection Builds Too Much

Finding:

- Root environment injection creates a complete dependency graph or many environment objects before the first screen.

Recommended direction:

- Inject lightweight handles, factories, or launch-specific dependencies.
- Create feature dependencies when the feature becomes visible.
- Avoid storing full app graphs in the environment unless construction is cheap and measured.

### Delegate Bridge Duplicates Startup

Finding:

- Startup is triggered from both `@UIApplicationDelegateAdaptor` and SwiftUI root lifecycle hooks.

Recommended direction:

- Assign one owner for each startup responsibility.
- Keep delegate-owned work limited to responsibilities that belong in the app delegate path.
- Keep view-owned work tied to visible UI or feature lifecycle.
- Add idempotency where multiple lifecycle events can legitimately request the same operation.

## Patch Guidance

When proposing code changes, prefer small, behavior-preserving patches:

- split app-type startup into critical and noncritical phases
- replace eager root service construction with lazy factories
- move heavy root model work from `init` to explicit async methods
- add idempotency to startup methods triggered by view lifecycle
- move feature-specific startup from the root shell to feature entry points
- reduce the first root environment to launch-critical dependencies
- add lightweight app-owned signposts around SwiftUI startup phases when measurement is missing

Do not patch by:

- moving work into a detached task without ownership, cancellation, priority, and failure reasoning
- replacing required security, auth, privacy, or routing decisions with unsafe placeholders
- changing SwiftUI ownership wrappers without checking lifecycle semantics
- hiding work behind a different initializer or singleton
- pushing all work to the background without considering first interaction correctness
- using `MainActor.run` as a blanket fix for isolation problems created by moved startup work

## Validation Handoff

This reference can recommend what to validate, but tool-specific details belong in `metrics-instruments-xctest-metrickit.md`.

For SwiftUI launch changes, validation should answer:

- Did the first visible app frame become earlier?
- Did early responsiveness improve, stay the same, or regress?
- Did root lifecycle-bound work move out of the critical path or merely shift jank later?
- Did the change accidentally duplicate work on resume, scene recreation, deep link launch, or notification launch?
- Did cancellation and idempotency still behave correctly?
- Did authenticated, unauthenticated, first-install, returning-user, upgraded-app, deep-link, and notification launches still route correctly?
- Did the change preserve SwiftUI state ownership semantics?

## Boundary With Other References

This reference should answer:

- Is SwiftUI app/root lifecycle code doing too much during launch?
- Is root observable state cheap enough to construct before first frame?
- Are `.task`, `.onAppear`, and `scenePhase` being used with correct lifecycle assumptions?
- Is SwiftUI-to-UIKit delegate bridging duplicating startup work?
- Can the first SwiftUI shell render without full app-wide readiness?

This reference should not answer:

- how dyld loads frameworks or runs static initializers
- whether a module should be static, dynamic, or mergeable
- how to restructure UIKit app/scene delegate code
- how to design a multi-step launch scheduler
- how to configure XCTest launch metrics or interpret MetricKit payloads
- how to optimize general SwiftUI list, animation, diffing, or scrolling performance outside launch
