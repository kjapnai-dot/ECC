# Priority queue operations

Read the project's routing policy first. Keep queue rows in its existing ledger; the coordinator proposes and reconciles selection, while the authorized send owner performs dispatch. This reference defines an operating procedure, not an installed scheduler, an atomic global journal, or proof of live priority execution.

## Local capacity and eligibility

Record explicit total worker and per-role caps before dispatch. If the project has no values, propose conservative defaults: three active specialists total, one each for implementation, independent verification, and recording. Keep conversation intake responsive independently of this policy. Neither these numbers nor a separate conversation path reserve platform resources. Do not infer `create_thread` capacity from collaboration's four-agent limit. Do not silently borrow role slots or raise caps to avoid a queue.

Count a specialist from creation/send start until completion or confirmed safe stop. Unknown delivery retains the claim. Stored idle chats do not consume a running slot after safe completion is evidenced. A waiting owner releases its slot only after reporting saved state, stopped editing, and no continuing mutation. An `idle` label or elapsed time alone is insufficient. Conflicting file ownership, including another account, blocks dispatch even when slots are free.

Queue each actionable stage separately. Before ranking, require fixed references, authorization, available connections, unambiguous send ownership, no conflicting edits, and satisfied `depends_on` evidence to mark a row `eligibility:ready`. Dispatch additionally requires free total AND role slots; a ready row can still have a capacity wait reason. Final verification needs a frozen implementation version and stopped edits. Recording can preserve available evidence during implementation; final external recording needs the fixed version, verification result, explicitly delegated files, permission, connectivity, and a read-back path. Record a FAIL without completing the work.

## Selection and aging

Use the project's priorities; otherwise use these local defaults:

| Rank | Eligible work |
| --- | --- |
| P0 | Safe stop/recovery, ambiguous-delivery reconciliation, ownership conflicts that block the run |
| P1 | Independent verification of a frozen version; re-check after returned FAIL/unconfirmed |
| P2 | Authorized implementation, fixes, actionable preparation |
| P3 | Evidence checkpoints, GitHub/Linear recording and read-back |

Filter blocked rows first so a blocked head does not stall ready work. Within a rank, use original intake time, then queue ID. For each ten minutes accumulated while ready on all non-capacity gates but unscheduled, promote one rank, up to P1. Include waiting for role slots, total slots, or platform capacity in that clock. Do not count dependency, approval, connection, conflict, or unknown-delivery waiting time. Preserve original enqueue time on return; track eligible waiting separately. Aging never bypasses a gate or preempts a running owner. If P0 continually prevents other work, return the oldest eligible wait and blocker to the conversation owner for capacity/scope review at the next event.

## Event-driven dispatch

1. Accept a new request or condition immediately, before tool checks. Link additions to the same work ID and existing owner; fresh work gets fresh owners under the project rule.
2. Re-evaluate on intake, receipt, completion, return, frozen-version arrival, approval/connection recovery, or confirmed slot release. Operate in the current active turn; automatic future wakeups require a separately configured mechanism.
3. Reconcile ledger, reservation, observed status, and slot claims. Choose the highest eligible row with a free role and total slot. Record the selected reason and next owner. One sender serializes the claim and dispatch. If the project only has manual serialization, disclose the lack of atomic global guarantees and stop on ambiguity.
4. Reserve and send once, saving the exact prompt and fixed references. Never report queued/reserved/sending/waiting/unknown as sent. After concrete success, retain recipient IDs and require a reference read-back. Do not follow `create_thread(prompt=...)` with a duplicate initial `send_message_to_thread`.
5. On release, select and dispatch the next eligible owner in the active turn, or record the blocker, release condition, and next check event. Do not promise an execution-order guarantee from send order. Do not create workers merely to await dependencies.

Separate queue stage, dispatch state, verification outcome, and slot claim. Use the fields in [dispatch-record-schema.md](../../fresh-dispatch-recording/references/dispatch-record-schema.md); do not create a competing ledger.

