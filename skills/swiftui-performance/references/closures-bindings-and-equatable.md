# Closures, Bindings, and Equatable Views

Use this reference when a SwiftUI performance review involves stored closures in views, capture lists, action handlers, gestures, menus, swipe actions, custom `Binding(get:set:)`, key-path bindings, `.equatable()`, or `Equatable` view inputs.

The goal is not to remove every closure, avoid every custom binding, or make every view equatable. The goal is to keep visual inputs, action routing, and equality boundaries easy to reason about in large or frequently updating view trees.

## Contents

- Core model
- Review procedure
- Stored closures in repeated views
- Capture lists and action inputs
- Action routing patterns
- Gestures, menus, and swipe actions
- Custom bindings
- Key-path bindings with Observation
- Equatable views
- Equatable and closures
- Validation
- Risk levels
- Common mistakes

## Core model

A SwiftUI view value may contain two kinds of input:

- visual input: text, numbers, flags, colors, images, render models, layout values
- behavioral input: closures, gesture handlers, menu actions, swipe actions, binding closures, service calls

Visual input is usually easier to compare, isolate, and test. Behavioral input is harder to compare, easier to over-capture, and more likely to hide dependencies.

This matters most when the view is:

- inside a large `List`, `LazyVStack`, or `ForEach`
- updated frequently by parent state
- gesture-heavy or menu-heavy
- already suspected of unnecessary row updates
- using `.equatable()` or custom equality
- using custom `Binding(get:set:)` in repeated content

Do not claim that closures automatically cause redraws. Treat closure-heavy rows as a review signal, not proof of a measured performance issue.

## Review procedure

When reviewing closure-heavy SwiftUI code, check:

1. 1Is this a small static view or a hot repeated view?
2. 2Are closures stored as properties of a visual row?
3. 3Does a closure capture the whole parent view implicitly?
4. 4Does the action capture a full domain model when only an ID is needed?
5. 5Are multiple non-visual actions mixed into otherwise simple visual row input?
6. 6Would a key-path binding express the same editable state more clearly?
7. 7Does a custom binding hide expensive derived reads or business rules?
8. 8Is `.equatable()` used only for cheap, complete, visual equality?
9. 9Would the refactor make update locality clearer without adding unnecessary architecture?

Focus on repeated views, broad captures, hidden work, and equality boundaries. Do not flag every closure in ordinary SwiftUI code.

## Stored closures in repeated views

A row that stores many closures mixes visual data with non-visual behavior.

Risky in a large or frequently updating collection:

```swift
struct PaymentRow: View {
    let row: PaymentRowModel
    let onOpen: () -> Void
    let onRetry: () -> Void
    let onCancel: () -> Void

    var body: some View {
        HStack {
            Text(row.title)
            Spacer()
            Text(row.amountText)
        }
        .contentShape(Rectangle())
        .onTapGesture(perform: onOpen)
        .swipeActions {
            Button("Retry", action: onRetry)
            Button("Cancel", role: .destructive, action: onCancel)
        }
    }
}
```

And the parent creates new closures while building every row:

```swift
ForEach(model.payments) { payment in
    PaymentRow(
        row: payment,
        onOpen: { model.openPayment(payment.id) },
        onRetry: { model.retryPayment(payment.id) },
        onCancel: { model.cancelPayment(payment.id) }
    )
}
```

This is not automatically wrong. It becomes suspicious when the collection is large, the parent updates often, rows are already expensive, or unnecessary row updates are suspected.

Prefer keeping the row focused on visual input, then attach interaction at a stable boundary when practical:

```swift
ForEach(model.payments) { payment in
    PaymentRow(row: payment)
        .contentShape(Rectangle())
        .onTapGesture { [model, id = payment.id] in
            model.openPayment(id)
        }
        .swipeActions {
            Button("Retry") { [model, id = payment.id] in
                model.retryPayment(id)
            }
            Button("Cancel", role: .destructive) { [model, id = payment.id] in
                model.cancelPayment(id)
            }
        }
}
```

This does not remove closures from the SwiftUI tree. It keeps the row's stored input visual and makes captured dependencies explicit.

## Capture lists and action inputs

Closures created inside `body` can accidentally capture the whole parent view value. In repeated content, prefer capture lists that make dependencies explicit and small.

Risky:

```swift
ForEach(model.transfers) { transfer in
    TransferRow(
        row: transfer,
        onSelect: {
            model.selectTransfer(transfer.id)
        }
    )
}
```

Prefer:

```swift
ForEach(model.transfers) { transfer in
    TransferRow(
        row: transfer,
        onSelect: { [model, id = transfer.id] in
            model.selectTransfer(id)
        }
    )
}
```

