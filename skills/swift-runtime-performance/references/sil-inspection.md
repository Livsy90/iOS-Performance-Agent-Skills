# SIL Inspection

Use this reference when source-level reasoning is not enough and the task needs optimized SIL evidence for allocation, ARC, dispatch, existential opening, closure creation, specialization, or inlining.

SIL inspection is a support tool. Use it to validate a narrow runtime hypothesis, not as a replacement for profiling and not as a reason to rewrite clear code.

## Contents

- [When to use this reference](#when-to-use-this-reference)
- [Core model](#core-model)
- [Guardrails](#guardrails)
- [Generate SIL](#generate-sil)
- [Choose the right SIL form](#choose-the-right-sil-form)
- [Inspection workflow](#inspection-workflow)
- [Search targets](#search-targets)
- [Pattern guide](#pattern-guide)
- [Before and after comparison](#before-and-after-comparison)
- [Common false conclusions](#common-false-conclusions)
- [Output guidance](#output-guidance)

## When to use this reference

Read this file when the task asks whether the Swift optimizer removed, kept, or transformed a suspected runtime cost.

Good triggers:

- "Does this closure allocate?"
- "Did this generic function specialize?"
- "Is this protocol call still a witness-table call?"
- "Did this class method devirtualize?"
- "Is this existential opened or boxed in the hot path?"
- "Why are there retains/releases here?"
- "Would `final`, generics, or `@inlinable` change optimized output?"

Use SIL when:

- the code is plausibly hot;
- the suspected cost is runtime-level;
- source-level reasoning is not enough;
- optimized compiler output would change the recommendation.

Prefer another skill when:

- task lifetime, cancellation, MainActor responsiveness, or actor contention is the main issue — use `swift-concurrency-performance`;
- SwiftUI invalidation, identity, layout, scrolling, or view lifecycle is the main issue — use `swiftui-performance`;
- launch time, dyld, static initializers, or framework loading is the main issue — use `ios-launch-performance`;
- the main task is choosing or interpreting Instruments, MetricKit, XCTest metrics, signposts, or traces — use `ios-performance-profiling`.

## Core model

SIL is Swift Intermediate Language. It sits between type-checked Swift source and lower-level compiler output.

SIL can expose Swift-specific performance signals:

- reference counting and ownership operations;
- class, Objective-C, and protocol witness dispatch;
- existential initialization, opening, and boxing;
- closure formation and captured context;
- generic specialization;
- inlining and direct calls;
- value copying and destruction;
- stack, box, and reference allocation patterns.

SIL is not a stable user-facing API. Instruction names and generated patterns can change across compiler versions, ownership modes, optimization levels, and module settings.

Use SIL to answer a narrow question:

> Does the optimized build still contain the suspected allocation, dispatch, retain/release, existential operation, closure creation, unspecialized generic call, or missed inline opportunity?

## Guardrails

- Inspect optimized SIL for Release performance questions.
- Do not make Release performance claims from `-Onone` SIL.
- Compare output with the same Swift version, optimization level, target architecture, library evolution setting, and module boundaries as the real build.
- Treat SIL instructions as clues. Confirm important claims with Instruments, benchmarks, signposts, XCTest performance tests, or production metrics.
- Do not quote large SIL dumps. Summarize the relevant pattern.
- Do not rewrite clear code because of a single scary-looking instruction.
- Do not assume every SIL instruction survives unchanged into final machine code.
- Do not use `@inline(__always)`, `@inlinable`, unsafe code, or architecture changes without a measured or clearly hot path.

## Generate SIL

For a small standalone file:

```bash
swiftc -O -emit-sil Example.swift > optimized.sil
```

For raw SIL directly after SILGen:

```bash
swiftc -Onone -emit-silgen Example.swift > raw.sil
```

For canonical SIL without performance optimization:

```bash
swiftc -Onone -emit-sil Example.swift > canonical-onone.sil
```

For optimized SIL with a module name:

```bash
swiftc -O -emit-sil -module-name MyModule *.swift > optimized.sil
```

Demangle symbols when needed:

```bash
swift-demangle '$s4Demo9makeValueSiyF'
```

When module boundaries matter, reproduce the real module structure instead of compiling one pasted file. Optimizer visibility, access control, whole-module optimization, library evolution, and public API boundaries can change the result.

## Choose the right SIL form

### Raw SIL

Generated with:

```bash
swiftc -Onone -emit-silgen Example.swift
```

Use raw SIL to understand initial lowering. Do not use it to judge final runtime performance.

### Canonical `-Onone` SIL

Generated with:

```bash
swiftc -Onone -emit-sil Example.swift
```

Use it to inspect mandatory lowering and ownership-related transformations. Do not treat it as Release performance evidence.

### Optimized SIL

Generated with:

```bash
swiftc -O -emit-sil Example.swift
```

Use optimized SIL for performance-relevant questions:

- whether allocation remains;
- whether a closure context is still created;
- whether retains/releases remain in a hot region;
- whether calls devirtualize;
- whether protocol calls specialize;
- whether existential opening or boxing remains;
- whether a helper function inlines;
- whether module boundaries block optimization.

Optimized SIL is the default form for this reference.

## Inspection workflow

1. 1State the exact question.

   Good questions:

  - "Does this `any Formatter` call remain in the hot loop?"
  - "Does this escaping closure allocate each time?"
  - "Does the generic helper specialize for `ImageRecord`?"
  - "Does marking this app-level class `final` change a hot call from `class_method` to direct call?"

   Weak questions:

  - "Is this code fast?"
  - "Is this SIL good?"
  - "Should I rewrite this abstraction?"

2. 2Confirm the build context.

   Check compiler version, optimization level, target architecture, whole-module optimization, library evolution, module boundaries, and whether the issue is Debug-only or Release-relevant.

3. 3Find the reviewed function.

   Ignore unrelated thunks, standard library noise, synthesized conformance helpers, and functions outside the suspected hot path.

4. 4Search for the suspected pattern.

   Do not scan every instruction mechanically. Search for the cost that matches the hypothesis.

5. 5Interpret the local pattern.

   Ask whether the instruction appears inside the hot loop or hot call path, not merely somewhere in the file.

6. 6Compare before and after.

   A useful refactor should remove, reduce, or move the suspected cost out of the hot path without damaging semantics.

7. 7Validate outside SIL.

   Use Instruments, a benchmark, signposts, XCTest performance tests, or production metrics when the result matters to users.

## Search targets

Use these as practical search strings. Instruction names can vary across compiler versions.

```text
alloc_ref
alloc_ref_dynamic
alloc_box
project_box
alloc_stack
partial_apply
thin_to_thick_function
convert_function
strong_retain
strong_release
retain_value
release_value
copy_value
destroy_value
copy_addr
destroy_addr
begin_borrow
end_borrow
class_method
super_method
objc_method
objc_super_method
witness_method
init_existential
open_existential
alloc_existential_box
project_existential_box
function_ref
apply
specialized
```

After searching, answer:

1. 1Is the pattern inside the hot path?
2. 2Is it expected for this source construct?
3. 3Did optimization remove it, move it, or keep it?
4. 4Does it explain measured allocation, ARC, dispatch, or copying cost?
5. 5Can a local semantic change remove it?
6. 6Would the fix require API, module, or architecture changes?
7. 7How will the change be validated?

## Pattern guide


## Before and after comparison

Use this workflow when proposing a change:

1. 1Capture the baseline:
  - source snippet;
  - optimized SIL;
  - build settings;
  - measurement or hot-path justification.

2. 2State the hypothesis:
  - "This existential remains in the hot loop."
  - "This closure is allocated repeatedly."
  - "This class call does not devirtualize."
  - "This generic function does not specialize across the module boundary."
  - "This helper does not inline, leaving witness calls and ARC traffic."

3. 3Make the smallest safe source-level change:
  - narrow a closure capture;
  - mark an app-internal non-inheritable class as `final`;
  - move a hot inner loop into a generic helper;
  - keep a hot implementation internal to a module;
  - remove type erasure from the inner loop while preserving it at the boundary;
  - reduce mutation after sharing;
  - use a borrowing or in-place API where semantics allow.

4. 4Inspect optimized SIL again.

5. 5Validate the performance impact.

Do not stop at "SIL looks cleaner" when the claim is user-visible performance.

## Common false conclusions

Avoid these mistakes:

- "`alloc_stack` means there is no cost."
- "`alloc_ref` means this class must become a struct."
- "`alloc_box` in raw SIL means the optimized build allocates a box."
- "`witness_method` means protocols are bad."
- "`open_existential` means `any Protocol` is always wrong."
- "`partial_apply` means closures are always too expensive."
- "`strong_retain` means ARC is the bottleneck."
- "`copy_value` means a deep copy happened."
- "No `witness_method` means no abstraction cost remains."
- "No obvious SIL issue means the code is fast."
- "Debug SIL explains Release performance."
- "`@inline(__always)` is the next step whenever a call remains."
- "`@inlinable` is safe because it can improve optimization."
- "Unsafe Swift is justified because optimized SIL still has overhead."

SIL explains compiler lowering and optimization opportunities. It does not replace runtime measurement.

## Output guidance

When using this reference, respond with:

```markdown
## SIL question

State the exact question SIL is being used to answer.

## Relevant SIL pattern

Summarize the important instruction pattern without pasting a large dump.

## Interpretation

Explain what the pattern likely means and what it does not prove.

## Recommended change

Suggest the smallest safe source-level change.

## Trade-offs

Mention API clarity, code size, module boundaries, ABI, readability, or maintainability.

## Validation

Recommend optimized SIL comparison plus Instruments, benchmark, signposts, XCTest performance tests, or assembly when needed.
```

If the code is not likely in a hot path, say that SIL inspection is optional and avoid low-level rewrites.
