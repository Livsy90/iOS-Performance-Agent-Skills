# SIL Inspection

Use this reference when source-level reasoning is not enough and the task needs optimized SIL evidence for allocation, ARC, dispatch, existential opening, closure creation, specialization, or inlining.

SIL inspection is a support tool. Use it to validate a narrow runtime hypothesis, not as a replacement for profiling and not as a reason to rewrite clear code.

Do not use this reference to judge code style, architecture, or performance from compiler output alone. Use it only when optimized compiler output would change the recommendation.

## Contents

* [When to use this reference](#when-to-use-this-reference)
* [Core model](#core-model)
* [Guardrails](#guardrails)
* [Generate SIL](#generate-sil)
* [Choose the right SIL form](#choose-the-right-sil-form)
* [Inspection workflow](#inspection-workflow)
* [Search targets](#search-targets)
* [Pattern guide](#pattern-guide)
* [Before and after comparison](#before-and-after-comparison)
* [Common false conclusions](#common-false-conclusions)
* [Output guidance](#output-guidance)

## When to use this reference

Read this file when the task asks whether the Swift optimizer removed, kept, or transformed a suspected runtime cost.

Good triggers:

* "Does this closure allocate?"
* "Did this generic function specialize?"
* "Is this protocol call still a witness-table call?"
* "Did this class method devirtualize?"
* "Is this existential opened or boxed in the hot path?"
* "Why are there retains/releases here?"
* "Would `final`, generics, `@inlinable`, or a module-boundary change affect optimized output?"

Use SIL when:

* the code is plausibly hot;
* the suspected cost is runtime-level;
* source-level reasoning is not enough;
* optimized compiler output would change the recommendation;
* the build configuration can be reproduced closely enough to make the result meaningful.

Prefer another skill or reference when:

* task lifetime, cancellation, `MainActor` responsiveness, actor design, or actor contention is the main issue — use `swift-concurrency-performance`;
* SwiftUI invalidation, identity, layout, scrolling, or view lifecycle is the main issue — use `swiftui-performance`;
* launch time, dyld, static initializers, or framework loading is the main issue — use `ios-launch-performance`;
* the main task is choosing or interpreting Instruments, MetricKit, XCTest metrics, signposts, or traces — use `ios-performance-profiling`;
* the main issue is API shape around `any`, `some`, generics, or type erasure — use `existentials-generics-opaque-types.md`;
* the main issue is COW mutation or large value copying — use `cow-and-large-values.md`;
* the main issue is ARC lifetime, captures, or reference cycles — use `arc-and-ownership.md`.

If one of these files does not exist yet, treat the link as intended routing and do not invent details from a missing reference.

## Core model

SIL is Swift Intermediate Language. It sits between type-checked Swift source and lower-level compiler output.

SIL can expose Swift-specific performance signals:

* reference counting and ownership operations;
* class, Objective-C, and protocol witness dispatch;
* existential initialization, opening, and boxing;
* closure formation and captured context;
* generic specialization;
* inlining and direct calls;
* value copying and destruction;
* stack, box, and reference allocation patterns;
* async lowering and executor-hop patterns when concurrency runtime cost is the hypothesis.

SIL is not a stable user-facing API. Instruction names and generated patterns can change across compiler versions, ownership modes, optimization levels, language modes, target platforms, and module settings.

Use SIL to answer a narrow question:

> Does the optimized build still contain the suspected allocation, dispatch, retain/release, existential operation, closure creation, unspecialized generic call, missed inline opportunity, boxed capture, or value-copying pattern in the hot path?

SIL can show that a suspected compiler-level cost remains. It does not prove that the cost is user-visible, dominant, or worth rewriting without measurement.

## Guardrails

* Inspect optimized SIL for Release performance questions.
* Do not make Release performance claims from `-Onone` SIL.
* Compare output with the same Swift version, optimization level, target architecture, SDK, library evolution setting, whole-module or cross-module optimization setting, and module boundaries as the real build.
* When the real build uses whole-module optimization, cross-module optimization, library evolution, or a package/framework boundary, reproduce that configuration before making claims about specialization or optimizer visibility.
* For Apple-platform code, prefer reproducing the target SDK, target triple, build configuration, module imports, and framework boundaries when the result depends on platform frameworks or ABI boundaries.
* Treat SIL instructions as clues. Confirm important claims with Instruments, benchmarks, signposts, XCTest performance tests, production metrics, or assembly when appropriate.
* Do not quote large SIL dumps. Summarize the relevant pattern.
* Do not rewrite clear code because of a single scary-looking instruction.
* Do not assume every SIL instruction survives unchanged into final machine code.
* Do not assume a single-file `swiftc -O -emit-sil Example.swift` result represents the real app build.
* Do not use `@inline(__always)`, `@inlinable`, unsafe code, or architecture changes without a measured or clearly hot path.

## Generate SIL

For a small standalone file:

```bash id="o3fc0w"
swiftc -O -emit-sil Example.swift > optimized.sil
```

For raw SIL directly after SILGen:

```bash id="cxua6n"
swiftc -Onone -emit-silgen Example.swift > raw.sil
```

For canonical SIL without performance optimization:

```bash id="3cwviq"
swiftc -Onone -emit-sil Example.swift > canonical-onone.sil
```

For optimized SIL with a module name:

```bash id="f89yyi"
swiftc -O -emit-sil -module-name MyModule *.swift > optimized.sil
```

Demangle symbols when needed:

```bash id="a7v3b5"
swift-demangle '$s4Demo9makeValueSiyF'
```

For targeted compiler debugging, advanced SIL dump options may be useful, such as printing a specific function or printing SIL before and after a pass. These are compiler-debugging tools; use them only when normal optimized SIL output is not enough.

Examples of useful directions:

```bash id="ce7tzd"
swiftc -O -emit-sil Example.swift -Xllvm -sil-print-functions=ranked
swiftc -O -emit-sil Example.swift -Xllvm -sil-print-function=<mangled-name>
```

When module boundaries matter, reproduce the real module structure instead of compiling one pasted file. Optimizer visibility, access control, whole-module optimization, library evolution, public API boundaries, target architecture, and SDK imports can change the result.

## Choose the right SIL form

### Raw SIL

Generated with:

```bash id="hyq8bp"
swiftc -Onone -emit-silgen Example.swift
```

Use raw SIL to understand initial lowering. Do not use it to judge final runtime performance.

Raw SIL is useful for questions such as:

* how a source construct is initially lowered;
* whether a closure, capture, existential, or box appears before optimization;
* what the compiler starts with before mandatory and performance passes.

### Canonical `-Onone` SIL

Generated with:

```bash id="9nsx5v"
swiftc -Onone -emit-sil Example.swift
```

Use it to inspect mandatory lowering and ownership-related transformations. Do not treat it as Release performance evidence.

It can help explain why a construct exists, but it does not answer whether optimization removes, moves, or simplifies it.

### Optimized SIL

Generated with:

```bash id="wmqsw2"
swiftc -O -emit-sil Example.swift
```

Use optimized SIL for performance-relevant questions:

* whether allocation remains;
* whether a closure context is still created;
* whether retains/releases remain in a hot region;
* whether calls devirtualize;
* whether protocol calls specialize;
* whether existential opening or boxing remains;
* whether a helper function inlines;
* whether module boundaries block optimization;
* whether value copying or ownership traffic remains around the suspected path.

Optimized SIL is the default form for this reference.

## Inspection workflow

1. State the exact question.

   Good questions:

   * "Does this `any Formatter` call remain in the hot loop?"
   * "Does this escaping closure allocate each time?"
   * "Does the generic helper specialize for `ImageRecord`?"
   * "Does marking this app-level class `final` change a hot call from `class_method` to direct call?"
   * "Does this value remain alive across the `await`?"

   Weak questions:

   * "Is this code fast?"
   * "Is this SIL good?"
   * "Should I rewrite this abstraction?"
   * "Can I make this SIL shorter?"

2. Confirm the build context.

   Check Swift version, optimization level, target architecture, SDK, target triple when relevant, whole-module optimization, cross-module optimization, library evolution, module boundaries, build configuration, and whether the issue is Debug-only or Release-relevant.

3. Find the reviewed function.

   Search for the source function name, demangle symbols when needed, and check compiler comments when available.

   Be careful with:

   * specialized clones;
   * reabstraction thunks;
   * protocol witness thunks;
   * Objective-C thunks;
   * closure functions;
   * generated conformance helpers;
   * async function fragments;
   * inlined functions that no longer appear as separate calls.

   The relevant cost may be in a specialized clone or closure body, not in the original source function symbol.

4. Search for the suspected pattern.

   Do not scan every instruction mechanically. Search for the cost that matches the hypothesis.

5. Interpret the local pattern.

   Ask whether the instruction appears inside the hot loop or hot call path, not merely somewhere in the file.

6. Compare before and after.

   A useful refactor should remove, reduce, or move the suspected cost out of the hot path without damaging semantics.

7. Validate outside SIL.

   Use Instruments, a benchmark, signposts, XCTest performance tests, production metrics, or assembly when the result matters to users.

## Search targets

Use these as practical search strings. Instruction names can vary across compiler versions.

```text id="dj0kpj"
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
hop_to_executor
```

Search by prefixes when useful. For example, existential operations can appear in forms such as `init_existential_*`, `open_existential_*`, `alloc_existential_box`, or `project_existential_box`.

This list is not exhaustive. Some operations appear under different names, some are optimized away, and some relevant cost is visible only after following calls, thunks, closure bodies, or specialized functions.

After searching, answer:

1. Is the pattern inside the hot path?
2. Is it expected for this source construct?
3. Did optimization remove it, move it, or keep it?
4. Does it explain measured allocation, ARC, dispatch, copying, or scheduling cost?
5. Can a local semantic change remove it?
6. Would the fix require API, module, ownership, or architecture changes?
7. How will the change be validated?

## Pattern guide

Use these patterns as investigation signals, not final proof.

| Pattern                                       | Possible meaning                                                       | Do not conclude                                   |
| --------------------------------------------- | ---------------------------------------------------------------------- | ------------------------------------------------- |
| `alloc_ref`                                   | class/object allocation remains                                        | the class must become a struct                    |
| `alloc_ref_dynamic`                           | dynamically typed allocation remains                                   | dynamic allocation is necessarily the bottleneck  |
| `alloc_box` / `project_box`                   | boxed mutable capture or boxed storage                                 | raw SIL box means optimized build allocates a box |
| `alloc_stack`                                 | stack storage or temporary address storage                             | there is no cost anywhere around this code        |
| `partial_apply`                               | closure, partial application, capture context, or reabstraction signal | closures are automatically too expensive          |
| `thin_to_thick_function` / `convert_function` | function representation conversion or adaptation                       | this is a user-visible cost by itself             |
| `strong_retain` / `strong_release`            | ARC traffic remains                                                    | ARC is the bottleneck                             |
| `retain_value` / `release_value`              | ownership traffic remains                                              | the value is too expensive to use                 |
| `copy_value` / `copy_addr`                    | ownership movement or value copying at SIL level                       | a deep copy definitely happened                   |
| `destroy_value` / `destroy_addr`              | end of ownership/lifetime for a value/address                          | destruction dominates runtime                     |
| `class_method`                                | class dispatch remains                                                 | the call is necessarily expensive                 |
| `objc_method`                                 | Objective-C dispatch boundary remains                                  | `@objc` should be removed                         |
| `witness_method`                              | protocol requirement dispatch remains                                  | protocols are bad                                 |
| `open_existential_*`                          | existential opening remains                                            | `any Protocol` is always wrong                    |
| `init_existential_*`                          | existential value/container formation                                  | every existential allocates                       |
| `alloc_existential_box`                       | existential boxing remains                                             | all existential use is wrong                      |
| `function_ref`                                | direct function reference                                              | all abstraction cost is gone                      |
| specialized function name                     | specialization likely happened                                         | the whole path is fast                            |
| `hop_to_executor`                             | executor transition in lowered async code                              | every `await` caused the hop or the hop dominates |

Additional interpretation notes:

* `alloc_stack` is usually less suspicious than heap allocation, but it is not proof that the surrounding code is free. Large stack values, address-only operations, repeated stack traffic, or copies around stack storage can still matter in narrow cases.
* `copy_value` and `copy_addr` often describe ownership movement or value copying at SIL level. For COW values, this may retain shared storage rather than copy the full buffer. Check the type and mutation boundary before claiming a deep copy.
* `partial_apply` is a closure or partial-application signal. Check whether it remains in optimized SIL, whether it is inside the hot path, and whether Allocations or Time Profiler show real cost.
* `function_ref` can show a direct reference, but the work inside the function may still dominate. Direct dispatch is not the same as fast code.
* No single SIL instruction proves a user-visible performance problem by itself.

## Before and after comparison

Use this workflow when proposing a change:

1. Capture the baseline:

   * source snippet;
   * optimized SIL;
   * Swift version and target architecture;
   * optimization mode and module settings;
   * build settings relevant to the hypothesis;
   * measurement or hot-path justification.

2. State the hypothesis:

   * "This existential remains in the hot loop."
   * "This closure is allocated repeatedly."
   * "This class call does not devirtualize."
   * "This generic function does not specialize across the module boundary."
   * "This helper does not inline, leaving witness calls and ARC traffic."
   * "This boxed mutable capture remains in optimized SIL."

3. Make the smallest safe source-level change:

   * narrow a closure capture;
   * mark an app-internal non-inheritable class as `final`;
   * move a hot inner loop into a generic helper;
   * keep a hot implementation internal to a module;
   * remove type erasure from the inner loop while preserving it at the boundary;
   * reduce mutation after sharing;
   * use a borrowing or in-place API where semantics allow;
   * batch repeated actor or boundary calls when semantics allow.

4. Inspect optimized SIL again.

5. Validate the performance impact.

Do not stop at "SIL looks cleaner" when the claim is user-visible performance.

## Common false conclusions

Avoid these mistakes:

* "`alloc_stack` means there is no cost."
* "`alloc_ref` means this class must become a struct."
* "`alloc_box` in raw SIL means the optimized build allocates a box."
* "`witness_method` means protocols are bad."
* "`open_existential` means `any Protocol` is always wrong."
* "`partial_apply` means closures are always too expensive."
* "`strong_retain` means ARC is the bottleneck."
* "`copy_value` means a deep copy happened."
* "`function_ref` means the path is fast."
* "`specialized` means no runtime cost remains."
* "No `witness_method` means no abstraction cost remains."
* "No obvious SIL issue means the code is fast."
* "Debug SIL explains Release performance."
* "Single-file SIL explains the real app build."
* "`@inline(__always)` is the next step whenever a call remains."
* "`@inlinable` is safe because it can improve optimization."
* "Unsafe Swift is justified because optimized SIL still has overhead."

SIL explains compiler lowering and optimization opportunities. It does not replace runtime measurement.

## Output guidance

When using this reference, respond with:

```markdown id="j0gqah"
## SIL question

State the exact question SIL is being used to answer.

## Build context

State Swift version, optimization level, module boundary, library evolution setting, target architecture, target SDK or platform when relevant, and whether this matches the real build.

## Relevant SIL pattern

Summarize the important instruction pattern without pasting a large dump.

## Interpretation

Explain what the pattern likely means and what it does not prove.

## Recommended change

Suggest the smallest safe source-level change.

## Trade-offs

Mention API clarity, code size, module boundaries, ABI, ownership, readability, maintainability, or concurrency semantics.

## Validation

Recommend optimized SIL comparison plus Instruments, benchmark, signposts, XCTest performance tests, production metrics, or assembly when needed.
```

If the code is not likely in a hot path, say that SIL inspection is optional and avoid low-level rewrites.

Use cautious language when evidence is incomplete:

* "This pattern suggests the closure may remain in the hot path; validate with Allocations or Time Profiler."
* "This single-file SIL result may not match the real module boundary."
* "This `copy_value` does not prove a deep copy; check the type and mutation boundary."
* "The SIL is cleaner, but the performance claim still needs a before/after measurement."
* "The remaining dynamic dispatch may be intentional API behavior."
