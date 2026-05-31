# Linking Strategy and Launch Cost

Use this reference when a launch investigation points to dynamic library loading, embedded framework count, modularization shape, static vs dynamic linkage, mergeable libraries, binary layout, or release-bundle dependency structure.

This file is about **linking choices that can affect app launch**. It is not a general modular architecture guide, a full dyld internals guide, or a build-time optimization manual.

## Scope Boundary

This file covers:

* static libraries and static frameworks
* dynamic frameworks and dynamic libraries
* mergeable libraries and merged release products
* embedded framework count and dependency depth
* first-party vs third-party dependency ownership
* release-build bundle inspection
* app/extension dependency duplication risk
* optional or feature-specific dependencies that are linked too early
* order files and binary layout as advanced launch-related linker topics

This file does not cover:

* detailed dyld internals, fixups, rebasing, binding, initializer order, or static initializer diagnosis; use `pre-main-dyld-and-static-initializers.md`
* Objective-C `+load`, `+initialize`, constructor functions, or initializer contents; use `pre-main-dyld-and-static-initializers.md`
* AppDelegate, SceneDelegate, root UI, dependency orchestration, or startup work deferral; use `appdelegate-scenedelegate-and-first-frame.md` and `launch-orchestration-and-dependency-graph.md`
* SwiftUI root app initialization; use `swiftui-app-launch.md`
* SDK initialization policy; use `third-party-sdks-at-launch.md`
* XCTest, Instruments, MetricKit, Organizer, CI baselines, or production monitoring; use `metrics-instruments-xctest-metrickit.md`

## Core Model

Linking strategy changes where cost is paid.

* **Static linking** happens at build time. Object code from libraries is linked into the consuming Mach-O image, subject to linker behavior such as symbol resolution and dead stripping.
* **Dynamic linking** happens at runtime. The app binary records dependencies on separate dynamic images, and dyld must load and connect those images when they are needed.
* **Mergeable libraries** are dynamic libraries with metadata that allows the linker to merge their contents into another binary, often giving teams dynamic-library ergonomics during development while reducing separate dynamic images in release builds.

Do not turn this into a simple rule like “static is always faster” or “dynamic is always bad.” The correct recommendation depends on the release product, dependency graph, device class, build configuration, app size, app-extension layout, vendor constraints, and measured launch phase.

## What the Agent Can Inspect

When repository access is available, inspect the project instead of guessing.

Look for:

* Xcode projects and workspaces
* Swift Package manifests and resolved package files
* CocoaPods manifests and lockfiles
* Carthage resolved dependencies
* Tuist, Bazel, Buck, or other project-generation manifests
* build scripts that copy, embed, strip, merge, sign, or remove frameworks
* vendored frameworks and XCFrameworks
* app extension targets and their shared dependencies
* separate Debug, Release, Profile, and CI build settings

When a built product is available, inspect the final `.app` bundle and the Mach-O binaries.

Useful questions:

* Which non-system frameworks are embedded under the app bundle?
* Which libraries does the main executable load directly?
* Which dependencies are loaded through another embedded framework?
* Which frameworks are present in Debug but not in Release?
* Which libraries were merged into the main executable or another framework?
* Which artifacts are resource bundles rather than runtime code?
* Are app extensions carrying duplicate copies of the same dependency?

Use command-line tools only as inspection aids. Bundle or binary inspection can show current structure, but it does not prove launch impact without a release-like launch measurement.

## System vs Embedded Frameworks

Do not treat Apple system frameworks and app-embedded frameworks the same way.

Apple platform frameworks are heavily optimized by the OS and commonly live in the dyld shared cache. They still appear as dependencies, but they are not usually the first optimization target for an app-level linking review.

For launch-cost review, focus first on:

* first-party embedded dynamic frameworks
* third-party embedded dynamic frameworks
* vendored binary SDKs
* dynamic framework chains created by internal modularization
* optional feature dependencies present in the main app binary graph
* debug-only or tooling frameworks accidentally included in release builds

## Classify Each Dependency

Before recommending a linkage change, classify each dependency.

### Ownership

* Apple system framework
* first-party internal module
* third-party source dependency
* third-party binary SDK
* vendored framework or XCFramework
* shared dependency used by app extensions
* optional feature dependency
* development-only or debug-only dependency