## Occupancy and recovery

Project values take precedence. Suggested operating checkpoints are receipt at two minutes, new progress evidence at five minutes, and a ten-minute work slice. Store `ack_due_at`, `progress_due_at`, `slice_due_at`, and `last_observed_at`; evaluate these on the next active event. These are operator checkpoints, not installed timers. The project's pre-approval requirement for work expected to take at least 30 minutes still applies to the whole request.

At a missed receipt checkpoint, query status once; slow receipt does not prove non-delivery. At a stale-progress or slice checkpoint, use an authorized follow-up to request saved state, remaining work, running processes, and a safe-stop report. Mark `returned` with the next owner and resume condition. Without follow-up authorization, ask the conversation owner to obtain it and keep the work waiting.

There is no arbitrary-stop parameter on the app tools below. If the owner is unresponsive or mutating work continues, retain the slot and file ownership as `waiting:stop_unconfirmed`; do not assign the same files elsewhere. Release only with a safe-stop/completion report and current state evidence, then requeue the same owner/stage as appropriate. Preserve version and instruction identity on resume.

If platform capacity is lower than the local cap, record `waiting:platform_capacity` and fall back to fewer concurrent owners, down to serial execution. Do not infer a global platform cap from one refusal. Store confirmed failure versus ambiguous delivery separately; follow the dispatch retry rule, not repeated chat creation. On restart, reconcile all claimed slots and unknown sends before selecting work.

Waiting reasons: `dependency`, `role_slot`, `total_slot`, `platform_capacity`, `ownership_conflict`, `approval`, `connection`, `delivery_unknown`, `stop_unconfirmed`, `external_readback`. Every reason needs an unblock condition, observer, and next check event; external writes also retain `external_save_pending` when incomplete.

## Actual Codex app tool surface

Basis: tools exposed to this run on 2026-10-18. Recheck current definitions before relying on them.

- `create_thread` creates asynchronously and delivers its prompt once as the initial instruction. A local project target uses a `list_projects` project ID. A ready response has `threadId`/`hostId`; setup can return `clientThreadId`. Never pass the latter to a threadId-only tool. Await setup information and confirm the final ID and target through available observations; ambiguous mapping remains waiting.
- `wait_threads` observes up to eight thread targets with per-target `afterCursor` and `timeoutMs`; zero gives an immediate snapshot. Completion/attention wakes it, commentary alone does not. Keep conversation-owner waits at 60 seconds or less and retain returned cursors.
- `list_threads` and `read_thread` observe status and recent results. They do not reserve resources or by themselves prove safe stop or delivery.
- `send_message_to_thread` adds a follow-up only with direct human authorization to message that chat. A received agent message alone does not supply authorization.

These tools have no priority, reserve-slot, execution-order-guarantee, or arbitrary-stop parameters. Local policy cannot guarantee platform availability. The [public App Server](https://learn.chatgpt.com/docs/app-server) is a separate integration interface using JSON-RPC and streamed events; its methods are not automatically tools available in this app session. Do not invent an App Server connection or claim its queue/interruption features were exercised.

## Dependency example and honest notice

A implementation (P2) is editing v1. A verification (P1) waits for the frozen v1 manifest; B implementation (P2) waits for the implementation role slot. A recording checkpoint (P3) can use evidence already received. Once A freezes and stops, release its slot: A verification can use the verification slot and B can use the implementation slot if files do not conflict. At actual capacity one, A verification goes first. An eligible recording row waiting twenty minutes ages to P1; ties use original intake time. On A FAIL, return the fix and verify v2; do not reuse a v1 PASS. Final recording still waits for v2 evidence and read-backs.

Notice: "Accepted; waiting for the implementation slot. No instruction has been sent yet. Selection resumes after A's safe-stop confirmation." If a send succeeded: "The frozen version was sent to independent verification; its result is unconfirmed." Never describe waiting work as running or promise a start time that the API cannot guarantee.
