# Identity and State

Use this reference when reviewing SwiftUI code that involves structural identity, explicit `.id(...)`, `ForEach` identity, conditional modifiers, local state lifetime, `@State`, `@StateObject`, `@ObservedObject`, or owned observable models.

This reference is about identity and state lifetime. Keep Observation dependency granularity, broad model reads, list pagination, closure-heavy rows, custom bindings, body cost, and profiling details in their dedicated references.

## Contents

* [Agent goal](#agent-goal)
* [Core mental model](#core-mental-model)
* [Structural identity](#structural-identity)
* [Conditional modifiers and identity](#conditional-modifiers-and-identity)
* [Explicit identity with `.id(...)`](#explicit-identity-with-id)
* [`ForEach` identity](#foreach-identity)
* [Local `@State` lifetime](#local-state-lifetime)
* [State ownership boundaries](#state-ownership-boundaries)
* [`@StateObject` and `@ObservedObject`](#stateobject-and-observedobject)
* [Owned `@Observable` models](#owned-observable-models)
* [State reset diagnostics](#state-reset-diagnostics)
* [Review language](#review-language)
* [Common mistakes](#common-mistakes)
* [Minimal checklist](#minimal-checklist)

## Agent goal

Help the user understand whether SwiftUI can preserve the right view identity and state across updates.

When reviewing code, answer:

* Does this view keep the same identity across normal updates?
* Is local state attached to a stable identity?
* Are collection rows identified by stable data identity rather than position or temporary values?
* Is the model owned by this view, injected into this view, or owned elsewhere?
* Would a branch, conditional modifier, explicit `.id(...)`, or unstable `ForEach` ID accidentally reset state?

Avoid vague claims such as "SwiftUI redraws everything." Prefer precise language about identity, lifetime, and ownership boundaries.

## Core mental model

SwiftUI view values are temporary descriptions. State is not stored inside the view value itself. SwiftUI preserves state by associating it with identity in the view hierarchy.

A view's identity usually comes from:

* its structural position in the view tree;
* the concrete view type at that position;
* explicit identity provided by APIs such as `ForEach` or `.id(...)`.

When identity changes, SwiftUI may treat the view as a different view. State attached to the previous identity can be discarded and new state can be created.

Review identity issues before suggesting lower-level optimization. Many SwiftUI correctness and performance bugs come from state being attached to the wrong lifetime boundary.

## Structural identity

SwiftUI normally derives identity from structure. Two branches that look visually similar can still represent different identities if they produce different concrete view types, wrappers, modifiers, or positions in the hierarchy.

Risky when the same conceptual view appears as different structure:

```swift
struct PaymentStatusView: View {
    let isPending: Bool

    var body: some View {
        if isPending {
            StatusCard(title: "Processing")
                .overlay {
                    ProgressView()
                }
        } else {
            StatusCard(title: "Complete")
        }
    }
}
```

This can be fine when the UI really has two separate states. But if the view should keep the same local state and only change values, prefer one stable structure:

```swift
struct PaymentStatusView: View {
    let isPending: Bool

    var body: some View {
        StatusCard(
            title: isPending ? "Processing" : "Complete",
            showsProgress: isPending
        )
    }
}
```

Prefer value-based modifiers for value changes such as opacity, disabled state, text, color, visibility, or simple styling when the same conceptual view should keep state. Keep structural branches when the UI is genuinely different.

Do not mechanically remove all branches. Branches are normal SwiftUI. Flag them only when they accidentally reset state, create unstable layout, or make a hot repeated view structurally unpredictable.

## Conditional modifiers and identity

Be careful with custom conditional modifier helpers such as:

```swift
extension View {
    @ViewBuilder
    func `if`<Content: View>(
        _ condition: Bool,
        transform: (Self) -> Content
    ) -> some View {
        if condition {
            transform(self)
        } else {
            self
        }
    }
}
```

This pattern is convenient, but it can create different structural identities for the modified and unmodified branches. When the condition changes dynamically, SwiftUI may treat the result as a different subtree. That can reset local `@State`, `@FocusState`, `@StateObject`, restart lifecycle-bound work, or produce surprising animation behavior.

Risky when the condition changes during the view lifetime:

```swift
Text(taskName)
    .padding()
    .if(isUrgent) { view in
        view
            .font(.system(size: 16, weight: .bold))
            .border(Color.red, width: 2)
    }
```

Prefer value-based modifiers when the conceptual view should remain the same and only modifier values change:

```swift
Text(taskName)
    .padding()
    .font(.system(size: 16, weight: isUrgent ? .bold : .regular))
    .border(isUrgent ? Color.red : Color.clear, width: 2)
```

Use conditional modifiers mainly when:

* the condition is static for the view lifetime;
* the modified and unmodified branches intentionally represent different structure;
* resetting local state and lifecycle work is acceptable;
* the helper improves readability without hiding identity changes.

Do not ban conditional modifiers mechanically. They are often fine for static configuration, platform-specific branches, feature flags that do not change while the view is alive, or structural differences that are intentional.

Flag them when the condition changes at runtime and the subtree contains local state, focus, animations, tasks, gestures, or row-local state.

## Explicit identity with `.id(...)`

Use `.id(...)` only when creating a deliberate identity boundary.

Good reasons:

* reset local state intentionally;
* make a detail or editor view start fresh for a different entity;
* define a scroll target;
* force a known lifecycle boundary after a meaningful identity change.

Risky:

```swift
InvoiceEditor(invoice: invoice)
    .id(UUID())
```

This creates a new identity every time the body is evaluated. It can reset local state, restart lifecycle work, and make update behavior difficult to reason about.

Prefer stable domain identity when a reset is intentional:

```swift
InvoiceEditor(invoice: invoice)
    .id(invoice.id)
```

Only use this if changing `invoice.id` should discard editor-local state such as draft fields, focus, validation state, temporary selections, or lifecycle-bound work.

Avoid using `.id(...)` as a generic refresh workaround. First check whether the real issue is stale derived state, wrong ownership, missing dependency updates, or async lifecycle behavior.

## `ForEach` identity

Dynamic collections need stable, unique, cheap identifiers. Row identity should come from the underlying data, not from the current array position or a temporary value.

Risky for mutable, filterable, reorderable, or pageable collections:

```swift
ForEach(Array(messages.enumerated()), id: \.offset) { _, message in
    MessageRow(message: message)
}
```

If items are inserted, removed, filtered, or reordered, offsets can change for existing rows. SwiftUI may associate old row state with the wrong item or recreate rows unnecessarily.

Prefer identity from the data:

```swift
ForEach(messages) { message in
    MessageRow(message: message)
}
```

or:

```swift
ForEach(messages, id: \.messageID) { message in
    MessageRow(message: message)
}
```

Avoid IDs that allocate or change on every access:

```swift
struct MessageRowModel: Identifiable {
    var id: UUID { UUID() }
    let title: String
}
```

Prefer stored identity:

```swift
struct MessageRowModel: Identifiable {
    let id: Message.ID
    let title: String
}
```

Use `id: \.self` only when the value is truly unique and stable for the lifetime of the collection. It is usually fine for a fixed list of unique strings or enum values. It is risky for duplicate values, mutable values, or values whose equality changes when visible content changes.

Index identity is acceptable only when identity is truly positional by product design: fixed tabs, fixed rating stars, fixed placeholders, or static settings rows with no insertion, deletion, filtering, or reordering.

## Local `@State` lifetime

Use `@State` for local value state owned by a stable view identity.

Good candidates:

* expansion state inside a component;
* local draft text;
* focus-related UI flags;
* temporary selection inside a picker-like component;
* small animation state;
* presentation toggles owned by the current view.

Keep `@State` private whenever possible. External code should not depend on a child view's private state storage.

Do not use `@State` as an accidental cache of parent input:

```swift
struct UserNameView: View {
    let userName: String
    @State private var displayedName: String

    init(userName: String) {
        self.userName = userName
        _displayedName = State(initialValue: userName)
    }

    var body: some View {
        Text(displayedName)
    }
}
```

This captures the initial value. Later changes to `userName` do not automatically update `displayedName`.

Prefer deriving directly when no local editing is needed:

```swift
struct UserNameView: View {
    let userName: String

    var body: some View {
        Text(userName)
    }
}
```

If local editing is needed, make the snapshot intentional:

```swift
struct EditableUserNameView: View {
    @State private var draftName: String

    init(initialName: String) {
        _draftName = State(initialValue: initialName)
    }

    var body: some View {
        TextField("Name", text: $draftName)
    }
}
```

In this case, `draftName` is a local draft. It should not automatically track every later parent value unless the product behavior requires that.

If the snapshot should reset for a different entity, attach the editor to a stable entity identity or handle the change explicitly. Do not expect `State(initialValue:)` to re-run for ordinary parent updates.

```swift
EditableUserNameView(initialName: user.name)
    .id(user.id)
```

Use this only if changing `user.id` should intentionally discard local draft state, focus, validation state, temporary selections, and lifecycle-bound work for the previous entity.

## State ownership boundaries

State should live at the smallest stable boundary that owns the behavior.

Prefer local ownership when state belongs to one component:

```swift
struct WalletScreen: View {
    @State private var selectedFilter = Filter.all

    var body: some View {
        VStack {
            WalletSearchField()
            BalanceCard()
            TransactionFilter(selection: $selectedFilter)
            NicknameEditor()
        }
    }
}
```

This does not guarantee fewer updates by itself. It makes lifetime, ownership, and dependency boundaries easier to reason about.

Lift state up when multiple components need the same source of truth or when the parent coordinates behavior. Do not push state down so far that synchronization becomes unclear.

## `@StateObject` and `@ObservedObject`

Use `@StateObject` when a SwiftUI view creates and owns an `ObservableObject`.

Risky:

```swift
struct RatesView: View {
    @ObservedObject private var model = RatesModel()

    var body: some View {
        RatesContent(model: model)
    }
}
```

The view declares ownership but uses a wrapper meant for externally owned observable objects.

Prefer:

```swift
struct RatesView: View {
    @StateObject private var model = RatesModel()

    var body: some View {
        RatesContent(model: model)
    }
}
```

Use `@ObservedObject` when the object is owned elsewhere and injected:

```swift
struct RatesContent: View {
    @ObservedObject var model: RatesModel
}
```

For `ObservableObject`, use `@ObservedObject` in the child when the child needs to observe changes. For Observation models, passing the model reference is enough for property access tracking. Use `@Bindable` only when the child needs bindings.

Do not create an owned, stateful observable model as a plain stored property in a view. SwiftUI view values can be recreated, and a plain stored reference does not express SwiftUI-managed lifetime.

A plain stored property is fine for immutable dependencies, externally owned services, value inputs, or references whose lifetime is intentionally managed elsewhere.

Do not claim `@StateObject` is faster than `@ObservedObject`. The main distinction is ownership and lifetime.

## Owned `@Observable` models

For iOS 17 and later, an `@Observable` model owned by a view can be stored with `@State`.

```swift
@MainActor
@Observable
final class RatesModel {
    var rows: [RateRowModel] = []
    var isRefreshing = false
}

struct RatesView: View {
    @State private var model = RatesModel()

    var body: some View {
        RatesContent(model: model)
    }
}
```

Use this when the view owns the model's lifetime.

For UI-facing observable models, prefer making main-actor isolation explicit when the model is mutated from async code or coordinates UI state. This does not mean every `@Observable` type in the app must be `@MainActor`; pure domain models or background data models may have different isolation.

If a parent owns the model, inject it into the child instead of creating it again:

```swift
struct RatesContent: View {
    let model: RatesModel
}
```

If the child needs editable bindings into an `@Observable` model, use `@Bindable`:

```swift
struct RatesContent: View {
    @Bindable var model: RatesModel

    var body: some View {
        Toggle("Refreshing", isOn: $model.isRefreshing)
    }
}
```

This reference covers ownership and lifetime. Detailed Observation dependency behavior belongs in `observation-and-dependencies.md`.

## State reset diagnostics

Suspect accidental identity changes when the user reports:

* text fields lose input unexpectedly;
* focus disappears during updates;
* scroll position resets without intent;
* rows lose expansion or selection state after filtering;
* async `.task` work restarts repeatedly;
* animations restart during unrelated updates;
* row-local state appears attached to the wrong item after insertion or deletion.

Check:

* Is `.id(UUID())` or another unstable ID used?
* Does a `ForEach` use offsets, indices, or mutable values as identity?
* Does a row model compute a new ID on every access?
* Does conditional structure replace one stateful view with another?
* Does a conditional modifier hide runtime structural changes?
* Is local state initialized from parent input and then expected to track later changes?
* Is an owned model created with `@ObservedObject` or a plain stored property?
* Is state owned too high or too low in the tree?

## Review language

Use precise wording:

```md
This `ForEach` identifies rows by offset. If the array is filtered, reordered, or prepended, existing row state can become associated with a different item. Use a stable domain ID instead.
```

```md
This `.id(UUID())` creates a fresh identity on every update. That can reset local state and restart lifecycle-bound work. Use a stable domain ID only if changing that ID should intentionally reset the subtree.
```

```md
This view creates its own observable model, so `@StateObject` is the correct ownership wrapper for `ObservableObject`. Use `@ObservedObject` only when the model is injected from an external owner.
```

```md
This `@State` value is initialized from a parent input, so it behaves like an initial snapshot. If the view should always show the latest parent value, derive it directly from the input instead.
```

```md
This conditional modifier helper can produce different structure when the condition changes. If this subtree needs stable local state or smooth animation, prefer value-based modifiers or an explicit structural branch with intentional identity.
```

## Common mistakes

* Do not add `.id(...)` everywhere. Explicit identity is a tool, not a default requirement.
* Do not replace every branch with value-based modifiers. Structural branching is fine when it expresses genuinely different UI.
* Do not hide runtime structural changes behind custom conditional modifier helpers when the view needs stable local state or smooth animation.
* Do not move all state to the parent. Local state is often better for local UI behavior.
* Do not move all state to children. Shared source of truth still belongs at the coordinating owner.
* Do not use offset or index identity for mutable or reorderable collections.
* Do not create IDs with `UUID()` in an `id` computed property.
* Do not use `@State(initialValue:)` as if it automatically tracks later parent input changes.
* Do not claim `@StateObject` is a performance optimization over `@ObservedObject`; it is an ownership rule.
* Do not claim Observation removes the need to think about identity. Observation changes dependency tracking, not identity rules.

## Minimal checklist

Before finishing an identity/state review, verify:

* IDs are stable, unique, and cheap.
* `.id(...)` is intentional and not used as a refresh hack.
* Mutable collections do not use offset or index identity unless identity is intentionally positional and order/membership are fixed.
* Conditional modifier helpers are used only for static conditions or intentional structural changes.
* Local `@State` is attached to a stable structural position.
* `@State` initialized from input is either an intentional snapshot or replaced with derived input.
* Snapshot state resets intentionally when the represented entity changes.
* `ObservableObject` created by the view uses `@StateObject`.
* `ObservableObject` owned elsewhere uses `@ObservedObject` or is passed through without claiming ownership.
* Owned `@Observable` models use `@State` when the deployment target and architecture support Observation.
* UI-facing observable models have clear actor isolation when mutated from async code.
* State is neither lifted too high nor pushed too low for the behavior being modeled.
