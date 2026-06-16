# Linking Strategy

Use this reference when a launch investigation points to dynamic library loading, embedded framework count, modularization shape, static vs dynamic linkage, mergeable libraries, binary layout, or release-bundle dependency structure.

This file is about **linking and packaging choices that can affect launch**. It should not be used as a general modularization guide or as a replacement for measurement.

## Scope Boundary

This reference covers:

* static libraries;
* dynamic frameworks;
* mergeable libraries;
* embedded framework count;
* dependency depth;
* release-bundle inspection;
* app and extension dependency duplication;
* optional feature dependencies in the main binary graph;
* vendored binary SDK packaging;
* order files and binary layout as advanced topics.

This reference does not cover:

* detailed dyld internals;
* `+load`, `+initialize`, constructor functions, or static initialization content;
* AppDelegate, SceneDelegate, or SwiftUI root setup;
* third-party SDK initialization policy;
* launch orchestration and dependency scheduling;
* XCTest, MetricKit, Organizer, or Instruments workflows.

Use the other references for those topics.

## Contents

* [Core Model](#core-model)
* [Review Procedure](#review-procedure)
* [What the Agent Can Inspect](#what-the-agent-can-inspect)
* [System vs Embedded Frameworks](#system-vs-embedded-frameworks)
* [Classify Each Dependency](#classify-each-dependency)
* [When Linking Strategy Is Likely Relevant](#when-linking-strategy-is-likely-relevant)
* [Static Linking](#static-linking)
* [Static Linking Risks](#static-linking-risks)
* [Dynamic Linking](#dynamic-linking)
* [Dynamic Linking Risks](#dynamic-linking-risks)
* [Mergeable Libraries](#mergeable-libraries)
* [Package Manager Notes](#package-manager-notes)
* [App Extensions and Shared Dependencies](#app-extensions-and-shared-dependencies)
* [Optional Features and Runtime Loading](#optional-features-and-runtime-loading)
* [Advanced: Order Files and Binary Layout](#advanced-order-files-and-binary-layout)
* [Safe Patch Heuristics](#safe-patch-heuristics)
* [Review Checklist](#review-checklist)
* [Agent Guidance](#agent-guidance)
* [Boundary With Other References](#boundary-with-other-references)

## Core Model

Linking strategy changes **where cost is paid**.

* Static linking can move code into a larger binary and reduce separate dynamic image loading.
* Dynamic linking can preserve runtime and distribution boundaries but may increase launch-time image loading and fixup work.
* Mergeable libraries can preserve dynamic-style development ergonomics while allowing release builds to reduce the number of separate dynamic images.
* Optional runtime loading can move cost out of launch, but only when the feature truly owns its loading, failure, and fallback behavior.

Do not use simplistic rules such as "static is always faster" or "dynamic is always bad." The right recommendation depends on the release product, dependency graph, device class, build configuration, app size, extension layout, vendor constraints, resource packaging, and measured launch phase.

Do not treat **declared**, **linked**, **embedded**, and **loaded at launch** as the same fact.

A dependency can be:

* declared in a package manifest;
* linked into the main executable;
* embedded in `App.app/Frameworks`;
* loaded because another embedded framework depends on it;
* present in the app bundle but not loaded during initial launch;
* runtime-loaded later by a feature.

A manifest dependency is not proof of a launch-loaded image. An embedded framework is not proof that it dominates launch. A runtime-loaded dependency may affect a later feature rather than the first frame.

## Review Procedure

When using this reference:

1. Confirm that the suspected bottleneck is related to linking, binary packaging, or release-bundle structure.
2. Inspect the shipped product when available, not only package manifests.
3. Separate Apple system frameworks from embedded first-party and third-party frameworks.
4. Classify each dependency by ownership, packaging, linkage, and launch relevance.
5. Look for dynamic images that are loaded at launch but only needed by later features.
6. Check whether the issue is linkage structure or initializer content inside a framework.
7. Consider static linking, mergeable libraries, dependency removal, feature splitting, or runtime loading only after classification.
8. Check app extension, resource bundle, duplicate symbol, Objective-C class/category, vendor SDK, and build-system constraints.
9. Recommend the smallest safe change.
10. Validate with release-like builds and launch measurements.

Do not recommend a linkage change just because launch is slow. Use this reference only when evidence points to binary/linkage structure.

## What the Agent Can Inspect

When repository access is available, inspect both source-level dependency declarations and the final release product.

Inspect project and package configuration:

* Xcode projects and workspaces;
* Swift Package manifests;
* `Package.resolved`;
* CocoaPods `Podfile` and lockfiles;
* Carthage files;
* Tuist, Bazel, Buck, or other project-generation manifests;
* build scripts that copy, strip, merge, or sign frameworks;
* vendored `.framework` and `.xcframework` artifacts;
* app extension targets and shared dependencies;
* Debug/Profile/Release build configuration differences.

Search for Swift Package linkage declarations:

```sh
rg "type:\s*\.dynamic|type:\s*\.static|library\(|binaryTarget|target\(" Package.swift
```

Search for CocoaPods and vendored framework configuration:

```sh
rg "use_frameworks!|vendored_frameworks|vendored_libraries|static_framework|modular_headers" .
```

Search for build settings and framework embedding:

```sh
rg "Embed Frameworks|FRAMEWORK_SEARCH_PATHS|LD_RUNPATH_SEARCH_PATHS|MACH_O_TYPE|DEAD_CODE_STRIPPING|OTHER_LDFLAGS" .
```

Search for Objective-C static-linking flags and forced loading:

```sh
rg "\-ObjC|\-all_load|\-force_load|OTHER_LDFLAGS" .
```

Search for scripts that modify the shipped binary layout:

```sh
rg "copy.*framework|embed.*framework|strip.*framework|merge.*framework|codesign|lipo|xcodebuild" .
```

Search for vendored binary artifacts:

```sh
find . \( -name "*.xcframework" -o -name "*.framework" -o -name "*.a" -o -name "*.dylib" \)
```

When a built app is available, inspect the shipped bundle:

```sh
find path/to/App.app -maxdepth 3 \( -name "*.framework" -o -name "*.dylib" \)
```

Inspect dynamic dependencies of the main executable:

```sh
otool -L path/to/App.app/AppName
```

Inspect dynamic dependencies of embedded frameworks too, because the main executable may not show the full transitive launch-loaded graph:

```sh
find path/to/App.app/Frameworks -name "*.framework" -maxdepth 2 -print
otool -L path/to/App.app/Frameworks/SomeFramework.framework/SomeFramework
```

If needed, inspect load commands and rpaths:

```sh
otool -l path/to/App.app/AppName
```

Use commands as leads, not proof. A dependency being present in a manifest does not prove it is launch-critical. A dependency being present in the app bundle does not prove it dominates launch. Validate with release-like measurement.

## System vs Embedded Frameworks

Separate Apple system frameworks from app-embedded frameworks.

Apple system frameworks are often in the dyld shared cache and are not the first place to optimize. Do not recommend removing or replacing system frameworks only because they appear in a dependency list.

Focus first on:

* first-party embedded dynamic frameworks;
* third-party embedded dynamic frameworks;
* vendored binary SDKs;
* internal dynamic framework chains;
* optional feature dependencies that are linked into the main app binary graph;
* debug-only or tooling frameworks accidentally included in Release;
* app extension dependency duplication.

If the investigation points to dyld internals or initializer work inside a framework, route to `pre-main-dyld-and-static-initializers.md`.

## Classify Each Dependency

For each dependency in the launch-loaded binary graph, classify it along these axes.

### Ownership

* first-party app module;
* first-party shared platform module;
* third-party open-source dependency;
* vendored binary SDK;
* Apple system framework;
* build-only or debug-only tooling.

### Packaging and Linkage

* static library;
* static framework;
* dynamic framework;
* mergeable library;
* XCFramework;
* Swift package product;
* CocoaPods product;
* Carthage product;
* runtime-loaded bundle or framework;
* unclear until release product inspection.

### Bundle and Load Status

* declared in manifest only;
* linked into the main executable;
* embedded in `App.app/Frameworks`;
* dependency of another embedded framework;
* present in the bundle but not known to be launch-loaded;
* explicitly runtime-loaded later;
* unclear until shipped product inspection or trace.

### Launch Relevance

* loaded by the main executable at launch;
* loaded through an embedded framework dependency;
* linked but not required before first frame;
* needed only after login;
* needed only after feature navigation;
* app-extension-only;
* loaded later by explicit runtime loading;
* unclear until measured.

This classification matters more than the raw number of modules.

## When Linking Strategy Is Likely Relevant

Linking strategy is likely relevant when:

* traces, logs, or metrics point to dyld or pre-main cost;
* the release app embeds many non-system dynamic frameworks;
* internal modularization ships many small modules as dynamic frameworks;
* a recent package-manager or modularization change correlates with a launch regression;
* a dependency required only by a later feature is linked into the main app launch graph;
* the app and extensions duplicate large shared dependencies;
* Debug-only or tooling frameworks are present in Release;
* a vendored SDK is linked into the main app but used only in an optional flow;
* mergeable-library adoption or removal changed the shipped binary structure.

Do not blame linking strategy just because launch is slow. First classify the launch scenario and suspected phase.

## Static Linking

Static linking can reduce the number of separate dynamic images that dyld must load during launch.

It may help when:

* many first-party modules are shipped as dynamic frameworks only for development ergonomics;
* dynamic framework boundaries are not required at runtime;
* the app embeds many small internal frameworks;
* optional internal modules can be folded without resource or symbol issues;
* a release product can safely reduce dynamic image count.

Static linking is a candidate, not an automatic fix.

Before recommending it, check:

* binary size impact;
* duplicate symbol risk;
* Objective-C class/category duplication risk;
* resource bundle lookup;
* app extension sharing;
* vendor distribution constraints;
* Swift package or build-system support;
* debug and CI build-time impact;
* symbolication and crash-reporting expectations;
* whether the shipped Release app actually changes.

Static linking can reduce image count but may increase the main executable size. Validate whether the trade-off improves the measured launch phase rather than assuming fewer images always wins.

## Static Linking Risks

Static linking can create or reveal problems:

* larger main binary;
* slower incremental builds;
* duplicate symbols;
* duplicate Objective-C classes;
* different category loading behavior;
* resource lookup failures when code assumes a framework bundle;
* app and extension binary duplication;
* harder vendor SDK distribution;
* different binary-size trade-offs;
* changed symbolication expectations;
* package-manager configuration complexity.

For Swift packages, verify `Bundle.module` resource access after linkage or packaging changes. For framework-based code, verify `Bundle(for:)`, `Bundle(identifier:)`, and hardcoded framework-bundle assumptions.

For Objective-C static libraries with categories, check whether linker flags such as `-ObjC`, `-all_load`, or `-force_load` are required or already present. These flags can affect correctness, binary size, duplicate symbols, and launch behavior.

Do not recommend converting vendored dynamic SDKs to static linkage unless the vendor explicitly supports it.

## Dynamic Linking

Dynamic frameworks can be legitimate.

A dynamic boundary may be needed for:

* vendored SDK distribution;
* binary stability or distribution constraints;
* shared code between app and extensions;
* plugin-like architecture;
* independent signing or packaging requirements;
* runtime loading;
* development workflow;
* separate resource bundles;
* build-system constraints.

Dynamic linking becomes suspicious when dynamic boundaries exist only because modularization grew historically and the release app pays for many separate launch-loaded images that are not needed before first frame.

## Dynamic Linking Risks

Dynamic linking can increase launch cost through:

* more images to load;
* deeper dependency chains;
* more binding and fixup work;
* more exported symbols and metadata;
* Objective-C runtime registration;
* Swift metadata and protocol conformance registration;
* framework-level static initialization;
* resource and bundle lookup overhead;
* accidental inclusion of feature-only dependencies in the main launch graph.

Do not treat the framework count alone as proof. The actual cost depends on image contents, dependency graph, binary layout, OS behavior, device class, and whether the images are loaded during launch.

If the issue is initializer content inside a dynamic framework, route to `pre-main-dyld-and-static-initializers.md`. Use this file only to review whether that framework should be a separate launch-loaded image at all.

## Mergeable Libraries

Mergeable libraries are a useful candidate when an app wants dynamic-library development ergonomics but fewer separate dynamic images in the shipped release product.

They may help when:

* many internal dynamic frameworks exist mostly for modularization;
* runtime separation is not required;
* the build system supports mergeable-library behavior;
* the shipped release product can reduce image count without breaking resources, symbols, extensions, or debugging expectations.

Do not present mergeable libraries as a guaranteed drop-in launch fix.

Review:

* minimum supported OS, Xcode version, and build-system support;
* whether Debug and Release behave differently;
* whether the final `.app` actually contains fewer separate images;
* whether app extensions still require separate products;
* whether resources are still found correctly;
* whether symbolication and debugging remain acceptable;
* whether CI and incremental build behavior remain acceptable;
* whether the launch trace shows improvement in the expected phase.

Mergeable libraries can preserve source/module ergonomics while changing the shipped binary shape. Do not assume every project can adopt them without release validation.

## Package Manager Notes

Keep package-manager advice review-oriented. This file should not become a full build-system tutorial.

### Swift Package Manager

When reviewing Swift Package Manager usage, inspect:

* `library` products declared as `.static`, `.dynamic`, or automatic;
* package products used by the main app target;
* binary targets;
* package products used only by extensions or tests;
* whether a package is pulled into the main launch graph through another dependency;
* whether optional feature packages are linked into the main app executable.

Automatic package behavior can differ depending on the build system and product shape. Do not infer the shipped binary structure only from `Package.swift`.

For SPM, inspect the built products because automatic linkage can be influenced by the consuming target, product type, build configuration, and package graph.

### CocoaPods

When reviewing CocoaPods usage, inspect:

* `use_frameworks!`;
* static framework settings;
* vendored frameworks;
* vendored libraries;
* transitive pod dependencies;
* pods included in the main app but only used by optional features;
* Release vs Debug pod integration differences.

Do not recommend global `use_frameworks!` changes without checking Swift/Objective-C compatibility, vendored SDK constraints, resource bundles, and app extension behavior.

### Carthage, XCFrameworks, and Vendored SDKs

When reviewing Carthage, XCFrameworks, or vendored SDKs, inspect:

* whether the dependency is embedded as a dynamic framework;
* whether static variants are provided and supported;
* whether unused slices or debug artifacts are stripped correctly;
* whether the SDK is required before first frame;
* whether the SDK is used only in a later feature;
* whether the SDK must remain dynamic for vendor, licensing, signing, or distribution reasons.

This file can question whether an SDK must be linked into the main launch-loaded binary graph. It should not decide whether the SDK must initialize before first frame. Route SDK startup policy to `third-party-sdks-at-launch.md`.

## App Extensions and Shared Dependencies

App extensions can complicate linking decisions.

Review:

* whether the main app and extensions duplicate large static dependencies;
* whether a shared dynamic framework is justified by extension reuse;
* whether an extension-only dependency is linked into the main app;
* whether app-only dependencies are linked into extensions;
* whether resource bundles are shared safely;
* whether extension-safe API constraints affect linkage choices.

Static duplication between app and extension is a trade-off, not automatically a bug. It may be acceptable when the duplicated code is small or when sharing a dynamic framework would add launch-loaded images to the main app.

Do not optimize the main app launch by breaking extension packaging, signing, or runtime behavior.

## Optional Features and Runtime Loading

A dependency needed only by a later feature should not automatically sit on the main launch-loaded path.

Candidates for removal from the launch graph:

* payment SDK used only after checkout starts;
* map SDK used only after opening a map screen;
* camera or scanning SDK used only after a specific flow;
* large ML model wrapper used only by an optional feature;
* internal feature module used only behind a later route.

Possible strategies:

* move the dependency behind a feature-owned module;
* split the feature target from the main launch path;
* use lazy initialization inside the feature;
* use runtime loading only when platform, signing, and App Store constraints allow it;
* show feature-local loading and failure states.

Before moving an optional dependency out of the main target, check compile-time symbol references, generated registries, storyboards/nibs, URL/deep-link routers, Objective-C runtime lookup, and feature flag code paths.

Runtime loading is an advanced option, not a generic fix. It can complicate signing, review, testing, error handling, symbolication, and crash diagnostics.

Runtime loading should usually mean loading code already shipped and signed with the app, not downloading executable code. Do not suggest downloadable executable plugins for App Store apps.

## Advanced: Order Files and Binary Layout

Order files and binary layout can matter in advanced performance work, but they should not be the first recommendation.

Consider this area only when:

* the app already has reliable launch measurements;
* higher-level launch work has been reduced;
* the team has enough build-system control;
* the binary is large enough for layout effects to matter;
* the team can validate improvements across devices and OS versions.

Do not recommend order-file work as a substitute for removing unnecessary launch work, reducing dynamic images, fixing static initializers, or simplifying first-frame dependencies.

Route tool-specific measurement details to `metrics-instruments-xctest-metrickit.md`.

## Safe Patch Heuristics

When the agent is allowed to edit code, prefer small, reversible changes.

Good patch candidates:

* remove debug-only frameworks from Release builds;
* remove unused embedded frameworks after confirming they are not referenced;
* move optional feature dependencies out of the main app target when architecture already supports feature boundaries;
* change first-party internal modules from dynamic to static when resources, extensions, distribution constraints, and duplicate-symbol risks are checked;
* evaluate mergeable libraries for internal dynamic frameworks;
* split a large optional feature SDK behind a feature-owned wrapper;
* add build checks that report shipped embedded framework count;
* add notes or scripts that inspect the final `.app` product;
* add measurement markers before and after a linkage experiment.

Risky patch candidates requiring extra care:

* changing vendored binary SDK linkage manually;
* changing linkage for dependencies used by app extensions;
* changing resource packaging without verifying bundle lookup;
* converting Objective-C-heavy dynamic frameworks to static without duplicate-symbol, duplicate-class, and category-loading checks;
* removing frameworks based only on manifest inspection;
* introducing runtime loading as a generic fix;
* changing global CocoaPods linkage settings without auditing all pods and targets;
* changing binary layout or order files without stable launch measurements.

If correctness is uncertain, recommend inspection and measurement first, then behavior-changing linkage changes after evidence is available.

## Review Checklist

When reviewing linking strategy for launch performance, check:

* [ ] Is linking strategy actually relevant to the measured launch phase?
* [ ] Was the launch scenario classified before blaming linkage?
* [ ] Was the shipped Release app inspected, not only manifests?
* [ ] Are declared, linked, embedded, and launch-loaded dependencies distinguished?
* [ ] Are Apple system frameworks separated from embedded app frameworks?
* [ ] Are first-party, third-party, vendored, and debug-only dependencies classified separately?
* [ ] Is each dependency's launch relevance known?
* [ ] Are optional feature dependencies kept out of the main launch graph where possible?
* [ ] Are dynamic framework counts and dependency depth visible in the release product?
* [ ] Are transitive dynamic dependencies of embedded frameworks inspected when relevant?
* [ ] Are static linking risks checked before recommending conversion?
* [ ] Are mergeable libraries considered only where supported and measurable?
* [ ] Are app extension constraints preserved?
* [ ] Are resource bundle lookup and symbolication expectations preserved?
* [ ] Are Swift package `Bundle.module` assumptions preserved?
* [ ] Are Objective-C class/category duplication and linker-flag risks checked?
* [ ] Are vendor SDK linkage constraints respected?
* [ ] Is the recommendation tied to release-like launch measurement?

## Agent Guidance

When applying this reference, produce a linkage-oriented review:

```markdown
### Linking relevance

Explain why linking strategy is or is not likely relevant to the launch issue.

### Shipped product structure

Summarize what is known about embedded frameworks, dynamic images, package products, and vendored binaries.

### Dependency classification

Classify relevant dependencies by ownership, linkage, packaging, bundle/load status, and launch relevance.

### Suspicious launch-loaded dependencies

List dependencies that appear to be loaded at launch but are only needed later.

### Candidate changes

Recommend static linking, mergeable libraries, dependency removal, feature splitting, runtime loading, or no linkage change.

### Correctness and build risks

Call out resources, extensions, duplicate symbols, Objective-C classes/categories, linker flags, vendor constraints, symbolication, and build-time risks.

### Validation

Explain what release-like measurement or shipped-product inspection should improve.
```

Keep recommendations tied to evidence. If only manifests are available, label findings as hypotheses until the shipped product or launch trace is inspected.

## Boundary With Other References

Use this reference for launch-related linking and packaging decisions.

Read `references/launch-taxonomy-and-targets.md` when the issue involves:

* cold, warm, prewarmed, resume, first install, or update launch terminology;
* launch target selection;
* measurement scenario classification.

Read `references/pre-main-dyld-and-static-initializers.md` when the issue involves:

* dyld internals;
* pre-main work;
* `+load`;
* `+initialize`;
* constructor functions;
* Objective-C categories;
* runtime registration;
* static initialization content inside a framework.

Read `references/launch-orchestration-and-dependency-graph.md` when the issue involves:

* critical path analysis;
* startup step dependencies;
* hidden ordering;
* safe parallelism;
* failure policy;
* dependency-chain optimization.

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
