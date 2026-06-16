# Pre-main, dyld, and Static Initializers

Use this reference when a launch investigation points to work that happens before the app reaches its own application lifecycle code, or when reviewing code that may execute while executable images are loaded, linked, and registered.

Keep this file focused on **pre-main and load-time behavior**. Do not use it as the primary guide for framework linking strategy, AppDelegate/SceneDelegate startup, SwiftUI root initialization, third-party SDK policy, launch orchestration, or measurement setup.

## Scope Boundary

This file covers:

* what pre-main means in an iOS launch investigation;
* dyld loading, binding/fixups, and initializer execution at a high level;
* Objective-C `+load` and `+initialize`;
* C/C++ constructors and clang constructor attributes;
* binary initializer sections such as `__mod_init_func`;
* Objective-C categories and runtime registration risk;
* Swift global/static initialization when it is touched during launch;
* explicit dynamic loading during startup when it behaves like launch-time image work;
* how to review code or binaries suspected of load-time work.

This file does not cover:

* whether modules should be static, dynamic, or mergeable;
* how to restructure `didFinishLaunching` or `scene(_:willConnectTo:)`;
* how to defer third-party SDK startup;
* how to configure XCTest, MetricKit, Organizer, or CI launch tests;
* how to design a launch dependency graph;
* how to restructure SwiftUI root view setup.

Use this file to diagnose initializer content and load-time behavior. Use `linking-strategy.md` to decide whether an image should be a separate launch-loaded dependency at all.

## Contents

