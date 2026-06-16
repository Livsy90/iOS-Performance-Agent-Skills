# Layout, Drawing, and Animation

Use this reference when a SwiftUI task involves modifier chains, `GeometryReader`, `PreferenceKey`, layout feedback, shadows, masks, blurs, `Canvas`, `TimelineView`, animation hitches, Core Animation, or drawing-heavy UI.

The goal is not to remove visual polish. The goal is to keep layout, drawing, compositing, and animation costs proportional to the UI surface, especially inside repeated or frequently updating content.

## Contents

* [Review mindset](#review-mindset)
* [Modifier chains and view hierarchy](#modifier-chains-and-view-hierarchy)
* [GeometryReader](#geometryreader)
* [PreferenceKey and layout feedback](#preferencekey-and-layout-feedback)
* [Visual effects and compositing](#visual-effects-and-compositing)
* [`drawingGroup()` and `compositingGroup()`](#drawinggroup-and-compositinggroup)
* [`Canvas` for dense drawing](#canvas-for-dense-drawing)
* [`TimelineView` for scheduled updates](#timelineview-for-scheduled-updates)
* [Animation scope](#animation-scope)
* [Layout-affecting vs compositor-friendly animation](#layout-affecting-vs-compositor-friendly-animation)
* [Matched geometry and transitions](#matched-geometry-and-transitions)
* [Validation](#validation)
* [Common red flags](#common-red-flags)
* [Preferred refactoring order](#preferred-refactoring-order)
* [Agent output guidance](#agent-output-guidance)

## Review mindset

First classify the symptom or risk:

* layout work: geometry reads, custom measurement, preference propagation, nested layout, layout feedback loops;
* drawing work: many shapes, gradients, charts, waveforms, decorative effects, custom drawing;
* compositing work: shadows, masks, blurs, clipping, transparency, materials, layered effects;
* animation work: broad animated state changes, layout-affecting transitions, repeated updates during animation;
* unrelated app work: main-thread CPU, image decoding, formatting, or data preparation that overlaps a visual interaction.

Do not assume every hitch is caused by SwiftUI diffing. Check whether the dominant cause is main-thread blocking, layout, drawing, compositing, state churn, image work, or animation scope.

Prefer precise language:

```md
This row performs a geometry read and applies a blur, mask, and shadow in repeated content. That may increase layout and compositing work during scrolling. Validate with Animation Hitches, Core Animation, Time Profiler, or the SwiftUI instrument if the issue is user-visible.
```

Avoid unsupported claims like “this blur causes a 30 ms frame” unless a trace, screen recording, metric, benchmark, or user-provided measurement supports it.

## Modifier chains and view hierarchy

Modifiers create additional view structure. Their order affects layout, drawing, hit testing, animation, and sometimes identity.

Review long modifier chains in hot paths, especially when they include:

* `background`, `overlay`, `mask`, `clipShape`, `shadow`, `blur`;
* `drawingGroup`, `compositingGroup`;
* gestures, animations, geometry reads, preference keys;
* wrappers that hide layout, drawing, or compositing work.

A long modifier chain is not automatically bad. It becomes suspicious when it is repeated many times, updates frequently, or hides layout/drawing work that should happen at a coarser boundary.

Risky in a large list:

```swift
HStack { rowContent }
    .padding(16)
    .background(.ultraThinMaterial)
    .clipShape(RoundedRectangle(cornerRadius: 18))
    .overlay { RoundedRectangle(cornerRadius: 18).stroke(.white.opacity(0.3)) }
    .shadow(radius: row.isHighlighted ? 12 : 4)
    .blur(radius: row.isLocked ? 1.5 : 0)
    .animation(.easeInOut, value: row.isHighlighted)
```

Prefer a simpler hot-path row when the design allows it:

```swift
HStack { rowContent }
    .padding(16)
    .background {
        RoundedRectangle(cornerRadius: 18)
            .fill(row.isHighlighted ? Color.accentColor.opacity(0.12) : Color.secondary.opacity(0.08))
    }
```

If the visual effect is essential, keep it and validate the cost instead of deleting design details blindly.

## GeometryReader

`GeometryReader` is useful when a view genuinely needs container size, coordinate space, or measured geometry. It is not inherently wrong.

Use it carefully because geometry reads can couple layout to state updates and make a repeated subtree sensitive to size changes.

Common risks:

* placing a geometry reader inside every row of a long list;
* changing row sizing behavior accidentally;
* storing frequently changing geometry values in broad observable state;
* using geometry to drive layout that standard SwiftUI layout can express directly;
* creating feedback loops where measurement changes state, which changes layout, which changes measurement;
* using geometry as a generic workaround instead of narrowing the layout problem.

Risky in repeated content:

```swift
List(cards) { card in
    GeometryReader { proxy in
        LoyaltyCardRow(card: card, availableWidth: proxy.size.width)
    }
}
```

Inside rows, `GeometryReader` can also change sizing behavior because it takes the proposed space from its parent. If it must be used in a row, make the expected height and alignment explicit.

Prefer reading geometry at a stable container boundary when every row needs the same value:

```swift
GeometryReader { proxy in
    List(cards) { card in
        LoyaltyCardRow(card: card, availableWidth: proxy.size.width)
    }
}
```

Even better, use layout APIs that avoid explicit geometry when possible:

```swift
LoyaltyCardRow(card: card)
    .frame(maxWidth: .infinity, alignment: .leading)
```

For complex reusable layout logic, consider a custom `Layout` instead of scattering `GeometryReader` and preference keys across many views. Respect the deployment target before recommending this.

## PreferenceKey and layout feedback

Preference keys are useful for child-to-parent communication, such as reporting measured size, anchor positions, scroll-related values, or header offsets.

They can also introduce extra update cycles because child layout produces a value, the parent reacts, and that reaction may cause another layout pass.

Use them carefully in large lists, nested scroll views, animated containers, resizing headers, rows that update often, and layouts where measured values are written back into state.

Risky:

```swift
.onPreferenceChange(HeaderHeightKey.self) { height in
    headerHeight = height
}
```

Prefer guarding state updates so tiny measurement changes do not trigger unnecessary work:

```swift
.onPreferenceChange(HeaderHeightKey.self) { height in
    guard abs(headerHeight - height) > 0.5 else { return }
    headerHeight = height
}
```

Agent guidance:

* Keep preference values small and stable.
* Reduce values intentionally.
* Avoid emitting per-row preferences when a container-level value is enough.
* Do not write preference results into broad shared models unless multiple independent views truly need them.
* Guard against oscillation and tiny floating-point changes.

## Visual effects and compositing

Visual effects can increase drawing and compositing work, especially when repeated, animated, or combined.

Review repeated content for large dynamic shadows, animated shadow radius or opacity, nested masks, repeated `blur`, translucent materials in many rows, clipping plus shadows, complex overlays/backgrounds, animated gradients, and large images with masks or rounded clipping.

Do not state that every shadow, blur, or mask is slow. The risk depends on size, frequency, device, content, and animation.

Prefer:

* applying expensive effects at a coarser boundary when visually acceptable;
* keeping row effects simple in large lists;
* avoiding animated blur/shadow changes during scrolling;
* pre-rendering decorative assets when the effect is static and complex;
* using simpler shapes for repeated masks and clips;
* validating with Instruments when the effect is suspected to cause hitches.

Pre-render static decorative assets only when they do not need dynamic type, localization, theme, or accessibility adaptation. Watch image scale and memory.

Risky in repeated animated content:

```swift
AvatarView(user: user)
    .clipShape(Circle())
    .shadow(color: .black.opacity(0.25), radius: isActive ? 16 : 4)
    .blur(radius: isDisabled ? 1 : 0)
    .animation(.spring(), value: isActive)
```

A simpler hot-path candidate:

```swift
AvatarView(user: user)
    .clipShape(Circle())
    .overlay { Circle().stroke(isActive ? Color.accentColor : Color.clear, lineWidth: 2) }
```

Use visual simplification as a candidate refactor, not as a universal rule.

## `drawingGroup()` and `compositingGroup()`

Use these modifiers deliberately.

`compositingGroup()` changes how SwiftUI composites a subtree. `drawingGroup()` can rasterize a subtree into an offscreen representation. This may help some complex vector drawing cases, but it can also increase memory use, offscreen rendering, and texture work.

Do not recommend `drawingGroup()` as a generic performance fix.

Potentially reasonable:

```swift
ComplexBadgeVector()
    .drawingGroup()
```

Risky if applied mechanically to every row:

```swift
ForEach(rows) { row in
    TransactionRow(row: row).drawingGroup()
}
```

Validate before and after because these modifiers can move cost rather than remove cost. Watch memory and animation behavior, not only CPU samples.

## `Canvas` for dense drawing

Use `Canvas` when the UI represents many small visual elements that update together and do not need to be individual SwiftUI subviews.

Good candidates include sparklines, mini charts, timelines, heat maps, audio waveforms, dense particles, compact financial graphs, and custom progress shapes.

Risky approach:

```swift
HStack(spacing: 1) {
    ForEach(samples.indices, id: \.self) { index in
        RoundedRectangle(cornerRadius: 1)
            .frame(width: 2, height: samples[index] * 40)
    }
}
```

Prefer drawing dense primitives in one surface:

```swift
Canvas { context, size in
    guard !samples.isEmpty else { return }

    let barWidth = size.width / CGFloat(samples.count)

    for index in samples.indices {
        let normalizedValue = min(max(samples[index], 0), 1)
        let height = size.height * CGFloat(normalizedValue)
        let rect = CGRect(
            x: CGFloat(index) * barWidth,
            y: size.height - height,
            width: max(1, barWidth - 1),
            height: height
        )

        context.fill(Path(rect), with: .color(.primary))
    }
}
.frame(height: 44)
```

Keep the renderer lightweight. Prepare normalized points, colors, labels, and expensive geometry outside the drawing closure.

Do not replace normal interactive UI controls with `Canvas`. Individual elements drawn inside a canvas are not automatically separate SwiftUI views with their own accessibility, hit testing, focus, or state lifecycle.

## `TimelineView` for scheduled updates

Use `TimelineView` when UI should redraw on a known schedule, such as clocks, countdowns, market session timers, live progress indicators, time-based visualizations, or animated drawings paired with `Canvas`.

Prefer scheduled redraws over manually driving repeated `@State` changes with `Timer` when the UI is naturally time-based.

Example:

```swift
TimelineView(.periodic(from: .now, by: 1)) { context in
    Text(remainingText(now: context.date))
        .monospacedDigit()
}
```

Do not use a high-frequency schedule when a slower cadence is enough:

```swift
TimelineView(.periodic(from: .now, by: 1.0 / 60.0)) { context in
    Text(expensiveFormattedText(for: context.date))
}
```

for a countdown that only displays whole seconds.

A high-frequency cadence can be valid for animation-like drawing, simulations, or `Canvas` content, but it should be intentional and the drawing work per tick should be small.

Agent guidance:

* Match the schedule to the visible precision.
* Avoid expensive formatting or data transformation inside every tick.
* Do not use `TimelineView` as a replacement for async loading or event subscriptions.

## Animation scope

Animation hitches often come from too much work happening during an animated transaction.

Review what state changes are animated, how broad the affected subtree is, whether the animation changes layout/drawing or only transform/opacity, whether rows update during the animation, whether expensive effects are animated, and whether high-frequency values live in broad state or environment.

Prefer narrow animation scope:

```swift
withAnimation(.easeInOut) {
    expandedTransactionID = row.id
}
```

and make sure only the relevant subtree depends on `expandedTransactionID`.

Avoid applying broad implicit animation to a large container when many unrelated values can change:

```swift
PortfolioScreen(model: model)
    .animation(.easeInOut, value: model)
```

Prefer animating specific values and moving animation closer to the component that visually changes.

## Layout-affecting vs compositor-friendly animation

Animations that change layout can be more expensive than animations that affect transform or opacity.

Potentially heavier:

* animating text size;
* animating dynamic content height across many rows;
* animating layout constraints through state changes;
* animating blur, mask, shadow, or gradient changes;
* expanding many rows at once.

Often cheaper:

* opacity;
* scale;
* translation;
* simple rotation.

These are often cheaper when the affected subtree is not itself expensive to draw or composite. Transform or opacity animation can still hitch if the subtree contains heavy effects, large images, materials, masks, shadows, or frequent state updates.

Do not force every design into transform-only animation. Use this distinction to reason about risk and decide what to measure.

## Matched geometry and transitions

`matchedGeometryEffect` and custom transitions can produce polished UI, but they can also make layout and animation dependencies harder to reason about.

Use them carefully when source and destination views live inside large, frequently updating containers; list rows are inserted or removed during the same animation; matched views have heavy masks/materials/shadows/dynamic text; or multiple matched elements animate at once.

Also check correctness: matched elements should use stable IDs and the intended namespace, and source/destination lifetimes should be clear. Ambiguous multiple sources for the same matched ID can produce surprising transitions.

Agent guidance:

* Keep matched elements visually focused.
* Avoid pairing matched geometry with broad unrelated state updates.
* Validate on target devices if the transition is central to the experience.
* If the animation hitches, test a simpler transition before redesigning the entire screen.

## Validation

Choose the smallest tool that answers the question.

Use Animation Hitches for scrolling stutter, skipped frames, visible gesture pauses, or janky transitions.

Use Animation Hitches and Core Animation instruments for layer/compositing pressure, offscreen rendering candidates, heavy masks/shadows/blurs/transparency, or large animated surfaces.

Use Time Profiler for main-thread CPU work, expensive formatting or image work during updates, custom layout/drawing code consuming CPU, or synchronous work that overlaps animation.

Use the SwiftUI instrument when available for long body updates, unnecessary body updates, broad SwiftUI update causes, or representable update cost.

Use signposts around animation start/end, geometry or preference updates, page append or row insertion, render model generation, expensive drawing preparation, and state changes that trigger visual updates.

Do not claim that Instruments proved a cause unless the trace or user-provided artifact actually shows it.

## Common red flags

Flag these patterns during layout, drawing, and animation review:

```swift
GeometryReader { proxy in
    Row(width: proxy.size.width)
}
```

inside every row of a large collection.

```swift
.onPreferenceChange(RowOffsetKey.self) { value in
    model.scrollOffset = value
}
```

when `model` is broad state observed by a large screen.

```swift
LargeList(rows: rows)
    .animation(.default, value: screenModel)
```

when many unrelated properties can change.

```swift
ForEach(rows) { row in
    Row(row: row)
        .shadow(radius: row.isSelected ? 20 : 4)
        .blur(radius: row.isDisabled ? 2 : 0)
}
```

in a frequently scrolling list.

```swift
ForEach(points.indices, id: \.self) { index in
    Circle().frame(width: 2, height: 2).position(points[index])
}
```

for dense chart-like drawing that could be a single `Canvas`.

```swift
TimelineView(.periodic(from: .now, by: 1.0 / 60.0)) { context in
    Text(expensiveFormattedText(for: context.date))
}
```

when the visible content does not require 60 updates per second.

## Preferred refactoring order

For layout, drawing, and animation issues, prefer this order:

1. Narrow the state dependency that triggers the visual update.
2. Remove expensive data preparation from the visual update path.
3. Simplify repeated row modifier chains.
4. Move geometry reads to stable container boundaries.
5. Replace layout feedback loops with simpler layout APIs when possible.
6. Guard preference-driven state updates.
7. Apply expensive effects at coarser boundaries when visually acceptable.
8. Replace dense subview trees with `Canvas` when the content is drawing-like.
9. Match `TimelineView` cadence to visible precision.
10. Narrow animation scope to the component that actually changes.
11. Prefer transform/opacity animation when it preserves the design intent.
12. Validate suspected hitches with the smallest suitable profiling tool.

## Agent output guidance

When responding to a code review involving this reference, include:

1. The likely layout, drawing, or animation risk.
2. Why it matters in SwiftUI.
3. Whether the issue is a static risk or measured finding.
4. A focused refactor.
5. What to measure if the user needs confirmation.

Avoid generic advice such as:

```md
Use fewer modifiers.
```

Prefer concrete advice:

```md
This `GeometryReader` runs inside every row even though all rows need the same container width. Move the width read to the list container or use `.frame(maxWidth: .infinity)` if the row only needs flexible width.
```
