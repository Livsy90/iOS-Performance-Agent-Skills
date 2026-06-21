# Modularization and Linking

Use this reference when the task involves Swift module-boundary optimizer visibility, public API resilience, `@inlinable`, `@usableFromInline`, `@frozen`, static vs dynamic libraries, mergeable libraries, binary size, or runtime trade-offs caused by modularization.

Do not use this reference as a general architecture guide. Use it to separate architectural boundaries from compiler visibility, ABI resilience, linking behavior, launch cost, binary size, and build-time/runtime trade-offs.

The goal is not to reduce the number of modules. The goal is to identify which boundary creates which cost, whether that boundary is required by the product or API, whether the optimizer can see enough implementation detail, and how the claim can be validated.

## Contents

* [Boundary with other skills](#boundary-with-other-skills)
* [When not to use this reference](#when-not-to-use-this-reference)
* [Core model](#core-model)
* [Review procedure](#review-procedure)
* [What the agent can inspect](#what-the-agent-can-inspect)
* [Module-boundary optimizer visibility](#module-boundary-optimizer-visibility)
* [Public API and hot paths](#public-api-and-hot-paths)
* [Resilience attributes](#resilience-attributes)
* [Library evolution](#library-evolution)
* [Optimization build modes](#optimization-build-modes)
* [Static, dynamic, and mergeable libraries](#static-dynamic-and-mergeable-libraries)
* [Binary size and build-time trade-offs](#binary-size-and-build-time-trade-offs)
* [Evidence signs](#evidence-signs)
* [Decision rules](#decision-rules)
* [Common mistakes](#common-mistakes)
* [Output guidance](#output-guidance)
* [Validation](#validation)

## Boundary with other skills

This reference belongs to `swift-runtime-performance` only when the question is about Swift runtime, optimizer, resilience, or linking consequences of module boundaries.

Prefer another skill when the main issue is different:

* Use `ios-launch-performance` for cold launch, pre-main, dyld work, framework loading, first frame, or first interaction.
* Use `ios-performance-profiling` for Instruments, MetricKit, signposts, trace interpretation, or measurement workflow.
* Use `swift-concurrency-performance` for actors, task lifetime, cancellation, priority, executors, or MainActor responsiveness.
* Use `swiftui-performance` for SwiftUI invalidation, identity, lists, layout, body cost, drawing, or view lifecycle work.

If a task mentions both modularization and launch, keep the responsibilities separate. This reference explains module and linking trade-offs. The launch skill decides whether those trade-offs are on the launch critical path.

## When not to use this reference

Do not use this reference as the primary source when:

* the question is only about architectural layering, ownership, team boundaries, or dependency direction with no runtime, optimizer, build, linking, or binary-size consequence;
* the issue is only dispatch, inlining, generics, existentials, or specialization inside a module — use `dispatch-and-specialization.md`;
* the issue is object layout, heap allocation, closure boxes, or storage location — use `allocation-and-layout.md`;
* the issue is retain/release traffic, closure lifetime, weak/unowned ownership, or reference cycles — use `arc-and-ownership.md`;
* the issue is app startup behavior itself — use `ios-launch-performance`.

If one of these files does not exist yet, treat the link as intended routing and do not invent details from a missing reference.

## Core model

A module boundary can affect several independent concerns:

* team ownership and dependency direction;
* public API design;
* optimizer visibility;
* generic specialization and inlining;
* ABI and library evolution;
* static or dynamic linking;
* binary size;
* app launch behavior;
* incremental and release build time.

Do not collapse these into one rule such as "more modules are slower" or "static linking is always faster."

Use this mental model:

* architecture decides where boundaries should exist;
* the compiler decides what it can see and optimize;
* the linker and loader decide how code is packaged and loaded;
* ABI resilience decides what clients may assume about public declarations;
* build settings decide how much optimization work happens and where.

A good recommendation preserves useful boundaries while removing accidental runtime cost from hot paths.

Keep these claims separate:

* **Runtime CPU cost:** missed inlining, missed specialization, indirect dispatch, type erasure, ARC traffic, or repeated boundary crossings.
* **Launch cost:** extra images, loader work, initialization work, code signing, framework embedding, or code on the launch critical path.
* **Binary size cost:** duplicated static code, specialization growth, public symbols, large dependencies, or retained unused code.
* **Build-time cost:** module count, dependency graph, whole-module optimization, cross-module optimization, link time, packaging, or binary distribution settings.

Do not claim improvement in one category only because another category improved.

## Review procedure

1. Identify the symptom: CPU time, allocation, ARC traffic, missed specialization, binary size, launch cost, build time, or API rigidity.
2. Locate the boundary: Swift module, package target, dynamic framework, static library, binary framework, public API, resilient API, or third-party dependency.
3. Check whether the hot path crosses that boundary repeatedly.
4. Check whether the implementation is visible to the optimizer in the relevant release configuration.
5. Decide whether the problem is source-level API shape, access control, build settings, linking mode, distribution model, or architecture.
6. Prefer design changes before attributes: move the hot loop, batch calls, keep helpers internal, add a concrete fast path, or reduce type erasure.
7. Use `@inlinable`, `@usableFromInline`, and `@frozen` only when the API and ABI commitment is intentional.
8. Explain trade-offs across runtime speed, launch cost, binary size, build time, API flexibility, binary compatibility, and team ownership.
9. Require validation before claiming a performance win.

## What the agent can inspect

Useful repository searches:

```bash
rg "@inlinable|@usableFromInline|@frozen" .
rg "BUILD_LIBRARY_FOR_DISTRIBUTION|MACH_O_TYPE|DEFINES_MODULE" .
rg "libraryEvolution|buildLibraryForDistribution|type:.*dynamic|type:.*static" Package.swift .
rg "public protocol|public struct|public enum|public final class|open class" Sources
rg "package protocol|package struct|package enum|package final class|package func" Sources
rg "any [A-Z][A-Za-z0-9_]*" Sources
rg "\.binaryTarget|binaryTarget" Package.swift
```

Useful inputs:

* `Package.swift` target graph and product types;
* Xcode build settings for library evolution and Mach-O type;
* `.xcconfig`, `.pbxproj`, CI scripts, and generated project settings;
* framework embedding and signing settings;
* public and package-level APIs in hot modules;
* binary framework boundaries;
* optimized SIL for suspected hot functions;
* release build configuration;
* linker map, app size report, or symbol-size report when binary size is the claim.

These searches are starting points, not proof. Xcode project files, `.xcconfig` files, generated projects, CI scripts, and package manager integration can change the effective build settings.

Do not infer runtime cost from file count, target count, or package count alone.

## Module-boundary optimizer visibility

Within one optimized Swift module, the compiler usually has more source visibility. That can help it inline small functions, specialize generics, devirtualize calls, remove abstraction, and reduce ARC traffic.

Across module boundaries, the compiler may see only the public interface. It may not see enough implementation detail to specialize a generic helper or inline a small wrapper unless the API exposes additional information or the build mode provides more visibility.

This matters most for:

* small generic helpers called inside large loops;
* protocol-heavy APIs in hot paths;
* collection transformations;
* parsing, serialization, image, geometry, and numeric code;
* type-erased wrappers;
* public convenience APIs used by performance-sensitive clients;
* repeated boundary crossings where one batched operation would be cheaper.

Review rules:

* Keep hot implementation details internal when possible.
* Avoid making APIs public only because the module split is inconvenient.
* Use `package` access when it fits Swift package boundaries and avoids unnecessary `public` API.
* Do not treat `package` access as a replacement for `@inlinable` or as a binary distribution tool.
* Check whether the caller can see the concrete type.
* Check whether optimized SIL shows specialization or witness dispatch.
* Redesign the boundary before adding ABI-visible attributes.

Visibility helps the optimizer, but it does not guarantee inlining, devirtualization, specialization, or speedup. Use optimized output and measurement when the performance claim depends on it.

## Public API and hot paths

A common mistake is turning an implementation detail into `public` API just so another module can call it.

Prefer this shape when possible:

* expose a public operation that matches the real use case;
* keep the hot loop and concrete implementation inside the defining module;
* expose configuration values instead of many tiny public callbacks;
* pass data in batches instead of repeatedly crossing the boundary;
* keep fast-path helpers internal or package-internal when possible;
* add a concrete fast path behind a public abstraction when dynamic behavior is still needed.

Use public protocols, existential parameters, and type erasure when the architecture needs runtime heterogeneity or decoupling. Do not remove dynamic behavior that is part of the product design.

When a hot path crosses a module boundary, ask:

* Is the function `private`, `internal`, `package`, `public`, or `open`?
* Is it generic?
* Does it accept `any Protocol` or a concrete type?
* Does the caller know the concrete implementation?
* Is the implementation body visible to the caller's optimizer?
* Is library evolution enabled?
* Is the dependency distributed as source or binary?
* Does optimized SIL show specialization, inlining, `witness_method`, `class_method`, or `open_existential_*`?

## Resilience attributes

### `@inlinable`

`@inlinable` exposes the body of a public or internal declaration as part of the module interface so the optimizer in client modules may use it.

It does not guarantee inlining. The optimizer may inline, specialize, perform other interprocedural analysis, or ignore the body depending on optimization level, code size, profitability, and build settings.

Good candidates:

* small public functions;
* stable utility functions;
* simple computed properties;
* thin forwarding functions;
* performance-critical generic wrappers;
* algorithms where client-side specialization is important.

Avoid it for large, unstable, frequently changing, implementation-revealing, or cold functions.

Ask:

* Is the function truly part of the external API, or only exposed because of the module split?
* Is the body small and stable enough to expose?
* Does optimized SIL or benchmarking show that cross-module visibility matters?
* Would moving the hot loop into the defining module avoid the attribute?
* Are source compatibility, binary compatibility, and downstream compile-time costs acceptable?

Do not use `@inlinable` as a general performance switch. It is a visibility and resilience decision, not just an optimization hint.

### `@usableFromInline`

`@usableFromInline` allows an internal declaration to be referenced from an `@inlinable` declaration.

It does not make the declaration source-public. It makes an internal declaration ABI-public enough that inlined client code can depend on its symbol or signature.

Use it only when an `@inlinable` function needs a helper that should not be source-public, and the helper's signature is stable enough for ABI exposure.

Avoid it when:

* the helper is unstable;
* the helper exposes private design;
* the helper exists only because `@inlinable` was added prematurely;
* moving the hot implementation into the defining module would be simpler;
* the ABI-visible commitment is not intentional.

### `@frozen`

`@frozen` is a library-evolution tool. It tells clients that the stored layout of a public struct or the cases of a public enum are stable across binary-compatible versions.

It can enable better client-side optimization because clients may make stronger assumptions about layout or enum cases. The cost is reduced evolution flexibility.

Use `@frozen` only for ABI-public structs or enums in library-evolution contexts when the layout or case set is intentionally stable. In app-internal modules or builds without library evolution, it is usually not a useful local performance annotation.

Good candidates:

* small value types with intentionally stable stored properties;
* low-level performance types;
* enums whose cases are part of a stable domain model;
* public types in binary frameworks where layout stability is acceptable.

Avoid it for:

* models likely to gain fields;
* enums likely to gain cases;
* DTOs controlled by changing backend payloads;
* normal app-internal types;
* types whose layout is not part of the intended long-term ABI contract.

Do not use `@frozen` as a casual optimization hint.

## Library evolution

Library evolution allows a binary framework to change implementation details without breaking already-built clients. This flexibility can reduce optimization opportunities because clients cannot assume all public implementation details.

Common effects of resilient public boundaries:

* public function bodies may be hidden unless made inlinable;
* public struct layout may be hidden unless frozen;
* public enum cases may be resilient unless frozen;
* some calls or field accesses may use less direct patterns;
* clients may have less room for inlining and specialization.

Ask:

* Is the module shipped as a binary framework?
* Does it need binary compatibility across versions?
* Is `BUILD_LIBRARY_FOR_DISTRIBUTION` enabled because distribution requires it?
* Is this an app-internal module where resilience is unnecessary?
* Are performance-sensitive APIs accidentally placed behind resilient public boundaries?

`BUILD_LIBRARY_FOR_DISTRIBUTION` is a distribution and resilience decision first. Do not enable it for every internal target by habit, and do not disable it for a binary framework that depends on stable distribution compatibility.

Do not disable library evolution just for speed if the framework's distribution model requires binary compatibility.

Module stability is not the same thing as library evolution. Module stability is about importing a module interface across compiler versions. Library evolution is about binary-compatible evolution of a framework.

## Optimization build modes

Whole-module optimization improves compiler visibility within a single module. It can help with inlining, specialization, devirtualization, ARC optimization, dead-code elimination, and abstraction removal.

It does not automatically solve cross-module visibility. A module boundary may still hide implementation details unless the build mode or API surface exposes them.

Cross-module optimization settings can improve visibility across modules in some build configurations. Treat them as build-system tools, not API design substitutes.

Ask:

* Is the suspected cost inside one module or across modules?
* Is the build optimized and release-like?
* Is whole-module optimization enabled where expected?
* Is cross-module optimization available and appropriate for owned source modules?
* Does the hot path involve third-party or binary dependencies where the setting cannot help?
* Is the release build-time impact acceptable?

Do not reason about optimizer behavior from Debug builds.

## Static, dynamic, and mergeable libraries

Static and dynamic linking are packaging choices with runtime, launch, build, and distribution consequences.

Static linking can reduce dynamic loader work and simplify the runtime dependency graph, but it can also increase binary size, duplicate code across dynamic products, slow clean release links, or complicate dependency management.

Dynamic frameworks can support binary distribution, independent framework boundaries, and large-team ownership, but they add runtime images, embedding/signing complexity, and may affect launch when they are on the critical path.

Mergeable libraries are an Apple/Xcode packaging feature, not a Swift language feature. Check platform, Xcode version, target type, and build settings before recommending them.

Mergeable libraries can preserve a library-like development model while allowing the final product to merge libraries for deployment. Treat them as a packaging tool, not as a cure-all.

Ask:

* Is separate dynamic loading or binary distribution required?
* Is the dependency app-internal?
* Is the measured problem launch, runtime CPU, binary size, or build time?
* Could static linking duplicate code across dynamic products?
* Would mergeable libraries reduce runtime image cost without damaging workflow?
* Is the real issue API shape, public access, type erasure, or missed specialization rather than packaging?
* Is the target platform and build system capable of using the proposed packaging mode?

Do not recommend static linking, dynamic frameworks, or mergeable libraries by default. Tie the recommendation to the measured cost and product constraints.

Static linking may reduce dynamic loader work, but it does not guarantee faster launch. Binary size, code signing, paging behavior, framework initialization, and launch-critical work also matter.

## Binary size and build-time trade-offs

Module and linking choices can affect binary size through duplicated static code, excessive specialization, aggressive inlining, many generic instantiations, retained public symbols, or unnecessary dependencies.

They can also affect build time:

* merging modules can improve optimizer visibility but increase recompilation scope;
* `@inlinable` can expose bodies and increase downstream compile work;
* static linking can affect clean release link time;
* cross-module optimization can increase release build time;
* excessive generic specialization can increase code size and compile time;
* binary distribution settings can add packaging and compatibility work.

Do not present runtime optimization as free. Include build, size, and maintainability cost when changing module structure.

## Evidence signs

Use evidence that matches the claim.

For optimized SIL, useful signs may include:

* specialized function variants;
* remaining `witness_method`, `open_existential_*`, or `class_method` in hot code;
* calls that did not inline across a module boundary;
* retained `partial_apply` wrappers;
* retained `strong_retain` / `strong_release` around public wrappers or erased values;
* generic functions that remain unspecialized in a hot path;
* public wrapper layers that survive optimization.

Treat SIL instruction names as implementation-level evidence, not stable API. Exact lowering can change across compiler versions, optimization levels, build settings, language modes, ownership modes, and target platforms.

For build settings, useful signs may include:

* `BUILD_LIBRARY_FOR_DISTRIBUTION`;
* Mach-O type;
* static or dynamic package product type;
* binary targets;
* framework embedding and signing settings;
* whole-module optimization or cross-module optimization settings.

For linking and binary size, useful signs may include:

* linker maps;
* symbol-size reports;
* app size reports;
* duplicate static dependency symbols across dynamic products;
* number and size of embedded dynamic frameworks;
* packaging differences between Debug, Release, and distribution builds.

For launch claims, use launch traces. Do not infer launch improvement only from linking mode.

## Decision rules

### If hot generic code crosses a module boundary

Prefer moving the hot loop or concrete implementation into one module before adding attributes. Use `@inlinable` only if the API is public or internal in a way that needs client visibility, the body is small and stable, and the optimization issue is measured or strongly justified.

### If an API is public only because another module needs it

Question the module boundary. Consider moving the caller, introducing a higher-level operation, using `package` access when appropriate, or keeping a concrete fast path internal.

### If `BUILD_LIBRARY_FOR_DISTRIBUTION` is enabled

Ask whether the module is actually distributed as a binary framework. If it is app-internal, the setting may add resilience constraints without benefit. If it is a binary framework that needs compatibility, do not disable the setting only for local speed.

### If dynamic frameworks are blamed for launch

Route launch-critical analysis to `ios-launch-performance`. In this reference, explain the packaging trade-off and avoid claiming causality without launch measurements.

### If someone proposes static linking

Check whether separate dynamic loading, binary distribution, or plugin-like behavior is needed. Also check duplicate code, binary-size risk, clean release link time, and launch evidence.

### If someone proposes mergeable libraries

Check whether the project, platform, Xcode version, and target type support them. Treat them as packaging configuration, not a replacement for API or module-boundary design.

### If someone proposes `@frozen`

Require a stable public layout or enum case set in a library-evolution context. Do not use it for models expected to evolve or for app-local performance decoration.

## Common mistakes

* Treating module count as a performance metric.
* Recommending fewer modules without identifying a measured cost.
* Treating `public` as harmless in app-internal modular code.
* Using `@inlinable` on unstable or large functions.
* Treating `@inlinable` as a guarantee that the function will inline.
* Adding `@usableFromInline` without understanding the ABI-visible commitment.
* Adding `@frozen` to types that are likely to evolve.
* Using `@frozen` as an app-local performance decoration.
* Disabling library evolution for a binary framework that needs compatibility.
* Enabling distribution-oriented settings for every internal target by habit.
* Assuming static linking always improves performance.
* Assuming dynamic frameworks are always the launch bottleneck.
* Treating mergeable libraries as a Swift language feature or universal fix.
* Ignoring binary-size and build-time costs from inlining and specialization.
* Drawing conclusions from Debug builds.
* Inferring effective build settings from one repository search.

## Output guidance

When responding to a modularization/runtime review, include:

1. The boundary involved: module, package target, framework, binary framework, public API, package API, or resilience boundary.
2. The suspected cost category: runtime CPU, allocation/ARC, missed specialization, indirect dispatch, type erasure, dynamic loading, launch cost, binary size, or build-time trade-off.
3. The evidence needed: optimized SIL, profiling, launch trace, binary-size report, linker map, build settings, or build-time measurement.
4. The safest design change before attributes or linking changes.
5. Any API, ABI, binary size, launch, build-time, distribution, or team-ownership trade-off.
6. A validation step.

Avoid saying "make it static," "add `@inlinable`," or "merge modules" without explaining why that change targets the measured cost.

Use cautious language when evidence is incomplete:

* "This boundary may limit specialization; check optimized SIL before changing the API."
* "Static linking may reduce loader work, but launch needs a launch trace."
* "`@inlinable` exposes implementation detail and does not guarantee inlining."
* "`@frozen` is a library-evolution commitment, not a local speed switch."
* "The repository search shows a candidate setting, not necessarily the effective build configuration."

## Validation

Use validation that matches the claim.

For runtime hot paths:

* profile release-like builds on a real device;
* inspect optimized SIL for specialization, inlining, witness dispatch, existential opening, and ARC traffic;
* benchmark the isolated operation if it is deterministic;
* compare before/after with the same input size and build settings.

For launch-related claims:

* use the launch skill's measurement workflow;
* compare cold launch and first-frame/first-interaction metrics;
* check whether the changed framework or library is on the launch critical path;
* verify with a launch trace rather than linking mode alone.

For binary-size claims:

* compare app size artifacts;
* inspect linker maps or symbol-size reports;
* check whether static dependencies are duplicated across products;
* compare the same configuration before and after the change.

For build-time claims:

* compare clean and incremental release builds;
* separate compile, link, and package time;
* include the cost of cross-module optimization or additional specialization.

A modularization change is successful only if it improves the targeted metric without creating unacceptable API, ABI, size, build, distribution, or ownership cost.
