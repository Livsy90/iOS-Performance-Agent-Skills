# Modularization and Linking

Use this reference when the task involves Swift module-boundary optimizer visibility, public API resilience, `@inlinable`, `@usableFromInline`, `@frozen`, static vs dynamic libraries, mergeable libraries, binary size, or runtime trade-offs caused by modularization.

Do not use this reference as a general architecture guide. Use it to separate architectural boundaries from compiler visibility, ABI resilience, linking behavior, launch cost, binary size, and build-time/runtime trade-offs.

## Contents

- [Boundary with other skills](#boundary-with-other-skills)
- [Core model](#core-model)
- [Review procedure](#review-procedure)
- [What the agent can inspect](#what-the-agent-can-inspect)
- [Module-boundary optimizer visibility](#module-boundary-optimizer-visibility)
- [Public API and hot paths](#public-api-and-hot-paths)
- [Resilience attributes](#resilience-attributes)
- [Library evolution](#library-evolution)
- [Optimization build modes](#optimization-build-modes)
- [Static, dynamic, and mergeable libraries](#static-dynamic-and-mergeable-libraries)
- [Binary size and build-time trade-offs](#binary-size-and-build-time-trade-offs)
- [Decision rules](#decision-rules)
- [Common mistakes](#common-mistakes)
- [Output guidance](#output-guidance)
- [Validation](#validation)

## Boundary with other skills

This reference belongs to `swift-runtime-performance` only when the question is about Swift runtime or optimizer consequences of module boundaries.

Prefer another skill when the main issue is different:

- Use `ios-launch-performance` for cold launch, pre-main, dyld work, framework loading, first frame, or first interaction.
- Use `ios-performance-profiling` for Instruments, MetricKit, signposts, trace interpretation, or measurement workflow.
- Use `swift-concurrency-performance` for actors, task lifetime, cancellation, priority, executors, or MainActor responsiveness.
- Use `swiftui-performance` for SwiftUI invalidation, identity, lists, layout, body cost, drawing, or view lifecycle work.

If a task mentions both modularization and launch, keep the responsibilities separate. This reference explains module and linking trade-offs. The launch skill decides whether those trade-offs are on the launch critical path.

## Core model

A module boundary can affect several independent concerns:

- team ownership and dependency direction;
- public API design;
- optimizer visibility;
- generic specialization and inlining;
- ABI and library evolution;
- static or dynamic linking;
- binary size;
- app launch behavior;
- incremental and release build time.

Do not collapse these into one rule such as "more modules are slower" or "static linking is always faster."

Use this mental model:

- architecture decides where boundaries should exist;
- the compiler decides what it can see and optimize;
- the linker and loader decide how code is packaged and loaded;
- ABI resilience decides what clients may assume about public declarations.

A good recommendation preserves useful boundaries while removing accidental runtime cost from hot paths.

## Review procedure

1. 1Identify the symptom: CPU time, allocation, ARC traffic, missed specialization, binary size, launch cost, build time, or API rigidity.
2. 2Locate the boundary: Swift module, package target, dynamic framework, binary framework, public API, resilient API, or third-party dependency.
3. 3Check whether the hot path crosses that boundary repeatedly.
4. 4Check whether the implementation is visible to the optimizer in the relevant release configuration.
5. 5Decide whether the problem is source-level API shape, build settings, linking mode, or architecture.
6. 6Prefer design changes before attributes: move the hot loop, batch calls, keep helpers internal, add a concrete fast path, or reduce type erasure.
7. 7Use `@inlinable`, `@usableFromInline`, and `@frozen` only when the API and ABI commitment is intentional.
8. 8Explain trade-offs across runtime speed, launch cost, binary size, build time, API flexibility, binary compatibility, and ownership.
9. 9Require validation before claiming a performance win.

## What the agent can inspect

Useful repository searches:

```bash
rg "@inlinable|@usableFromInline|@frozen" .
rg "BUILD_LIBRARY_FOR_DISTRIBUTION|MACH_O_TYPE|DEFINES_MODULE" .
rg "libraryEvolution|buildLibraryForDistribution|type:.*dynamic|type:.*static" Package.swift .
rg "public protocol|public struct|public enum|public final class|open class" Sources
rg "any [A-Z][A-Za-z0-9_]*" Sources
rg "\.binaryTarget|binaryTarget" Package.swift
```

Useful inputs:

- `Package.swift` target graph and product types;
- Xcode build settings for library evolution and Mach-O type;
- framework embedding settings;
- public APIs in hot modules;
- binary framework boundaries;
- optimized SIL for suspected hot functions;
- release build configuration.

Do not infer runtime cost from file count or target count alone.

## Module-boundary optimizer visibility

Within one optimized Swift module, the compiler usually has more source visibility. That can help it inline small functions, specialize generics, devirtualize calls, remove abstraction, and reduce ARC traffic.

Across module boundaries, the compiler may see only the public interface. It may not see enough implementation detail to specialize a generic helper or inline a small wrapper unless the API exposes additional information or the build mode provides more visibility.

This matters most for:

- small generic helpers called inside large loops;
- protocol-heavy APIs in hot paths;
- collection transformations;
- parsing, serialization, image, geometry, and numeric code;
- type-erased wrappers;
- public convenience APIs used by performance-sensitive clients;
- repeated boundary crossings where one batched operation would be cheaper.

Review rules:

- Keep hot implementation details internal when possible.
- Avoid making APIs public only because the module split is inconvenient.
- Check whether the caller can see the concrete type.
- Check whether optimized SIL shows specialization or witness dispatch.
- Redesign the boundary before adding ABI-visible attributes.

## Public API and hot paths

A common mistake is turning an implementation detail into `public` API just so another module can call it.

Prefer this shape when possible:

- expose a public operation that matches the real use case;
- keep the hot loop and concrete implementation inside the defining module;
- expose configuration values instead of many tiny public callbacks;
- pass data in batches instead of repeatedly crossing the boundary;
- keep fast-path helpers internal;
- add a concrete fast path behind a public abstraction when dynamic behavior is still needed.

Use public protocols, existential parameters, and type erasure when the architecture needs runtime heterogeneity or decoupling. Do not remove dynamic behavior that is part of the product design.

When a hot path crosses a module boundary, ask:

- Is the function public, internal, package, or open?
- Is it generic?
- Does it accept `any Protocol` or a concrete type?
- Does the caller know the concrete implementation?
- Is the implementation body visible to the caller's optimizer?
- Is library evolution enabled?
- Is the dependency distributed as source or binary?
- Does optimized SIL show specialization, inlining, `witness_method`, `class_method`, or `open_existential`?

## Resilience attributes

### `@inlinable`

`@inlinable` exposes the body of a public declaration to clients so the client module's optimizer can use it.

Good candidates:

- small public functions;
- stable utility functions;
- simple computed properties;
- thin forwarding functions;
- performance-critical generic wrappers;
- algorithms where client-side specialization is important.

Avoid it for large, unstable, frequently changing, or implementation-revealing functions.

Ask:

- Is the function truly public API, or only public because of the module split?
- Is the body small and stable enough to expose?
- Does optimized SIL or benchmarking show that cross-module visibility matters?
- Would moving the hot loop into the defining module avoid the attribute?

Do not use `@inlinable` as a general performance switch. It is a visibility and resilience decision, not just an optimization hint.

### `@usableFromInline`

`@usableFromInline` allows an internal declaration to be used from an `@inlinable` declaration.

It is not ordinary `internal`. The declaration is not source-public, but its signature becomes ABI-visible enough for inlined client code to depend on it.

Use it only when an `@inlinable` function needs a helper that should not be source-public, and the helper's signature is stable enough for ABI exposure.

Avoid it when the helper is unstable, exposes private design, or exists only because `@inlinable` was added prematurely.

### `@frozen`

`@frozen` is a library-evolution tool. It tells clients that the stored layout of a public struct or the cases of a public enum are stable across binary-compatible versions.

It can enable better client-side optimization because clients may make stronger assumptions about layout or enum cases. The cost is reduced evolution flexibility.

Good candidates:

- small value types with intentionally stable stored properties;
- low-level performance types;
- enums whose cases are part of a stable domain model;
- public types in binary frameworks where layout stability is acceptable.

Avoid it for models likely to gain fields, enums likely to gain cases, DTOs controlled by changing backend payloads, or normal app-internal types.

Do not use `@frozen` as a casual optimization hint.

## Library evolution

Library evolution allows a binary framework to change implementation details without breaking already-built clients. This flexibility can reduce optimization opportunities because clients cannot assume all public implementation details.

Common effects of resilient public boundaries:

- public function bodies may be hidden unless made inlinable;
- public struct layout may be hidden unless frozen;
- public enum cases may be resilient unless frozen;
- some calls or field accesses may use less direct patterns;
- clients may have less room for inlining and specialization.

Ask:

- Is the module shipped as a binary framework?
- Does it need binary compatibility across versions?
- Is `BUILD_LIBRARY_FOR_DISTRIBUTION` enabled because distribution requires it?
- Is this an app-internal module where resilience is unnecessary?
- Are performance-sensitive APIs accidentally placed behind resilient public boundaries?

Do not disable library evolution just for speed if the framework's distribution model requires binary compatibility.

Module stability is not the same thing as library evolution. Module stability is about importing a module interface across compiler versions. Library evolution is about binary-compatible evolution of a framework.

## Optimization build modes

Whole-module optimization improves compiler visibility within a single module. It can help with inlining, specialization, devirtualization, ARC optimization, dead-code elimination, and abstraction removal.

It does not automatically solve cross-module visibility. A module boundary may still hide implementation details unless the build mode or API surface exposes them.

Cross-module optimization settings can improve visibility across modules in some build configurations. Treat them as build-system tools, not API design substitutes.

Ask:

- Is the suspected cost inside one module or across modules?
- Is the build optimized and release-like?
- Is whole-module optimization enabled where expected?
- Is cross-module optimization available and appropriate for owned source modules?
- Does the hot path involve third-party or binary dependencies where the setting cannot help?
- Is the release build-time impact acceptable?

Do not reason about optimizer behavior from Debug builds.

## Static, dynamic, and mergeable libraries

Static and dynamic linking are packaging choices with runtime, launch, build, and distribution consequences.

Static linking can reduce dynamic loader work and simplify the runtime dependency graph, but it can also increase binary size, duplicate code across dynamic products, slow clean release links, or complicate dependency management.

Dynamic frameworks can support binary distribution, independent framework boundaries, and large-team ownership, but they add runtime images, embedding/signing complexity, and may affect launch when they are on the critical path.

Mergeable libraries can preserve a library-like development model while allowing the final product to merge libraries for deployment. Treat them as a packaging tool, not as a cure-all.

Ask:

- Is separate dynamic loading or binary distribution required?
- Is the dependency app-internal?
- Is the measured problem launch, runtime CPU, binary size, or build time?
- Could static linking duplicate code across dynamic products?
- Would mergeable libraries reduce runtime image cost without damaging workflow?
- Is the real issue API shape or missed specialization rather than packaging?

Do not recommend static linking, dynamic frameworks, or mergeable libraries by default. Tie the recommendation to the measured cost and product constraints.

## Binary size and build-time trade-offs

Module and linking choices can affect binary size through duplicated static code, excessive specialization, aggressive inlining, many generic instantiations, retained public symbols, or unnecessary dependencies.

They can also affect build time:

- merging modules can improve optimizer visibility but increase recompilation scope;
- `@inlinable` can expose bodies and increase downstream compile work;
- static linking can affect clean release link time;
- cross-module optimization can increase release build time;
- excessive generic specialization can increase code size and compile time.

Do not present runtime optimization as free. Include build, size, and maintainability cost when changing module structure.

## Decision rules

### If hot generic code crosses a module boundary

Prefer moving the hot loop or concrete implementation into one module before adding attributes. Use `@inlinable` only if the API is public, small, stable, and measured or strongly justified.

### If an API is public only because another module needs it

Question the module boundary. Consider moving the caller, introducing a higher-level operation, or keeping a concrete fast path internal.

### If `BUILD_LIBRARY_FOR_DISTRIBUTION` is enabled

Ask whether the module is actually distributed as a binary framework. If it is app-internal, the setting may add resilience constraints without benefit.

### If dynamic frameworks are blamed for launch

Route launch-critical analysis to `ios-launch-performance`. In this reference, explain the packaging trade-off and avoid claiming causality without launch measurements.

### If someone proposes static linking

Check whether separate dynamic loading, binary distribution, or plugin-like behavior is needed. Also check duplicate code and binary-size risk.

### If someone proposes `@frozen`

Require a stable public layout or enum case set. Do not use it for models expected to evolve.

## Common mistakes

- Treating module count as a performance metric.
- Recommending fewer modules without identifying a measured cost.
- Treating `public` as harmless in app-internal modular code.
- Using `@inlinable` on unstable or large functions.
- Adding `@usableFromInline` without understanding the ABI-visible commitment.
- Adding `@frozen` to types that are likely to evolve.
- Disabling library evolution for a binary framework that needs compatibility.
- Enabling distribution-oriented settings for every internal target by habit.
- Assuming static linking always improves performance.
- Assuming dynamic frameworks are always the launch bottleneck.
- Ignoring binary-size and build-time costs from inlining and specialization.
- Drawing conclusions from Debug builds.

## Output guidance

When responding to a modularization/runtime review, include:

1. 1The boundary involved: module, package target, framework, binary framework, public API, or resilience boundary.
2. 2The suspected runtime cost: missed specialization, indirect dispatch, type erasure, ARC traffic, dynamic loading, binary size, or build trade-off.
3. 3The evidence needed: optimized SIL, profiling, launch trace, binary-size report, linker map, or build settings.
4. 4The safest design change before attributes or linking changes.
5. 5Any API, ABI, binary size, launch, build-time, or team-ownership trade-off.
6. 6A validation step.

Avoid saying "make it static," "add `@inlinable`," or "merge modules" without explaining why that change targets the measured cost.

## Validation

Use validation that matches the claim.

For runtime hot paths:

- profile release-like builds on a real device;
- inspect optimized SIL for specialization, inlining, witness dispatch, existential opening, and ARC traffic;
- benchmark the isolated operation if it is deterministic;
- compare before/after with the same input size and build settings.

For launch-related claims:

- use the launch skill's measurement workflow;
- compare cold launch and first-frame/first-interaction metrics;
- check whether the changed framework or library is on the launch critical path.

For binary-size claims:

- compare app size artifacts;
- inspect linker maps or symbol-size reports;
- check whether static dependencies are duplicated across products.

For build-time claims:

- compare clean and incremental release builds;
- separate compile, link, and package time;
- include the cost of cross-module optimization or additional specialization.

A modularization change is successful only if it improves the targeted metric without creating unacceptable API, ABI, size, build, or ownership cost.