* [Mental Model](#mental-model)
* [Why This Phase Matters](#why-this-phase-matters)
* [Review Procedure](#review-procedure)
* [What Counts as Pre-main Risk](#what-counts-as-pre-main-risk)
* [Common Sources of Pre-main Work](#common-sources-of-pre-main-work)
* [Code Review Checklist](#code-review-checklist)
* [Safer Design Directions](#safer-design-directions)
* [Diagnostic Hints](#diagnostic-hints)
* [What the Agent Can Inspect](#what-the-agent-can-inspect)
* [Safe Patch Heuristics](#safe-patch-heuristics)
* [Recommendation Language](#recommendation-language)
* [Boundary With Other References](#boundary-with-other-references)

## Mental Model

Pre-main is the part of launch before the app reaches its own explicit application lifecycle code.

Treat pre-main as an investigation boundary rather than a single source-code location.

In UIKit lifecycle apps, this usually means before control reaches app-owned lifecycle callbacks such as `application(_:willFinishLaunchingWithOptions:)`, `application(_:didFinishLaunchingWithOptions:)`, or scene connection.

App delegate object creation and stored-property initialization can still be early launch work, but treat it separately from dyld/load-time initializer work unless evidence shows it belongs to the pre-main region.

In SwiftUI lifecycle apps, pre-main usually means before control reaches the `@main App` type, its stored properties, `App.init`, and scene construction.

Work in SwiftUI `App` stored properties, `App.init`, `Scene` construction, root view creation, or environment setup is launch-critical, but classify it as SwiftUI app launch unless trace evidence places it in pre-main or load-time initialization.

Work in this phase can include:

* loading the main executable;
* loading dynamic libraries and frameworks;
* mapping code and data;
* applying dyld fixups and bindings;
* registering Objective-C and Swift runtime metadata;
* running load-time initializers;
* running C/C++ constructors;
* preparing enough runtime state to transfer control into the app entry path.

Do not treat all launch work as pre-main. Work in `UIApplicationDelegate`, `UISceneDelegate`, SwiftUI `App`, root view setup, or first-screen rendering is launch-critical, but it is not automatically pre-main.

## Why This Phase Matters

Pre-main work is paid before the app can run its own startup policy.

That means the app cannot show a placeholder UI, schedule work later, or handle failure gracefully until this phase completes. If expensive work is hidden here, moving code out of `AppDelegate`, `SceneDelegate`, or SwiftUI `.task` will not address it.

The review goal is to answer:

1. Is the slow time truly before app lifecycle code?
2. Which image or initializer owns the time?
3. Does that work have to run before first frame?
4. Can it be removed, reduced, delayed, or made explicit?
5. How will the same launch scenario be measured after the change?

## Review Procedure

When using this reference:

1. Confirm the boundary.
2. Identify the owner: app code, first-party framework, third-party source dependency, vendored binary SDK, Objective-C category, C/C++ code, or Swift static/global initialization.
3. Classify the initializer source.
4. Decide whether the work is truly required before app lifecycle code.
5. Check whether the work can move to explicit startup, lazy feature setup, or a smaller readiness condition.
6. Identify correctness risks: swizzling, runtime registration, crash/security/compliance, routing, SDK vendor requirements, ABI/setup expectations, or first-use behavior.
7. Recommend the smallest safe change.
8. Validate with the same launch scenario and a release-like build.

Do not recommend broad rewrites until the owner and boundary are clear.

## What Counts as Pre-main Risk

Treat the following as pre-main or load-time risk until measured otherwise:

* Objective-C `+load`;
* C/C++ constructor functions;
* clang constructor attributes;
* nontrivial functions referenced from binary initializer sections;
* heavy Objective-C runtime registration caused by categories or large class graphs;
* swizzling performed from `+load`;
* static initialization triggered by load-time hooks;
* dynamic frameworks loaded before app lifecycle code;
* explicit runtime loading performed before first frame or first interaction;
* vendored binary SDKs that execute work while being loaded.

Do not automatically treat the following as pre-main work:

* every Swift `static let`;
* every Swift global declaration;
* every Objective-C category;
* every dynamic framework;
* every linked dependency;
* work that runs in `didFinishLaunching`;
* work that runs in SwiftUI `App.init`;
* work that runs in `.task`, `.onAppear`, or first-screen code.

If the code runs after the app lifecycle has started, route to the lifecycle, SwiftUI, orchestration, SDK, or first-frame reference.

## Common Sources of Pre-main Work

### Objective-C `+load`

`+load` runs when the Objective-C runtime loads the class or category. It can execute before app lifecycle code and is a common source of hidden launch work.

Treat every `+load` implementation as launch-critical until proven otherwise.

High-risk work inside `+load`:

* networking;
* disk I/O;
* database setup;
* keychain access;
* dependency graph construction;
* analytics startup;
* SDK startup;
* large parsing or decoding;
* broad notification registration;
* method swizzling;
* synchronization that can block;
* calling into other subsystems with unclear readiness.

Prefer explicit registration or lazy setup when possible.

If swizzling is required, check whether it must happen before the first message to the affected class. Moving swizzling later can change behavior if objects or methods were already used.

### Objective-C `+initialize`

`+initialize` is different from `+load`. It is invoked before the class receives its first message, not simply because the image was loaded.

This can delay work, but it is not a universal fix.

Do not recommend replacing `+load` with `+initialize` as a blanket rule. `+initialize` can still occur early, may run on a sensitive path, and can introduce ordering or locking surprises.

Do not design new launch behavior around `+initialize`. Treat it mainly as legacy behavior to understand or remove, not as a preferred modern startup mechanism.

Prefer explicit, named initialization when the app needs predictable startup behavior.

### C/C++ Constructors

C/C++ constructors can run before app lifecycle code.

Look for:

* global objects with nontrivial constructors;
* `__attribute__((constructor))`;
* library-level registration code;
* setup performed by static objects;
* logging, analytics, storage, or runtime hooks initialized from constructors.

Constructors may come from app code, C++ libraries, Objective-C runtime helpers, generated registries, or vendored SDKs. If the source is vendored, prefer vendor documentation or measurement before suggesting code-level changes.

Keep constructors trivial. Move app-owned setup to explicit startup or lazy feature initialization.

### Binary Initializer Sections

Executable images may contain initializer sections such as `__mod_init_func`.

The presence of an initializer section is a signal, not a diagnosis. The important questions are:

* which binary owns it;
* which function runs;
* whether the function is app-owned or vendored;
* whether the function does meaningful work;
* whether the work is required before app lifecycle code;
* whether it can be removed, reduced, or moved.

Inspect both the main executable and embedded frameworks when the shipped app contains multiple images.

Use binary inspection as a lead, then connect it back to source, vendor documentation, or trace evidence when possible.

### Objective-C Categories and Runtime Registration

Objective-C categories can contribute to runtime registration and may contain `+load`.

Not every category is a problem. The risk increases when categories:

* define `+load`;
* perform swizzling;
* target broad UIKit or Foundation classes;
* are spread across many dynamic images;
* are part of large SDKs;
* register observers, handlers, or services at load time.

Focus on categories with behavior, not categories merely existing.

### Swift Globals and Static Properties

Do not label every Swift global, `static let`, or static property as pre-main work.

Swift initialization can be lazy depending on how the value is declared and accessed. Swift `static let` initialization is commonly lazy and thread-safe on first access, but the first access can still occur during launch and can block the accessing thread or actor.

The risk is not the declaration alone. The risk is expensive first access on a launch-critical path.

Review:

* whether the static/global value is expensive;
* whether it is touched before app lifecycle code;
* whether it is touched during app delegate, scene delegate, SwiftUI `App`, root UI, or first-screen setup;
* whether it performs I/O, parsing, locking, or large allocation;
* whether it creates a dependency graph;
* whether it can become a cheap placeholder plus lazy factory.

Use precise language: “this static is expensive if touched during launch” rather than “all statics are pre-main.”

### Dynamic Loading During Startup

Dynamic frameworks and libraries can contribute to pre-main or early launch cost through image loading, fixups, runtime registration, and initializer execution.

Explicit runtime loading during early startup can behave like launch-time image work if it happens before first frame or first interaction.

Look for:

* `dlopen`;
* `Bundle(path:)`;
* `Bundle.load()`;
* `loadAndReturnError`;
* plugin-style registries;
* manually loaded frameworks or bundles.

Do not automatically blame a dynamic framework. Determine whether the cost is:

* many separate images;
* deep image dependency chains;
* initializer content;
* Objective-C or Swift metadata registration;
* runtime hooks;
* dynamic framework boundaries that exist only for historical modularization reasons.

Use this file for initializer content and load-time behavior.

Use `linking-strategy.md` when the main decision is whether to keep, merge, static-link, remove, or move a framework in the shipped release product.

## Code Review Checklist

When reviewing source code for pre-main risk, check:

* [ ] Are there Objective-C `+load` methods?
* [ ] Do any `+load` methods perform app-level work?
* [ ] Do any `+load` methods call into SDKs, dependency containers, databases, keychain, networking, analytics, logging, or feature registries?
* [ ] Are there C/C++ constructors or clang constructor attributes?
* [ ] Are there global C++ objects with nontrivial constructors?
* [ ] Are there Objective-C categories that define `+load` or swizzle broad classes?
* [ ] Are there expensive Swift globals/statics touched from load-time hooks or early launch paths?
* [ ] Are binary initializer sections present in app-owned or vendored images?
* [ ] Are dynamic frameworks doing meaningful initializer work?
* [ ] Is explicit runtime loading performed before first frame or first interaction?
* [ ] Is the owner app-owned, first-party framework, third-party source dependency, or vendored binary?
* [ ] Is the evidence trace-based, binary-inspection-based, source-search-based, or only suspected?
* [ ] Is the cost initializer content, image count/linking, or early app lifecycle work?
* [ ] Is the suspected code truly before app lifecycle code?
* [ ] Is the recommendation validated with the same launch scenario?

## Safer Design Directions

### Move app-owned work out of `+load`

Prefer:

* explicit registration from a known startup phase;
* lazy registration when a feature opens;
* dependency injection from app-owned startup code;
* small static tables instead of runtime work;
* one-time setup guarded by explicit readiness.

Avoid:

* network, file, keychain, database, analytics, or SDK startup from `+load`;
* broad swizzling from `+load`;
* hidden dependency graph construction from load-time hooks.

When a `+load` hook cannot be removed immediately, reduce it to a cheap registration stub and move expensive work behind explicit setup.

### Keep constructors trivial

C/C++ constructors should not perform broad app setup.

Prefer:

* compile-time constants;
* cheap table registration;
* explicit setup calls;
* lazy initialization;
* moving heavyweight work behind app lifecycle control.

### Make Swift static initialization cheap and explicit

If a Swift static/global value is expensive and touched early, prefer:

* storing a lightweight descriptor;
* using a lazy factory;
* moving heavy parsing or I/O behind an explicit method;
* splitting cheap metadata from expensive runtime state;
* making dependency creation visible at the call site.

### Keep swizzling narrow and auditable

Swizzling is sometimes used by SDKs or legacy infrastructure, but it is high-risk during launch because it often runs from `+load` and changes global runtime behavior.

Review:

* why swizzling is needed;
* whether it must run before first frame;
* whether it must run before the first message to the affected class;
* whether it can be moved to explicit setup safely;
* whether the affected class is broad, such as common UIKit/Foundation types;
* whether multiple SDKs swizzle the same method;
* whether failure or ordering behavior is understood.

## Diagnostic Hints

Use diagnostics to locate the owner of work before recommending rewrites.

Helpful signals:

* App Launch trace shows a large pre-main or static-initializer region.
* dyld Activity attributes time to static initialization or image loading.
* Time Profiler shows CPU inside runtime registration, constructor functions, or framework startup before app lifecycle code.
* App-controlled signposts begin later than expected, suggesting time before the first app-controlled marker.
* Debug-only dyld or Objective-C runtime logging shows many loaded images or load methods.
* Binary inspection reveals initializer sections or unexpected linked dependencies.

Potential debug tools and signals:

* Instruments App Launch template;
* dyld Activity instrument on Xcode versions that include it;
* Time Profiler with system libraries visible when needed;
* app-level signposts around the first app-controlled startup points;
* debug-only environment variables such as `DYLD_PRINT_LIBRARIES` or `OBJC_PRINT_LOAD_METHODS`, when supported by the current run environment;
* binary inspection tools such as `otool`, `nm`, `dyld_info`, or link maps for advanced investigations.

Use this section to identify useful signals. For detailed tool setup and metric interpretation, route to `metrics-instruments-xctest-metrickit.md`.

Do not rely on dyld/Objective-C debug environment variables being available or meaningful in every device, simulator, OS, signing, or sandbox configuration.

Do not overfit to one run. Pre-main measurements can vary by device state, OS version, install state, cache warmth, app update state, and build configuration.

## What the Agent Can Inspect

When repository access is available, inspect concrete load-time patterns instead of giving generic advice.

Search for Objective-C load and initialize methods:

```sh
rg "\+\s*\(void\)load|\+\s*\(void\)initialize" .
```

Search for C/C++ constructors and clang constructor attributes:

```sh
rg "__attribute__\s*\(\(constructor\)\)|constructor\s*\(" .
```

Search for swizzling and broad Objective-C runtime hooks:

```sh
rg "method_exchangeImplementations|class_replaceMethod|objc_getClass|objc_allocateClassPair|NSClassFromString|performSelector" .
```

Search for explicit runtime loading:

```sh
rg "dlopen|Bundle\(path:|Bundle\.load|loadAndReturnError|NSCreateObjectFileImageFromFile" .
```

Search for static or global values that may be touched early:

```sh
rg "static let|static var|static .* =" .
```

Search for likely expensive statics, globals, singletons, registries, and shared objects:

```sh
rg "static let|static var|shared|default|container|registry|formatter|decoder|JSONDecoder|PropertyListDecoder|DateFormatter|FileManager|Bundle" .
```

Use static/global searches as broad leads only after finding an early access path. These searches can be noisy and should not be treated as proof of pre-main work.

Search for expensive work near load-time hooks:

```sh
rg "Data\(|contentsOf:|FileManager|Keychain|SecItem|JSONDecoder|PropertyListDecoder|sqlite|migrate|URLSession|wait\(|semaphore|sync\(" .
```

When a built binary is available, inspect load-time sections and symbols in the main executable:

```sh
otool -l path/to/App.app/AppName | rg "__mod_init_func|__objc"
```

Inspect embedded frameworks too:

```sh
otool -l path/to/App.app/Frameworks/SomeFramework.framework/SomeFramework | rg "__mod_init_func|__objc"
```

Search symbols when needed:

```sh
nm -m path/to/binary | rg " load| initialize|constructor|mod_init|__mod_init"
```

Symbol search can produce false positives. Confirm whether the symbol is an Objective-C class method, a constructor, or an unrelated name before treating it as load-time work.

Use command matches as leads, not proof. Confirm that the code or binary actually contributes to pre-main or load-time cost.

The agent can:

* identify app-owned `+load`, `+initialize`, constructors, and swizzling;
* find expensive work called from load-time hooks;
* suggest moving app-owned work to explicit startup or lazy setup;
* recommend binary inspection when source is not available;
* ask for App Launch traces or pre-main evidence when the boundary is unclear.

The agent cannot reliably:

* rewrite vendor SDK load-time behavior without vendor guidance;
* prove production impact from source search alone;
* treat every static/global as pre-main work;
* treat every dynamic framework as the bottleneck;
* promise a fixed millisecond improvement without measurement.

## Safe Patch Heuristics

When the agent is allowed to edit code, prefer small, reversible changes.

Good patch candidates:

* move app-owned noncritical work from `+load` to explicit startup;
* reduce a load-time hook to a cheap registration stub, then move expensive work behind explicit setup;
* replace constructor-based app setup with explicit initialization;
* split heavy static initialization into a cheap static descriptor plus lazy factory;
* remove disk, network, keychain, database, parsing, or analytics work from load-time hooks;
* add a small explicit registration call where ordering and failure are visible;
* narrow app-owned swizzling or move it behind explicit setup when safe;
* add signposts around the first app-controlled startup point;
* add comments documenting why unavoidable load-time work must remain early.

Risky patch candidates requiring extra care:

* changing vendor SDK `+load` behavior without vendor support;
* replacing `+load` with `+initialize` as a blanket fix;
* changing swizzling order;
* moving crash reporting, security, fraud, payment, privacy, or compliance setup without product review;
* changing Objective-C categories used for runtime behavior;
* removing C/C++ constructors from libraries without understanding ABI or setup requirements;
* changing dynamic/static linkage when the real issue is initializer content;
* hiding work behind lazy initialization without a loading, failure, or retry plan.

If correctness is uncertain, recommend measurement, isolation, or vendor guidance before behavior-changing edits.

## Recommendation Language

Use precise wording:

* Say "this code can run before app lifecycle code" when discussing `+load` or constructors.
* Say "move this out of load time" when the issue is unconditional pre-main execution.
* Say "make initialization explicit or lazy" instead of "replace `+load` with `+initialize`" unless that Objective-C trade-off is justified.
* Say "measure the same launch scenario again" instead of predicting a fixed millisecond gain without trace evidence.
* Say "this is launch-critical but not necessarily pre-main" when the code runs in app delegate, scene delegate, SwiftUI `App`, or root UI setup.

Avoid language that overclaims:

* Do not say every Swift static value is pre-main work.
* Do not say every Objective-C category is expensive.
* Do not say a dynamic framework is always the main cause of pre-main cost.
* Do not promise a fixed improvement from removing a single initializer.
* Do not treat debug-only environment variable output as a production metric.

## Boundary With Other References

Use this reference for pre-main and load-time behavior.

Read `references/launch-taxonomy-and-targets.md` when the issue involves:

* cold, warm, prewarmed, resume, first install, or update launch terminology;
* launch target selection;
* measurement scenario classification;
* whether two numbers are comparable.

Read `references/linking-strategy.md` when the issue involves:

* deciding between dynamic frameworks, static libraries, and mergeable libraries;
* modularization and binary-size trade-offs;
* whether a framework should remain a separate launch-loaded image;
* release-bundle dependency structure;
* app/extension dependency duplication.

Read `references/launch-orchestration-and-dependency-graph.md` when the issue involves:

* ordered startup steps;
* dependency graphs;
* critical path analysis;
* safe parallelism;
* launch step failure handling;
* longest-chain optimization.

Read `references/appdelegate-scenedelegate-and-first-frame.md` when the issue involves:

* work inside `didFinishLaunching`;
* scene connection;
* dependency containers;
* root UI creation;
* first-frame readiness;
* main-thread lifecycle work.

Read `references/swiftui-app-launch.md` when the issue involves:

* SwiftUI `App`;
* root `Scene`;
* root view initialization;
* observable state;
* `.task`;
* `.onAppear`;
* `scenePhase`;
* `@UIApplicationDelegateAdaptor`.

Read `references/third-party-sdks-at-launch.md` when the issue involves:

* vendor-specific startup policies;
* deferred SDK modes;
* crash reporting;
* attribution;
* ads;
* analytics;
* remote config;
* push;
* security;
* feature flags.

Read `references/metrics-instruments-xctest-metrickit.md` when the issue involves:

* detailed tool usage;
* launch metric interpretation;
* CI baselines;
* MetricKit;
* Organizer;
* production monitoring.

Do not read all references by default.
