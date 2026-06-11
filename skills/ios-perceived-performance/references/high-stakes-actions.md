# High-Stakes Actions

Use this reference when reviewing flows where the app should not predict final success locally before server, backend, or another authoritative system confirms the result.

This reference is about confirmation, trust, and preventing misleading UI in financial, legal, destructive, irreversible, security-sensitive, medical, identity-related, or server-authoritative actions.

For reversible optimistic UI, read `references/optimistic-updates.md`. For loading indicators and state transitions, read `references/loading-states.md`. For perceived-performance validation, read `references/validation-and-testing.md`.

## Contents

- [Core idea](#core-idea)
- [What the agent can and cannot decide](#what-the-agent-can-and-cannot-decide)
- [High-stakes candidates](#high-stakes-candidates)
- [Decision rules](#decision-rules)
- [Recommended state model](#recommended-state-model)
- [Confirmation before starting](#confirmation-before-starting)
- [Progress during submission](#progress-during-submission)
- [Duplicate submission protection](#duplicate-submission-protection)
- [Server-authoritative result](#server-authoritative-result)
- [Pending and unknown outcomes](#pending-and-unknown-outcomes)
- [Destructive actions](#destructive-actions)
- [Security-sensitive changes](#security-sensitive-changes)
- [Trustworthy copy](#trustworthy-copy)
- [Deliberate delays](#deliberate-delays)
- [Agent review questions](#agent-review-questions)
- [Implementation checklist](#implementation-checklist)
- [Validation](#validation)
- [Review output guidance](#review-output-guidance)

## Core idea

High-stakes actions should not appear complete until the authoritative system confirms the result.

In low-risk flows, optimistic UI can improve perceived performance by showing the expected local result immediately. In high-stakes flows, premature success can mislead the user, create trust problems, or cause product, legal, financial, security, medical, or support issues.

The goal is not to make the action feel instant. The goal is to make the state honest, clear, and trustworthy.

## What the agent can and cannot decide

The agent can inspect and improve:

- UI that shows success before authoritative confirmation;
- missing progress state during submission, authorization, verification, or deletion;
- missing duplicate-submission protection;
- missing failure, pending, or unknown-outcome state;
- unclear final confirmation;
- destructive actions without an explicit confirmation step;
- local state that can diverge from server-authoritative state;
- optimistic UI where rollback would be unsafe or misleading;
- copy that implies completion too early.

The agent cannot decide alone:

- whether a product action is legally high-stakes;
- whether a financial, medical, security, or identity action has compliance requirements;
- whether a business domain permits local prediction;
- whether a delay, confirmation step, or audit trail is legally required;
- whether server-side semantics make an action reversible.

When risk is unclear, recommend confirming the behavior with product, design, backend, legal, security, or compliance owners.

## High-stakes candidates

Treat these as high-stakes by default:

- money transfer, payment authorization, purchase confirmation, trading, investment, or loan actions;
- account deletion or irreversible content deletion;
- legal consent, medical submission, identity verification, or eligibility submission;
- password, email, phone, recovery-method, two-factor, permission, privacy, or access-level changes;
- booking scarce inventory or actions that affect another person or organization;
- server-calculated pricing, fees, eligibility, authorization, verification, or final status.

Do not show final success for these actions until the authoritative response arrives.

## Decision rules

- Prefer explicit progress over optimistic final success.
- Treat `pending` as a separate state from `confirmed` when the backend can accept a request without completing it.
- Treat `unknown` as a separate state when the app may lose the final outcome after submission.
- Use the server response as the canonical result for receipts, IDs, pricing, fees, eligibility, authorization, and security state.
- Prevent repeated submission in the UI and guard against duplicate requests in the model.
- Suggest backend idempotency or request identifiers when duplicate submission would be harmful.
- Do not encourage retry when the operation may already have been submitted unless the backend is idempotent or the app can safely check status.
- Use confirmation prompts only when the consequence is meaningful, destructive, difficult to undo, or trust-sensitive.
- Do not claim a flow is safe unless success, failure, duplicate, timeout, and unknown-outcome paths are handled or tested.

## Recommended state model

High-stakes flows need explicit states that represent intent, progress, confirmed success, failure, pending, and unknown outcome when relevant.

Risky:

```swift
enum TransferState {
    case idle
    case done
}
```

This cannot distinguish waiting, confirmed success, failure, retry, pending review, or unknown outcome.

Prefer:

```swift
enum TransferState {
    case idle
    case reviewing(TransferDraft)
    case submitting
    case pending(SubmissionID)
    case confirmed(TransferReceipt)
    case failed(TransferError)
    case unknown(SubmissionID?)
}
```

Use names that match the backend semantics. Avoid names that imply certainty before confirmation.

## Confirmation before starting

For destructive, irreversible, or significant actions, add a separate confirmation step before the request starts.

A good confirmation step should:

- name the specific object or consequence;
- explain whether the action is reversible;
- avoid vague copy like “Are you sure?” without context;
- use destructive styling for the final confirmation action when the platform supports it;
- start the irreversible operation only after explicit confirmation.

Do not add confirmation prompts to every small action. Overusing confirmation makes users ignore them and can hurt responsiveness without improving trust.

## Progress during submission

High-stakes actions should acknowledge that work is happening immediately after the user confirms the action.

Use accurate progress states such as:

- submitting;
- authorizing;
- verifying;
- processing;
- waiting for confirmation;
- deleting;
- saving security change.

Avoid copy that implies final success too early. For example, do not show “Transfer complete…” while the request is still running. Prefer “Submitting transfer…” or “Waiting for bank confirmation…”.

For unknown duration, use an indeterminate progress indicator. For known multi-step work, show real step progress when available. Do not fake precise progress if the app does not know how much work remains.

## Duplicate submission protection

High-stakes flows should prevent accidental duplicate requests.

Check both layers:

- UI: disable or replace the confirmation control while submission is in progress;
- model: guard against repeated calls while a request is already active;
- backend: use idempotency or request identifiers when duplicate submission would be harmful.

Client-side disabling is useful for responsiveness and accidental taps, but it is not a complete safety guarantee. The agent can suggest idempotency, but cannot implement server guarantees from the client alone.

## Server-authoritative result

High-stakes flows should prefer the authoritative response as the canonical result.

Avoid inventing final local receipts, final prices, final eligibility, final authorization, final security state, or final deletion state before the server confirms them.

The server may calculate fees, reject the action, return a pending status, require additional verification, provide a canonical identifier, or return a final state that differs from the local draft.

## Pending and unknown outcomes

Some high-stakes actions may be accepted but not completed.

Examples:

- payment authorized but not captured;
- transfer submitted but pending review;
- booking requested but awaiting provider confirmation;
- identity verification submitted but under review;
- account change requested but awaiting email confirmation.

Represent pending separately from confirmed success. Do not collapse pending into success unless the product explicitly defines pending as a successful final state and communicates it clearly.

Unknown outcome needs a separate plan when:

- the request times out;
- the app loses connectivity after submission;
- the server accepts the request but the response is lost;
- the app is terminated during submission;
- an external authorization provider does not return a clear result.

For unknown outcomes, prefer checking status from the server, showing “checking status”, preventing immediate duplicate submission, and providing a safe support, history, or activity-link path.

Avoid showing success without confirmation, showing failure when the operation may have succeeded, or automatically retrying an operation that may already be submitted.

## Destructive actions

Destructive actions need extra care because rollback may be impossible, costly, or only visually simulated.

Review:

- Is the action clearly labeled?
- Is the destructive consequence clear?
- Is there a confirmation step before the operation starts?
- Is there a real undo path, or only a temporary visual rollback?
- Is server confirmation required before permanently removing the item from UI?
- What happens if deletion fails?
- What happens if the app closes during deletion?

For low-risk reversible deletion, optimistic removal with undo may be acceptable. For irreversible deletion, prefer explicit confirmation before starting the operation and final success only after authoritative confirmation.

If deletion fails, keep the item visible or restore it from a known local snapshot. Do not present the item as permanently deleted until the authoritative system confirms the result.

## Security-sensitive changes

Security-sensitive changes should not appear complete until confirmed.

Examples include changing password, email, phone number, two-factor settings, recovery method, account permissions, device access, privacy settings, or organization-level access.

Prefer pending or verification states when the final security state is not immediately confirmed:

```swift
enum EmailChangeState {
    case idle
    case submitting
    case verificationRequired(maskedEmail: String)
    case confirmed(String)
    case failed(Error)
}
```

Displayed account state should reflect the confirmed server state unless the product explicitly supports pending local changes and labels them clearly.

## Trustworthy copy

Copy should match the real state of the operation.

Avoid premature certainty:

- “Done” before confirmation;
- “Paid” before authorization;
- “Deleted” while deletion is pending;
- “Verified” before verification completes;
- “Your changes are saved” before the server accepts them.

Prefer accurate status:

- “Submitting…”;
- “Authorizing payment…”;
- “Waiting for confirmation…”;
- “Deleting…”;
- “Verification required”;
- “Request submitted”;
- “Pending review”;
- “Confirmed”.

For unknown results, use copy such as “We could not confirm the status”, “Checking status…”, or “Do not retry until the status is checked.”

Do not use scary copy unless the risk is real. High-stakes copy should be clear, not dramatic.

## Deliberate delays

Do not add artificial delays as a default way to make high-stakes operations feel trustworthy.

A high-stakes flow should feel trustworthy because it shows accurate states, confirmation, progress, pending state, final success after authoritative response, and safe failure recovery.

A short deliberate delay may be a product decision in narrow cases, but the agent should not introduce it unless the product requirement is explicit.

## Agent review questions

When reviewing a high-stakes action, ask:

1. What is the authoritative source of truth?
2. Can the local app safely predict success?
3. Is the action reversible?
4. What happens if the request fails?
5. What happens if the outcome is unknown?
6. Can repeated taps submit duplicate operations?
7. Is final success shown only after confirmation?
8. Does the copy distinguish submitting, pending, confirmed, failed, and unknown?
9. Is retry safe?
10. Is rollback real or only visual?
11. Does the operation need a confirmation step before starting?
12. Does the backend need idempotency or request tracking?
13. Is product, legal, security, or compliance review needed?

## Implementation checklist

When proposing changes for high-stakes actions, check:

- [ ] Optimistic final success is avoided.
- [ ] There is an explicit submitting, authorizing, verifying, deleting, or processing state.
- [ ] Final success appears only after authoritative confirmation.
- [ ] Pending is represented separately from confirmed success when needed.
- [ ] Unknown outcome is represented when possible.
- [ ] Duplicate submissions are prevented in the UI and guarded in the model.
- [ ] Backend idempotency or request tracking is suggested when duplicate submission would be harmful.
- [ ] Destructive intent is confirmed before the operation starts.
- [ ] Failure recovery is clear and safe.
- [ ] Retry behavior is safe.
- [ ] Copy is accurate and not prematurely certain.
- [ ] Server-authoritative data is not replaced by invented local final state.
- [ ] Compliance, product, legal, security, or backend unknowns are called out instead of guessed.

## Validation

Validate high-stakes flows with scenarios that cover success and uncertainty:

- normal success;
- backend rejection;
- network failure before submission;
- network failure after submission;
- timeout with unknown result;
- repeated taps;
- app backgrounding during submission;
- app termination during submission;
- retry after failure;
- retry after unknown outcome;
- server returns pending instead of confirmed;
- server returns a canonical result different from the local draft;
- destructive action cancellation;
- accessibility review for confirmation and error states.

Correct phrasing:

- “This avoids showing final success before confirmation.”
- “This makes the pending state explicit.”
- “Validate duplicate taps, timeout, and unknown-result behavior.”

Avoid:

- “This makes the operation faster.”
- “This guarantees the operation is safe.”
- “The user can just retry if something fails.”
- “Optimistic rollback is enough for this financial, destructive, or security action.”

For broader device, runtime, and production validation, read `references/validation-and-testing.md`.

## Review output guidance

When using this reference, explain:

```markdown
## Finding

The flow predicts success locally before the authoritative system confirms the result.

## User impact

The user may believe a financial, destructive, legal, security-sensitive, or otherwise high-stakes action completed when it is still pending, failed, or unknown.

## Recommended change

Use explicit confirmation, submitting, pending, confirmed, failed, and unknown states. Show final success only after the authoritative result arrives. Prevent duplicate submission and define safe failure recovery.

## Safety checks

Call out whether the action is reversible, whether retry is safe, whether idempotency is needed, and whether product/legal/security/compliance review is required.

## Validation

Test success, rejection, timeout, unknown outcome, repeated taps, retry, and app interruption.
```