Prefer capturing:

- a stable model reference
- a stable service or action handler
- a stable row ID
- a small immutable value needed by the action

Avoid capturing:

- the whole parent view implicitly
- a large domain model when only an ID is needed
- a mutable row object when identity is enough
- a broad handler container when a narrower dependency is available
- values recomputed during every render

Passing the full model is fine when the action genuinely needs the full immutable value. Do not mechanically replace all values with IDs if that forces extra lookups or makes the code less correct.

## Action routing patterns

When a child must report several interactions to a parent, a compact action surface can be clearer than many independent closure properties.

```swift
enum CardAction {
    case open(Card.ID)
    case favorite(Card.ID)
    case dismiss(Card.ID)
}

struct CardRow: View {
    let row: CardRowModel
    let send: (CardAction) -> Void

    var body: some View {
        Button(row.title) {
            send(.open(row.id))
        }
    }
}
```

This still stores a closure. It is not a magic performance fix. Use this pattern when it reduces API noise, clarifies behavior, or prevents many separate stored closures from spreading through row APIs.

For very hot rows, also consider whether actions can be attached outside the purely visual row.

## Gestures, menus, and swipe actions

Gestures, menus, context menus, and swipe actions are interaction surfaces. In large collections, treat them as part of row complexity.

Review whether:

- every row really needs the interaction surface
- actions capture only stable IDs or narrow dependencies
- menu content is lightweight
- repeated action builders hide expensive work
- gesture state is owned locally where possible
- destructive actions have clear confirmation or state handling

Risky:

```swift
FeedRow(row: row)
    .contextMenu {
        ForEach(model.availableActions(for: row)) { action in
            Button(action.title) {
                model.perform(action, on: row)
            }
        }
    }
```

Prefer preparing lightweight action models before rendering when action computation is non-trivial:

```swift
struct FeedRowModel: Identifiable, Equatable {
    let id: FeedItem.ID
    let title: String
    let availableActions: [FeedActionModel]
}

FeedRow(row: row)
    .contextMenu {
        ForEach(row.availableActions) { action in
            Button(action.title) { [model, itemID = row.id, actionID = action.id] in
                model.perform(actionID, on: itemID)
            }
        }
    }
```

Do not add indirection just because a row has one tap gesture. Apply this guidance when repeated interaction builders become heavy or hard to reason about.

## Custom bindings

Prefer key-path bindings when no transformation is needed.

Good:

```swift
Toggle("Push notifications", isOn: $settings.pushNotificationsEnabled)
```

Risky as a default style:

```swift
Toggle(
    "Push notifications",
    isOn: Binding(
        get: { settings.pushNotificationsEnabled },
        set: { settings.pushNotificationsEnabled = $0 }
    )
)
```

A custom `Binding(get:set:)` is not automatically wrong. It often introduces fresh closures, broader captures, and hidden transformation logic inside rendering code.

Use a custom binding when the binding genuinely needs transformation, validation, optional handling, clamping, logging, routing, or compatibility with an API shape.

Valid use case:

```swift
TextField(
    "Limit",
    value: Binding(
        get: { draft.dailyLimit ?? 0 },
        set: { draft.dailyLimit = $0 == 0 ? nil : $0 }
    ),
    format: .number
)
```

When custom binding logic grows, move the behavior to a named helper or model method so the view does not hide business rules inside `body`.

## Key-path bindings with Observation

For Observation-based models, prefer `@Bindable` when a view needs editable bindings into an `@Observable` model.

```swift
@Observable
final class NotificationSettings {
    var pushEnabled = false
    var weeklySummaryEnabled = true
}

struct NotificationSettingsForm: View {
    @Bindable var settings: NotificationSettings

    var body: some View {
        Toggle("Push", isOn: $settings.pushEnabled)
        Toggle("Weekly summary", isOn: $settings.weeklySummaryEnabled)
    }
}
```

This keeps the binding relationship explicit and avoids replacing simple key-path access with custom closure bindings.

## Binding red flags

Flag these patterns in performance-sensitive SwiftUI code:

```swift
Binding(
    get: { model.expensiveDerivedValue },
    set: { model.apply($0) }
)
```

```swift
Binding(
    get: { formatter.string(from: value) },
    set: { text in model.update(text) }
)
```

```swift
Binding(
    get: { largeParentModel.items[index].isEnabled },
    set: { largeParentModel.items[index].isEnabled = $0 }
)
```

The issue is not the `Binding` type itself. The issue is hidden work, broad captures, index-based mutation risk, or transformation logic running on a hot rendering path.

## Equatable views

