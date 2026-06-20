# Unsafe Swift

Use this reference when the task involves unsafe pointers, buffer access, memory binding, alignment, aliasing, manual lifetime, unsafe wrappers, C or Objective-C interop, or replacing safe APIs with unsafe code.

Unsafe Swift is not a default performance strategy. Treat it as a narrow boundary that must be justified, documented, tested, and hidden behind safe APIs whenever possible.

Use this file only when unsafe constructs are already present or are being proposed.

## Contents

* [Core principle](#core-principle)
* [When unsafe code is justified](#when-unsafe-code-is-justified)
* [When not to use this reference](#when-not-to-use-this-reference)
* [When to push back](#when-to-push-back)
* [Safety dimensions](#safety-dimensions)
* [Pointer families](#pointer-families)
* [Lifetime of `withUnsafe...` APIs](#lifetime-of-withunsafe-apis)
* [Memory binding](#memory-binding)
* [Alignment, layout, and external bytes](#alignment-layout-and-external-bytes)
* [Manual allocation](#manual-allocation)
* [`unsafeBitCast`](#unsafebitcast)
* [`Unmanaged`](#unmanaged)
* [`unowned(unsafe)`](#unownedunsafe)
* [`nonisolated(unsafe)` and `@unchecked Sendable`](#nonisolatedunsafe-and-unchecked-sendable)
* [C, C++, Objective-C, and Core Foundation boundaries](#c-c-objective-c-and-core-foundation-boundaries)
* [Safe wrapper shape](#safe-wrapper-shape)
* [Modern safer memory tools](#modern-safer-memory-tools)
* [Review workflow](#review-workflow)
* [Validation](#validation)
* [Checklist](#checklist)
* [Common gotchas](#common-gotchas)
* [Output guidance](#output-guidance)

## Core principle

Unsafe code should be a boundary, not a style.

Use this order:

1. Start from the safe implementation.
2. Identify the measured cost or required interop contract.
3. Isolate the unsafe operation.
4. Document the invariants that make it correct.
5. Expose a safe API to callers.
6. Validate correctness before claiming a performance win.

Do not recommend unsafe code before considering:

* algorithmic changes;
* data layout changes;
* borrowing or in-place APIs;
* standard library APIs;
* explicit parsing;
* specialization;
* batching;
* safe wrappers;
* safer non-escapable memory views when available.

Unsafe code can hide aliasing, lifetime, or exclusivity information from the optimizer. It is not automatically faster than safe Swift.

If unsafe code is proposed only because optimized SIL still shows overhead, route to `sil-inspection.md` and profiling first. SIL overhead alone is not enough to justify unsafe code.

## When unsafe code is justified

Unsafe constructs may be justified when:

* a safe API cannot express the required operation;
* C, C++, Objective-C, or Core Foundation interop requires pointer or ownership control;
* a hot parser, codec, graphics, crypto, storage, or networking path needs controlled buffer access;
* profiling shows the safe abstraction is a real bottleneck;
* layout, lifetime, alignment, binding, aliasing, and ownership assumptions are documented;
* invalid inputs are rejected before unsafe access;
* tests cover bounds, malformed input, cleanup, and concurrency assumptions.

If unsafe code is introduced for performance, require measurement. If it is introduced for interop or representation, require a correctness contract.

## When not to use this reference

Do not use this reference as the primary source when:

* the issue is ordinary ARC lifetime, weak/unowned ownership, retain/release traffic, or reference cycles — use `arc-and-ownership.md`;
* the issue is allocation shape, closure boxes, existential boxes, or stack vs heap storage — use `allocation-and-layout.md`;
* the issue is COW mutation, large values, collection buffers, or copy-after-sharing behavior — use `cow-and-large-values.md`;
* the issue is dispatch, generics, specialization, devirtualization, or inlining — use `dispatch-and-specialization.md`;
* the issue is actor isolation, `Sendable` design, task cancellation, executor behavior, or `MainActor` responsiveness — use `swift-concurrency-performance`;
* the issue is only that SIL still contains overhead — use `sil-inspection.md` plus measurement first.

If one of these files does not exist yet, treat the link as intended routing and do not invent details from a missing reference.

## When to push back

Push back when unsafe code is used to:

* avoid a safe initializer or conversion API;
* reinterpret file, network, or IPC bytes as Swift structs without validating layout;
* store a pointer from a temporary `withUnsafe...` closure;
* bypass `Sendable`, actor isolation, or exclusivity without synchronization;
* skip bounds checks for untrusted input;
* avoid ARC without an explicit ownership-transfer contract;
* spread pointer handling through high-level app code;
* make a speculative optimization with no hot path evidence;
* silence compiler diagnostics without documenting the invariant that makes the code safe.

## Safety dimensions

Classify every unsafe review by the assumption that can fail.

### Lifetime safety

Ask whether every access happens while the memory is valid.

Red flags:

* pointer escapes a `withUnsafe...` closure;
* pointer is stored after the owner can deallocate or reallocate;
* buffer pointer is used after collection mutation;
* C API stores a pointer longer than Swift expects;
* `unowned(unsafe)` can outlive the referenced object;
* cancellation skips cleanup for memory or external resources.

### Bounds safety

Ask whether every read and write stays inside the valid allocation.

Red flags:

* byte count is confused with element count;
* `capacity` is treated as initialized count;
* pointer arithmetic can cross the buffer end;
* null termination is assumed without checking;
* external length fields are trusted blindly;
* integer overflow is possible when computing offsets, counts, or nested lengths.

### Type and binding safety

Ask whether memory is accessed as a type it is actually bound to and initialized as.

Red flags:

* `assumingMemoryBound(to:)` is used to make memory become a type;
* arbitrary bytes are loaded as a Swift struct;
* `unsafeBitCast` is used as a semantic conversion;
* memory is rebound outside a valid temporary scope;
* a typed pointer is used after memory was rebound to another unrelated type.

### Initialization safety

Ask whether memory is initialized before reads and cleaned up correctly.

Red flags:

* reading uninitialized memory;
* double-initializing typed storage;
* deallocating initialized storage without deinitializing when needed;
* reporting the wrong initialized count from uninitialized-capacity APIs;
* partially initialized buffers are not cleaned up on failure.

### Alignment and layout safety

Ask whether the address satisfies the alignment and layout requirements of the type.

Red flags:

* data comes from `Data`, files, sockets, compression buffers, memory maps, or packed structs;
* raw bytes are loaded as multi-byte values without considering alignment;
* Swift struct layout is treated as a portable binary format;
* padding, endianness, or versioning is ignored;
* external formats are assumed to match the current platform ABI.

### Thread safety

Ask whether shared mutable memory is synchronized.

Red flags:

* raw or typed mutable pointers cross task or thread boundaries;
* `@unchecked Sendable` wraps unsafe storage without synchronization;
* `nonisolated(unsafe)` exposes mutable state;
* actor-isolated storage leaks as an unsafe pointer;
* C callbacks access Swift state concurrently;
* cancellation or teardown can race with pointer use.

## Pointer families

### Typed pointers

Examples: `UnsafePointer<T>`, `UnsafeMutablePointer<T>`, `UnsafeBufferPointer<T>`, `UnsafeMutableBufferPointer<T>`.

Review questions:

* Is the memory bound to `T`?
* Are the elements initialized?
* Is capacity measured in elements, not bytes?
* Is mutation exclusive?
* Does the pointer remain valid for the whole access?
* Can the pointer escape its intended lifetime?

### Raw pointers

Examples: `UnsafeRawPointer`, `UnsafeMutableRawPointer`, `UnsafeRawBufferPointer`, `UnsafeMutableRawBufferPointer`.

Review questions:

* Is the operation truly byte-oriented?
* If typed values are loaded, are alignment and layout requirements satisfied?
* Is memory binding handled correctly?
* Is unaligned input possible?
* Is external binary data parsed explicitly rather than reinterpreted?
* Does byte-level access avoid typed aliasing violations?

### Buffer pointers

Buffer pointers combine a base address and a count.

Review questions:

* Is `count` bytes or elements?
* Can `baseAddress` be nil for an empty buffer?
* Are indexes checked?
* Is the buffer used only during its valid lifetime?
* Is the buffer mutated while another view exists?
* Is the initialized element count known separately from capacity?

Do not treat `UnsafeBufferPointer` like `Array`. Its API shape may look collection-like, but the safety contract still depends on valid lifetime, correct count, initialized memory, binding, and no pointer escape.

## Lifetime of `withUnsafe...` APIs

Pointers produced by `withUnsafePointer`, `withUnsafeMutablePointer`, `withUnsafeBytes`, `withUnsafeBufferPointer`, and similar APIs are generally valid only during the closure unless the API explicitly documents otherwise.

Avoid:

```swift id="m0u60z"
func leakedPointer(from data: Data) -> UnsafeRawPointer? {
    data.withUnsafeBytes { buffer in
        buffer.baseAddress
    }
}
```

Prefer doing the work inside the closure:

```swift id="zy97hn"
enum ByteReadError: Error {
    case empty
}

func firstByte(in data: Data) throws -> UInt8 {
    try data.withUnsafeBytes { buffer in
        guard buffer.count > 0 else {
            throw ByteReadError.empty
        }

        return buffer[0]
    }
}
```

Review rule: do not let unsafe pointers escape `withUnsafe...` closures unless ownership transfer or lifetime extension is explicitly guaranteed by the API contract.

If a C API stores the pointer beyond the call, a closure-scoped pointer is usually not enough. The code needs an explicit owner for the memory and a matching cleanup path.

## Memory binding

Memory binding tells Swift what type a memory region may be accessed as.

Common operations:

* `bindMemory(to:capacity:)` binds raw memory to a type.
* `assumingMemoryBound(to:)` assumes memory is already bound to a type.
* `withMemoryRebound(to:capacity:_:)` temporarily accesses memory through another compatible type during a closure.

Decision rules:

* Do not use `assumingMemoryBound(to:)` to bind memory.
* Do not use `withMemoryRebound` for arbitrary type punning.
* Do not read external bytes directly as Swift structs unless binding, layout, alignment, initialization, and byte order are guaranteed.
* Prefer explicit parsing for file, network, IPC, and persisted formats.
* Keep memory binding changes local and auditable.

`withMemoryRebound(to:capacity:_:)` is for temporary rebinding when the original and rebound access are valid for the duration of the closure. Do not let the rebound pointer escape. Do not use the original typed pointer inconsistently while memory is rebound.

Suspicious:

```swift id="f3hmgy"
func parseRecord(_ raw: UnsafeRawPointer) -> Record {
    raw.assumingMemoryBound(to: Record.self).pointee
}
```

This is valid only if the memory is already bound to `Record`, initialized as `Record`, properly aligned, and valid for the access.

## Alignment, layout, and external bytes

Unsafe pointer casts do not solve byte order, alignment, padding, or versioning.

For external binary formats, check:

* byte order;
* integer size;
* signedness;
* padding;
* alignment;
* struct layout;
* versioning;
* declared lengths;
* nested offsets;
* malformed input behavior;
* integer overflow during offset calculation.

Avoid assuming this is portable:

```swift id="3aar1v"
let value = rawPointer.load(as: UInt32.self)
```

It may be wrong if the bytes are unaligned, use a different endianness, are not initialized as a `UInt32`, or do not represent the format version expected by the code.

When unaligned input is possible and the type is appropriate for byte loading, prefer APIs that explicitly support unaligned loads, or parse bytes manually. Still handle endianness, bounds, and format validation separately. Unaligned loading solves only the alignment precondition; it does not validate the external format.

Prefer explicit conversion for external formats:

```swift id="3y4g0o"
enum BinaryReadError: Error {
    case tooShort
}

func readBigEndianUInt32(_ bytes: ArraySlice<UInt8>) throws -> UInt32 {
    guard bytes.count >= 4 else {
        throw BinaryReadError.tooShort
    }

    let i0 = bytes.startIndex
    let i1 = bytes.index(after: i0)
    let i2 = bytes.index(after: i1)
    let i3 = bytes.index(after: i2)

    let b0 = UInt32(bytes[i0])
    let b1 = UInt32(bytes[i1])
    let b2 = UInt32(bytes[i2])
    let b3 = UInt32(bytes[i3])

    return b0 << 24 | b1 << 16 | b2 << 8 | b3
}
```

Use lower-level loads only when the input representation, bounds, byte order, and alignment are intentionally controlled.

## Manual allocation

When code manually allocates memory, review allocation, initialization, use, deinitialization, and deallocation as one lifecycle.

Checklist:

* Is capacity correct?
* Is alignment correct?
* Is memory initialized before typed reads?
* Is every initialized element deinitialized exactly once when needed?
* Is memory deallocated exactly once?
* Are error paths and partial initialization handled?
* Is ownership transfer explicit?
* Can cleanup run on cancellation, early return, or thrown error?
* Is deallocation performed with the matching API and allocation size/alignment assumptions?

Prefer a small wrapper that centralizes allocation, initialization, bounds checks, deinitialization, and deallocation. The lifecycle should be auditable in one place, not spread across call sites.

## `unsafeBitCast`

`unsafeBitCast` should be rare.

Use it only when:

* there is no safe conversion API;
* source and destination representations are layout-compatible;
* size and alignment assumptions are known;
* the result is valid for the destination type;
* lifetime and ownership assumptions are valid;
* tests cover the representation assumptions.

Do not use it for:

* numeric conversion;
* pointer lifetime extension;
* arbitrary bytes;
* bypassing generics or protocol design;
* avoiding a safe initializer;
* working around concurrency diagnostics;
* changing ownership or initialization state.

## `Unmanaged`

`Unmanaged` is for explicit ownership transfer across APIs that cannot express Swift ARC ownership.

Review questions:

* Who owns the object before the call?
* Does the callee retain, release, store, or only borrow the pointer?
* Is `passRetained` balanced with a consuming release path?
* Is `passUnretained` used only when the object outlives the callback?
* Can the callback fire zero, one, or many times?
* Is cancellation or early failure handled?
* Does the callback occur synchronously or asynchronously?
* Is the retained object released on every error path?

Do not use `Unmanaged` just to avoid ARC. Use it only when the external ownership contract is explicit.

## `unowned(unsafe)`

`unowned(unsafe)` disables the runtime safety check that normal `unowned` provides.

Use it only when:

* the lifetime relationship is formally guaranteed;
* dangling access is impossible by construction;
* measurement proves normal `unowned`, `weak`, or a strong reference model is a problem;
* destruction order is tested;
* the invariant is documented next to the declaration.

Prefer:

* strong references when ownership is intended;
* `weak` when `nil` is valid;
* normal `unowned` when lifetime is guaranteed but a runtime check is acceptable.

Do not use `unowned(unsafe)` only to avoid optional handling or to make a reference look cheaper.

## `nonisolated(unsafe)` and `@unchecked Sendable`

`nonisolated(unsafe)` and `@unchecked Sendable` bypass different compiler checks. Do not treat them as the same tool.

`@unchecked Sendable` is a manual promise that a type is safe to transfer across concurrency domains even though the compiler cannot verify the implementation.

`@unchecked Sendable` does not make stored pointers, references, caches, locks, or mutable state thread-safe. It is a promise that the type already provides the required safety through immutability, synchronization, value semantics, or another documented mechanism.

`nonisolated(unsafe)` bypasses actor-isolation checking for a declaration or access pattern.

`nonisolated(unsafe)` does not make actor-isolated state safe to access concurrently. It removes compiler checking at that boundary, so the immutability proof, synchronization, or external isolation rule must be explicit.

Review questions:

* What synchronization protects the accessed state?
* Can the value be read and written concurrently?
* Is the declaration immutable, thread-safe, or externally synchronized?
* Is this hiding a real data race?
* Could the design use actor isolation, a lock, an immutable snapshot, normal `nonisolated`, or a safer `Sendable` design instead?
* Is the invariant documented close to the unsafe declaration?
* Are there tests or stress tests for concurrent access?

Do not use either construct to silence diagnostics without a synchronization story.

## C, C++, Objective-C, and Core Foundation boundaries

Interop imports external memory and ownership contracts into Swift.

Review questions:

* Does the API borrow the pointer only during the call?
* Does it store the pointer?
* Does it require mutable memory or null termination?
* Does it expect a byte count, element count, or sentinel?
* Does it write into the buffer?
* Who owns returned memory and how is it freed?
* Are callbacks synchronous or asynchronous?
* Are imported structs packed, aligned, or pointer-containing?
* Are Swift objects passed through `void *` with correct retain/release behavior?
* Can the callback fire after cancellation, deallocation, or teardown?
* Does the external API require thread affinity?

Prefer small adapter layers that translate external contracts into safe Swift values immediately.

## Safe wrapper shape

Unsafe code should usually be hidden behind a safe API.

A safe wrapper should:

* validate input sizes;
* reject malformed input;
* avoid pointer escape;
* preserve lifetime;
* enforce bounds;
* handle errors;
* hide raw pointers from callers;
* document representation assumptions;
* keep unsafe expressions small;
* expose value-oriented Swift results;
* centralize cleanup;
* provide tests for the unsafe invariant.

Example safe public contract:

```swift id="z4y6k2"
enum PacketError: Error {
    case tooShort
    case invalidVersion
}

struct PacketHeader {
    var version: UInt8
    var payloadLength: UInt16
}

func parsePacketHeader(from bytes: [UInt8]) throws -> PacketHeader {
    guard bytes.count >= 3 else {
        throw PacketError.tooShort
    }

    let version = bytes[0]
    guard version == 1 else {
        throw PacketError.invalidVersion
    }

    let payloadLength = UInt16(bytes[1]) << 8 | UInt16(bytes[2])
    return PacketHeader(version: version, payloadLength: payloadLength)
}
```

If a lower-level implementation is later required, keep the public contract safe and swap the internals only after measurement.

## Modern safer memory tools

When the project uses a Swift toolchain that supports them, consider safer memory-oriented APIs before raw pointers:

* non-escapable views such as `Span` and `RawSpan`;
* mutable or output span APIs when available;
* borrowing APIs;
* noncopyable types;
* standard library APIs that avoid unnecessary copies.

`Span` and `RawSpan` are Swift 6.2-era non-escapable contiguous-memory views. Use them only when the project’s toolchain, deployment assumptions, and API surface support them. Do not recommend them as a drop-in replacement for every pointer API.

Use safer memory tools when they express the same lifetime and performance goal without pointer escape or invalid memory access. Do not present them as mandatory fixes when the target Swift version does not support them.

## Review workflow

When unsafe code appears in a performance review:

1. Identify the safe baseline.
2. Locate the measured cost or required interop contract.
3. Identify the unsafe assumption: lifetime, bounds, binding, alignment, initialization, aliasing, ownership, or thread safety.
4. Check whether the assumption is locally enforceable.
5. Prefer a safe wrapper with a small unsafe region.
6. Validate correctness and speed with tests, sanitizers, release builds, and before/after benchmarks.

## Validation

For unsafe wrappers, recommend:

* unit tests for normal cases;
* tests for empty, short, malformed, and boundary inputs;
* tests for large inputs;
* tests for declared-length overflow or nested-offset overflow;
* tests for repeated allocation and deallocation;
* tests for cancellation and cleanup paths;
* tests for concurrent access if memory is shared;
* fuzz tests for parsers and binary formats;
* Address Sanitizer where applicable;
* Thread Sanitizer where applicable;
* Undefined Behavior Sanitizer or runtime diagnostics where applicable;
* release-build benchmarks for performance claims.

Debug builds are useful for diagnostics, but do not use Debug performance as evidence for unsafe optimization.

## Checklist

Use this checklist when reviewing unsafe Swift.

* [ ] Is unsafe code necessary?
* [ ] Is there a safe baseline implementation?
* [ ] Is there measurement if unsafe was introduced for performance?
* [ ] Is the unsafe region small?
* [ ] Is unsafe code hidden behind a safe API?
* [ ] Are lifetime assumptions documented?
* [ ] Are bounds checked?
* [ ] Are alignment requirements satisfied?
* [ ] Is memory binding correct?
* [ ] Is memory initialized before reads?
* [ ] Is memory deinitialized and deallocated correctly?
* [ ] Is ownership transfer explicit?
* [ ] Is pointer escape prevented?
* [ ] Are C callbacks and cleanup paths handled?
* [ ] Is shared mutable memory synchronized?
* [ ] Are `@unchecked Sendable`, `nonisolated(unsafe)`, and `unowned(unsafe)` justified?
* [ ] Is `unsafeBitCast` avoided or heavily justified?
* [ ] Are invalid, short, malformed, boundary, and concurrent cases tested?
* [ ] Is the toolchain/version assumption documented for newer safer memory APIs?

## Common gotchas

* A pointer from `withUnsafeBytes` must not escape the closure unless the API explicitly guarantees it.
* `baseAddress` can be nil for an empty buffer.
* Buffer count may mean bytes or elements depending on the type.
* `capacity` is not initialized count.
* Raw memory is not automatically bound to the type you want.
* `assumingMemoryBound(to:)` does not bind memory.
* `withMemoryRebound` is temporary and scoped to the closure.
* Pointer casts do not fix alignment or endianness.
* Unaligned loading does not validate byte order, bounds, or external format semantics.
* Swift struct layout is not a portable file or network format by default.
* `unsafeBitCast` is not a conversion API.
* `Unmanaged` requires explicit ownership balance.
* `unowned(unsafe)` can create dangling references without a runtime trap.
* `nonisolated(unsafe)` can hide data races.
* `@unchecked Sendable` does not make unsafe storage thread-safe.
* Unsafe code can make compiler optimization harder, not easier.
* Unsafe code should be easier to audit than the safe code it replaces.
* SIL overhead alone does not justify unsafe code.

## Output guidance

When this reference is used, include:

```markdown id="6ki6qp"
## Unsafe boundary

Identify the unsafe construct and why it is being used.

## Safety assumptions

List lifetime, bounds, binding, alignment, initialization, ownership, aliasing, and thread-safety assumptions.

## Risk

Explain what can go wrong if the assumptions are false.

## Safer alternative

Suggest a safe API, safe wrapper, safer memory view, explicit parsing, or a narrower unsafe region.

## Validation

Recommend tests, sanitizers, release-build benchmarks, and before/after measurement when performance is the reason.
```

If unsafe code is not justified by correctness or measurement, recommend removing it or isolating it behind a safe wrapper.

Use cautious language when evidence is incomplete:

* "This unsafe operation is only justified if the external API requires this ownership/lifetime contract."
* "This pointer must not escape the closure unless the callee explicitly stores and owns it."
* "This load handles alignment only if the API supports unaligned access; it does not handle endianness or format validation."
* "This `@unchecked Sendable` conformance needs a synchronization or immutability story."
* "Unsafe code is not a valid next step until profiling shows the safe version is the bottleneck."
