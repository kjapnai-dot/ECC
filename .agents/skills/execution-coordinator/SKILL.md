---
name: execution-coordinator
description: Turn an accepted request into a tracked implementation, independent verification, and recording run with fresh owners, evidence, and explicit failure recovery.
---

# Execution Coordinator

Use this skill between the conversation owner and the implementation, independent verification, and recording owners. It is for work that needs explicit delegation, versioned handoffs, progress visibility, or GitHub/Linear recording. The coordinator plans and reconciles the run; it does not implement, verify its own implementation, or perform external writes.

Read the project's routing rules and the [fresh-dispatch-recording](../fresh-dispatch-recording/SKILL.md) skill before dispatching. Use the project's existing ledger and prompt directory when they exist; do not create a competing source of truth.

## Boundaries

- The conversation owner accepts the request, owns the project canonical specifications, and gives the coordinator the work ID and authorized scope.
- The conversation owner keeps the first reply short and does not absorb long audits, implementation, independent verification, GitHub saves, or Linear updates; those activities remain delegated and asynchronous.
- The coordinator owns the decomposition card, fresh-chat assignments, state transitions, timestamped summaries, and handoffs.
- The implementation owner edits only the assigned scope and reports backup or recovery information and self-checks.
- The independent verifier uses a separate chat and a separate procedure or environment. Its `PASS`, `FAIL`, or `unconfirmed` result is independent evidence.
- The recording owner collects evidence and may write only explicitly delegated documents or reports. GitHub and Linear changes require the stated permission and a read-back.

The coordinator must not edit application code, tests, dependencies, shared specifications, or project state to make a handoff pass. It must not convert a missing report, a timeout, or a self-check into `PASS`.

## Decomposition card

Create one card per work ID before creating chats. Keep plan and observed result separate. Use one row for each owner and separate completion fields for GitHub and Linear, even when one recording owner handles both:

```text
Work ID / intake time / coordinator:
Objective / expected result / target / out of scope / acceptance conditions:
Reference version: path + commit, fixed snapshot, or artifact hash:
Permissions, approvals, and sharing method:

Implementation: new chat ID / prompt path / normalized instruction SHA256
  scope / state / started-at / changed files / backup / self-check / evidence / next action
Independent verification: different new chat ID / prompt path / SHA256
  version checked / separate procedure or environment / state / PASS|FAIL|unconfirmed / evidence / next action
GitHub recording: new chat ID / prompt path / SHA256
  delegated documents / write state / commit or remote / read-back / next action
Linear recording: new chat ID or the recording chat / issue / write state / read-back / next action

Unconfirmed, failures, and returns: scope / evidence / next owner / re-verification condition
Rollback: previous version or backup / executor / trigger / recovery check
Timestamped short log: JST time / state / owner / fact / evidence / next action
Final outcome: PASS|FAIL|unconfirmed; external save: complete|external_save_pending
```

Use the dispatch states from the fresh-dispatch skill: `planned`, `not_sent`, `sending`, `sent`, `received`, `working`, `verifying`, `returned`, `waiting`, `external_save_pending`, and `complete`. Keep `delivery_unknown` and `failed` visible when supported by the ledger. These dispatch states are separate from the outcome values `PASS`, `FAIL`, and `unconfirmed`.

## Dispatch and handoff

1. Confirm the work ID, objective, target, out-of-scope area, acceptance conditions, reference version, permissions, and existing ledger entries. If the work ID already has an initial dispatch, stop duplicate creation and route only an additional condition to the existing owner.
2. Split the request into implementation, independent verification, and recording. State the owner, allowed files or systems, evidence location, and return condition for each. Treat GitHub and Linear as separate recording gates; use separate fresh chats if their permissions or owners differ.
3. Normalize each exact instruction as UTF-8 with LF line endings, Unicode NFC, no trailing line whitespace, and one final LF. Reserve `(work_id, role, normalized_instruction_sha256)` with one send owner before sending. A hash or a manual check alone is not atomic duplicate prevention; without a lock or transaction, record the design as unconfirmed and wait.
4. Create a new chat for every assigned work. Reuse is allowed only with explicit user authorization recorded in the card. Save the exact prompt under the configured prompt directory. Send the short handoff card and minimum fixed references, then record a concrete send-success signal and the recipient's read-back. Never report `sent` for `reserved`, `sending`, `delivery_unknown`, or `waiting`.
5. On timeout, query the operation or chat ID. Do not create another chat or resend while delivery is unknown. Retry at most once in the same chat with the same reservation key, and only after non-delivery is confirmed. A second failure or unresolvable state returns to the conversation owner.
6. When implementation is complete, freeze the checked version and hand that version, changed files, backup, self-checks, unresolved items, and evidence location to the independent verifier. Do not ask the verifier to repeat the implementation owner's conclusion. After verification, hand the report and evidence to the recording owner.
7. Record progress as a short JST line containing state, owner, observed fact, evidence, and one next action. If a result is not available, write `unconfirmed` or `waiting`; do not describe an unstarted or blocked task as running in the background.

## Failure, rollback, and completion

For `FAIL` or `unconfirmed`, preserve the evidence, affected scope, next owner, and condition for re-verification. Return the work to the implementation or verification owner instead of fixing it inside the coordinator role. Re-run affected verification against the changed version; a prior `PASS` never transfers to a new version.

If rollback is required, name the previous fixed version or backup, trigger, authorized executor, and recovery check. The implementation owner performs an application rollback; the recording owner handles any authorized correction to external records. The coordinator records the result and does not run a destructive rollback itself. If recovery is unavailable, retain the failure and mark the work unresolved.

The card can be returned as complete only when the implementation evidence and recovery information exist, the independent verifier has a result, delegated documents are recorded, required GitHub and Linear read-backs are present, and no `FAIL`, `unconfirmed`, `delivery_unknown`, or `external_save_pending` remains. Missing external permission or connectivity is an explicit `external_save_pending` outcome, not completion.

Keep the card as an index of references, not a copied specification. Do not include secrets, personal data, or full conversation history in prompts or reports.
