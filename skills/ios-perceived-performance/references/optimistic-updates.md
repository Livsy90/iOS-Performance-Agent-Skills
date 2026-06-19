# Optimistic Updates

Use this reference when reviewing actions that can update the UI before authoritative confirmation because the expected result is predictable, low-risk, reversible, and easy to reconcile.

This reference is about perceived responsiveness for reversible user actions. It covers optimistic local changes, pending state, rollback, reconciliation, repeated taps, stale responses, offline behavior, restart behavior, and conflict handling.

For loading states, read `references/loading-states.md`.
For staged real-content rendering, read `references/progressive-rendering.md`.
For financial, legal, destructive, irreversible, server-authoritative, or trust-sensitive flows, read `references/high-stakes-actions.md`.
For broader validation strategy, read `references/validation-and-testing.md`.

## Contents

* [Core idea](#core-idea)
* [What the agent can and cannot prove](#what-the-agent-can-and-cannot-prove)
* [Candidate filter](#candidate-filter)
* [Prefer set-style mutations](#prefer-set-style-mutations)
* [Implementation model](#implementation-model)
* [State model](#state-model)
* [Display-safe errors](#display-safe-errors)
* [Pending state](#pending-state)
* [Rollback and reconciliation](#rollback-and-reconciliation)
* [Derived state reconciliation](#derived-state-reconciliation)
* [Conflict and ordering](#conflict-and-ordering)
* [Repeated taps](#repeated-taps)
* [Retry, offline, and restart](#retry-offline-and-restart)
* [Persisted mutation metadata](#persisted-mutation-metadata)
* [Cancellation](#cancellation)
* [High-stakes boundary](#high-stakes-boundary)
* [Implementation checklist](#implementation-checklist)
* [Validation](#validation)
* [Review output guidance](#review-output-guidance)

## Core idea

An optimistic update applies the expected local UI change immediately, sends the authoritative request afterward, and reconciles the UI when the authoritative system responds.

It improves perceived latency because the user receives visual feedback without waiting for a network round trip. It does not make the backend operation faster and it does not remove the need for confirmation, rollback, or reconciliation.

Use optimistic updates only when the local prediction is safe enough and the app has a clear recovery path.

A good optimistic update answers four questions:

* What local value should appear immediately?
* How does the UI show that the value is not confirmed yet?
* What happens if the authoritative system rejects or changes the result?
* What happens if requests complete out of order, fail, retry, or survive app restart?

## What the agent can and cannot prove

The agent can inspect and improve:

* actions that wait for backend confirmation before changing simple local UI;
* missing pending, syncing, failed, retry, rollback, queued, or reconciliation state;
* duplicate tap behavior;
* local state that can become inconsistent with authoritative state;
* optimistic updates used for unsafe actions;
* state models that cannot represent pending, synced, failed, and queued sync states;
* requests that can complete out of order and overwrite newer local state;
* use of toggle-style APIs where set-style mutations would be safer;
* missing handling for cancellation, offline state, app restart, or stale responses;
* derived UI state that is not reconciled, such as counts, badges, list membership, or cached collections.

The agent cannot prove from code alone:

* that the failure rate is low enough;
* that the backend conflict model is safe;
* that users will understand the pending state;
* that optimistic UI is acceptable for the product domain;
* that rollback behavior feels good in real use;
* that cancellation of a local task cancels a mutation already sent to the backend;
* that retry is safe without understanding idempotency or mutation semantics.

Do not claim:

* “This is always safe.”
* “This removes the need for server confirmation.”
* “This makes the operation complete instantly.”
* “Rollback is enough for any failed optimistic update.”
* “Cancelling the task cancels the backend mutation.”

Prefer:

* “This moves visual feedback out of the network round trip.”
* “This requires pending state, rollback, and reconciliation.”
* “This is appropriate only if the action is reversible and low-risk.”
* “Validate with failure simulation, repeated interaction tests, and out-of-order response tests.”

## Candidate filter

Optimistic updates are usually good candidates when the action is:

* reversible;
* local or user-specific;
* low-risk;
* easy to retry or roll back;
* unlikely to fail;
* unlikely to conflict with another user or device;
* not legally, financially, medically, security-sensitive, identity-related, or trust-sensitive.

Good candidates may include:

* save or unsave an article;
* mute or unmute a topic;
* mark an item as read;
* dismiss a local tip;
* change a local display preference;
* reorder local draft content;
* archive a low-risk notification when undo or recovery is available;
* follow or unfollow a non-critical feed when product risk is low and conflicts are acceptable.

Treat these as product-dependent, not automatically safe:

* follow or unfollow people, organizations, or private feeds;
* archive, hide, or delete server-backed records;
* actions that affect notifications, privacy, visibility, permissions, recommendations, or another user’s experience.

Avoid optimistic final success when the action is:

* financial, legal, destructive, irreversible, security-sensitive, medical, or identity-related;
* server-authoritative and difficult to reconcile;
* likely to fail validation;
* dependent on inventory, balance, eligibility, pricing, authorization, or permissions;
* difficult to roll back;
* likely to conflict across devices or users;
* meaningful to another person or organization;
* likely to create support, legal, financial, privacy, or trust issues if the UI is wrong.

Risky examples:

* sending money;
* deleting an account;
* confirming a purchase;
* accepting legal terms;
* changing security settings;
* submitting medical information;
* verifying identity;
* booking a scarce resource;
* showing final eligibility before server confirmation.

For these flows, prefer explicit progress and final confirmation. Read `references/high-stakes-actions.md`.

## Prefer set-style mutations

Optimistic updates are safer when the authoritative API accepts the desired final state.

Prefer set-style mutations:

```swift id="vdn3nm"
try await articleAPI.setSaved(true, for: id)
```

over toggle-style mutations:

```swift id="4sxl83"
try await articleAPI.toggleSaved(id: id)
```

Set-style mutations are easier to reason about because the request says what final state the client wants. They are usually safer for:

* retries;
* duplicate requests;
* out-of-order responses;
* idempotency;
* reconciliation;
* latest-value-wins behavior.

Toggle-style APIs can be risky because retrying or duplicating the same request may produce the opposite of the intended state.

If the backend only exposes a toggle operation, the agent should call out the risk and recommend one of these mitigations:

* add a set-style endpoint;
* include expected previous value;
* include mutation ID or version;
* refetch canonical state after mutation;
* avoid optimistic UI if wrong final state would be confusing or unsafe.

## Implementation model

Risky non-optimistic interaction:

```swift id="4p31g5"
@MainActor
func toggleSaved(id: Article.ID) async {
    let isSaved = try await articleAPI.toggleSaved(id: id)
    articles.update(id) { $0.isSaved = isSaved }
}
```

The UI waits for the backend before reflecting the tap, and the toggle-style API can be hard to reconcile if retry or duplication occurs.

Prefer optimistic feedback with desired final state when the action is reversible:

```swift id="8udkqo"
@MainActor
func setSaved(_ desired: Bool, id: Article.ID) {
    let previous = articles[id].isSaved
    let mutationID = UUID()

    articles.update(id) { article in
        article.isSaved = desired
        article.pendingMutationID = mutationID
        article.syncState = .syncing(previous: previous)
    }

    syncTasks[id]?.cancel()
    syncTasks[id] = Task {
        await syncSavedState(
            id: id,
            desired: desired,
            previous: previous,
            mutationID: mutationID
        )
    }
}
```

This gives immediate visual feedback while preserving enough information to reconcile, roll back, or ignore stale responses.

Use this shape only when the task is intentionally owned by the model or screen. If the action already runs inside an async parent scope, prefer structured concurrency.

Unstructured `Task` is reasonable only when the model intentionally owns the mutation lifecycle beyond the immediate view callback. Do not recommend detached or unstructured work as a default pattern.

## State model

Optimistic UI needs more than a final boolean.

Risky:

```swift id="8plkbr"
struct Article {
    var isSaved: Bool
}
```

This cannot show whether the local value is synced, pending, failed, or queued.

Prefer explicit sync state:

```swift id="g7s5zv"
struct Article {
    var id: Article.ID
    var isSaved: Bool
    var syncState: SyncState<Bool>
    var pendingMutationID: UUID?
}

enum SyncState<Value> {
    case synced
    case syncing(previous: Value)
    case failed(previous: Value, error: DisplayError)
    case queued(previous: Value)
}
```

Keep the model as simple as the product needs. Do not add complex sync state for trivial local-only actions.

For small reversible actions, a minimal state may be enough:

```swift id="fd06ex"
enum SyncState {
    case synced
    case syncing
    case failed(DisplayError)
}
```

Use richer state when rollback, retry, offline queue, or reconciliation needs the previous value.

## Display-safe errors

Do not expose raw technical errors directly through user-facing optimistic state.

Risky:

```swift id="dil7jk"
enum SyncState<Value> {
    case synced
    case syncing(previous: Value)
    case failed(previous: Value, error: Error)
}
```

Raw `Error` values may be:

* too technical;
* not localized;
* not `Equatable`;
* unsafe to display;
* missing recovery guidance;
* inconsistent across service layers.

Prefer display-safe error values at the UI boundary:

```swift id="zsc1au"
struct DisplayError: Equatable {
    let title: String
    let message: String
    let retryTitle: String
}
```

Keep technical errors available for logging, analytics, or diagnostics when appropriate, but do not make visible UI copy depend on raw implementation details.

## Pending state

An optimistic update should not pretend the authoritative system already confirmed the result.

Show a pending state when confirmation matters:

* subtle spinner near the changed control;
* disabled repeated tap while syncing;
* temporary “Saving…” label;
* inline sync indicator;
* local pending badge;
* queued action state;
* undo affordance when appropriate.

Example:

```swift id="x5nr5g"
Button {
    model.setSaved(!article.isSaved, id: article.id)
} label: {
    HStack {
        Image(systemName: article.isSaved ? "bookmark.fill" : "bookmark")

        if case .syncing = article.syncState {
            ProgressView()
        }
    }
}
.disabled(article.isInteractionDisabled)
```

Avoid noisy pending indicators for very low-risk actions where visible pending state would create more friction than value. Make the trade-off explicit.

If the UI shows the optimistic value immediately, ensure the product is comfortable with the value looking selected before confirmation. For some actions, a subtle pending marker is enough; for others, the state must remain visibly unconfirmed.

## Rollback and reconciliation

Every optimistic update needs rollback or reconciliation.

Rollback is usually appropriate when:

* the failed action has no local value without authoritative confirmation;
* the previous state is still valid;
* the change is easy to reverse;
* keeping the optimistic value would mislead the user.

Simple rollback shape:

```swift id="8jqku6"
@MainActor
func rollbackSavedState(
    id: Article.ID,
    previous: Bool,
    error: DisplayError
) {
    articles.update(id) { article in
        article.isSaved = previous
        article.syncState = .failed(previous: previous, error: error)
        article.pendingMutationID = nil
    }

    errorPresenter.show("Could not save. Restored the previous state.")
}
```

Rollback may be wrong when:

* the user has made additional changes after the original optimistic update;
* the server accepted a different canonical value;
* the same value changed on another device;
* the app supports offline-first behavior;
* the action should remain queued for later sync;
* the optimistic value affects derived state that also needs correction.

In these cases, reconcile with the authoritative canonical value or keep a visible queued or failed state.

When the authoritative system responds, prefer applying the canonical result over assuming the local value is final:

```swift id="5zyhsg"
let response = try await articleAPI.setSaved(next, for: id)

articles.update(id) { article in
    article.isSaved = response.isSaved
    article.pendingMutationID = nil
    article.syncState = .synced
}
```

If the server returns only success without canonical state, document that the client assumes the requested value was accepted.

## Derived state reconciliation

Optimistic changes often affect more than one property.

Review derived state such as:

* counts;
* badges;
* section membership;
* list membership;
* filter results;
* cached collections;
* search results;
* related detail screens;
* recommendation state;
* notification state;
* local widgets or app extensions.

Example: saving an article may affect both the row and the saved count.

Risky:

```swift id="b4w5hx"
article.isSaved = true
```

if the screen also shows:

```swift id="2j6563"
savedArticlesCount
```

Prefer updating or invalidating related local state intentionally:

```swift id="0xc708"
articles.update(id) { $0.isSaved = true }
savedArticlesCount += 1
```

Then reconcile both values when the server responds:

```swift id="jr64xn"
let response = try await articleAPI.setSaved(true, for: id)

articles.update(id) { $0.isSaved = response.article.isSaved }
savedArticlesCount = response.savedArticlesCount
```

Do not let optimistic derived values become permanent when the canonical server result differs.

If derived state is complex, prefer refetching the affected summary, invalidating a cache, or applying a server-provided canonical response.

## Conflict and ordering

Optimistic UI can conflict with authoritative state.

Review what happens when:

* the server rejects the change;
* the server returns a different canonical value;
* another device changed the same item;
* the user taps repeatedly before the first request completes;
* the app goes offline after the optimistic update;
* the app terminates before confirmation;
* a later request completes before an earlier one;
* a local task is cancelled after the backend request was already sent.

Risky:

```swift id="hd88s1"
articles.update(id) { $0.isSaved = next }

Task {
    try? await articleAPI.setSaved(next, for: id)
}
```

This ignores failure, ordering, cancellation, and reconciliation.

Use a mutation identifier, version, or server token when requests can complete out of order:

```swift id="jye5jg"
@MainActor
func applyServerResult(
    id: Article.ID,
    mutationID: UUID,
    confirmed: ArticleState
) {
    articles.update(id) { article in
        guard article.pendingMutationID == mutationID else { return }

        article.isSaved = confirmed.isSaved
        article.pendingMutationID = nil
        article.syncState = .synced
    }
}
```

This prevents an older request from overwriting a newer local action.

When the backend supports versioning, prefer using server versions, ETags, revision IDs, or mutation tokens for stronger reconciliation.

## Repeated taps

Repeated taps are common in optimistic UI.

Choose one policy intentionally:

* disable the control while syncing;
* coalesce multiple taps into the latest desired value;
* send a new set-style request for the latest desired value;
* queue mutations in order;
* allow immediate toggles but reconcile with mutation IDs;
* provide undo instead of repeated toggles.

For a simple toggle, latest-value-wins is often reasonable. For actions where every tap is meaningful, do not coalesce without product approval.

Be careful when cancelling a previous local sync task. Cancelling local waiting does not guarantee that a request already sent to the backend was cancelled.

If the first request may still complete on the backend, the app must still handle its response, ignore it using mutation identity, or reconcile with canonical state later.

## Retry, offline, and restart

Failure recovery should be explicit.

Options:

* automatic retry for transient network failures;
* manual retry from an inline failed state;
* undo to previous state;
* keep queued change for offline sync;
* show error and restore previous state;
* fetch canonical authoritative state.

Do not automatically retry forever. Use bounded retries and avoid hidden background work that users cannot understand or control.

If optimistic state can outlive the current screen, decide whether to persist it.

Ask:

* Should pending mutations survive app restart?
* Should failed mutations remain visible?
* Should the app retry when connectivity returns?
* Should local state be replaced by authoritative state on next launch?
* Should the user be warned before leaving with unsynced changes?
* Does the backend provide idempotency keys, versions, or mutation IDs?
* Can the app safely determine whether a mutation was already submitted?

For local-only draft flows, persisting pending state can be correct. For server-authoritative state, blindly persisting optimistic values can mislead users.

## Persisted mutation metadata

When optimistic mutations can survive screen disappearance, offline mode, retry, or app restart, persist enough metadata to recover safely.

Useful metadata may include:

* mutation ID;
* target entity ID;
* operation type;
* desired final value;
* previous value;
* created timestamp;
* retry count;
* current status;
* idempotency key;
* server version or revision when available;
* account or user scope;
* safe display summary.

Example:

```swift id="zra7uv"
struct PendingMutation<Value: Codable>: Codable, Identifiable {
    let id: UUID
    let entityID: String
    let operation: String
    let desiredValue: Value
    let previousValue: Value
    let createdAt: Date
    var retryCount: Int
    var status: PendingMutationStatus
    let idempotencyKey: UUID
}

enum PendingMutationStatus: Codable {
    case queued
    case syncing
    case failed
}
```

On app restart, the app should be able to:

* show queued or failed optimistic changes accurately;
* retry safely when appropriate;
* fetch canonical state when retry is unsafe;
* avoid duplicating a mutation that may already have been submitted;
* clear resolved mutations after reconciliation.

Do not persist optimistic state without a recovery plan.

## Cancellation

Cancellation policy must be explicit.

When the screen disappears, should the sync request continue?

Possible answers:

* cancel it because the action belongs only to the screen;
* continue it because the user already changed account-level state;
* queue it because the app supports offline-first sync;
* revert it because the action is not meaningful without immediate confirmation.

Do not assume that screen disappearance always cancels optimistic work. For account-level preferences, cancelling on disappearance may be wrong.

Do not assume that cancelling a Swift `Task` cancels a mutation already accepted by the backend. It may only cancel local waiting.

Risky:

```swift id="x6zs25"
syncTask.cancel()
article.syncState = .synced
```

This may hide an unresolved mutation.

Prefer one of these explicit outcomes:

* keep the mutation pending until the backend confirms;
* mark it as queued for later retry;
* refetch canonical state;
* roll back if the request was not submitted;
* show unknown or failed sync state if the outcome cannot be confirmed.

For high-stakes operations, local cancellation is especially sensitive. Read `references/high-stakes-actions.md`.

## High-stakes boundary

Do not use optimistic final success for high-stakes actions.

Risky:

```swift id="yamgr7"
func confirmPayment() {
    state = .paid

    Task {
        try await paymentService.confirm()
    }
}
```

This can mislead the user.

Prefer explicit progress and final confirmation after the authoritative system responds:

```swift id="0hfn3m"
func confirmPayment() async {
    state = .submitting

    do {
        let receipt = try await paymentService.confirm()
        state = .confirmed(receipt)
    } catch {
        state = .failed(mapError(error))
    }
}
```

For financial, legal, destructive, irreversible, medical, identity-related, or security-sensitive operations, read `references/high-stakes-actions.md`.

## Implementation checklist

When proposing optimistic updates, check:

* [ ] Is the action reversible, low-risk, and predictable?
* [ ] Is the action outside financial, legal, destructive, medical, identity, security, or other high-stakes domains?
* [ ] Does the API express desired final state rather than ambiguous toggle semantics?
* [ ] Is the previous value captured before mutation?
* [ ] Is pending, syncing, failed, or queued state represented when needed?
* [ ] Is rollback or reconciliation implemented?
* [ ] Is failure visible to the user when it matters?
* [ ] Is retry behavior defined?
* [ ] Are repeated taps handled intentionally?
* [ ] Are stale or out-of-order responses ignored or reconciled?
* [ ] Does the server return canonical state?
* [ ] Are derived values such as counts, badges, lists, and caches reconciled?
* [ ] Is conflict handling defined?
* [ ] Is offline behavior defined when relevant?
* [ ] Is app restart behavior defined when relevant?
* [ ] Is persisted mutation metadata stored when needed?
* [ ] Is cancellation policy explicit?
* [ ] Are local task cancellation and backend mutation cancellation treated separately?
* [ ] Is the improvement validated with failure simulation and interaction tests?

## Validation

Validate optimistic updates with both success and failure paths.

Recommended validation:

* normal success path;
* simulated network failure;
* slow network response;
* repeated tap during pending state;
* latest-value-wins behavior;
* queued mutation behavior if supported;
* screen disappearance during sync;
* app restart with pending mutation if persistence is supported;
* server rejection;
* server returns a different canonical value;
* derived count, badge, and list reconciliation;
* offline mode if supported;
* retry after failure;
* out-of-order response if multiple mutations can overlap;
* cancellation after request submission;
* canonical state refetch after uncertainty.

Do not claim that optimistic UI is safe without testing failure and conflict paths.

Correct phrasing:

* “This gives immediate feedback while the backend request continues.”
* “This requires rollback and pending-state handling.”
* “This uses desired final state, which is safer than a toggle-style mutation.”
* “Validate success, failure, repeated taps, out-of-order responses, and derived state reconciliation.”

Avoid:

* “This makes the operation instant.”
* “This removes the need for backend confirmation.”
* “This is safe because the UI can always roll back.”
* “Cancelling the task cancels the backend mutation.”
* “A toggle request is fine because the UI already changed.”

For broader validation strategy, read `references/validation-and-testing.md`.

## Review output guidance

When using this reference, explain:

```markdown id="7xi9tt"
## Finding

The UI waits for authoritative confirmation before reflecting a reversible, low-risk user action.

## User impact

The interaction feels slower because the user does not receive immediate feedback.

## Recommended change

Apply the expected local state immediately, mark it as pending, send a set-style authoritative request, then reconcile with the canonical response. Roll back, retry, queue, or show a failed sync state if the request fails.

## Safety checks

Confirm that the action is low-risk, reversible, and not financially, legally, destructively, medically, identity-related, security-sensitive, or otherwise trust-sensitive. Check repeated taps, cancellation, offline behavior, restart behavior, idempotency, and derived state reconciliation.

## Validation

Test success, failure, repeated taps, cancellation or disappearance, retry, offline or restart behavior when relevant, server conflict, out-of-order responses, and canonical reconciliation.
```
