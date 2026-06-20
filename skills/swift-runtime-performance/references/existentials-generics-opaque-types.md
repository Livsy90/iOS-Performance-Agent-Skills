# Existentials, Generics, and Opaque Types

Use this reference when the task involves `any Protocol`, `some Protocol`, generic constraints, type erasure, protocol witness dispatch, opaque result types, heterogeneous collections, associated type relationships, or replacing existential-heavy hot paths.

The goal is not to ban existentials or force generics everywhere. The goal is to choose the abstraction that matches the design while understanding storage, dispatch, specialization, API, and validation trade-offs.

Do not rewrite abstraction style for performance unless the path is hot, the abstraction is unnecessary for the design, and profiling, optimized SIL, or a benchmark suggests that storage, dispatch, allocation, ARC, or missed specialization matters.

## Contents

* [Core model](#core-model)
* [When not to use this reference](#when-not-to-use-this-reference)
* [Quick decision table](#quick-decision-table)
* [Concrete types](#concrete-types)
* [Generics](#generics)
* [Opaque types with `some`](#opaque-types-with-some)
* [Existentials with `any`](#existentials-with-any)
* [Type erasure wrappers](#type-erasure-wrappers)
* [`any` vs `some` vs generics](#any-vs-some-vs-generics)
* [Hot-path refactoring patterns](#hot-path-refactoring-patterns)
* [SIL and Instruments signals](#sil-and-instruments-signals)
* [Decision rules](#decision-rules)
* [Common gotchas](#common-gotchas)
* [Output guidance](#output-guidance)

## Core model

Swift has several ways to express abstraction over types:

* concrete types;
* generic parameters;
* opaque types with `some`;
* existential types with `any`;
* manual type-erasure wrappers;
* class inheritance or Objective-C dynamic dispatch.

Each answers a different design question:

* **Concrete type:** this exact type is part of the implementation or API.
* **Generic parameter:** the caller chooses one concrete type that satisfies constraints.
* **Opaque result type:** the implementation chooses one concrete type but exposes only protocol capabilities.
* **Opaque parameter type:** the caller chooses the concrete type through an unnamed generic parameter.
* **Existential type:** the concrete type is erased and can vary at runtime.
* **Type-erasure wrapper:** a concrete wrapper hides another value behind a stable API.
* **Class hierarchy:** behavior varies through inheritance and reference identity.

Do not treat these as interchangeable syntax choices. They differ in storage, dispatch, optimizer visibility, type relationships, API evolution, source compatibility, and code size.

A good review asks:

1. Who chooses the concrete type: the caller, the implementation, or runtime composition?
2. Is runtime heterogeneity required?
3. Are associated type or same-type relationships important?
4. Is the abstraction local, public API, or a module boundary?
5. Is this path hot enough for dispatch, storage, or specialization to matter?
6. Is there evidence from profiling, optimized SIL, allocation data, or benchmarks?

## When not to use this reference

Do not use this reference as the primary source when:

* the issue is only method dispatch, inlining, devirtualization, witness dispatch, or module-boundary specialization — use `dispatch-and-specialization.md`;
* the issue is existential storage layout, closure boxes, object layout, or unexpected heap allocation — use `allocation-and-layout.md`;
* the issue is retain/release traffic, capture lifetime, reference cycles, weak/unowned ownership, or Objective-C bridging lifetime — use `arc-and-ownership.md`;
* the issue is collection mutation, COW buffers, large snapshots, or copy-after-sharing behavior — use `cow-and-large-values.md`;
* the question is general API design with no performance, storage, or abstraction trade-off.

If one of these files does not exist yet, treat the link as intended routing and do not invent details from a missing reference.

## Quick decision table

| Need                                                          | Prefer                                        | Why                                                       |
| ------------------------------------------------------------- | --------------------------------------------- | --------------------------------------------------------- |
| One local implementation, no runtime substitution             | Concrete type                                 | Simplest model and best optimizer visibility              |
| Caller chooses one concrete type per call                     | Generic `<T: Protocol>`                       | Preserves concrete type and type relationships            |
| Implementation returns one hidden concrete type               | `some Protocol` in return position            | Hides concrete type while preserving concrete identity    |
| Parameter has a simple one-off constraint                     | `some Protocol` in parameter position         | Shorthand for an unnamed generic parameter                |
| Multiple parameters must share the same concrete type         | Explicit generic parameter                    | Gives the relationship a name                             |
| Runtime heterogeneity is required                             | `any Protocol`                                | Concrete type can vary at runtime                         |
| Stable nominal wrapper or extra forwarding behavior is needed | Type-erasure wrapper                          | Provides API shape or behavior beyond a plain existential |
| Associated type relationships must be preserved               | Explicit generics or constrained existentials | Keeps important type information visible                  |
| Hot homogeneous inner loop                                    | Concrete or generic path                      | Allows specialization and inlining when visible           |
| Public API wants to hide a verbose implementation type        | Opaque result type or wrapper                 | Hides implementation without exposing unstable detail     |
| Plugin, dependency, or screen composition boundary            | `any Protocol` or type-erasure wrapper        | Late binding and heterogeneity may be the design          |

Use this table as a routing aid, not as a mechanical rewrite rule.

## Concrete types

Concrete types give the compiler and reader the most direct information.

Prefer concrete types when:

* the implementation is local;
* runtime substitution is not required;
* the code is performance-sensitive;
* the exact type does not create an API burden;
* abstraction does not improve testing, architecture, replacement, or source stability.

Example:

```swift id="aaj076"
struct JSONMessageDecoder {
    func decode(_ data: Data) throws -> Message {
        try JSONDecoder().decode(Message.self, from: data)
    }
}

func loadMessages(
    data: [Data],
    decoder: JSONMessageDecoder
) throws -> [Message] {
    try data.map { try decoder.decode($0) }
}
```

This is appropriate when there is only one local decoding strategy. Introducing a protocol may add indirection without solving a design problem.

Review questions:

* Is a protocol used only because flexibility feels cleaner?
* Is this path internal enough that a concrete type is simpler?
* Is the value used in a measured hot path?
* Would the protocol boundary help testing, replacement, or architecture?
* Does exposing the concrete type leak unstable implementation detail?

Do not force concrete types across public API boundaries when abstraction is part of the design.

## Generics

A generic parameter preserves the concrete type selected by the caller while expressing constraints through protocols.

Example:

```swift id="nstco3"
protocol MessageDecoder {
    func decode(_ data: Data) throws -> Message
}

func decodeBatch<D: MessageDecoder>(
    _ batch: [Data],
    using decoder: D
) throws -> [Message] {
    try batch.map { try decoder.decode($0) }
}
```

Here `D` is one concrete type for a given call. The compiler may specialize the function for that concrete type when the implementation is visible enough and specialization is considered profitable.

Prefer generics when:

* the call path is homogeneous;
* one concrete conforming type is used per call;
* the caller should choose the concrete type;
* type relationships must be preserved;
* associated types or same-type relationships matter;
* performance depends on specialization or inlining;
* the algorithm should work for many concrete types.

Generics are not automatically free. They help most when specialization happens and when the generic shape preserves useful type information.

Use explicit generic parameters instead of parameter-position `some` when the relationship needs a name:

```swift id="eagjuq"
func merge<S: Sequence>(
    primary: S,
    secondary: S
) -> [S.Element] {
    Array(primary) + Array(secondary)
}
```

The explicit `S` makes the same-type relationship between `primary` and `secondary` visible.

Two separate parameter-position `some Protocol` parameters introduce separate unnamed generic parameters. They do not automatically mean "the same concrete type":

```swift id="3aqbqr"
func compare(
    lhs: some Sequence<Int>,
    rhs: some Sequence<Int>
) {
    // lhs and rhs may be different concrete sequence types.
}
```

Use explicit generics when multiple parameters must share the same type, when associated type relationships must be named, or when the type appears in the return value.

## Opaque types with `some`

An opaque type hides the concrete type from the API surface while preserving an underlying concrete type for the compiler.

In return position, `some Protocol` means the implementation chooses the concrete type.

```swift id="fo3g6y"
protocol TimelineRenderer {
    func render(_ item: TimelineItem) -> RenderedRow
}

struct CompactTimelineRenderer: TimelineRenderer {
    func render(_ item: TimelineItem) -> RenderedRow {
        RenderedRow(title: item.title, subtitle: item.subtitle)
    }
}

func makeTimelineRenderer() -> some TimelineRenderer {
    CompactTimelineRenderer()
}
```

Prefer opaque result types when:

* callers only need protocol capabilities;
* the implementation should hide a verbose concrete type;
* the result has one stable underlying concrete type for that declaration;
* preserving concrete type identity matters;
* existential storage is unnecessary;
* the API wants abstraction without runtime heterogeneity.

Do not use return-position `some` when:

* the function must directly return unrelated concrete types from ordinary runtime branches;
* the value must be stored with other different conforming types;
* the API needs runtime heterogeneity;
* callers need to choose the concrete type.

A plain `some Protocol` return type cannot directly express ordinary runtime choice between unrelated concrete return types:

```swift id="e61fdp"
func makeFormatter(kind: ExportKind) -> some ExportFormatter {
    switch kind {
    case .pdf:
        PDFFormatter()
    case .csv:
        CSVFormatter()
    }
}
```

Use `any`, a type-erasure wrapper, a common concrete wrapper such as an enum, or a result-builder-based API when that is the intended model.

Runtime choice often belongs to `any` or a type-erasure wrapper:

```swift id="7vzhzt"
func makeFormatter(kind: ExportKind) -> any ExportFormatter {
    switch kind {
    case .pdf:
        PDFFormatter()
    case .csv:
        CSVFormatter()
    }
}
```

Parameter-position `some Protocol` is shorthand for an unnamed generic parameter. The caller supplies the concrete type for that parameter.

Prefer parameter-position `some` for simple one-off constraints:

```swift id="855bsc"
func render(_ renderer: some TimelineRenderer, item: TimelineItem) -> RenderedRow {
    renderer.render(item)
}
```

Prefer explicit generics when same-type or associated-type relationships matter.

## Existentials with `any`

An existential value stores a value whose concrete type is erased at that point.

```swift id="fmfx28"
protocol NotificationChannel {
    func send(_ message: NotificationMessage) async throws
}

let channels: [any NotificationChannel] = [
    EmailChannel(),
    PushChannel()
]
```

The array intentionally stores different concrete types. This is a good use of `any`.

Existentials may involve:

* erased concrete type identity;
* metadata and witness tables;
* protocol witness dispatch;
* inline existential storage;
* possible boxing or indirect storage;
* pointer indirection;
* ARC traffic;
* less optimizer visibility than concrete or specialized generic code.

Not every existential allocates. Small values may fit inline in the existential container. Larger or address-only values may require indirect storage. The exact representation is an implementation detail and should be validated when the performance claim depends on it.

Prefer `any Protocol` when:

* runtime heterogeneity is required;
* different concrete types must share one storage location;
* the value crosses a dependency or composition boundary;
* a plugin-like architecture needs late binding;
* the API intentionally erases implementation details;
* the code is not a measured hot path;
* the existential makes the design simpler and more stable.

Avoid unnecessary existentials when:

* the concrete type is homogeneous;
* the value is used in a tight loop;
* the abstraction is local and not needed;
* associated type relationships are erased and later reconstructed with casts;
* optimized SIL or Instruments shows boxing, witness calls, allocation, pointer indirection, or ARC traffic as relevant cost.

Constrained existentials are useful when existential storage is needed but some primary associated type information should remain visible:

```swift id="0a68l4"
func describeStrings(_ values: any Collection<String>) -> [String] {
    values.map { "Value: \($0)" }
}
```

Use constrained existentials when runtime erasure is useful but erasing all type information would make the API weaker.

Constrained existential syntax depends on the language version, deployment target, and whether the protocol exposes the relevant primary associated type relationships. When support is unavailable or the relationship is more complex, prefer explicit generics.

## Type erasure wrappers

Manual type erasure wraps an underlying value in a stable concrete type.

```swift id="6oggjz"
struct AnyMessageHandler<Message> {
    private let _handle: (Message) async throws -> Void

    init<H: MessageHandler>(_ handler: H) where H.Message == Message {
        self._handle = handler.handle
    }

    func handle(_ message: Message) async throws {
        try await _handle(message)
    }
}
```

Type erasure can be useful when:

* public API should expose one concrete wrapper;
* stored properties need a stable nominal type;
* associated type relationships should be preserved through wrapper generics;
* the implementation needs custom forwarding, caching, cancellation, `Sendable`, equality, hashing, or lifecycle behavior;
* source or binary compatibility depends on an existing wrapper;
* the wrapper provides useful conformances or behavior beyond forwarding.

Type erasure is not automatically faster than `any`. It often uses closures, boxes, references, forwarding, or ARC traffic. Capturing `handler.handle` may retain or copy `handler` depending on the underlying type and representation.

Treat type erasure as an API design tool, not a default performance optimization.

Prefer language existentials when the wrapper only forwards protocol calls and provides no additional behavior. Do this only when source compatibility, binary compatibility, custom conformances, conditional conformances, `Sendable` behavior, or additional wrapper behavior are not part of the wrapper’s purpose.

## `any` vs `some` vs generics

Use this compact model:

* `some Protocol` in return position: the implementation chooses one hidden concrete type.
* `some Protocol` in parameter position: the caller chooses a concrete type through an unnamed generic parameter.
* `any Protocol`: the concrete type is intentionally dynamic or erased.
* `<T: Protocol>`: the caller chooses one concrete type and type relationships can be named.
* `AnyProtocol` wrapper: a nominal container provides API stability or custom behavior.

Common replacements:

* Replace `any` with a generic when the path is homogeneous and hot.
* Replace `any` with return-position `some` when the implementation returns one stable hidden type.
* Replace parameter-position `some` with explicit generics when type relationships need names.
* Replace a forwarding `Any...` wrapper with `any` when the wrapper adds no behavior and compatibility permits it.
* Keep `any` when runtime heterogeneity is the design.
* Keep generics when associated type relationships are central.
* Keep type erasure when a stable wrapper, custom behavior, or compatibility requirement matters.

Do not rewrite abstraction style only because one form looks more modern.

## Hot-path refactoring patterns

### Move existential dispatch out of the inner loop

```swift id="cj1fmy"
func rank<R: ScoreRule>(
    candidates: [Candidate],
    rule: R
) -> [Candidate] {
    candidates.sorted {
        rule.score($0) > rule.score($1)
    }
}
```

Prefer this shape only when ranking is hot and `rule` is one concrete type for the call. Keep `any ScoreRule` if ranking intentionally accepts runtime-varying rules at a composition boundary.

### Keep dynamic composition at the edge

```swift id="gm86a7"
let modules: [any FeedModule] = [
    NewsModule(),
    AdsModule(),
    RecommendationsModule()
]
```

This is fine for screen-level composition. If the inner rendering loop repeatedly opens existential values, consider moving the dynamic boundary outward and keeping the repeated work concrete or generic.

### Preserve associated type relationships

```swift id="fo5ipl"
protocol SnapshotProvider {
    associatedtype Snapshot

    func snapshot() -> Snapshot
    func restore(_ snapshot: Snapshot)
}

func roundTrip<P: SnapshotProvider>(_ provider: P) {
    let snapshot = provider.snapshot()
    provider.restore(snapshot)
}
```

The generic version guarantees that `restore` receives exactly the snapshot type produced by the same provider.

### Use an enum when runtime choice is finite and local

```swift id="iq0iv8"
enum ExportFormatter {
    case pdf(PDFFormatter)
    case csv(CSVFormatter)

    func format(_ document: Document) throws -> Data {
        switch self {
        case .pdf(let formatter):
            try formatter.format(document)
        case .csv(let formatter):
            try formatter.format(document)
        }
    }
}
```

This can preserve a concrete storage model while still representing runtime choice. Use it only when the set of cases is finite and owned by the module. Use `any` or type erasure for open plugin-style extension.

## SIL and Instruments signals

When source-level reasoning is not enough, inspect optimized SIL.

Look for:

* `init_existential_*`: existential container creation;
* `open_existential_*`: opening an existential;
* `witness_method`: protocol witness dispatch;
* `partial_apply`: closure creation or type-erased forwarding;
* `alloc_box`: boxed captured state;
* `alloc_ref`: class or box allocation;
* `strong_retain` / `strong_release`: ARC traffic;
* `retain_value` / `release_value`: ownership traffic;
* generic code that remains unspecialized in a hot path;
* specialized function names or specialization attributes.

Treat SIL instruction names as implementation-level signals, not stable API. Exact lowering can change across compiler versions, optimization levels, build settings, language modes, ownership modes, and target platforms.

In Allocations, look for repeated existential boxes, type-erasure wrapper objects, closure contexts from `Any...` wrappers, and allocation spikes from converting concrete collections to existential collections.

In Time Profiler, look for protocol witness dispatch, forwarding through wrappers, small methods that did not inline, retain/release traffic around erased values, or hot sorting, mapping, rendering, parsing, or ranking loops that call protocol requirements.

Use SIL and Instruments to explain measured behavior, not to justify speculative rewrites by themselves.

## Decision rules

### If you see `any Protocol`

Ask:

* Is runtime heterogeneity required?
* Is the value stored, passed briefly, or used inside a loop?
* Is the concrete type actually homogeneous?
* Does the code rely on associated type relationships?
* Would a generic, opaque type, or concrete type preserve the design?
* Is boxing, witness dispatch, allocation, ARC traffic, or missed specialization visible in Instruments or optimized SIL?

Recommendation style:

* Keep `any` when it expresses a real dynamic boundary.
* Replace it only when the path is homogeneous and performance-sensitive.
* Consider constrained existentials when some type information should remain visible.
* Move dynamic composition outward when only the inner loop needs optimization.

### If you see `some Protocol`

Ask:

* Is this return-position opacity or parameter-position generic shorthand?
* For return position, is the underlying type stable for this declaration?
* Is the implementation trying to directly return unrelated concrete types from ordinary runtime branches?
* Would callers need to store heterogeneous values?
* For parameter position, do multiple values need to share the same concrete type?
* Does `some` hide implementation detail without weakening the API?

Recommendation style:

* Keep return-position `some` when the implementation chooses one hidden concrete type.
* Use `any`, an enum, a result builder, or type erasure when concrete type must vary at runtime.
* Use explicit generics when type relationships need names.

### If you see a generic function

Ask:

* Does the generic parameter preserve an important type relationship?
* Is the function internal or public across a module boundary?
* Does specialization happen in optimized SIL?
* Could many concrete instantiations increase code size?
* Would a concrete type be simpler for a local hot path?
* Is `@inlinable` being considered only because of a real cross-module optimization issue?

Recommendation style:

* Keep generics for reusable homogeneous behavior.
* Use concrete types where abstraction adds no value.
* Use `@inlinable` only after identifying a real cross-module optimization issue and accepting the API/ABI trade-off.

### If you see type erasure

Ask:

* Is the wrapper needed for API stability, storage, or behavior?
* Does it preserve associated type relationships?
* Does it store escaping closures?
* Does it allocate or retain unexpectedly?
* Would `any Protocol` be simpler in modern Swift?
* Is the wrapper used in a hot path?
* Does the wrapper provide conformances, compatibility, or lifecycle behavior that a plain existential would not?

Recommendation style:

* Keep type erasure when it provides a meaningful wrapper or behavior.
* Prefer `any` when the wrapper only forwards protocol calls and compatibility permits the change.
* Prefer generics or concrete types inside hot implementation paths.

## Common gotchas

* `any Protocol` is an existential type, not a generic constraint.
* `some Protocol` is not the same as `any Protocol`.
* Return-position `some` preserves one hidden concrete underlying type.
* Parameter-position `some` is an unnamed generic parameter.
* Two separate parameter-position `some` parameters do not automatically share the same concrete type.
* Generics help most when specialization happens.
* Existentials may allocate, but they do not always allocate.
* Type erasure wrappers can allocate or capture closures.
* A plain `some` return type cannot directly express ordinary runtime choice between unrelated concrete return types.
* `any` is often correct at dependency, plugin, and composition boundaries.
* Associated type relationships are often clearer with generics.
* Constrained existentials can be better than erasing all type information, when supported.
* Cross-module boundaries can limit specialization.
* `@inlinable` is an API and ABI commitment, not a default performance fix.
* Changing abstraction style can affect API stability, code size, source compatibility, binary compatibility, and maintainability.

## Output guidance

When this reference is used, include:

```markdown id="obyavz"
## Abstraction model

State whether the code uses concrete types, generics, return-position `some`, parameter-position `some`, `any`, type erasure, or inheritance.

## Runtime cost

Explain possible costs: witness dispatch, existential storage, boxing, closure capture, ARC, missed specialization, code size, or module-boundary limits.

## Design reason

Explain what flexibility, storage model, type relationship, compatibility, or runtime heterogeneity the current abstraction provides.

## Recommendation

Suggest whether to keep the abstraction or change it to concrete, generic, opaque, existential, enum-backed, or type-erased form.

## Trade-offs

Call out API stability, runtime heterogeneity, source compatibility, binary compatibility, code size, optimizer visibility, and maintainability.

## Validation

Recommend optimized SIL, Time Profiler, Allocations, a benchmark, code-size inspection, or before/after trace.
```

If the abstraction is not in a hot path and expresses the design well, say so and avoid rewriting it only for theoretical performance.
