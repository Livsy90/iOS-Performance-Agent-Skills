---
name: swiftui-performance
description: Use this skill when reviewing or fixing SwiftUI performance issues, including unnecessary invalidation, unstable identity, broad state dependencies, expensive body work, heavy rows, scrolling hitches, layout/drawing cost, or async lifecycle work. Do not use it for general SwiftUI syntax, styling, UIKit-only performance, or generic profiling unless SwiftUI update behavior is central.
---

# SwiftUI Performance

## Purpose

Use this skill to review and refactor SwiftUI code with a performance-first mental model focused on change locality.

SwiftUI performance work should explain which change invalidates which part of the UI, why that work matters, and how to keep updates local.

## Scope

Use this skill for:

* SwiftUI screen, list, row, layout, drawing, animation, and lifecycle reviews
* targeted refactors that reduce unnecessary updates or rendering work
* profiling plans only when they validate a SwiftUI performance hypothesis

Do not use this skill for:

* UIKit-only performance problems
* app launch performance unless the issue is inside SwiftUI root view or scene construction
* networking, backend latency, database performance, or caching unless they affect SwiftUI updates
* generic Instruments walkthroughs without a SwiftUI code path
* broad architecture rewrites without a concrete SwiftUI performance risk

## Core Model

Answer three questions first:

1. What changed?
2. Which views depend on that change?
3. How much UI work happens because of it?

Then inspect:

* Identity: does SwiftUI preserve the same logical view across updates?
* Lifetime: is state owned by the smallest stable component that needs it?
* Dependencies: does each view read only the data it needs to render?
* Rendering: is `body` cheap, predictable, and free from heavy transformation work?
* Lifecycle: is async work tied to `.task`, `.onAppear`, or explicit actions rather than started from `body`?

A SwiftUI view is a value description of UI. Do not treat it as a long-lived UIKit-style object.

## Review Workflow

Inspect the smallest relevant code path before suggesting broad changes.

Check in this order:

1. Stable and intentional identity.
2. Expensive work in `body`.
3. Broad state or model reads.
4. State ownership and lifetime.
5. List, row, and pagination structure.
6. Closure-heavy inputs, custom bindings, and unnecessary type erasure.
7. Layout, drawing, and animation cost in repeated content.
8. Async lifecycle, cancellation, and main-actor work.
9. Profiling or debug probes when confirmation is useful.

Prefer targeted refactors over architectural rewrites.

## Evidence Rule

Always separate:

* static code review findings
* likely risks
* hypotheses
* measured results
* user-provided evidence
* tool-generated evidence

Never invent timing numbers or claim profiling results unless the user provided evidence or a profiling command was actually run.

Prefer:

> This parent view reads the whole model, so changes to that model can invalidate a large part of the screen.

Avoid:

> This costs 500 ms.

## Common Red Flags

Flag these patterns when they appear on an update or scrolling path:

* unstable IDs such as `.id(UUID())` or `var id: UUID { UUID() }`
* index-based identity for mutable collections
* sorting, filtering, grouping, formatting, parsing, decoding, or database reads inside `body`
* filtering inside `ForEach` instead of preparing visible rows first
* passing one giant model into every child view
* broad computed properties that hide many state reads
* `AnyView` in large repeated collections
* custom `Binding(get:set:)` when a key-path binding would work
* many stored non-visual closures in row views
* `GeometryReader`, preferences, heavy shadows, masks, blurs, or overlays in every row
* async work started directly from `body`
* unguarded `.onAppear` pagination triggers inside rows

## Refactor Guidelines

Use the smallest fix that addresses the cause:

* keep identity stable
* move expensive transformations out of `body`
* prepare render-ready models outside repeated rendering paths
* move state closer to the component that owns it
* split large views into dependency-focused subviews
* simplify row inputs and repeated structure
* prefer key-path bindings over custom closure bindings when no transformation is needed
* use `.equatable()` only when visual equality is clear and cheaper than recomputing
* use `.task(id:)` when work is tied to a changing input and should cancel/restart automatically
* consider UIKit only for genuinely hot paths that need lower-level control

Do not use `.equatable()`, memoization, or caching as blanket fixes. Explain the invalidation or rendering issue first.

## Reference Routing

Read references only when the task needs more detail:

* `references/identity-and-state.md` — identity, `.id`, `ForEach`, `@State`, `@StateObject`, `@ObservedObject`, owned observable models
* `references/observation-and-dependencies.md` — Observation, `ObservableObject`, broad dependencies, environment reads
* `references/body-cost-and-render-models.md` — sorting, filtering, formatting, derived data, render models
* `references/lists-pagination-and-rows.md` — `List`, `ScrollView`, `LazyVStack`, rows, pagination, scrolling
* `references/closures-bindings-and-equatable.md` — closures, bindings, `.equatable()`, action handlers
* `references/layout-drawing-and-animation.md` — layout, drawing, modifiers, animation hitches
* `references/async-lifecycle-and-mainactor.md` — `.task`, `.onAppear`, cancellation, `MainActor`
* `references/profiling-validation.md` — Instruments, `xctrace`, signposts, XCTest, MetricKit

## Response Format

For code reviews, answer with:

1. Main issue
2. Why it matters in SwiftUI
3. Risk level
4. Suggested refactor
5. Code example when useful
6. What to measure if confirmation is needed

For profiling requests, answer with:

1. Scenario
2. Hypothesis
3. Tool or probe
4. What to look for
5. How to compare before and after

Keep the answer concrete. Do not say “SwiftUI is slow.” Explain the specific identity, dependency, rendering, layout, drawing, animation, or lifecycle issue.

## Final Principle

SwiftUI performance improves when code makes change locality obvious.
