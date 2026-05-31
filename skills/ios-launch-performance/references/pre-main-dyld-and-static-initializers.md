# Pre-main, dyld, and Static Initializers

Use this reference when a launch investigation points to work that happens before the app reaches its own application lifecycle code, or when reviewing code that may execute while executable images are loaded, linked, and registered.

Keep this file focused on **pre-main and load-time behavior**. Do not use it as the primary guide for framework linking strategy, AppDelegate/SceneDelegate startup, SwiftUI root initialization, third-party SDK policy, launch orchestration, or measurement setup. Those topics belong in their own reference files.

## Scope Boundary

This file covers:

- what pre-main means in an iOS launch investigation
- dyld loading, binding/fixups, and initializer execution at a high level
- Objective-C `+load` and `+initialize`
- C/C++ constructors and clang constructor attributes
- binary initializer sections such as `__mod_init_func`
- Objective-C categories and runtime registration risk
- Swift global/static initialization when it is touched during launch
- how to review code or binaries suspected of load-time work

This file does not cover:

- whether modules should be static, dynamic, or mergeable
- how to restructure `didFinishLaunching` or `scene(_:willConnectTo:)`
- how to defer third-party SDK startup
- how to configure XCTest, MetricKit, Organizer, or CI launch tests
- how to build a launch dependency graph or launch orchestrator

## Mental Model

Pre-main is the part of launch that happens before the app enters its explicit application lifecycle code.

In UIKit apps, this means before the app delegate and scene delegate startup path begins. In SwiftUI apps, this means before the SwiftUI `App` startup path begins. The exact implementation details can vary by toolchain, OS version, app type, and how the entry point is generated, so treat **pre-main** as an investigation boundary rather than a single source-code location.

At a high level, this area can include:

1. Loading the main executable and dependent images.
2. Mapping code and data into the process.
3. Applying dyld fixups and connecting references between images.
4. Registering runtime metadata needed by loaded images.
5. Running load-time initializers such as Objective-C `+load`, C/C++ constructors, clang constructor functions, and functions in binary initializer sections.
6. Handing control to the app entry path.

Do not treat all launch work as pre-main. Code inside `UIApplicationMain`, `@main App`, `AppDelegate`, `SceneDelegate`, root view/model initialization, or first-frame rendering is launch-critical, but it is not pre-main.

## Why This Phase Matters

Pre-main work is paid before the app can start building its first UI. If this phase is expensive, later deferral in app delegate, scene delegate, or SwiftUI lifecycle code cannot recover the time already spent.

The main review goal is to remove hidden eager work from load time and make initialization explicit, lazy, measurable, and scoped to the feature that actually needs it.

A good pre-main review answers four questions:

1. Is the slow time truly before app lifecycle code?
2. Which image or initializer owns the time?
3. Does that work have to run before first frame?
4. Can it be removed, reduced, delayed, or made explicit?

## What Counts as Pre-main Risk

Pre-main risk usually comes from work that is:

- unconditional
- hidden behind language/runtime hooks
- triggered by image loading rather than feature use
- expensive before the app can render UI
- difficult to order, observe, or test
- introduced by third-party binaries or generated code

The problem is not only raw CPU time. Load-time work can also create ordering hazards, unexpected dependency setup, main-thread assumptions, lock contention, I/O before the app is ready, and startup behavior that is hard for later maintainers to understand.

## Common Sources of Pre-main Work

### Objective-C `+load`

`+load` is called when a class or category is loaded into the Objective-C runtime. It is not tied to whether the app uses that class on the first screen.

Treat every `+load` implementation as launch-critical until proven otherwise.

Review for:

- method swizzling
- runtime scanning
- service registration
- singleton creation
- dependency lookup
- logging or analytics setup
- file, keychain, database, or network access
- locks, semaphores, or synchronous dispatch
- large allocations, decoding, parsing, or reflection-like work

Prefer:

- explicit registration from a known startup point
- lazy registration on first feature use
- narrow one-time setup guarded by a cheap path
- compile-time configuration when possible
- vendor-supported minimal initialization modes for SDKs

Acceptable `+load` work should be tiny, deterministic, independent of app state, and free of I/O. Avoid depending on ordering between unrelated classes, categories, or libraries.

### Objective-C `+initialize`

`+initialize` is lazy compared with `+load`; it runs before a class receives its first message. This can reduce unconditional launch work in legacy Objective-C code, but it is not a universal replacement.

Use it carefully because it still hides work behind runtime behavior and can move latency to the first call site. It may also make ordering and failure behavior harder to reason about.

Prefer this order when changing code:

1. Remove the need for load-time initialization.
2. Move required registration to explicit startup code if it is truly launch-critical.
3. Make setup lazy and feature-scoped when first-frame correctness does not require it.
4. Use `+initialize` only when maintaining Objective-C code where its semantics are appropriate.

Do not recommend replacing `+load` with `+initialize` as a blanket rule.

### C and C++ Constructors

C++ global objects with nontrivial constructors and functions marked with constructor attributes can run before the app entry path.

Review for:

