# AppDelegate, SceneDelegate, and First Frame

Use this reference when a launch investigation points to UIKit lifecycle code, scene lifecycle code, root UI creation, first-screen preparation, launch routing, dependency setup called from lifecycle callbacks, or main-thread work that delays the first visible app-rendered frame.

Keep this file focused on the path after the app enters its own lifecycle code and before the first useful UI is visible and responsive.

Do not use this file as the primary guide for dyld, linking, SwiftUI `App`, vendor SDK policy, large launch orchestration systems, or measurement tooling. Route to the focused reference for those topics.

## Scope Boundary

This file covers:

* `UIApplicationDelegate` launch callbacks;
* `UISceneDelegate` connection and activation callbacks when they affect startup;
* apps with and without the scene lifecycle;
* `UIWindow` and root view controller setup;
* first-screen coordinator or view controller creation;
* launch options, universal links, notification launches, shortcuts, handoff, and restoration when they affect initial routing;
* app-owned dependency setup called directly from lifecycle code;
* main-thread work before first app-rendered frame and early responsiveness;
* deciding what must stay on the first-frame path and what can move later.

This file does not cover:

* dyld, pre-main work, Objective-C `+load`, `+initialize`, constructor functions, or static initializer contents; use `pre-main-dyld-and-static-initializers.md`;
* static vs dynamic linking, mergeable libraries, framework count, or binary layout; use `linking-strategy.md`;
* large ordered launch step systems, dependency graph scheduling, or safe parallel startup orchestration; use `launch-orchestration-and-dependency-graph.md`;
* SwiftUI `@main App`, root SwiftUI views, `.task`, `.onAppear`, `scenePhase`, or observable state setup; use `swiftui-app-launch.md`;
* SDK-specific startup decisions for analytics, ads, crash reporting, attribution, push, remote config, feature flags, or security vendors; use `third-party-sdks-at-launch.md`;
* Instruments, XCTest, MetricKit, Organizer, signpost design, or CI baselines; use `metrics-instruments-xctest-metrickit.md`;
* general rendering, scrolling, memory, networking, or architecture review unless the work is on the launch path.

## Contents