### Packaging and linkage

* static library archive
* static framework
* dynamic framework
* dynamic library
* mergeable library or mergeable framework
* Swift Package product with explicit static or dynamic type
* Swift Package product with automatic linkage
* CocoaPods source pod
* CocoaPods vendored framework or library
* Carthage or manually integrated binary artifact
* resource bundle without runtime code

### Launch relevance

* loaded by the main executable at launch
* loaded through an embedded framework dependency
* linked but not required before first frame
* needed only after login or feature navigation
* used only by an extension
* loaded later by an explicit runtime-loading mechanism
* unclear until the release product is inspected

## When Linking Strategy Is Likely Relevant

Investigate linking strategy when:

* Instruments or launch logs point to high dyld or pre-main time.
* The release app embeds many non-system dynamic frameworks.
* Internal modularization ships many small first-party modules as dynamic frameworks.
* Dynamic frameworks depend on other dynamic frameworks, creating deep loading chains.
* Recent package-manager or modularization changes correlate with launch regression.
* A dependency is included in the main app even though it is only used by a later feature.
* The app and one or more extensions include overlapping dependencies.
* Release bundle inspection shows debug-only or unused frameworks.
* A vendor SDK is linked into the main app but initialized only for an optional flow.

Do not blame linking strategy just because launch is slow. If the trace points to app delegate work, root UI creation, database startup, keychain access, blocking waits, network work, or first-frame rendering, route to the relevant reference.

## Static Linking

Static linking can reduce dynamic loader work because fewer separate app-embedded images need to be loaded and connected at launch.

It is often worth considering for:

* small first-party modules
* internal shared modules without a runtime boundary requirement
* launch-critical modules used immediately by the main app
* source dependencies controlled by the team
* feature modules that became dynamic through historical project setup
* modular apps with many embedded internal dynamic frameworks

Agent guidance:

* Suggest static linkage as a candidate, not as an automatic fix.
* Start with first-party modules because the team controls the build settings and integration details.
* Prefer removing an unnecessary dependency before changing its linkage.
* Validate app size, extension size, build time, symbolication, resources, and launch metrics after the change.
* Avoid hard thresholds such as a maximum number of allowed frameworks.

## Static Linking Risks

Static linking can introduce trade-offs or correctness problems.

Check for:

* slower clean or incremental links
* larger consuming binaries
* duplicate code across the main app and extensions
* duplicate symbols through multiple dependency paths
* Objective-C class duplication when the same static library is linked into multiple dynamic frameworks
* resource packaging changes when converting from dynamic framework to static library or static framework
* Swift module distribution requirements
* binary SDK support and licensing constraints
* exported symbol expectations
* debug and symbolication workflow changes
* longer CI build times

Do not recommend converting a vendored binary framework manually unless the vendor provides a supported static, dynamic, or mergeable variant.

## Dynamic Linking

Dynamic frameworks are not inherently wrong. They may be appropriate when the project needs:

* faster local iteration for large modules
* binary distribution through a vendor SDK
* a clear runtime boundary between components
* a shared binary used by multiple executables
* independent delivery or signing requirements
* a plugin-like architecture where the platform and product constraints allow it
* a development workflow that depends on dynamic library boundaries

Agent guidance:

* Ask whether the dynamic boundary is intentional or accidental.
* Separate development-time ergonomics from release-time launch cost.
* Verify whether a dependency remains a separate dynamic image in the shipped app.
* Prefer reducing unnecessary dynamic images over removing useful module boundaries.
* Avoid universal per-framework timing claims. Measure the app.

## Dynamic Linking Risks

Dynamic linking can affect launch because each dependent app-embedded dynamic image can add work for dyld and the kernel.

The actual cost depends on:

* device and OS version
* debug vs release configuration
* image count and dependency depth
* exported symbols and metadata
* Objective-C and Swift metadata
* static initializers and load-time hooks inside the image
* fixup format and linker output
* dyld shared cache behavior for system libraries
* whether the library is embedded, merged, or loaded later

Review dynamic frameworks for:

* unnecessary presence in the main app target
* deep dependency chains
* optional feature code linked into the launch path
* debug-only frameworks in release builds
* large vendor SDKs used by a narrow feature
* frameworks with load-time initializers
* duplicated dependencies across app and extensions
* package-manager defaults that accidentally produce dynamic frameworks