- global objects that allocate memory
- constructors that open files, databases, sockets, or caches
- constructors that create threads, queues, locks, or observers
- registration systems that scan types, modules, or plugins
- hidden dependency setup inside native libraries

Prefer explicit initialization functions that the app calls at a chosen point. If a constructor is unavoidable, keep it local, fast, deterministic, and free of app-level dependencies.

### Binary Initializer Sections

Functions linked into initializer sections, such as `__mod_init_func`, are part of the same load-time risk area. They may come from C, C++, Objective-C, Swift interoperability code, generated code, package dependencies, or third-party binaries.

When a trace shows static initializer time but source code does not obviously contain constructors or `+load`, inspect linked dependencies and generated/runtime support code as well as application code.

### Objective-C Categories and Runtime Registration

Objective-C classes, categories, protocols, selectors, and method metadata must be registered as images load. Categories with `+load` are especially important because their load methods can run even when the category's methods are not used on the launch path.

Do not assume categories are free just because they contain no visible app startup code. They can still contribute metadata, and when they include `+load`, they can execute launch-time behavior.

Review category-heavy modules for:

- categories on very common Foundation/UIKit classes
- swizzling performed from category `+load`
- duplicated registration across modules
- generated Objective-C bridging code from dependencies
- categories bundled in third-party SDKs that are linked for unrelated features

### Swift Globals and Static Properties

Do not label every Swift `static let`, global value, or static property as pre-main work. Swift global and static stored values are commonly initialized lazily on first access.

The launch risk appears when:

- a Swift global or static value is touched by a load-time initializer
- app startup code touches it before first frame
- initialization performs heavy work
- initialization builds a large dependency graph
- initialization has side effects beyond simple value creation
- initialization crosses into Objective-C/C/C++ code that performs load-time work

Prefer small, side-effect-free globals. Keep expensive setup behind explicit boundaries that match when the feature is actually needed.

### Dynamic Loading During Startup

Dynamic loading that happens during startup is not necessarily pre-main if the app explicitly triggers it after entering lifecycle code. However, it can behave like launch-critical work if it happens before first frame or first interaction.

If the task involves choosing static vs dynamic frameworks, mergeable libraries, resource bundles, or modularization trade-offs, route to `linking-strategy.md`. Use this file only to identify whether load-time behavior is part of the observed cost.

## Review Procedure

When inspecting a pre-main suspicion, follow this sequence.

### 1. Confirm the boundary

First verify whether the time is actually pre-main.

Use evidence such as:

- App Launch trace phase breakdown
- dyld Activity/static-initializer attribution
- Time Profiler showing work before app lifecycle frames
- app-level signposts that start at the first app-controlled point and show a gap before app code begins

Do not infer pre-main cost only from a slow first frame. First-frame delay can come from app delegate work, scene setup, SwiftUI root creation, layout, drawing, data access, or early post-launch blocking.

### 2. Identify the owner

Attribute the cost to the most specific owner available:

- main executable
- internal framework
- third-party SDK
- generated code
- Swift/Objective-C interoperability layer
- C/C++ library
- system framework
- unknown binary image

If the owner is a system framework, avoid recommending changes to system internals. Instead, inspect whether application code or linked dependencies cause the system framework to load or initialize earlier than necessary.

### 3. Classify the initializer

Classify the source of the work:

- Objective-C `+load`
- Objective-C `+initialize`
- C/C++ constructor
- clang constructor function
- function in an initializer section
- runtime metadata registration
- explicit dynamic loading
- unknown static initializer

The classification determines the fix. For example, `+load` usually calls for removing hidden load-time behavior, while explicit dynamic loading may require moving the call to a later app phase.

### 4. Decide whether the work is truly launch-critical

Ask:

- Is this needed before first app frame?
- Is it needed before first interaction?
- Is it needed only for routing, crash capture, security, auth, or feature flags?
- Can the first screen render without it?
- Can the setup happen on first feature use?
- Can the SDK or subsystem start in a smaller mode?

If the work is not required before first frame, move it out of load time. If it is required, reduce it to the smallest safe operation.

### 5. Validate with the same scenario

Re-measure using the same launch scenario, device class, OS version, build configuration, and app state. Avoid mixing cold, warm, prewarmed, and resume measurements.

## Code Review Checklist

Use this checklist when reviewing Objective-C, C/C++, mixed Swift/Objective-C modules, or binary dependencies on the launch path.

- [ ] No app-level work is hidden inside Objective-C `+load`.
- [ ] Any remaining `+load` methods are tiny, deterministic, and documented.
- [ ] `+load` is not used for networking, disk I/O, database setup, keychain access, analytics setup, dependency graph construction, or large parsing work.
- [ ] Method swizzling from `+load` is justified, minimal, and isolated.
- [ ] There are no C/C++ global constructors doing expensive work.
- [ ] Constructor attributes are not used as a substitute for explicit startup APIs.
- [ ] Heavy Swift globals or static values are not touched by pre-main hooks.
- [ ] Objective-C categories with `+load` are audited, especially in third-party frameworks.
- [ ] Initializer ordering assumptions are avoided.
- [ ] Third-party binaries with static initializer cost are tracked separately from application code.
- [ ] Debug-only dyld or Objective-C runtime environment variables are not treated as production behavior.
- [ ] Any change to load-time behavior is validated with the same launch scenario and build configuration.

