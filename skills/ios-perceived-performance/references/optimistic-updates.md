# Optimistic Updates

Use this reference when reviewing actions that can update the UI before server confirmation because the expected result is predictable, low-risk, and reversible.

This reference is about perceived responsiveness for reversible user actions. For loading states, read `references/loading-states.md`. For staged real-content rendering, read `references/progressive-rendering.md`. For financial, legal, destructive, irreversible, or trust-sensitive flows, read `references/high-stakes-actions.md`. For broader validation strategy, read `references/validation-and-testing.md`.

## Contents

- [Core idea](#core-idea)
- [What the agent can and cannot prove](#what-the-agent-can-and-cannot-prove)
- [Candidate filter](#candidate-filter)
- [Implementation model](#implementation-model)
- [State model](#state-model)
- [Pending state](#pending-state)
- [Rollback and reconciliation](#rollback-and-reconciliation)
- [Conflict and ordering](#conflict-and-ordering)
- [Repeated taps](#repeated-taps)
- [Retry, offline, and restart](#retry-offline-and-restart)
- [Cancellation](#cancellation)
- [High-stakes boundary](#high-stakes-boundary)
- [Implementation checklist](#implementation-checklist)
- [Validation](#validation)
- [Review output guidance](#review-output-guidance)

## Core idea

An optimistic update applies the expected local UI change immediately, sends the backend request afterward, and reconciles the UI when the backend responds.

It improves perceived latency because the user receives visual feedback without waiting for a network round trip. It does not make the backend operation faster and it does not remove the need for server confirmation.

Use optimistic updates only when the local prediction is safe enough and the app has a clear recovery path.

## What the agent can and cannot prove

The agent can inspect and improve:

- actions that wait for backend confirmation before changing simple local UI;
- missing pending, syncing, failed, retry, rollback, or reconciliation state;
- duplicate tap behavior;
- local state that can become inconsistent with server state;
- optimistic updates used for unsafe actions;
- state models that cannot represent pending, synced, and failed sync states.

The agent cannot prove from code alone:

- that the failure rate is low enough;
- that the backend conflict model is safe;
- that users will understand the pending state;
- that optimistic UI is acceptable for the product domain;
- that rollback behavior feels good in real use.

Do not claim:

- “This is always safe.”
- “This removes the need for server confirmation.”
- “This makes the operation complete instantly.”

Prefer:

- “This moves visual feedback out of the network round trip.”
- “This requires rollback and failure handling.”
- “This is appropriate only if the action is reversible and low-risk.”
- “Validate with failure simulation and repeated interaction tests.”

## Candidate filter

Optimistic updates are usually good candidates when the action is:

- reversible;
- local or user-specific;
- low-risk;
- easy to retry or roll back;
- unlikely to fail;
- unlikely to conflict with another user or device;
- not legally, financially, medically, or security significant.

Good examples:

- save or unsave an article;
- mute or unmute a topic;
- mark an item as read;
- dismiss a local tip;
- follow or unfollow a non-critical feed;
- change a local display preference;
- reorder local draft content;
- archive a low-risk notification with undo.

Avoid optimistic final success when the action is:

- financial, legal, destructive, irreversible, security-sensitive, medical, or identity-related;
- server-authoritative;
- likely to fail validation;
- dependent on inventory, balance, eligibility, or permissions;
- difficult to roll back;
- likely to conflict across devices or users.

Risky examples:

- sending money;
- deleting an account;
- confirming a purchase;
- accepting legal terms;
- changing security settings;
- submitting medical information;
- verifying identity;
- booking a scarce resource;
- showing final eligibility before server confirmation.

For these flows, prefer explicit progress and final confirmation. Read `references/high-stakes-actions.md`.

## Implementation model

Risky non-optimistic interaction:

```swift
@MainActor
func toggleSaved(id: Article.ID) async {
    let isSaved = try await articleAPI.toggleSaved(id: id)
    articles.update(id) { $0.isSaved = isSaved }
}
```

The UI waits for the backend before reflecting the tap.

Prefer optimistic feedback when the action is reversible:

```swift
@MainActor
func toggleSaved(id: Article.ID) {
    let previous = articles[id].isSaved
    let next = !previous
    let mutationID = UUID()

    articles.update(id) { article in
        article.isSaved = next
        article.pendingMutationID = mutationID
        article.syncState = .syncing(previous: previous)
    }

    syncTasks[id]?.cancel()
    syncTasks[id] = Task {
        await syncSavedState(
            id: id,
            desired: next,
            previous: previous,
            mutationID: mutationID
        )
    }
}
```

This gives immediate visual feedback while preserving enough information to reconcile, roll back, or ignore stale responses.

Use this shape only when the task is intentionally owned by the model or screen. If the action already runs inside an async parent scope, prefer structured concurrency.

## State model

Optimistic UI needs more than a final boolean.

Risky:

```swift
struct Article {
    var isSaved: Bool
}
```

This cannot show whether the local value is synced, pending, or failed.

Prefer explicit sync state:

```swift
struct Article {
    var id: Article.ID
    var isSaved: Bool
    var syncState: SyncState<Bool>
    var pendingMutationID: UUID?
}

enum SyncState<Value> {
    case synced
    case syncing(previous: Value)
    case failed(previous: Value, error: Error)
}
```

Keep the model as simple as the product needs. Do not add complex sync state for trivial local-only actions.

## Pending state

An optimistic update should not pretend the server already confirmed the result.

Show a pending state when confirmation matters:

- subtle spinner near the changed control;
- disabled repeated tap while syncing;
- temporary “Saving…” label;
- inline sync indicator;
- local pending badge;
- queued action state;
- undo affordance when appropriate.

Avoid noisy pending indicators for very low-risk actions where visible pending state would create more friction than value. Make the trade-off explicit.

## Rollback and reconciliation

Every optimistic update needs rollback or reconciliation.

Rollback is usually appropriate when:

- the failed action has no local value without server confirmation;
- the previous state is still valid;
- the change is easy to reverse;
- keeping the optimistic value would mislead the user.

Simple rollback shape:

```swift
func rollbackSavedState(id: Article.ID, previous: Bool, error: Error) {
    articles.update(id) { article in
        article.isSaved = previous
        article.syncState = .failed(previous: previous, error: error)
    }

    errorPresenter.show("Could not save. Restored the previous state.")
}
```

Rollback may be wrong when:

- the user has made additional changes after the original optimistic update;
- the server accepted a different canonical value;
- the same value changed on another device;
- the app supports offline-first behavior;
- the action should remain queued for later sync.

In these cases, reconcile with the server’s canonical value or keep a visible queued/failed state.

When the server responds, prefer applying the canonical result over assuming the local value is final:

```swift
let response = try await articleAPI.setSaved(next, for: id)

articles.update(id) { article in
    article.isSaved = response.isSaved
    article.syncState = .synced
}
```

If the server returns only success without canonical state, document that the client assumes the requested value was accepted.

## Conflict and ordering

Optimistic UI can conflict with server state.

Review what happens when:

- the server rejects the change;
- the server returns a different canonical value;
- another device changed the same item;
- the user taps repeatedly before the first request completes;
- the app goes offline after the optimistic update;
- the app terminates before confirmation;
- a later request completes before an earlier one.

Risky:

```swift
articles.update(id) { $0.isSaved = next }

Task {
    try? await articleAPI.setSaved(next, for: id)
}
```

This ignores failure, ordering, and reconciliation.

Use a mutation identifier, version, or server token when requests can complete out of order:

```swift
func applyServerResult(id: Article.ID, mutationID: UUID, confirmed: ArticleState) {
    articles.update(id) { article in
        guard article.pendingMutationID == mutationID else { return }
        article.isSaved = confirmed.isSaved
        article.pendingMutationID = nil
        article.syncState = .synced
    }
}
```

This prevents an older request from overwriting a newer local action.

## Repeated taps

Repeated taps are common in optimistic UI.

Choose one policy intentionally:

- disable the control while syncing;
- coalesce multiple taps into the latest desired value;
- cancel the previous request and send the latest value;
- queue mutations in order;
- allow immediate toggles but reconcile with mutation IDs;
- provide undo instead of repeated toggles.

For a simple toggle, latest-value-wins is often reasonable. For actions where every tap is meaningful, do not coalesce without product approval.

## Retry, offline, and restart

Failure recovery should be explicit.

Options:

- automatic retry for transient network failures;
- manual retry from an inline failed state;
- undo to previous state;
- keep queued change for offline sync;
- show error and restore previous state;
- fetch canonical server state.

Do not automatically retry forever. Use bounded retries and avoid hidden background work that users cannot understand or control.

If optimistic state can outlive the current screen, decide whether to persist it.

Ask:

- Should pending mutations survive app restart?
- Should failed mutations remain visible?
- Should the app retry when connectivity returns?
- Should local state be replaced by server state on next launch?
- Should the user be warned before leaving with unsynced changes?
- Does the backend provide idempotency keys or mutation IDs?

For local-only draft flows, persisting pending state can be correct. For server-authoritative state, blindly persisting optimistic values can mislead users.

## Cancellation

Cancellation policy must be explicit.

When the screen disappears, should the sync request continue?

Possible answers:

- cancel it because the action belongs only to the screen;
- continue it because the user already changed account-level state;
- queue it because the app supports offline-first sync;
- revert it because the action is not meaningful without immediate confirmation.

Do not assume that screen disappearance always cancels optimistic work. For account-level preferences, cancelling on disappearance may be wrong.

## High-stakes boundary

Do not use optimistic final success for high-stakes actions.

Risky:

```swift
func confirmPayment() {
    state = .paid

    Task {
        try await paymentService.confirm()
    }
}
```

This can mislead the user.

Prefer explicit progress and final confirmation after the authoritative system responds:

```swift
func confirmPayment() async {
    state = .submitting

    do {
        let receipt = try await paymentService.confirm()
        state = .confirmed(receipt)
    } catch {
        state = .failed(error)
    }
}
```

For financial, legal, destructive, irreversible, or security-sensitive operations, read `references/high-stakes-actions.md`.

## Implementation checklist

When proposing optimistic updates, check:

- [ ] Is the action reversible, low-risk, and predictable?
- [ ] Is the previous value captured before mutation?
- [ ] Is pending, syncing, failed, or queued state represented when needed?
- [ ] Is rollback or reconciliation implemented?
- [ ] Is failure visible to the user?
- [ ] Is retry behavior defined?
- [ ] Are repeated taps handled intentionally?
- [ ] Are stale or out-of-order responses ignored or reconciled?
- [ ] Does the server return canonical state?
- [ ] Is conflict handling defined?
- [ ] Is offline behavior defined when relevant?
- [ ] Is app restart behavior defined when relevant?
- [ ] Is cancellation policy explicit?
- [ ] Are high-stakes actions excluded?
- [ ] Is the improvement validated with failure simulation and interaction tests?

## Validation

Validate optimistic updates with both success and failure paths.

Recommended validation:

- normal success path;
- simulated network failure;
- slow network response;
- repeated tap during pending state;
- screen disappearance during sync;
- app restart with pending mutation if persistence is supported;
- server rejection;
- server returns a different canonical value;
- offline mode if supported;
- out-of-order response if multiple mutations can overlap.

Do not claim that optimistic UI is safe without testing failure and conflict paths.

Correct phrasing:

- “This gives immediate feedback while the backend request continues.”
- “This requires rollback and pending-state handling.”
- “Validate success, failure, repeated taps, and out-of-order responses.”

Avoid:

- “This makes the operation instant.”
- “This removes the need for backend confirmation.”
- “This is safe because the UI can always roll back.”

For broader validation strategy, read `references/validation-and-testing.md`.

## Review output guidance

When using this reference, explain:

```markdown
## Finding

The UI waits for server confirmation before reflecting a reversible user action.

## User impact

The interaction feels slower because the user does not receive immediate feedback.

## Recommended change

Apply the expected local state immediately, mark it as pending, send the backend request, then reconcile with the server response. Roll back or show a failed sync state if the request fails.

## Safety checks

Confirm that the action is low-risk, reversible, and not financially, legally, destructively, medically, or security sensitive.

## Validation

Test success, failure, repeated taps, cancellation or disappearance, and server conflict behavior.
```