## Mergeable Libraries

Mergeable libraries are a strong candidate for large modular apps that want dynamic-style development while reducing release-time dynamic loading work.

They can help when the team wants:

* dynamic-library ergonomics in development
* fewer separate dynamic images in release builds
* a gradual migration path from many embedded internal frameworks
* reduced release bundle complexity
* smaller dependency chains loaded by dyld
* better launch characteristics without flattening the entire module architecture

Agent guidance:

* Suggest mergeable libraries when the project uses a toolchain and deployment setup where they are supported.
* Treat them as a candidate solution, not a guaranteed drop-in fix.
* Verify Debug and Release behavior separately.
* Inspect the shipped app to confirm which libraries were actually merged.
* Ensure merged dependencies are not still embedded unnecessarily.
* Check app-extension boundaries before merging shared dependencies.
* Validate symbolication, debugging, exported symbols, and CI behavior.

Do not give detailed setup instructions unless the user asks for implementation. For this skill, the important launch question is whether release builds still ship many separate dynamic images and whether merging can safely reduce them.

## Swift Package Manager Notes

When reviewing Swift Package dependencies:

* Inspect whether products explicitly request static or dynamic linkage.
* Treat omitted linkage as automatic rather than proof of static or dynamic behavior.
* Inspect the generated project or final build product when the package manager chooses linkage automatically.
* Check whether a package is linked by the main app, an extension, or both.
* Watch for package products that group many targets into a dynamic boundary unnecessarily.
* Avoid changing third-party package linkage if the package author documents constraints.

If a package appears launch-relevant, recommend confirming the final Mach-O shape rather than relying only on the manifest.

## CocoaPods Notes

When reviewing CocoaPods:

* Inspect whether pods are built as static libraries, static frameworks, dynamic frameworks, or vendored binaries.
* Check global linkage configuration and per-pod overrides.
* Review vendored frameworks and vendored libraries separately from source-built pods.
* Watch for pods embedded into the app even when only a later feature needs them.
* Check whether multiple pods wrap or re-export the same binary dependency.
* Be careful with resource bundles when changing linkage.

Do not assume all pods behave the same way. The final product matters more than the presence of a pod in the dependency graph.

## Carthage, XCFrameworks, and Vendored SDKs

When reviewing binary dependencies:

* Inspect the actual slice used by the app.
* Determine whether the binary is static, dynamic, or mergeable.
* Check whether the framework must be embedded and signed.
* Respect vendor-supported integration modes.
* Avoid manual binary surgery as a launch optimization recommendation.
* Ask whether the vendor provides a lighter, static, modular, or lazy-loadable variant.

For XCFrameworks, do not infer linkage from the package extension alone. An XCFramework can contain different kinds of library artifacts, but each integrated slice must be inspected to know what the app actually ships.

## App Extensions and Shared Dependencies

Be careful when the app has extensions.

Before recommending static linkage, merging, or framework removal, check:

* whether the dependency is used by the main app, an extension, or both
* whether static linking duplicates code into multiple executables
* whether a merged dependency increases extension size
* whether the extension needs only a smaller subset of the dependency
* whether the dependency uses APIs unavailable to extensions
* whether signing, distribution, or vendor integration requires a separate framework

There may be no single best answer. Explain the trade-off and recommend measuring both main app launch and extension size/startup behavior when relevant.

## Optional Features and Runtime Loading

If a dependency is only needed after the user opens a specific feature, it may not belong on the initial launch path.

Possible directions:

* remove the dependency from the main app target if it is unused
* move the dependency behind a feature module boundary
* split a large SDK wrapper into launch-critical and feature-specific pieces
* use a vendor-supported lazy initialization mode
* consider runtime loading only when the platform, App Store constraints, code signing, architecture, and failure behavior are understood

Do not recommend runtime loading as a generic iOS launch fix. It can complicate signing, testing, failure handling, symbolication, and architecture. Prefer normal linking cleanup, static/mergeable linkage, or lazy initialization first.

## Advanced: Order Files and Binary Layout

Order files and binary layout can be launch-related because launch-critical code scattered across many pages may increase paging and cache pressure.

Use this as an advanced topic, not as a default recommendation.

Consider it only when:

