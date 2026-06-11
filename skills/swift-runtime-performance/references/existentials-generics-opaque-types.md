# Existentials, Generics, and Opaque Types

Use this reference when the task involves `any Protocol`, `some Protocol`, generic constraints, type erasure, protocol witness dispatch, opaque result types, heterogeneous collections, associated type relationships, or replacing existential-heavy hot paths.

The goal is not to ban existentials or force generics everywhere. The goal is to choose the abstraction that matches the design while understanding storage, dispatch, specialization, API, and validation trade-offs.

## Contents

- [Core model](#core-model)
- [Quick decision table](#quick-decision-table)
- [Concrete types](#concrete-types)
- [Generics](#generics)
- [Opaque types with `some`](#opaque-types-with-some)
- [Existentials with `any`](#existentials-with-any)
- [Type erasure wrappers](#type-erasure-wrappers)
- [`any` vs `some` vs generics](#any-vs-some-vs-generics)
- [Hot-path refactoring patterns](#hot-path-refactoring-patterns)
- [SIL and Instruments signals](#sil-and-instruments-signals)
- [Decision rules](#decision-rules)
- [Common gotchas](#common-gotchas)
- [Output guidance](#output-guidance)

## Core model

Swift has several ways to express abstraction over types:

- concrete types;
- generic parameters;
- opaque types with `some`;
- existential types with `any`;
- manual type-erasure wrappers;
- class inheritance or Objective-C dynamic dispatch.

Each answers a different design question:

- **Concrete type:** this exact type is part of the implementation or API.
- **Generic parameter:** the caller chooses one concrete type that satisfies constraints.
- **Opaque result type:** the implementation chooses one concrete type but exposes only protocol capabilities.
- **Existential type:** the concrete type is erased and can vary at runtime.
- **Type-erasure wrapper:** a concrete wrapper hides another value behind a stable API.
- **Class hierarchy:** behavior varies through inheritance and reference identity.

Do not treat these as interchangeable syntax choices. They differ in storage, dispatch, optimizer visibility, type relationships, and API evolution.

## Quick decision table


## Concrete types

Concrete types give the compiler and reader the most direct information.

Prefer concrete types when:

- the implementation is local;
- runtime substitution is not required;
- the code is performance-sensitive;
- the exact type does not create an API burden;
- abstraction does not improve testing, architecture, or replacement.

Example:

```swift
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

- Is a protocol used only because flexibility feels cleaner?
- Is this path internal enough that a concrete type is simpler?
- Is the value used in a measured hot path?
- Would the protocol boundary help testing, replacement, or architecture?
- Does exposing the concrete type leak unstable implementation detail?

## Generics

A generic parameter preserves the concrete type selected by the caller while expressing constraints through protocols.

Example:

```swift
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

Here `D` is one concrete type for a given call. The compiler may specialize the function for that concrete type when the implementation is visible enough.

Prefer generics when:

- the call path is homogeneous;
- one concrete conforming type is used per call;
- the caller should choose the concrete type;
- type relationships must be preserved;
- performance depends on specialization or inlining;
- the algorithm should work for many concrete types.

Generics are not automatically free. They help most when specialization happens and when the generic shape preserves useful type information.

Use explicit generic parameters instead of parameter-position `some` when the relationship needs a name:

```swift
func merge<S: Sequence>(
    primary: S,
    secondary: S
) -> [S.Element] {
    Array(primary) + Array(secondary)
}
```

The explicit `S` makes the same-type relationship between `primary` and `secondary` visible.

## Opaque types with `some`

An opaque type hides the concrete type from the API surface while preserving one underlying concrete type for the compiler.

In return position, `some Protocol` means the implementation chooses the concrete type.

```swift
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

- callers only need protocol capabilities;
- the implementation should hide a verbose concrete type;
- the result has one stable underlying concrete type for that declaration;
- preserving type identity matters;
- existential storage is unnecessary.

Do not use `some` when:

- the function must return unrelated concrete types from normal runtime branches;
- the value must be stored with other different conforming types;
- the API needs runtime heterogeneity;
- callers need to choose the concrete type.

Runtime choice usually belongs to `any` or a type-erasure wrapper:

```swift
func makeFormatter(kind: ExportKind) -> any ExportFormatter {
    switch kind {
    case .pdf:
        PDFFormatter()
    case .csv:
        CSVFormatter()
    }
}
```

Parameter-position `some Protocol` is usually shorthand for an implicit generic parameter. Prefer it for simple one-off constraints, and prefer explicit generics when same-type or associated-type relationships matter.

## Existentials with `any`

An existential value stores a value whose concrete type is erased at that point.

```swift
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

- erased concrete type identity;
- protocol witness dispatch;
- existential storage;
- possible boxing or indirect storage;
- pointer indirection;
- less optimizer visibility than concrete or specialized generic code.

Prefer `any Protocol` when:

- runtime heterogeneity is required;
- different concrete types must share one storage location;
- the value crosses a dependency or composition boundary;
- a plugin-like architecture needs late binding;
- the API intentionally erases implementation details;
- the code is not a measured hot path;
- the existential makes the design simpler and more stable.

Avoid unnecessary existentials when:

- the concrete type is homogeneous;
- the value is used in a tight loop;
- the abstraction is local and not needed;
- associated type relationships are erased and later reconstructed with casts;
- optimized SIL or Instruments shows boxing, witness calls, allocation, or ARC traffic as relevant cost.

Constrained existentials are useful when existential storage is needed but some type information should remain visible:

```swift
func describeStrings(_ values: any Sequence<String>) -> [String] {
    values.map { "Value: \($0)" }
}
```

Use constrained existentials when runtime erasure is useful but erasing all type information would make the API weaker.

## Type erasure wrappers

Manual type erasure wraps an underlying value in a stable concrete type.

```swift
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

- public API should expose one concrete wrapper;
- stored properties need a stable nominal type;
- associated type relationships should be preserved through wrapper generics;
- the implementation needs custom forwarding, caching, cancellation, or lifecycle behavior;
- source compatibility depends on an existing wrapper.

Type erasure is not automatically faster than `any`. It often uses closures, boxes, references, forwarding, or ARC traffic. Treat it as an API design tool, not a default performance optimization.

Prefer language existentials when the wrapper only forwards protocol calls and provides no additional behavior.

## `any` vs `some` vs generics

Use this compact model:

- `some Protocol`: the implementation chooses one hidden concrete type.
- `any Protocol`: the concrete type is intentionally dynamic or erased.
- `<T: Protocol>`: the caller chooses one concrete type and type relationships can be named.
- `AnyProtocol` wrapper: a nominal container provides API stability or custom behavior.

Common replacements:

- Replace `any` with a generic when the path is homogeneous and hot.
- Replace `any` with `some` when the implementation returns one stable hidden type.
- Replace a forwarding `Any...` wrapper with `any` when the wrapper adds no behavior.
- Keep `any` when runtime heterogeneity is the design.
- Keep generics when associated type relationships are central.

Do not rewrite abstraction style only because one form looks more modern.

## Hot-path refactoring patterns

### Move existential dispatch out of the inner loop

```swift
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

```swift
let modules: [any FeedModule] = [
    NewsModule(),
    AdsModule(),
    RecommendationsModule()
]
```

This is fine for screen-level composition. If the inner rendering loop repeatedly opens existential values, consider moving the dynamic boundary outward and keeping the repeated work concrete or generic.

### Preserve associated type relationships

```swift
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

## SIL and Instruments signals

When source-level reasoning is not enough, inspect optimized SIL.

Look for:

- `init_existential_*`: existential container creation;
- `open_existential_*`: opening an existential;
- `witness_method`: protocol witness dispatch;
- `partial_apply`: closure creation or type-erased forwarding;
- `alloc_box`: boxed captured state;
- `alloc_ref`: class or box allocation;
- `strong_retain` / `strong_release`: ARC traffic;
- generic code that remains unspecialized in a hot path;
- specialized function names or specialization attributes.

In Allocations, look for repeated existential boxes, type-erasure wrapper objects, closure contexts from `Any...` wrappers, and allocation spikes from converting concrete collections to existential collections.

In Time Profiler, look for protocol witness dispatch, forwarding through wrappers, small methods that did not inline, retain/release traffic around erased values, or hot sorting/mapping/rendering loops that call protocol requirements.

Use SIL and Instruments to explain measured behavior, not to justify speculative rewrites by themselves.

## Decision rules

### If you see `any Protocol`

Ask:

- Is runtime heterogeneity required?
- Is the value stored, passed briefly, or used inside a loop?
- Is the concrete type actually homogeneous?
- Does the code rely on associated type relationships?
- Would a generic, opaque type, or concrete type preserve the design?
- Is boxing or witness dispatch visible in Instruments or optimized SIL?

Recommendation style:

- Keep `any` when it expresses a real dynamic boundary.
- Replace it only when the path is homogeneous and performance-sensitive.
- Consider constrained existentials when some type information should remain visible.

### If you see `some Protocol`

Ask:

- Is the underlying type stable for this declaration?
- Is the implementation trying to return unrelated types from runtime branches?
- Would callers need to store heterogeneous values?
- Is this return-position opacity or parameter-position generic shorthand?
- Does `some` hide implementation detail without weakening the API?

Recommendation style:

- Keep `some` when the implementation chooses one hidden concrete type.
- Use `any` or type erasure when concrete type must vary at runtime.
- Use explicit generics when type relationships need names.

### If you see a generic function

Ask:

- Does the generic parameter preserve an important type relationship?
- Is the function internal or public across a module boundary?
- Does specialization happen in optimized SIL?
- Could many concrete instantiations increase code size?
- Would a concrete type be simpler for a local hot path?

Recommendation style:

- Keep generics for reusable homogeneous behavior.
- Use concrete types where abstraction adds no value.
- Use `@inlinable` only after identifying a real cross-module optimization issue.

### If you see type erasure

Ask:

- Is the wrapper needed for API stability, storage, or behavior?
- Does it preserve associated type relationships?
- Does it store escaping closures?
- Does it allocate or retain unexpectedly?
- Would `any Protocol` be simpler in modern Swift?
- Is the wrapper used in a hot path?

Recommendation style:

- Keep type erasure when it provides a meaningful wrapper or behavior.
- Prefer `any` when the wrapper only forwards protocol calls.
- Prefer generics or concrete types inside hot implementation paths.

## Common gotchas

- `any Protocol` is an existential type, not a generic constraint.
- `some Protocol` is not the same as `any Protocol`.
- `some` preserves one concrete underlying type; `any` erases it.
- Generics help most when specialization happens.
- Existentials may allocate, but they do not always allocate.
- Type erasure wrappers can allocate or capture closures.
- `some` cannot express ordinary runtime choice between unrelated return types.
- `any` is often correct at dependency, plugin, and composition boundaries.
- Parameter-position `some` is usually an implicit generic, not existential storage.
- Associated type relationships are often clearer with generics.
- Constrained existentials can be better than erasing all type information.
- Cross-module boundaries can limit specialization.
- `@inlinable` is an API and ABI commitment, not a default performance fix.
- Changing abstraction style can affect API stability, code size, and maintainability.

## Output guidance

When this reference is used, include:

```markdown
## Abstraction model

State whether the code uses concrete types, generics, `some`, `any`, type erasure, or inheritance.

## Runtime cost

Explain possible costs: witness dispatch, existential storage, boxing, closure capture, ARC, missed specialization, or code size.

## Design reason

Explain what flexibility, storage model, or type relationship the current abstraction provides.

## Recommendation

Suggest whether to keep the abstraction or change it to concrete, generic, opaque, existential, or type-erased form.

## Validation

Recommend optimized SIL, Time Profiler, Allocations, a benchmark, or code-size inspection.
```

If the abstraction is not in a hot path and expresses the design well, say so and avoid rewriting it only for theoretical performance.
