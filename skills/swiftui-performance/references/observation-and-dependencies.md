# Observation and Dependencies

Use this reference when reviewing SwiftUI code where the performance question is about dependency scope: `@Observable`, `ObservableObject`, `@Published`, environment reads, computed properties, or large shared models.

Do not use this reference for identity, list pagination, layout, drawing, animation, or async lifecycle issues unless the root cause is dependency scope.

The goal is to keep invalidation local: a state change should affect only the views that actually depend on that changed data.

## Core Rule

When reviewing dependencies, ask:

1. What data does this view read while building `body`?
2. Which changes can invalidate this view?
3. Can the read move closer to the smaller subview that actually needs it?

Explain the concrete dependency. Do not optimize by guessing.

Prefer this language:

> This view reads `model.user.name`, so its dependency should be scoped to the user name, not the entire app model.

Avoid vague language:

> SwiftUI redraws everything.

## `@Observable` vs `ObservableObject`

Prefer `@Observable` for iOS 17+ code paths when the deployment target, architecture, and existing Combine integration allow it.

`ObservableObject` with `@Published` has object-level invalidation. A view observing the object can be updated when any published property changes, even if the view visually uses only one field.

```swift
final class ViewModel: ObservableObject {
    @Published var name = ""
    @Published var count = 0
    @Published var items: [Item] = []
}

struct NameView: View {
    @ObservedObject var vm: ViewModel

    var body: some View {
        Text(vm.name)
    }
}
```

`NameView` visually needs only `name`, but changes to `count` or `items` still publish through the same observed object.

`@Observable` uses access-based tracking. SwiftUI tracks observable properties that are accessed while evaluating the view body.

```swift
@Observable
final class ViewModel {
    var name = ""
    var count = 0
    var items: [Item] = []
}

struct NameView: View {
    let vm: ViewModel

    var body: some View {
        Text(vm.name)
    }
}
```

Here, `NameView` reads `vm.name`. A change to `count` is not expected to invalidate this view through the `vm.name` dependency alone.

Still avoid absolute claims. The view can be re-evaluated for other reasons: parent updates, environment changes, identity changes, transactions, or other dependencies.

Prefer:

> With Observation, this view reads `vm.name`, so the dependency can stay scoped to `name`.

Avoid:

> This view redraws only when `name` changes.

"Redraw" is too vague. Talk about dependencies, invalidation, and body re-evaluation instead.

## Migration Rule

When reviewing code that can use Observation:

* prefer `@Observable` for new observable models
* remove unnecessary `@Published`
* pass observable models as regular values when no binding is needed
* use `@Bindable` only where a view needs bindings to mutable properties
* store a view-owned observable model in `@State`

Example:

```swift
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

Do not replace every `ObservableObject` blindly. Keep it when the code needs Combine publishers, supports older deployment targets, or relies on existing `objectWillChange` behavior.

## Pre-iOS 17 Rule

When the project cannot use `@Observable`, split large `ObservableObject` models into smaller dependency islands if unrelated changes are reaching the same views.

Bad:

```swift
final class AppViewModel: ObservableObject {
    @Published var title = ""
    @Published var items: [Item] = []
    @Published var selectedID: Item.ID?
    @Published var isSyncing = false
    @Published var errorMessage: String?
}

struct HeaderView: View {
    @ObservedObject var vm: AppViewModel

    var body: some View {
        Text(vm.title)
    }
}
```

`HeaderView` only needs `title`, but it observes the whole app model.

Better:

```swift
final class HeaderViewModel: ObservableObject {
    @Published var title = ""
}

final class ListViewModel: ObservableObject {
    @Published var items: [Item] = []
    @Published var selectedID: Item.ID?
}

struct HeaderView: View {
    @ObservedObject var vm: HeaderViewModel

    var body: some View {
        Text(vm.title)
    }
}
```

Do not split models just to make the architecture look cleaner. Split when it reduces a real dependency surface.

## Dependency Islands

A dependency island is a small view subtree that owns the reads for a specific part of observable state.

The key question is not whether to pass a model or a value. The key question is where the observable property is read.

If the parent reads the property:

```swift
ProductHeader(title: model.title)
```

then the parent `body` depends on `model.title`.

If the child reads the property:

```swift
ProductHeader(model: model)

struct ProductHeader: View {
    let model: ProductModel

    var body: some View {
        Text(model.title)
    }
}
```

then the child `body` depends on `model.title`.

Prefer passing narrow values when:

- the child is a reusable presentational component
- the parent already needs the same value
- the value is cheap and stable
- the dependency living in the parent is acceptable

Prefer passing an observable model when:

- the parent does not otherwise need the property
- you want the property read to belong to the child
- the child needs bindings through `@Bindable`
- the child owns a small dependency island
- passing separate values would force the parent to read many unrelated properties

Do not pass entire models automatically. Passing a model can hide dependencies in the child API. Passing values can move dependencies into the parent.

Choose based on where the dependency should live.

## Collection Dependencies

Be careful when a small view reads a whole collection.

```swift
struct BadgeView: View {
    let model: InboxModel

    var body: some View {
        Text("\(model.messages.filter(\.isUnread).count)")
    }
}
```

This ties the badge to the whole `messages` collection and performs work in `body`.

Prefer render-ready state with clear invalidation rules:

```swift
@Observable
final class InboxModel {
    var messages: [Message] = []
    var unreadCount = 0
}

struct BadgeView: View {
    let model: InboxModel

    var body: some View {
        Text("\(model.unreadCount)")
    }
}
```

For lists, avoid filtering, sorting, grouping, or formatting inside repeated content. Prepare visible rows before rendering or in a model with clear invalidation rules.

## Nested Models

Observation can track properties across observable references read in `body`, but do not use this as a reason to pass one giant object graph everywhere.

Prefer explicit, narrow dependencies when it makes the update path easier to reason about.

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

The second version hides the real dependency behind a broad app model.

## Red Flags

Flag these patterns:

* a view reads a whole model but uses one field
* a parent view extracts many model properties before passing values to children
* a large `ObservableObject` has many unrelated `@Published` properties
* a computed property does filtering, sorting, grouping, formatting, parsing, or database access
* a list reads global state in every row
* an observable model mixes UI state, services, caches, navigation, and networking
* `@ObservationIgnored` is used to hide real UI state changes
* migration to `@Observable` keeps old `ObservableObject` ownership patterns unnecessarily

## Measurement Ideas

When dependency scope is uncertain, suggest lightweight validation:

* add temporary `Self._printChanges()` in suspicious views
* add signposts around the interaction that changes state
* compare body update logs before and after moving reads into smaller subviews
* use Time Profiler only when static review and debug logs are not enough

Do not claim measured improvement unless a measurement was actually taken.

## Final Rule

Prefer the model shape that makes dependencies visible.

A good SwiftUI state design makes it obvious which view depends on which property.
