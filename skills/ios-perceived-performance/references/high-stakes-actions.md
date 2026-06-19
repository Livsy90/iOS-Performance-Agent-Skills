# High-Stakes Actions

Use this reference when reviewing flows where the app must not present final success before an authoritative system confirms the result.

This reference is about confirmation, trust, honest progress, duplicate-submission protection, safe retry behavior, and preventing misleading UI in financial, legal, destructive, irreversible, security-sensitive, medical, identity-related, or otherwise authoritative actions.

For reversible optimistic UI, read `references/optimistic-updates.md`.
For loading indicators and state transitions, read `references/loading-states.md`.
For perceived-performance validation, read `references/validation-and-testing.md`.

## Contents

* [Core idea](#core-idea)
* [Authoritative source of truth](#authoritative-source-of-truth)
* [What the agent can and cannot decide](#what-the-agent-can-and-cannot-decide)
* [High-stakes candidates](#high-stakes-candidates)
* [Decision rules](#decision-rules)
* [Recommended state model](#recommended-state-model)
* [Failure vs unknown outcome](#failure-vs-unknown-outcome)
* [Persisted operation identity](#persisted-operation-identity)
* [Confirmation before starting](#confirmation-before-starting)
* [Progress during submission](#progress-during-submission)
* [Duplicate submission protection](#duplicate-submission-protection)
* [Server-authoritative result](#server-authoritative-result)
* [Pending and unknown outcomes](#pending-and-unknown-outcomes)
* [Cancellation semantics](#cancellation-semantics)
* [Destructive actions](#destructive-actions)
* [Security-sensitive changes](#security-sensitive-changes)
* [Trustworthy copy](#trustworthy-copy)
* [Privacy and sensitive data](#privacy-and-sensitive-data)
* [Deliberate delays](#deliberate-delays)
* [Agent review questions](#agent-review-questions)
* [Implementation checklist](#implementation-checklist)
* [Validation](#validation)
* [Review output guidance](#review-output-guidance)

## Core idea

High-stakes actions should not appear complete until the authoritative system confirms the result.

In low-risk flows, optimistic UI can improve perceived performance by showing the expected local result immediately. In high-stakes flows, premature success can mislead the user, create trust problems, or cause product, legal, financial, security, medical, or support issues.

The goal is not to make the action feel instant. The goal is to make the state honest, clear, recoverable, and trustworthy.

## Authoritative source of truth

The authoritative source of truth is the system that owns the final state of the operation.

It may be:

* a backend service;
* a payment provider;
* a banking or trading system;
* an identity or verification provider;
* a security provider;
* a legal or consent system;
* a secure local store such as Keychain or Secure Enclave;
* a local database in an offline-first app;
* another app, extension, device, or external service.

Do not assume that the local UI state is authoritative unless the product architecture explicitly defines it that way.

When the authoritative system is local, the same rule still applies: do not show final success until the authoritative local write, deletion, verification, or state transition has completed.

## What the agent can and cannot decide

The agent can inspect and improve:

* UI that shows success before authoritative confirmation;
* missing progress state during submission, authorization, verification, or deletion;
* missing duplicate-submission protection;
* missing failure, pending, or unknown-outcome state;
* unclear final confirmation;
* destructive actions without an explicit confirmation step;
* local state that can diverge from the authoritative state;
* optimistic UI where rollback would be unsafe or misleading;
* copy that implies completion too early;
* retry behavior that may duplicate an already-submitted operation;
* missing persisted request, submission, or idempotency identifiers.

The agent cannot decide alone:

* whether a product action is legally high-stakes;
* whether a financial, medical, security, or identity action has compliance requirements;
* whether a business domain permits local prediction;
* whether a delay, confirmation step, audit trail, receipt, or consent record is legally required;
* whether server-side semantics make an action reversible;
* whether retry is safe without knowing backend idempotency and status-check behavior.

When risk is unclear, recommend confirming the behavior with product, design, backend, legal, security, or compliance owners.

## High-stakes candidates

Treat these as high-stakes by default:

* money transfer, payment authorization, purchase confirmation, trading, investment, or loan actions;
* account deletion or irreversible content deletion;
* legal consent, medical submission, identity verification, or eligibility submission;
* password, email, phone, recovery-method, two-factor, permission, privacy, or access-level changes;
* booking scarce inventory or actions that affect another person or organization;
* server-calculated pricing, fees, eligibility, authorization, verification, or final status;
* actions where retry may create duplicates, charge twice, submit twice, delete twice, or change another person’s access.

Do not show final success for these actions until the authoritative response arrives or the authoritative local operation completes.

## Decision rules

* Prefer explicit progress over optimistic final success.
* Treat `pending` as a separate state from `confirmed` when the authoritative system can accept a request without completing it.
* Treat `unknown` as a separate state when the app may lose the final outcome after submission.
* Distinguish failure before submission from unknown outcome after submission.
* Use the authoritative response as the canonical result for receipts, IDs, pricing, fees, eligibility, authorization, verification, and security state.
* Persist operation identifiers when they are needed to check status after timeout, app termination, relaunch, or external authorization.
* Prevent repeated submission in the UI and guard against duplicate requests in the model.
* Suggest backend idempotency or request identifiers when duplicate submission would be harmful.
* Do not encourage retry when the operation may already have been submitted unless the backend is idempotent or the app can safely check status first.
* Use confirmation prompts only when the consequence is meaningful, destructive, difficult to undo, or trust-sensitive.
* Do not claim a flow is safe unless success, rejection, duplicate, cancellation, timeout, unknown, and relaunch paths are handled or tested.

## Recommended state model

High-stakes flows need explicit states that represent intent, progress, confirmed success, rejection, failure, pending, and unknown outcome when relevant.

Risky:

```swift
enum TransferState {
    case idle
    case done
}
```

This cannot distinguish waiting, confirmed success, server rejection, retry, pending review, cancellation, or unknown outcome.

Prefer:

```swift
enum TransferState {
    case idle
    case reviewing(TransferDraft)
    case submitting(requestID: UUID)
    case pending(SubmissionID)
    case confirmed(TransferReceipt)
    case rejected(TransferError)
    case failedBeforeSubmission(Error)
    case unknown(requestID: UUID?, submissionID: SubmissionID?)
}
```

Use names that match the authoritative system semantics. Avoid names that imply certainty before confirmation.

For example, `rejected` is usually clearer than `failed` when the authoritative system explicitly declines the operation. `unknown` is clearer than `failed` when the app cannot prove whether the operation completed.

## Failure vs unknown outcome

Do not collapse every error into the same UI state.

A high-stakes operation can fail in different ways:

```swift
enum SubmissionFailure {
    case validationFailed
    case rejectedByAuthoritativeSystem
    case networkFailedBeforeSubmission
    case timedOutAfterSubmission(requestID: UUID)
    case responseLostAfterSubmission(requestID: UUID, submissionID: SubmissionID?)
}
```

These states should not produce the same retry behavior.

A validation error or explicit rejection can usually allow correction and resubmission.

A network error before submission may allow retry if the request was never sent.

A timeout, lost response, app termination, or external-provider interruption after submission should usually move to an unknown or checking-status state before retry is allowed.

Avoid copy such as “Payment failed” when the app only knows that it could not confirm the payment status.

## Persisted operation identity

When an operation can remain pending or unknown, preserve the information needed to recover.

This may include:

* idempotency key;
* client request ID;
* backend submission ID;
* payment intent ID;
* authorization session ID;
* external provider reference;
* local database transaction ID;
* timestamp;
* operation type;
* masked destination or safe display summary.

Persist this identity before or at the moment the request is submitted, depending on the architecture.

After relaunch, the app should be able to:

* detect an unresolved operation;
* check authoritative status;
* prevent unsafe duplicate submission;
* show a clear pending, checking, confirmed, rejected, or unknown state;
* route the user to history, activity, support, or a safe recovery path.

Do not rely only on in-memory state for operations where timeout, app termination, or external authorization can happen.

## Confirmation before starting

For destructive, irreversible, or significant actions, add a separate confirmation step before the request starts.

A good confirmation step should:

* name the specific object or consequence;
* explain whether the action is reversible;
* avoid vague copy like “Are you sure?” without context;
* use destructive styling for the final confirmation action when the platform supports it;
* start the irreversible operation only after explicit confirmation.

Do not add confirmation prompts to every small action. Overusing confirmation makes users ignore them and can hurt responsiveness without improving trust.

When a real undo path exists, consider whether undo is better than a confirmation prompt. Do not call a visual rollback “undo” if the authoritative operation cannot actually be reversed.

## Progress during submission

High-stakes actions should acknowledge that work is happening immediately after the user confirms the action.

Use accurate progress states such as:

* submitting;
* authorizing;
* verifying;
* processing;
* waiting for confirmation;
* deleting;
* saving security change;
* checking status.

Avoid copy that implies final success too early.

Do not show:

```text
Transfer complete…
```

while the request is still running.

Prefer:

```text
Submitting transfer…
```

or:

```text
Waiting for bank confirmation…
```

For unknown duration, use an indeterminate progress indicator. For known multi-step work, show real step progress when available. Do not fake precise progress if the app does not know how much work remains.

## Duplicate submission protection

High-stakes flows should prevent accidental duplicate requests.

Check all layers:

* UI: disable or replace the confirmation control while submission is in progress;
* model: guard against repeated calls while a request is already active;
* persistence: store unresolved operation identity when needed;
* backend: use idempotency or request identifiers when duplicate submission would be harmful.

Client-side disabling is useful for responsiveness and accidental taps, but it is not a complete safety guarantee. The agent can suggest idempotency, but cannot implement server guarantees from the client alone.

Example client-side guard:

```swift
@MainActor
final class TransferModel {
    private(set) var state: TransferState = .idle

    func submit(_ draft: TransferDraft) async {
        guard case .idle = state else { return }

        let requestID = UUID()
        state = .submitting(requestID: requestID)

        do {
            let receipt = try await transferService.submit(
                draft,
                requestID: requestID
            )
            state = .confirmed(receipt)
        } catch let error as TransferRejection {
            state = .rejected(error)
        } catch {
            state = .unknown(requestID: requestID, submissionID: nil)
        }
    }
}
```

This example is only the client-side part. It does not replace backend idempotency or status checks.

## Server-authoritative result

High-stakes flows should prefer the authoritative response as the canonical result.

Avoid inventing final local receipts, final prices, final eligibility, final authorization, final security state, or final deletion state before the authoritative system confirms them.

The authoritative system may calculate fees, reject the action, return a pending status, require additional verification, provide a canonical identifier, or return a final state that differs from the local draft.

If the server returns a canonical result, render that result. Do not keep displaying stale local draft values as if they were final.

## Pending and unknown outcomes

Some high-stakes actions may be accepted but not completed.

Examples:

* payment authorized but not captured;
* transfer submitted but pending review;
* booking requested but awaiting provider confirmation;
* identity verification submitted but under review;
* account change requested but awaiting email confirmation.

Represent pending separately from confirmed success. Do not collapse pending into success unless the product explicitly defines pending as a successful final state and communicates it clearly.

Unknown outcome needs a separate plan when:

* the request times out;
* the app loses connectivity after submission;
* the authoritative response is lost;
* the app is terminated during submission;
* the app is backgrounded during an external authorization flow;
* an external authorization provider does not return a clear result;
* cancellation stops local waiting but does not prove that the operation was cancelled remotely.

For unknown outcomes, prefer checking status from the authoritative system, showing “checking status”, preventing immediate duplicate submission, and providing a safe support, history, or activity-link path.

Avoid:

* showing success without confirmation;
* showing failure when the operation may have succeeded;
* automatically retrying an operation that may already be submitted;
* clearing unresolved operation identity before the final status is known.

## Cancellation semantics

Cancellation of local UI work is not always cancellation of the operation.

In Swift, cancelling a `Task` usually cancels local waiting or local async work. It does not automatically cancel a server-side transfer, payment authorization, booking, deletion, or identity verification that has already been submitted.

Do not treat local cancellation as final remote cancellation unless the authoritative system explicitly confirms cancellation.

Risky copy:

```text
Transfer cancelled
```

when the app only cancelled local waiting.

Prefer:

```text
We stopped waiting for confirmation.
```

or:

```text
Checking transfer status…
```

when the operation may still be in progress.

If the product supports cancellation after submission, represent it as a separate authoritative operation with its own submitting, confirmed, rejected, pending, and unknown states.

## Destructive actions

Destructive actions need extra care because rollback may be impossible, costly, or only visually simulated.

Review:

* Is the action clearly labeled?
* Is the destructive consequence clear?
* Is there a confirmation step before the operation starts?
* Is there a real undo path, or only a temporary visual rollback?
* Is authoritative confirmation required before permanently removing the item from UI?
* What happens if deletion fails?
* What happens if deletion is pending?
* What happens if the app closes during deletion?
* Can retry delete the wrong object or duplicate the operation?

For low-risk reversible deletion, optimistic removal with undo may be acceptable. For irreversible deletion, prefer explicit confirmation before starting the operation and final success only after authoritative confirmation.

If deletion fails, keep the item visible or restore it from a known local snapshot. Do not present the item as permanently deleted until the authoritative system confirms the result.

## Security-sensitive changes

Security-sensitive changes should not appear complete until confirmed.

Examples include changing password, email, phone number, two-factor settings, recovery method, account permissions, device access, privacy settings, or organization-level access.

Prefer pending or verification states when the final security state is not immediately confirmed:

```swift
enum EmailChangeState {
    case idle
    case submitting(requestID: UUID)
    case verificationRequired(maskedEmail: String)
    case confirmed(String)
    case rejected(Error)
    case unknown(requestID: UUID?)
}
```

Displayed account state should reflect the confirmed authoritative state unless the product explicitly supports pending local changes and labels them clearly.

For example, if an email change requires verification, do not replace the confirmed account email with the new email as if it were already active. Show it as pending verification.

## Trustworthy copy

Copy should match the real state of the operation.

Avoid premature certainty:

* “Done” before confirmation;
* “Paid” before authorization;
* “Deleted” while deletion is pending;
* “Verified” before verification completes;
* “Your changes are saved” before the authoritative system accepts them;
* “Cancelled” when only local waiting was cancelled.

Prefer accurate status:

* “Submitting…”;
* “Authorizing payment…”;
* “Waiting for confirmation…”;
* “Deleting…”;
* “Verification required”;
* “Request submitted”;
* “Pending review”;
* “Checking status…”;
* “Confirmed”.

For unknown results, use copy such as:

* “We could not confirm the status.”
* “Checking status…”
* “Do not retry until the status is checked.”
* “This may already have been submitted.”

Do not use scary copy unless the risk is real. High-stakes copy should be clear, not dramatic.

## Privacy and sensitive data

High-stakes states often involve sensitive information.

Avoid exposing unnecessary details in progress, error, pending, or success states.

Be careful with:

* full account numbers;
* full card numbers;
* full email or phone values when masking is expected;
* identity document data;
* medical details;
* security settings;
* organization access details;
* internal backend error messages;
* raw provider failure codes.

Prefer safe summaries:

```text
Transfer to •••• 1234 is pending.
```

instead of exposing full destination details.

Error copy should be useful but not leak sensitive implementation or security information.

## Deliberate delays

Do not add artificial delays as a default way to make high-stakes operations feel trustworthy.

A high-stakes flow should feel trustworthy because it shows accurate states, confirmation, progress, pending state, final success after authoritative response, and safe failure recovery.

A short deliberate delay may be a product decision in narrow cases, but the agent should not introduce it unless the product requirement is explicit.

Do not make a real operation slower to simulate seriousness.

## Agent review questions

When reviewing a high-stakes action, ask:

1. What is the authoritative source of truth?
2. Can the local app safely predict success?
3. Is the action reversible?
4. What happens if validation fails?
5. What happens if the authoritative system rejects the request?
6. What happens if the request fails before submission?
7. What happens if the outcome is unknown after submission?
8. Can repeated taps submit duplicate operations?
9. Is final success shown only after confirmation?
10. Does the copy distinguish submitting, pending, confirmed, rejected, failed, cancelled, and unknown?
11. Is retry safe?
12. Is rollback real or only visual?
13. Does the operation need a confirmation step before starting?
14. Does the backend need idempotency or request tracking?
15. Is operation identity persisted for relaunch or status checks?
16. Does local cancellation actually cancel the authoritative operation?
17. Is product, legal, security, or compliance review needed?

## Implementation checklist

When proposing changes for high-stakes actions, check:

* [ ] Optimistic final success is avoided.
* [ ] There is an explicit submitting, authorizing, verifying, deleting, processing, or checking-status state.
* [ ] Final success appears only after authoritative confirmation.
* [ ] Pending is represented separately from confirmed success when needed.
* [ ] Unknown outcome is represented when possible.
* [ ] Failure before submission is distinguished from unknown outcome after submission.
* [ ] Duplicate submissions are prevented in the UI and guarded in the model.
* [ ] Backend idempotency or request tracking is suggested when duplicate submission would be harmful.
* [ ] Operation identity is persisted when timeout, relaunch, app termination, or external authorization can happen.
* [ ] Destructive intent is confirmed before the operation starts.
* [ ] Failure recovery is clear and safe.
* [ ] Retry behavior is safe.
* [ ] Cancellation semantics are not misleading.
* [ ] Copy is accurate and not prematurely certain.
* [ ] Sensitive information is masked or omitted where appropriate.
* [ ] Authoritative data is not replaced by invented local final state.
* [ ] Compliance, product, legal, security, or backend unknowns are called out instead of guessed.

## Validation

Validate high-stakes flows with scenarios that cover success and uncertainty:

* normal success;
* authoritative rejection;
* validation error before submission;
* network failure before submission;
* network failure after submission;
* timeout with unknown result;
* repeated taps;
* app backgrounding during submission;
* app termination during submission;
* app relaunch with unresolved operation;
* status check after relaunch;
* retry after clear rejection;
* retry after unknown outcome;
* cancellation before submission;
* cancellation after submission;
* external authorization interruption;
* authoritative system returns pending instead of confirmed;
* authoritative system returns a canonical result different from the local draft;
* destructive action cancellation before submission;
* destructive action failure after submission;
* accessibility review for confirmation, progress, pending, error, and success states;
* privacy review for sensitive data in copy and receipts.

Correct phrasing:

* “This avoids showing final success before confirmation.”
* “This makes the pending state explicit.”
* “This distinguishes failed-before-submission from unknown-after-submission.”
* “This preserves request identity so the app can check status after relaunch.”
* “Validate duplicate taps, timeout, cancellation, relaunch, and unknown-result behavior.”

Avoid:

* “This makes the operation faster.”
* “This guarantees the operation is safe.”
* “The user can just retry if something fails.”
* “Cancelling the task cancels the payment.”
* “Optimistic rollback is enough for this financial, destructive, or security action.”

For broader device, runtime, and production validation, read `references/validation-and-testing.md`.

## Review output guidance

When using this reference, explain:

```markdown
## Finding

The flow predicts success locally before the authoritative system confirms the result.

## User impact

The user may believe a financial, destructive, legal, security-sensitive, or otherwise high-stakes action completed when it is still pending, rejected, failed, cancelled locally, or unknown.

## Recommended change

Use explicit confirmation, submitting, pending, checking-status, confirmed, rejected, failed-before-submission, and unknown states. Show final success only after the authoritative result arrives. Prevent duplicate submission and define safe failure recovery.

## Safety checks

Call out whether the action is reversible, whether retry is safe, whether idempotency is needed, whether operation identity should be persisted, whether cancellation semantics are clear, and whether product/legal/security/compliance review is required.

## Validation

Test success, rejection, timeout, unknown outcome, repeated taps, cancellation, retry, app interruption, relaunch, and status recovery.
```