* larger launch-path work has already been addressed
* launch traces suggest instruction paging or binary layout contributes meaningfully
* the team can automate generation and validation
* the order file can be tied to representative launch scenarios
* CI can keep the file up to date as binaries change

Agent guidance:

* Do not include hand-written symbol lists.
* Do not claim a universal percentage improvement.
* Treat stale order files as a risk.
* Prefer automated generation from representative launch tests.
* Validate with release builds and real devices.

Implementation details belong in a build-system or profiling-specific reference if the user asks for them.

## Review Procedure

When asked to review linking strategy for launch performance:

1. **Identify evidence**

   * Is there a launch trace, dyld/pre-main regression, release-build measurement, or only a suspicion?
   * Is the issue observed in Debug, Release, TestFlight, or production?

2. **Inspect the shipped shape**

   * List non-system dynamic images in the final app bundle.
   * Separate Debug-only artifacts from Release/TestFlight artifacts.
   * Confirm whether mergeable libraries were actually merged.

3. **Map ownership and necessity**

   * Mark dependencies as first-party, third-party, vendor, optional, launch-critical, feature-specific, or extension-only.

4. **Find accidental boundaries**

   * Look for small first-party modules that are dynamic by history or tooling default rather than design.
   * Look for dynamic boundaries that could become static or mergeable in release builds.

5. **Check duplication and resource risks**

   * App extensions, multiple dynamic frameworks, static libraries through multiple paths, resource bundles, and vendored SDKs need special care.

6. **Recommend the smallest safe change**

   * Remove unused links or embeds first.
   * Narrow target membership before changing linkage.
   * Convert first-party modules before touching vendor SDKs.
   * Consider mergeable libraries for larger modular architectures.

7. **Require validation**

   * Compare release-like builds before and after.
   * Validate launch metrics, pre-main/dyld trace shape, bundle contents, app size, extension size, build time, and symbolication.

## Decision Rules

* Prefer removing an unnecessary dependency over changing its linkage.
* Prefer narrowing target membership over linking a dependency into every executable.
* Prefer first-party modules as the first optimization target.
* Prefer static or mergeable linkage for small internal modules that do not need a runtime boundary.
* Prefer dynamic linkage when independent distribution, runtime loading, app/extension sharing, or vendor constraints matter more than launch cost.
* Prefer mergeable libraries when the team wants modular development but fewer release-time dynamic images.
* Do not convert all modules at once.
* Do not use arbitrary numeric limits for framework count.
* Do not invent per-framework launch cost estimates.
* Do not assume Debug bundle contents represent Release/TestFlight behavior.
* Do not assume package-manager labels prove the final binary shape.
* Do not optimize linking before verifying that the slow phase is pre-main/dyld or release-bundle image loading.

## Common Findings

Use findings like these, tied to evidence:

* The release app embeds several first-party dynamic frameworks that appear to be small internal modules without an obvious runtime boundary. Consider static or mergeable linkage for those modules first.
* A dependency is linked by the main app but only used after login or feature navigation. Consider moving it behind a narrower feature boundary or lazy setup path.
* A framework is embedded as a separate runtime image, but the release product may not need that boundary. Verify whether it can be merged, statically linked, or removed from embedding.
* The main app and extension both consume the same dependency. A static conversion may duplicate code, so validate extension size and startup behavior before changing it.
* The project uses a vendor binary SDK. Do not rewrite its linkage manually; check vendor-supported variants and initialization modes.
* The launch regression appeared after a package-manager change. Inspect the final release bundle before assuming the manifest reflects the shipped binary shape.

## Output Guidance

When this reference is used, include a focused section in the final review:

```markdown
### Linking strategy

**Current shape:** static / dynamic / mergeable / mixed / unclear.

**Evidence:** launch trace, release bundle inspection, project settings, package manager files, or hypothesis.

**Launch risk:** why the dependency graph may affect launch.

**Safe next check:** what to inspect in project settings, generated project, or built app.

**Candidate change:** remove unused link/embed, narrow target membership, convert a first-party module to static, enable mergeable libraries, or keep dynamic.

**Trade-offs:** build time, app size, extension size, resources, SDK support, debugging, signing, and symbolication.

**Validation:** release-like launch measurement plus bundle and binary inspection.
```

Keep recommendations tied to evidence. If there is no launch trace or built-product inspection, label the finding as a hypothesis.