Use `Equatable` only when a view has clear, cheap, visual inputs and its body is expensive enough to justify the equality check.

Good candidate:

```swift
struct RateBadge: View, Equatable {
    let currencyCode: String
    let valueText: String
    let direction: Direction

    static func == (lhs: Self, rhs: Self) -> Bool {
        lhs.currencyCode == rhs.currencyCode &&
        lhs.valueText == rhs.valueText &&
        lhs.direction == rhs.direction
    }

    var body: some View {
        HStack {
            Text(currencyCode)
            Text(valueText)
            DirectionIcon(direction: direction)
        }
    }
}
```

Use `.equatable()` only when equality covers all visible inputs and is cheaper than recomputing the body.

Include in equality:

- all visual values that affect rendering
- style flags that change visible output
- values that affect layout, text, color, images, accessibility labels, visibility, or animation state

Avoid including:

- non-visual closures
- services
- model objects whose internal changes are not represented by compared values
- large arrays when equality is more expensive than recomputing the view
- volatile values that change every update

Do not exclude a value from equality if changing it should visibly update the view.

## Equatable and closures

Do not make action closures part of `Equatable` comparison. Swift closures generally do not have meaningful value equality.

Avoid `.equatable()` on a view whose behavior depends on changing closure values. If equality says the view is unchanged but the action changed, the code becomes harder to reason about.

Prefer a purely visual equatable row plus actions attached outside that row:

```swift
ContactRow(row: row)
    .equatable()
    .contentShape(Rectangle())
    .onTapGesture { [model, id = row.id] in
        model.openContact(id)
    }
```

For complex rows, it is often better to make the render model `Equatable` than to manually compare many view properties.

```swift
struct TransactionRowModel: Identifiable, Equatable {
    let id: Transaction.ID
    let title: String
    let subtitle: String
    let amountText: String
    let isPending: Bool
}
```

This works best when the render model is already prepared outside rendering and contains only display-ready values.

## When not to use `.equatable()`

Avoid `.equatable()` when:

- the view body is cheap
- equality is expensive
- inputs are large collections
- the view reads broad external state internally
- equality would omit values that affect visible output
- the view has hidden dependencies through environment, model references, or custom bindings
- the performance issue is actually identity, layout, drawing, or main-actor work

`Equatable` is a local optimization boundary, not a substitute for good dependency design.

## Validation

Use validation when unnecessary updates remain suspected after code review.

Useful checks:

- Add temporary `_printChanges()` to the suspected row or component.
- Count body invocations for the repeated view during the target interaction.
- Use the SwiftUI instrument to inspect body count, body duration, and unexpected updates.
- Use Time Profiler when action builders, binding transformations, menu construction, or equality checks perform non-trivial CPU work.
- Use Allocations when closures, action models, temporary strings, or render models are rebuilt repeatedly.
- Use signposts around filter changes, page appends, menu construction, or major state assignments.

Do not claim a numeric performance gain unless a trace, benchmark, signpost log, or user-provided measurement supports it.

## Risk levels

Low risk:

- a closure appears in a small static view
- a custom binding performs a tiny necessary transformation
- `.equatable()` uses complete and cheap visual equality

Medium risk:

- repeated rows store several action closures
- custom bindings capture a parent model in a large form or list
- closure capture lists are missing in a frequently updating parent
- `.equatable()` compares multiple values manually and could become incomplete over time

High risk:

- row closures capture large mutable models or whole parent views in a large collection
- custom bindings perform expensive derived reads or index-based mutations in repeated content
- `.equatable()` omits visible state
- `.equatable()` is used to hide broad invalidation instead of fixing dependencies
- action builders do non-trivial work for every row during rendering

## Suggested refactoring order

1. 1Keep visual row data separate from action behavior.
2. 2Capture stable IDs and narrow dependencies in closures.
3. 3Replace unnecessary custom bindings with key-path bindings.
4. 4Move binding transformation logic out of `body` when it grows.
5. 5Reduce multiple closure properties to a smaller action surface when useful.
6. 6Use `Equatable` only after visual inputs are clear and equality is cheap.
7. 7Validate with profiling or temporary probes only when unnecessary updates remain suspected.

## Common mistakes

- Do not say closures always make SwiftUI rows redraw.
- Do not say custom bindings are bad for performance.
- Do not add `.equatable()` as a default fix for list performance.
- Do not compare non-visual closures in equality.
- Do not omit visible state from equality to suppress updates.
- Do not add action-routing architecture for one simple tap handler.
- Do not move expensive binding transformation from `body` into another computed property that is still read by `body`.
- Do not call an optimization successful without a validation path.