## Safer Design Directions

### Replace hidden load-time work with explicit registration

Prefer a named registration point that is called by the app when the capability is actually required.

Good registration points are:

- easy to find in code review
- scoped to the feature or subsystem
- safe to call once
- clear about failure behavior
- free of unrelated diagnostics or secondary setup

Avoid combining route registration, analytics, cache warming, dependency graph construction, and network setup in the same load-time hook.

### Keep constructors trivial

A constructor that only registers a tiny local table is different from a constructor that opens storage, starts queues, reads files, creates threads, or initializes a subsystem.

When constructor behavior is unavoidable:

- keep it small
- avoid I/O
- avoid locks that can interact with app startup
- avoid app state assumptions
- avoid calling into high-level application services
- document why explicit initialization is not possible

### Keep Swift static initialization cheap

Static values are not inherently bad. The problem is hiding large or side-effecting work behind static access that happens during launch.

Prefer:

- simple constants
- small immutable values
- factories that create only launch-critical services
- feature-scoped lazy state
- explicit async setup for expensive work

Avoid:

- building the full service graph from a static property
- touching storage/network/keychain from static initialization
- static initialization that changes global process state
- static initialization whose first access happens indirectly from `+load` or constructor code

### Keep swizzling narrow and auditable

Swizzling is sometimes used by SDKs or legacy infrastructure, but it is high-risk during launch because it often runs from `+load` and changes global runtime behavior.

Review:

- why swizzling is needed
- whether it must run before first frame
- whether it can be moved to explicit setup
- whether the affected class is broad, such as common UIKit/Foundation types
- whether multiple SDKs swizzle the same method
- whether failure or ordering behavior is understood

## Diagnostic Hints

Use diagnostics to locate the owner of work before recommending rewrites.

Helpful signals:

- App Launch trace shows a large pre-main or static-initializer region.
- dyld Activity attributes time to static initialization or image loading.
- Time Profiler shows CPU inside runtime registration, constructor functions, or framework startup before app lifecycle code.
- App-controlled signposts begin later than expected, suggesting time before the first app-controlled marker.
- Debug-only dyld or Objective-C runtime logging shows many loaded images or load methods.
- Binary inspection reveals initializer sections or unexpected linked dependencies.

Potential debug tools and signals:

- Instruments App Launch template
- dyld Activity instrument on Xcode versions that include it
- Time Profiler with system libraries visible when needed
- app-level signposts around the first app-controlled startup points
- debug-only environment variables such as `DYLD_PRINT_LIBRARIES` or `OBJC_PRINT_LOAD_METHODS`, when supported by the current run environment
- binary inspection tools such as `otool`, `nm`, `dyld_info`, or link maps for advanced investigations

Do not overfit to one run. Pre-main measurements can vary by device state, OS version, install state, cache warmth, app update state, and build configuration.

## Recommendation Language

Use precise wording:

- Say "this code can run before app lifecycle code" when discussing `+load` or constructors.
- Say "move this out of load time" when the issue is unconditional pre-main execution.
- Say "make initialization explicit or lazy" instead of "replace `+load` with `+initialize`" unless that Objective-C trade-off is justified.
- Say "measure the same launch scenario again" instead of predicting a fixed millisecond gain without trace evidence.
- Say "this is launch-critical but not necessarily pre-main" when the code runs in app delegate, scene delegate, SwiftUI `App`, or root UI setup.

Avoid language that overclaims:

- Do not say every Swift static value is pre-main work.
- Do not say every Objective-C category is expensive.
- Do not say a dynamic framework is always the main cause of pre-main cost.
- Do not promise a fixed improvement from removing a single initializer.
- Do not treat debug-only environment variable output as a production metric.

## Boundaries With Other References

Route to another reference when the main issue is outside pre-main:

- Use `launch-taxonomy-and-targets.md` for cold/warm/prewarmed/resume definitions, first-frame targets, and measurement comparability.
- Use `linking-strategy.md` for deciding between dynamic frameworks, static libraries, mergeable libraries, modularization, binary size, or build-time trade-offs.
- Use `launch-orchestration-and-dependency-graph.md` for ordered startup steps, dependency graphs, critical path analysis, safe parallelism, and launch step failure handling.
- Use `appdelegate-scenedelegate-and-first-frame.md` for work inside `didFinishLaunching`, scene connection, dependency containers, root UI creation, or main-thread deferral.
- Use `swiftui-app-launch.md` for `@main App`, root `Scene`, root view initialization, observable state, `.task`, or `.onAppear` work.
- Use `third-party-sdks-at-launch.md` for vendor-specific startup policies, deferred SDK modes, crash reporting, attribution, ads, analytics, remote config, push, security, or feature flags.
- Use `metrics-instruments-xctest-metrickit.md` for detailed tool usage, launch metric interpretation, CI baselines, MetricKit, Organizer, or production monitoring.
