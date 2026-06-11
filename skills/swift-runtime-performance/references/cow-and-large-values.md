# Copy-on-Write and Large Values

Use this reference when the task involves copy-on-write collections, large structs, repeated collection mutation, custom COW storage, uniqueness checks, or value-semantic API design.

The goal is not to avoid value types. The goal is to understand when value semantics are cheap, when they hide reference-backed storage, when mutation triggers copying, and when another ownership boundary would make performance and correctness clearer.

## Contents

- [Core model](#core-model)
- [When to use this reference](#when-to-use-this-reference)
- [Review workflow](#review-workflow)
- [Common COW-backed values](#common-cow-backed-values)
- [Mutation is the important boundary](#mutation-is-the-important-boundary)
- [Repeated collection mutation](#repeated-collection-mutation)
- [Large structs and snapshots](#large-structs-and-snapshots)
- [Intermediate collections](#intermediate-collections)
- [Custom COW storage](#custom-cow-storage)
- [`isKnownUniquelyReferenced`](#isknownuniquelyreferenced)
- [Thread safety and concurrency boundaries](#thread-safety-and-concurrency-boundaries)
- [Evidence to look for](#evidence-to-look-for)
- [Decision rules](#decision-rules)
- [Common mistakes](#common-mistakes)
- [Output guidance](#output-guidance)

## Core model

Copy-on-write, or COW, is a storage strategy where a value can share backing storage until mutation.

At the API level, the value behaves as if each copy were independent. Internally, storage may be shared while values are only read. Before mutation, the implementation checks whether the storage is uniquely referenced. If it is not unique, the implementation copies storage first.

Keep these models separate:

- Value semantics means independent observable values.
- COW means storage may be shared as an optimization.
- Assignment may copy only a small value header.
- Mutation after sharing may allocate and copy backing storage.
- COW preserves value semantics only if shared mutable storage does not leak.

## When to use this reference

Use this reference when code review or investigation involves:

- `Array`, `Dictionary`, `Set`, `String`, `Data`, or other value types with reference-backed storage;
- large structs or large state snapshots;
- repeated `map`, `filter`, `reduce`, `sorted`, `append`, `remove`, or subscript mutation in a hot path;
- copy-then-mutate APIs such as `var next = state; next.items.append(...)`;
- custom structs with private reference storage;
- `isKnownUniquelyReferenced`;
- replacing value types with classes for performance reasons;
- actor methods, tasks, or closures moving large values around.

Do not use this reference for general algorithm complexity unless copy, storage, mutation, ownership, or allocation behavior is part of the issue.

## Review workflow

1. 1Identify the value being copied, mutated, stored, or returned.
2. 2Decide whether it is plain inline storage, standard COW storage, custom reference-backed storage, or a large snapshot.
3. 3Locate the mutation boundary. COW costs usually appear at mutation after sharing, not at assignment alone.
4. 4Check for intermediate collections, repeated derived copies, and predictable capacity growth.
5. 5Check whether value semantics are required by the domain.
6. 6Propose the smallest change that preserves ownership clarity.
7. 7Recommend validation with allocation, time, SIL, benchmark, or signpost evidence.

## Common COW-backed values

Common Swift and Apple-platform value types often use reference-backed or COW-like storage internally:

- `Array`
- `Dictionary`
- `Set`
- `String`
- `Data`
- custom value types with private reference storage

Do not assume every struct is COW-backed. Most ordinary structs store their fields directly. COW appears when a type is intentionally implemented with shared reference storage.

Do not assume every value copy is expensive. For COW values, a copy may share storage until mutation. The suspicious path is usually repeated mutation, repeated buffer growth, or frequent large snapshot replacement.

## Mutation is the important boundary

COW cost often appears at mutation points.

```swift
func adding(_ item: Item, to items: [Item]) -> [Item] {
    var result = items
    result.append(item)
    return result
}
```

This shape is not automatically wrong. It becomes suspicious when `items` is large, the function runs frequently, the original value must stay alive, capacity growth repeats, or the same update could happen once at the owning boundary.

Prefer reviewing the whole pipeline rather than one assignment in isolation.

## Repeated collection mutation

Repeated copy-then-mutate patterns can create unnecessary intermediate buffers.

Risky shape:

```swift
func buildSections(from groups: [MessageGroup]) -> [Section] {
    var sections: [Section] = []

    for group in groups {
        var rows = sections.last?.rows ?? []
        rows.append(contentsOf: group.messages.map(Row.init))
        sections.append(Section(title: group.title, rows: rows))
    }

    return sections
}
```

The problem is not that arrays are bad. The problem is that the algorithm repeatedly derives a mutable value from another COW value, appends to it, and stores another copy.

Prefer building each owned value directly:

```swift
func buildSections(from groups: [MessageGroup]) -> [Section] {
    var sections: [Section] = []
    sections.reserveCapacity(groups.count)

    for group in groups {
        var rows: [Row] = []
        rows.reserveCapacity(group.messages.count)

        for message in group.messages {
            rows.append(Row(message))
        }

        sections.append(Section(title: group.title, rows: rows))
    }

    return sections
}
```

Review rule: avoid repeatedly deriving a mutable copy from an existing COW value. Prefer constructing the final value at the ownership boundary.

## Large structs and snapshots

Large structs are not automatically wrong. They are often the right model for immutable snapshots.

```swift
struct PortfolioSnapshot {
    var accounts: [AccountSummary]
    var positions: [PositionSummary]
    var alerts: [RiskAlert]
    var generatedAt: Date
}
```

This is a reasonable value model if the snapshot is created deliberately and consumed as a stable view of data. It may become expensive if every small update rebuilds, copies, publishes, or diffs the entire snapshot many times per second.

Ask:

- Is the value intended to be an immutable snapshot?
- Is only a small part changing?
- Is the whole value captured by escaping closures or tasks?
- Is the value sent across actor boundaries frequently?
- Would a page, delta, or targeted update reduce copying and invalidation?
- Would splitting state improve ownership, or only add complexity?

Do not split large values blindly. Split when it reduces real copying, recomputation, invalidation, or lifetime retention.

## Intermediate collections

Chained transformations can create intermediate arrays unless optimized away or expressed lazily.

```swift
func visibleTitles(from items: [FeedItem]) -> [String] {
    items
        .filter { $0.isVisible }
        .map { $0.title }
        .sorted()
}
```

This is fine for small or infrequent data. For large or frequent pipelines, consider whether intermediate storage matters.

A direct loop can reduce temporary storage and make capacity explicit:

```swift
func visibleTitles(from items: [FeedItem]) -> [String] {
    var titles: [String] = []
    titles.reserveCapacity(items.count)

    for item in items where item.isVisible {
        titles.append(item.title)
    }

    titles.sort()
    return titles
}
```

Do not rewrite every `map` or `filter` chain. Rewrite only when allocation, CPU, or repeated traversal is relevant.

Lazy sequences can help when only part of the result is consumed, but they are not a default performance decoration. They can be slower when consumed multiple times, when abstraction blocks optimization, or when repeated traversal performs work again.

## Custom COW storage

Custom COW is useful when a value type needs large mutable storage while preserving value semantics.

```swift
struct DocumentBuffer {
    private var storage: Storage

    var lines: [Line] { storage.lines }

    mutating func replaceLine(at index: Int, with line: Line) {
        makeUniqueStorage()
        storage.lines[index] = line
    }

    private mutating func makeUniqueStorage() {
        if !isKnownUniquelyReferenced(&storage) {
            storage = storage.copy()
        }
    }
}

private final class Storage {
    var lines: [Line]

    init(lines: [Line]) { self.lines = lines }
    func copy() -> Storage { Storage(lines: lines) }
}
```

A custom COW type should have these properties:

- storage is private;
- public API exposes values, not mutable storage identity;
- mutation goes through `mutating` methods;
- uniqueness is checked before mutation;
- copied storage preserves the intended value semantics;
- nested mutable references are copied or made immutable;
- tests cover copy, mutation, aliasing, and boundary behavior.

Do not add custom COW for small simple structs. The implementation complexity is justified only when storage size, mutation pattern, or API semantics make it worthwhile.

## `isKnownUniquelyReferenced`

Use `isKnownUniquelyReferenced` to check whether reference storage is uniquely owned before mutating it.

Correct mental model:

- It works with class instances.
- It is used through `inout` storage.
- It answers whether the object is known to have a single strong reference.
- A false result means the implementation should copy before mutation.
- It does not make storage thread-safe.
- It must be used under normal Swift exclusivity and synchronization rules.

Guardrails:

- Do not use it as synchronization.
- Do not expose the storage object and still expect value semantics.
- Do not mutate storage before uniqueness is ensured.
- Do not skip deep copy of nested mutable references.
- Do not use weak or unowned references to fake uniqueness semantics.

## Thread safety and concurrency boundaries

COW is not a synchronization mechanism. It protects value semantics under normal exclusive access rules. It does not make concurrent mutation of the same variable safe.

Returning a COW value from an actor can be a valid snapshot API:

```swift
actor MessageArchive {
    private var messages: [Message] = []

    func snapshot() -> [Message] {
        messages
    }
}
```

The caller cannot mutate the actor's stored property directly. If the caller mutates the returned array and storage is shared, COW should create separate storage.

Potential costs include frequent large snapshots, ARC traffic from repeated snapshots, buffer copy on caller mutation, tasks retaining large values, and broad UI invalidation from replacing whole snapshots.

For deep concurrency design, route to `swift-concurrency-performance` if available. Use this reference only for the value/storage cost of moving large COW values across concurrency boundaries.

## Evidence to look for

In Allocations:

- repeated array or dictionary growth;
- temporary collection creation;
- large buffer copies;
- custom storage copies;
- snapshot creation during scrolling or high-frequency updates;
- type-erased wrappers allocating around value storage.

In Time Profiler:

- retain/release traffic around collection-heavy code;
- sorting or transformation dominating the pipeline;
- bridging or conversion between collection representations;
- repeated mutation or copying near the hot path;
- expensive element copying or destruction.

In optimized SIL:

- `copy_value` and `destroy_value` around large values;
- `strong_retain` / `strong_release` around COW storage;
- `alloc_ref` for custom storage allocation;
- `partial_apply` capturing large values;
- calls that remain abstract or unspecialized in collection-heavy code.

Use SIL to explain likely cost, not as the only proof. Prefer user-visible measurements when possible.

## Decision rules

- Preserve value semantics when they match the domain.
- Optimize the mutation boundary before replacing values with references.
- Build large values once at the owner boundary when possible.
- Reserve capacity when growth is predictable and measured as relevant.
- Avoid repeated copy-then-mutate loops in hot paths.
- Keep custom COW storage private.
- Check uniqueness before mutating reference storage.
- Copy nested mutable reference state when it participates in the logical value.
- Use reference identity only when identity, sharing, or independent mutation is part of the model.
- Treat theoretical copies differently from measured hot-path costs.

## Common mistakes

- Assuming `struct` means no heap traffic.
- Assuming every value copy is expensive.
- Assuming COW is free.
- Replacing value types with classes only to avoid hypothetical copies.
- Publishing huge state snapshots for tiny high-frequency changes.
- Rebuilding intermediate collections inside nested loops.
- Adding custom COW for small simple values.
- Exposing custom COW storage.
- Using `isKnownUniquelyReferenced` as a thread-safety primitive.
- Skipping nested mutable references during custom COW copy.
- Rewriting clear collection pipelines without evidence.
- Adding unsafe buffers before checking algorithm, capacity, and ownership boundaries.

## Output guidance

When this reference is used, include:

```markdown
## COW / large-value model

Explain whether the relevant value is plain inline storage, standard COW storage, custom reference-backed storage, or a large snapshot.

## Suspected cost

Identify copy-after-sharing, buffer growth, ARC traffic, custom storage copy, intermediate collections, actor snapshotting, or broad state replacement.

## Why it matters

Tie the value behavior to the measured or likely performance problem.

## Safer alternative

Suggest the smallest change that preserves value semantics and ownership clarity.

## Validation

Recommend Allocations, Time Profiler, optimized SIL, signposts, XCTest performance tests, or a project benchmark.
```

If the concern is theoretical and the value is not in a hot path, say so directly. Do not recommend low-level rewrites without evidence.