* [Core Model](#core-model)
* [Lifecycle Facts to Preserve](#lifecycle-facts-to-preserve)
* [Review Procedure](#review-procedure)
* [App Delegate Review](#app-delegate-review)
* [Scene Delegate Review](#scene-delegate-review)
* [Apps Without Scene Lifecycle](#apps-without-scene-lifecycle)
* [Startup Task Classification](#startup-task-classification)
* [Work That May Need to Stay Early](#work-that-may-need-to-stay-early)
* [Root UI and First Frame](#root-ui-and-first-frame)
* [Routing, Launch Options, and State Restoration](#routing-launch-options-and-state-restoration)
* [Dependency Setup on the Launch Path](#dependency-setup-on-the-launch-path)
* [Main-Thread Startup Work](#main-thread-startup-work)
* [Deferral Patterns](#deferral-patterns)
* [Multiple Scenes and Idempotency](#multiple-scenes-and-idempotency)
* [What the Agent Can Inspect](#what-the-agent-can-inspect)
* [Safe Patch Heuristics](#safe-patch-heuristics)
* [Review Checklist](#review-checklist)
* [Output Guidance](#output-guidance)

## Core Model

After pre-main work finishes, UIKit asks the app to prepare itself for execution and user interaction.

In a UIKit app, launch-time app code commonly flows through:

```text
App delegate creation and stored-property initialization
→ willFinishLaunching / didFinishLaunching
→ scene session configuration, if scenes are used
→ scene connection, if a UI scene is created or restored
→ window and root controller creation
→ first-screen state preparation
→ first layout, draw, and frame commit
→ early responsiveness after visible UI
```

In this file, “first frame” means the first app-rendered frame after the system launch screen, not the static launch screen itself. When product correctness matters, prefer “first useful UI” or “first safe UI” over a purely visual first-frame target.

The goal is not to make lifecycle methods empty. The goal is to keep only work required for a correct first frame or first interaction on this path.

## Lifecycle Facts to Preserve

Use these facts when reviewing code:

* UIKit creates the app delegate early in the app launch cycle.
* Apps that use scenes still use the app delegate for process-level launch and termination responsibilities.
* `application(_:willFinishLaunchingWithOptions:)` and `application(_:didFinishLaunchingWithOptions:)` are app-launch callbacks, not a general place for all app setup.
* Treat `willFinishLaunching` as an even narrower hook than `didFinishLaunching`. Avoid moving work earlier into `willFinishLaunching` unless the platform or product requirement genuinely needs it before normal launch completion.
* `application(_:configurationForConnecting:options:)` can be called when UIKit needs a scene configuration. It should choose or return configuration cheaply, not build the full UI graph.
* `scene(_:willConnectTo:options:)` is called when UIKit creates or restores an instance of the app UI. This can happen on initial launch, but also when creating or restoring additional scenes.
* A scene owns a UI instance: windows, view controllers, and scene-specific coordination belong there when the app uses the scene lifecycle.
* Multiple scenes share the same app process and app-wide services. App-wide bootstrap must not accidentally run once per scene.
* Do not treat `sceneDidBecomeActive(_:)` or `sceneWillEnterForeground(_:)` as one-time launch hooks. They can run repeatedly and should not redo process bootstrap or heavy first-launch setup unless guarded by explicit state.
* If a remote notification launches a non-running app, launch information can be delivered through launch options or notification response handling depending on the launch path. Do not assume the normal notification callback runs before launch handling.
* For user-tapped notifications, keep launch options and notification response handling coordinated. For silent pushes or background launches, platform callbacks and timing differ; do not assume the same route as a foreground user launch.

## Review Procedure

When this reference is relevant, follow this procedure:

1. Confirm the scenario is launch-related, not only resume, scene restoration, foreground refresh, or prewarmed-process confusion.
2. Identify which lifecycle owner runs the work: app delegate, scene delegate, root UI factory, root coordinator, or first-screen setup.
3. Separate app-wide process bootstrap from scene-specific UI construction.
4. Classify each startup task by necessity: first frame, first interaction, soon after launch, feature-lazy, or background maintenance.
5. Look for synchronous or hidden eager work on the main thread.
6. Check whether routing, notification, universal link, shortcut, or restoration input forces early decisions.
7. Preserve correctness requirements such as auth, privacy lock, security, crash reporting, state restoration, multi-window behavior, and compliance.
8. Recommend the smallest safe change and a concrete validation path.

## App Delegate Review

Use the app delegate for process-level startup and app-wide lifecycle coordination.

Review:

* app delegate `init` and stored properties;
* `application(_:willFinishLaunchingWithOptions:)`;
* `application(_:didFinishLaunchingWithOptions:)`;
* `application(_:configurationForConnecting:options:)`;
* bootstrap objects called from these methods;
* app-wide service containers created from the app delegate.

Appropriate launch-time responsibilities may include:

* installing a small bootstrap coordinator;
* configuring app-wide invariants that must exist before any UI code runs;
* selecting high-level local routing state when the decision is cheap;
* registering required process-level handlers;
* preparing scene configuration without building scene UI;
* returning quickly so scene connection and first-frame work can continue.

High-risk patterns:

* full dependency graph construction;
* opening or migrating databases before the first screen needs them;
* synchronous keychain-heavy checks;
* waiting for network, reachability, token refresh, remote config, or feature flag fetches;
* large file reads, JSON parsing, or local index construction;
* registering every observer, router, notification pipeline, and feature service eagerly;
* starting feature modules that are unreachable from the first screen;
* creating UI in the app delegate when a scene delegate also owns the window;
* doing work here because the method is a convenient central place rather than because the work is launch-critical.

Prefer:

* thin app delegate methods;
* explicit process-level bootstrap with narrow responsibilities;
* cheap local state over remote decisions;
* one-time app-wide setup that is idempotent;
* lightweight factory registration rather than eager instance construction;
* deferring feature-specific setup to the feature boundary.

## Scene Delegate Review

Use the scene delegate for scene-specific UI creation and scene lifecycle coordination.

Review:

* `scene(_:willConnectTo:options:)`;
* scene delegate `init` and stored properties;
* custom scene coordinators called during connection;
* window creation and root controller factories;
* `sceneDidBecomeActive(_:)` and `sceneWillEnterForeground(_:)` when they rerun launch-like work.

Appropriate launch-time responsibilities may include:

* creating a `UIWindow` for the provided `UIWindowScene`;
* installing a minimal root controller, shell, or coordinator;
* applying scene-specific routing inputs from connection options;
* making the window visible;
* scheduling scene-specific secondary work outside the first-frame path.

High-risk patterns:

* recreating app-wide services per scene;
* treating every scene connection as a full app launch;
* building every tab and navigation branch before first frame;
* restoring complex screens synchronously;
* completing deep-link navigation before a root shell is visible;
* loading first-screen data synchronously before any valid UI exists;
* creating multiple complete navigation stacks during startup;
* doing heavy foreground-refresh work in both launch and activation callbacks.

Prefer:

* minimal root UI that can appear quickly;
* cheap route selection from local state;
* progressive scene-specific data loading;
* idempotent scene setup;
* lazy construction of tabs, routes, and feature coordinators;
* clear separation between app-wide services and scene-owned UI state.

## Apps Without Scene Lifecycle

Some apps still create their root window from the app delegate.

When scenes are not used, review the app delegate for both process-level setup and root UI setup.

The same launch-performance rules apply, but the app delegate has more responsibility:

* create the window;
* install the minimal root controller;
* make the window visible;
* avoid broad dependency setup before the first frame;
* defer feature-specific or maintenance work.

Do not recommend moving work to `UISceneDelegate` unless the app actually uses or is migrating to the scene lifecycle.

## Startup Task Classification

Before recommending deferral, classify each task by when the app truly needs it.

### Required before first frame

Keep only work required to show a correct first visible UI.

Examples:

* creating the window and minimal root controller;
* choosing login, locked, onboarding, or main shell from small local state;
* setting invariants required before UI code runs;
* registering a process-level component that must observe the earliest launch behavior;
* parsing launch input only enough to decide the first safe UI.

Even required work should be checked for main-thread blocking, unnecessary breadth, and hidden eager construction.

### Required before first interaction

This work does not need to block the first frame, but the UI may need it before controls become enabled.

Examples:

* preparing minimal first-screen view model state;
* reading cached data required by immediate controls;
* validating a local session timestamp;
* restoring lightweight navigation context;
* enabling observers needed for first-screen actions.

If interaction depends on this work, the UI needs a clear loading, disabled, or placeholder state.

### Needed soon after launch

This work is useful shortly after launch but should not block visible UI.

Examples:

* refreshing cached user data;
* warming small first-navigation caches;
* refreshing local configuration that has a safe default;
* preparing background sync state;
* scheduling noncritical observers.

Schedule this work after a clear readiness point and verify it does not compete with early user interaction.

### Feature-specific and lazy

Feature-specific services should start when the feature is opened or when its data is actually needed.

Examples:

* search index setup for a later screen;
* map, camera, media, payment, or document modules not visible at launch;
* secondary tab data providers;
* optional personalization or recommendations;
* admin, debug, or diagnostics tools.

Do not initialize feature modules from lifecycle callbacks just because they are globally reachable.

### Background maintenance

Maintenance work should not block the first frame or first interaction.

Examples:

* cache cleanup;
* log compaction;
* old file deletion;
* analytics batch upload;
* nonurgent sync repair;
* database vacuuming that is not required for immediate correctness.

Use appropriate priority, cancellation, and battery/network awareness. Avoid starting all maintenance immediately after first frame if it causes visible jank.

## Work That May Need to Stay Early

Avoid blanket deferral. Some work may have legitimate launch-time requirements.

Treat these as context-sensitive rather than automatically deferrable:

* crash reporting needed before any code can fail silently;
* security, fraud, compliance, or jailbreak/root detection required before sensitive UI;
* session lock or privacy state needed before showing user data;
* local feature flags or kill switches required to choose safe UI;
* deep link or notification parsing required to choose the initial shell;
* background task registration that the platform expects during launch;
* minimal analytics or logging required for regulated audit trails.

For each case, ask whether the full setup is required or whether a smaller early mode can run before the rest is deferred.

Separate background task registration from background task work. Registration may need to happen early; cleanup, sync repair, uploads, and maintenance should not automatically start on the first-frame path.

For SDK-specific startup strategy, route to `third-party-sdks-at-launch.md`. In this file, only decide whether lifecycle code blocks first frame or first interaction.

## Root UI and First Frame

The first frame should be minimal, correct, safe, and visually stable. It does not need to contain every final piece of data.

Review root UI creation for:

* expensive root view controller initializers;
* view models that start work in `init`;
* coordinators that create many screens immediately;
* tab controllers that eagerly build all tab contents;
* navigation restoration that reconstructs heavy screens before any shell appears;
* synchronous image loading, decoding, resizing, or asset processing;
* large table or collection data sets loaded before first draw;
* complex first-screen Auto Layout or view hierarchy setup;
* blocking state restoration.

Prefer:

* shell-first UI with progressive loading;
* cached, placeholder, or skeleton content when valid for the product;
* minimal route state before deeper navigation;
* lazy tab and feature construction;
* narrow root view model dependencies;
* delayed enrichment after visible UI;
* explicit loading, locked, or disabled states instead of launch blocking.

Do not show an empty or misleading shell just to win the first-frame metric. The first UI must still be correct for the user state and safe for the product domain.

## Routing, Launch Options, and State Restoration

Launch input can force early routing decisions, but it should not force full feature initialization.

Relevant inputs include:

* universal links;
* custom URL schemes;
* remote or local notification launches;
* quick actions;
* handoff and user activities;
* state restoration;
* scene restoration activities.

Recommended model:

1. Parse launch input cheaply.
2. Store a minimal pending route or initial intent.
3. Show a safe root UI.
4. Resolve expensive authorization, data, or module requirements after the root UI exists.
5. Complete deep-link or notification navigation only when the required state is available.

Review for:

* deep-link handlers that build full feature modules before root UI;
* notification handlers that fetch remote data synchronously during launch;
* restoration code that recreates complex screens before showing a shell;
* authentication refresh that blocks route selection;
* router/session/root UI dependency cycles;
* scene restoration that reruns app-wide bootstrap.

Prefer pending-route models over blocking route completion during app delegate or scene connection.

## Dependency Setup on the Launch Path

In this file, inspect dependency setup only when it is called directly from AppDelegate, SceneDelegate, root controller creation, or first-screen setup.

Dependency containers often hurt launch because they make whole-app construction easy.

Review for:

* `build`, `registerAll`, `resolveAll`, `assemble`, or similar all-app setup during launch;
* registrations that construct concrete instances instead of factories;
* eager singletons that open files, start threads, subscribe to streams, or touch storage;
* root view models that resolve broad dependencies;
* feature modules registered before they are reachable;
* hidden work in property initializers of dependency objects.

Prefer:

* lightweight factory registration;
* a separate launch-critical dependency slice;
* constructing only the root dependencies needed by the first screen;
* lazy feature module registration or construction;
* explicit async preparation after visible UI where needed;
* narrow protocols for first-screen dependencies.

Factory registration is useful only if it is actually lightweight. A registration phase that scans modules, decodes configuration, touches storage, or eagerly evaluates feature initializers can still be launch-critical work.

If the dependency graph is large, ordered, shared across features, or needs dependency scheduling, route to `launch-orchestration-and-dependency-graph.md` instead of trying to solve orchestration inside this reference.

## Main-Thread Startup Work

Lifecycle callbacks commonly lead to main-thread work. UI setup must happen on the main thread, but blocking or broad non-UI work should be challenged.

High-risk patterns:

* synchronous file reads or writes;
* keychain calls on the first-frame path;
* database open or migration on the main thread;
* large JSON or property list decoding;
* image decoding, resizing, or asset processing;
* waiting on semaphores, dispatch groups, locks, or synchronous dispatch;
* synchronous network wrappers;
* expensive logging or analytics formatting;
* broad notification registration with side effects;
* large diffing or data transformation before first frame;
* localization, theme, or configuration rebuilds with wide side effects.

Keychain or secure storage reads may be required for lock, auth, or privacy routing, but keep them minimal, measured, and separated from broader account loading or token refresh work.

When moving work off the main thread, verify:

* the API is thread-safe;
* UI state updates return to the correct actor or thread;
* the first screen has a valid loading or disabled state;
* cancellation and priority are appropriate;
* background work does not immediately compete with first interaction;
* failure handling is clear.

Do not recommend backgrounding work simply to hide it. Work required for correctness may require a better first-frame design rather than a thread change.

## Deferral Patterns

Use explicit deferral points instead of vague “do it later” advice.

Possible deferral points:

* after the minimal root UI is installed;
* after the first screen appears;
* after the first user interaction;
* after authentication or local session state is known;
* after the first critical data request completes;
* when a specific tab, route, or feature is opened;
* when the app is idle enough for maintenance work;
* during a background-friendly maintenance window.

When suggesting deferral, specify:

* what moves;
* why it is not required before first frame;
* which event triggers it;
* what state the UI shows before completion;
* how cancellation and failure are handled;
* how the improvement will be measured.

Avoid using `DispatchQueue.main.async` as proof that work happens after the first displayed frame. It only moves work out of the current synchronous call stack. Use lifecycle points, readiness signals, signposts, or traces to confirm timing.

Do not move all work to “after first frame” as one burst. Stagger noncritical work by priority and user interaction risk.

For full measurement setup, route to `metrics-instruments-xctest-metrickit.md`.

## Multiple Scenes and Idempotency

Scene-based apps can have more than one UI instance. Launch reviews should not assume a single global scene.

Check:

* whether app-wide bootstrap runs once or once per scene;
* whether services are safe when two scenes connect close together;
* whether scene restoration recreates global state;
* whether a second window triggers first-launch-only work;
* whether scene-specific state is stored globally by accident;
* whether foregrounding one scene restarts work intended for process launch.

Prefer:

* app-wide services owned by a process-level container;
* scene-owned coordinators for UI state;
* idempotent setup methods;
* explicit one-time guards for process bootstrap;
* scene-specific pending routes and restoration state.

## What the Agent Can Inspect

When repository access is available, inspect concrete launch-path code instead of giving generic advice.

Search for lifecycle entry points:

```sh
rg "didFinishLaunching|willFinishLaunching|configurationForConnecting|willConnectTo|sceneDidBecomeActive|sceneWillEnterForeground" .
```

Search for bootstrap and dependency setup language:

```sh
rg "bootstrap|configure|start|initialize|setup|register|migrate|open|load|restore|resolve|container|coordinator|assemble" .
```

Search for blocking or expensive operations near startup paths:

```sh
rg "Data\(|contentsOf:|FileManager|Keychain|SecItem|UserDefaults|JSONDecoder|PropertyListDecoder|sleep|wait\(|semaphore|sync\(|performAndWait|DispatchQueue\.main\.sync" .
```

Search for Swift concurrency startup work that may still be awaited before visible UI:

```sh
rg "Task\(|Task\.detached|async let|withTaskGroup|await .*load|await .*start|await .*bootstrap|await .*configure|await .*initialize" .
```

Search for blocking synchronization more explicitly:

```sh
rg "DispatchSemaphore|DispatchGroup|group\.wait|semaphore\.wait|NSLock|lock\(|MainActor\.run" .
```

Search for root UI creation and potential duplication:

```sh
rg "UIWindow\(|rootViewController|makeKeyAndVisible|UINavigationController\(|UITabBarController\(|UIHostingController" .
```

Use search results as leads, not proof. Confirm whether matched code actually runs before first frame and whether it blocks the main thread or early interaction.

The agent can:

* trace lifecycle call chains from app and scene delegates into bootstrap code;
* identify synchronous work on the launch path;
* classify startup tasks by necessity;
* suggest explicit readiness points;
* propose lazy initialization for feature-scoped services;
* recommend minimal root UI or placeholder state where appropriate;
* add or recommend lightweight app-owned launch phase markers;
* propose small local code patches when repository is available and the change is safe.

The agent cannot reliably:

* prove first-frame timing without measurement;
* assume `DispatchQueue.main.async` runs after the first displayed frame;
* know third-party SDK internal startup cost without traces, symbols, docs, or vendor guidance;
* decide that security, crash reporting, routing, compliance, or auth requirements can be deferred without product context;
* convert synchronous dependencies to async/lazy setup without checking call-site semantics.

## Safe Patch Heuristics

When the agent is allowed to edit code, prefer small, reversible changes.

Good patch candidates:

* split a large startup method into named phases;
* isolate process bootstrap from scene UI creation;
* move clearly noncritical work behind an explicit post-root-UI method;
* replace eager service construction with factories when call sites already tolerate laziness;
* add a lightweight placeholder, locked, or loading state for the first screen;
* delay feature module creation until route selection actually needs it;
* add app-owned phase markers around lifecycle and root UI construction;
* remove duplicate root UI setup when the correct lifecycle owner is clear;
* make setup idempotent when scene creation can happen more than once.

Risky patch candidates requiring extra care:

* changing authentication, privacy lock, security, crash reporting, payment, or compliance startup order;
* making a synchronous API async across many call sites;
* moving database migration or keychain logic without checking correctness;
* changing deep-link or notification routing order;
* changing multi-window behavior;
* replacing root navigation architecture;
* deferring feature flags or remote config when they gate visible behavior;
* changing background task registration timing.

If correctness is uncertain, recommend instrumentation or decomposition first, then behavior-changing optimization after evidence is available.

## Review Checklist

Use this checklist when reviewing UIKit lifecycle startup code.

* [ ] Is the scenario launch rather than resume, scene restoration, foreground refresh, or warm/prewarmed confusion?
* [ ] Is app-wide bootstrap separated from scene-specific UI creation?
* [ ] Does the app delegate avoid broad synchronous initialization?
* [ ] Does scene connection create a minimal valid root UI?
* [ ] Is root UI creation owned by one lifecycle path, not duplicated?
* [ ] Are launch options parsed cheaply before expensive route completion?
* [ ] Are notification, URL, shortcut, and restoration inputs represented as pending intent when needed?
* [ ] Are dependency containers registering lightweight factories instead of constructing the whole graph?
* [ ] Are factory registrations actually lightweight?
* [ ] Are first-screen view models cheap to create?
* [ ] Are database, keychain, file, parsing, migration, and network work absent from the first-frame path unless required?
* [ ] Are required early services reduced to their minimal early mode?
* [ ] Are multiple scenes handled without rerunning app-wide bootstrap?
* [ ] Are background task registration and background task work separated?
* [ ] Are background tasks scheduled with priority and correctness in mind?
* [ ] Does every deferred task have a trigger, fallback UI, and failure behavior?
* [ ] Is the recommendation connected to a launch phase and validation plan?

## Output Guidance

When this reference is used, report findings in this shape:

```markdown
### Lifecycle phase

AppDelegate / SceneDelegate / root UI / first frame / early responsiveness.

### Critical-path work

Concrete calls or responsibilities that appear to block first frame or first interaction.

### Necessity classification

First-frame / first-interaction / soon-after-launch / feature-lazy / maintenance.

### Recommended change

Smallest safe change plus the deferral point or lazy boundary.

### Correctness risk

Routing, auth, privacy, crash reporting, security, state restoration, multi-window, background tasks, or compliance concerns.

### Unknowns / evidence needed

What is not proven yet: missing trace, unclear lifecycle call path, product requirement unknown, SDK/internal cost unknown, or launch scenario not reproduced.

### Validation

The trace, phase marker, launch metric, or manual comparison that should improve.
```

Keep recommendations tied to evidence. If the code path or measurement boundary is unclear, label the finding as a hypothesis and ask for the missing trace, file, or launch scenario.
