# Observation and Dependencies

Use this reference when reviewing SwiftUI code where the performance question is about dependency scope: `@Observable`, `ObservableObject`, `@Published`, environment reads, computed properties, broad model dependencies, or splitting screens into dependency islands.

Do not use this reference for identity, list pagination, body cost, layout, drawing, animation, or async lifecycle issues unless the root cause is dependency scope.

The goal is to keep invalidation local: a state change should affect only the views that actually depend on the changed data.

## Contents

* [Agent goal](#agent-goal)
* [Core model](#core-model)
* [ObservableObject vs Observation](#observableobject-vs-observation)
* [Precise language](#precise-language)
* [Migration rule](#migration-rule)
* [Pre-iOS 17 rule](#pre-ios-17-rule)
* [Dependency islands](#dependency-islands)
* [Passing values vs passing models](#passing-values-vs-passing-models)
* [Environment dependencies](#environment-dependencies)
* [Collection dependencies](#collection-dependencies)
* [Computed properties as hidden dependencies](#computed-properties-as-hidden-dependencies)
* [Nested object graphs](#nested-object-graphs)
* [Validation](#validation)
* [Common mistakes](#common-mistakes)
* [Review checklist](#review-checklist)
* [Final rule](#final-rule)

## Agent goal

Help the user understand which state reads can cause a SwiftUI view to be re-evaluated and how to move those reads to the smallest useful boundary.

When reviewing code, answer these questions:

1. What data does this view read while building `body`?
2. Which changes can affect that dependency?
3. Is the read located in the smallest view that needs it?
4. Would passing a value, passing an observable model, or splitting a model make the dependency clearer?
5. How can the user validate the change without pretending it was measured?

## Core model

SwiftUI updates are driven by dependencies, identity, environment, transactions, and parent changes.

For this reference, focus on dependencies:

* A view depends on the observable values it reads while evaluating `body`.
* A broad read can make a large view subtree sensitive to unrelated state changes.
* Moving a read into a smaller child view can reduce the amount of view work affected by that state.
* Moving a read upward can accidentally make the parent depend on data that only a child needs.

Do not optimize by guessing. Identify the actual read that creates the dependency.

Prefer this language:

> This parent view reads `model.user.name`, so changes to `user.name` can re-evaluate the parent even though only the header needs the value.

Avoid vague language:

> SwiftUI redraws everything.

## ObservableObject vs Observation

`ObservableObject` with `@Published` commonly behaves at object granularity. A view observing the object can be updated when any published property changes, even if the view visually uses only one field.

Risky:

```swift
final class ProfileViewModel: ObservableObject {
    @Published var name = ""
    @Published var unreadCount = 0
    @Published var isSyncing = false
}

struct ProfileNameView: View {
    @ObservedObject var model: ProfileViewModel

    var body: some View {
        Text(model.name)
    }
}
```

`ProfileNameView` visually needs only `name`, but it observes the whole object.

Observation can track property reads more precisely. A view that reads one observable property does not need to depend on every property of the same model through that read alone.

Preferred for iOS 17+ when it fits the project:

```swift
@MainActor
@Observable
final class ProfileModel {
    var name = ""
    var unreadCount = 0
    var isSyncing = false
}

struct ProfileNameView: View {
    let model: ProfileModel

    var body: some View {
        Text(model.name)
    }
}
```

Here, the body reads `model.name`. A change to `unreadCount` is not expected to invalidate this view through the `model.name` dependency alone.

For UI-facing observable models, prefer explicit main-actor isolation when the model is mutated from async code or coordinates UI state. Pure domain models or background data models may have different isolation.

Still avoid absolute claims. A view can be re-evaluated for other reasons: parent updates, environment changes, identity changes, transactions, animations, or other dependencies.

## Precise language

Use dependency language, not vague redraw language.

Prefer:

> With Observation, this child view reads `model.name`, so the dependency can stay scoped to the child and to that property read.

Avoid:

> This view redraws only when `name` changes.

Prefer:

> This parent reads `model.title` before passing it down, so the parent now depends on `title` even if only the child displays it.

Avoid:

> Passing values is always faster than passing models.

## Migration rule

When reviewing code that can use Observation:

* prefer `@Observable` for new SwiftUI-facing observable models on iOS 17+ code paths when the project does not need Combine-based observation or older deployment support;
* make UI-facing observable model isolation explicit when async mutations or UI state coordination are involved;
* remove unnecessary `@Published` from migrated models;
* pass observable models as regular values when no binding is needed;
* use `@Bindable` only where a view needs bindings to mutable properties;
* store a view-owned observable model in `@State`;
* do not keep old `ObservableObject` ownership patterns automatically.

Example:

```swift
@MainActor
@Observable
final class ProfileModel {
    var name = ""
    var isPremium = false
}

struct ProfileScreen: View {
    @State private var model = ProfileModel()

    var body: some View {
        ProfileNameEditor(model: model)
    }
}

struct ProfileNameEditor: View {
    @Bindable var model: ProfileModel

    var body: some View {
        TextField("Name", text: $model.name)
    }
}
```

Use `@Bindable` only at the view boundary that creates bindings such as `$model.name`. Read-only child views should receive the model as a regular value.

Do not replace every `ObservableObject` blindly. Keep it when the code needs Combine publishers, supports older deployment targets, interacts with UIKit or Combine consumers, or relies on existing `objectWillChange` behavior.

## Pre-iOS 17 rule

When the project cannot use Observation, split large `ObservableObject` models into smaller observable objects if unrelated changes are reaching the same views.

Risky:

```swift
final class AppViewModel: ObservableObject {
    @Published var title = ""
    @Published var items: [Item] = []
    @Published var selectedID: Item.ID?
    @Published var isSyncing = false
    @Published var errorMessage: String?
}

struct HeaderView: View {
    @ObservedObject var model: AppViewModel

    var body: some View {
        Text(model.title)
    }
}
```

Better:

```swift
final class HeaderViewModel: ObservableObject {
    @Published var title = ""
}

final class ItemsViewModel: ObservableObject {
    @Published var items: [Item] = []
    @Published var selectedID: Item.ID?
}

struct HeaderView: View {
    @ObservedObject var model: HeaderViewModel

    var body: some View {
        Text(model.title)
    }
}
```

Do not split models just to make the architecture look cleaner. Split when it reduces a real dependency surface.

## Dependency islands

A dependency island is a small view subtree that owns the reads for a specific part of observable state.

The key question is not whether passing a model or a value is universally better. The key question is where the observable property is read.

If the parent reads the property:

```swift
struct ProductScreen: View {
    let model: ProductModel

    var body: some View {
        VStack {
            ProductHeader(title: model.title)
            ProductPrice(priceText: model.priceText)
        }
    }
}
```

then the parent `body` depends on `model.title` and `model.priceText`.

If children read the properties:

```swift
struct ProductScreen: View {
    let model: ProductModel

    var body: some View {
        VStack {
            ProductHeader(model: model)
            ProductPrice(model: model)
        }
    }
}

struct ProductHeader: View {
    let model: ProductModel

    var body: some View {
        Text(model.title)
    }
}

struct ProductPrice: View {
    let model: ProductModel

    var body: some View {
        Text(model.priceText)
    }
}
```

then each child owns its own property read.

This is useful only when it narrows dependencies or makes the update path clearer. It is not a rule to pass models everywhere.

## Passing values vs passing models

Prefer passing narrow values when:

* the child is a reusable presentational component;
* the parent already needs the same value;
* the value is cheap and stable;
* the dependency living in the parent is acceptable;
* the child should not know about the model type.

Prefer passing an observable model when:

* the parent does not otherwise need the property;
* the property read should belong to the child;
* the child needs bindings through `@Bindable`;
* the child owns a small dependency island;
* passing separate values would force the parent to read many unrelated properties.

Do not pass entire models automatically. Passing a model can hide dependencies in the child API. Passing values can move dependencies into the parent.

Choose based on where the dependency should live.

## Environment dependencies

Environment reads are dependencies too.

Risky:

```swift
struct ToolbarTitle: View {
    @Environment(AppModel.self) private var appModel

    var body: some View {
        Text(appModel.session.currentUser.displayName)
    }
}
```

This hides the dependency behind a broad environment model.

Prefer explicit dependencies for small presentational views:

```swift
struct ToolbarTitle: View {
    let displayName: String

    var body: some View {
        Text(displayName)
    }
}
```

Or pass a smaller observable model when the child should own the read:

```swift
struct ToolbarTitle: View {
    let user: UserModel

    var body: some View {
        Text(user.displayName)
    }
}
```

For Observation environment values, `@Environment(AppModel.self)` can track property reads more precisely, but broad environment access still hides dependencies and makes update paths harder to audit.

For legacy `@EnvironmentObject`, remember that the observed object usually has object-level invalidation.

## Collection dependencies

Be careful when a small view reads a whole collection.

Risky:

```swift
struct InboxBadge: View {
    let model: InboxModel

    var body: some View {
        Text("\(model.messages.filter(\.isUnread).count)")
    }
}
```

This has two problems:

* the badge depends on the whole `messages` collection;
* it performs filtering work during body evaluation.

Prefer render-ready state with clear invalidation rules:

```swift
@MainActor
@Observable
final class InboxModel {
    var messages: [Message] = []
    var unreadCount = 0

    func setMessages(_ newMessages: [Message]) {
        messages = newMessages
        unreadCount = newMessages.lazy.filter(\.isUnread).count
    }
}

struct InboxBadge: View {
    let model: InboxModel

    var body: some View {
        Text("\(model.unreadCount)")
    }
}
```

When storing aggregate values such as `unreadCount`, make the invalidation and update rule explicit. Derived state improves dependency scope only if it stays consistent with the source data.

For lists, avoid filtering, sorting, grouping, or formatting inside repeated content. Prepare visible rows or aggregate values before rendering. Use `body-cost-and-render-models.md` when the main issue is expensive transformation work rather than dependency scope.

## Computed properties as hidden dependencies

Computed properties can hide broad reads.

Risky:

```swift
@MainActor
@Observable
final class DashboardModel {
    var user: User
    var accounts: [Account]
    var marketState: MarketState

    var headerTitle: String {
        "\(user.name) · \(accounts.count) · \(marketState.statusText)"
    }
}

struct HeaderView: View {
    let model: DashboardModel

    var body: some View {
        Text(model.headerTitle)
    }
}
```

`HeaderView` looks like it reads one value, but `headerTitle` reads several pieces of state.

Prefer making the dependency visible:

```swift
struct HeaderView: View {
    let name: String
    let accountCount: Int
    let statusText: String

    var body: some View {
        Text("\(name) · \(accountCount) · \(statusText)")
    }
}
```

Pass explicit values when the parent already owns those reads or when they come from a prepared render model. If only the header needs the values, moving the reads into a smaller dependency island may be better.

Or build a render-ready header model when the formatting or aggregation is non-trivial.

Use `body-cost-and-render-models.md` when the computed property is expensive. Use this reference when the computed property hides dependencies.

## Nested object graphs

Observation can track reads through observable references when those references and properties participate in Observation. Do not assume every nested object graph gives precise invalidation automatically.

Good:

```swift
struct AccountHeader: View {
    let account: AccountModel

    var body: some View {
        Text(account.name)
    }
}
```

Risky:

```swift
struct AccountHeader: View {
    let appModel: AppModel

    var body: some View {
        Text(appModel.session.currentAccount.name)
    }
}
```

The second version hides the real dependency behind a broad app model. It also makes the component harder to reuse and harder to reason about.

Prefer explicit, narrow dependencies when they make the update path clearer.

## Validation

When dependency scope is uncertain, suggest lightweight validation before large refactors.

Useful checks:

* add temporary `Self._printChanges()` in suspicious views;
* log which state mutation happened before an unexpected update;
* add signposts around the interaction that changes state;
* compare body update logs before and after moving reads into smaller subviews;
* use the SwiftUI instrument when available to inspect body invocation count and update scope;
* use Time Profiler only when static review and debug logs are not enough.

Use `_printChanges()` only as a temporary local debugging probe and remove it before shipping.

Do not claim measured improvement unless a measurement was actually taken.

Prefer:

> This refactor should narrow the dependency from the parent screen to the header view. Validate by adding `_printChanges()` to both views and changing `model.title` before and after the refactor.

Avoid:

> This will make the screen 50% faster.

## Common mistakes

* Treating `@Observable` as a magic performance fix while leaving broad reads in large parent views.
* Passing a whole model into every child without checking where the properties are read.
* Passing only derived values when that forces the parent to read state that only the child needs.
* Splitting models only for architecture aesthetics, not dependency reduction.
* Hiding broad reads inside computed properties.
* Reading global environment models inside small presentational views.
* Reading an entire collection to display one aggregate value.
* Storing derived aggregate state without a clear consistency rule.
* Using `@ObservationIgnored` to hide state that should update the UI.
* Claiming a view updates only for one reason when parent updates, environment changes, identity, transactions, or animations can also re-evaluate it.

Use `@ObservationIgnored` only for state that should not drive UI updates, such as caches, services, diagnostics, or derived implementation details with explicit invalidation elsewhere.

## Review checklist

When reviewing dependency scope, check:

* Does the parent read properties that only children display?
* Does a small child observe a large `ObservableObject`?
* Does a computed property hide reads from multiple model fields?
* Does an environment read hide a broad dependency?
* Does a badge, header, or footer read a whole collection?
* Does the code use Observation but keep old object-level design patterns?
* Does the view need bindings, or can it receive values?
* Would moving the read into a child narrow updates?
* Would moving the read into a parent make updates broader?
* Is a proposed split tied to a real update problem?
* Is derived aggregate state kept consistent with its source data?
* Is there a validation step for suspected broad invalidation?

## Final rule

Prefer the model shape that makes dependencies visible.

A good SwiftUI state design makes it obvious which view depends on which property, and why that dependency lives there.
