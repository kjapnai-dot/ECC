---
name: fresh-dispatch-recording
description: Route a scoped request into fresh implementation, independent verification, and recording tasks, with hash-backed dispatch state and auditable GitHub/Linear write-back.
---

# Fresh Dispatch Recording

Use this skill when a request must be split across separate implementation, independent verification, and recording owners. Keep the work bounded to the user's request and preserve the distinction between planning, dispatch, verification, and external recording.

Read [references/dispatch-record-schema.md](references/dispatch-record-schema.md) when creating the dispatch ledger, normalizing instruction text, or recording handoff and completion state. For priority, capacity, eligibility, timeouts, and Codex app capabilities, read [priority queue operations](../execution-coordinator/references/priority-queue.md). Use the project's existing routing policy and ledger.

## Dispatch

1. Acknowledge intake before tool calls or worker-slot waits. Assign a non-sensitive work ID and record the objective, expected result, target, out-of-scope area, owner, and acceptance conditions. Check the ledger before creating anything.
2. Queue fresh owners for implementation, independent verification, and recording. Create each new chat only when its first actionable stage is dependency-ready and both local total and role slots are free. Keep blocked stages queued rather than starting owners to wait. Reuse an existing chat only when the user explicitly authorizes it and the exception is recorded.
3. Before each send, atomically reserve `work_id + role + normalized_instruction_sha256` with one send owner. Store the exact instruction, its fixed reference version, reservation/operation IDs, and current state. A hash or a manual check alone is not an atomic duplicate-prevention mechanism. If the project authorizes a single-sender manual fallback, disclose that limitation and stop on ambiguous ownership/delivery; do not claim an atomic global journal exists.
4. Send only a short handoff card and the minimum versioned references. Do not pass the full conversation history or create a copied source of truth. Record the actual prompt under the configured prompt directory and link it from the ledger.
5. Report `sent` only after the send has a concrete success signal. `reserved`, `not_sent`, `sending`, `delivery_unknown`, and `waiting` are not sent. On timeout, query the operation or chat ID; do not create a second chat or send a second initial prompt while delivery is unknown. Retry at most once, in the same chat and with the same reservation key, only after non-delivery is confirmed. Unknown delivery retains the local slot; `waiting` is not permission to create a replacement or send another initial instruction. A `create_thread` prompt is the initial delivery, so do not repeat it as a follow-up.

On confirmed completion or safe stop, release the local slot and choose the next eligible owner in the active turn. Record waiting reason, unblock condition, and next event when none is eligible. Apply queue aging without bypassing dependencies or preempting running work. If actual capacity is lower, reduce concurrency or use serial dispatch; never treat local caps as platform guarantees. Long occupancy returns through saved checkpoints and confirmed stops; no automatic scheduler or forced-stop capability is implied.

The implementation owner changes only the assigned scope and reports changed files, backup or recovery information, checks run, and unresolved items. The verifier receives the implementation version and performs a separate procedure or environment; it must not convert the implementer's self-check into an independent PASS. The recorder collects evidence while work is in progress and keeps implementation and verification ownership separate.

## External recording and outcome

The recorder may save only the explicitly delegated report/evidence documents. If authorization and connectivity exist, save the approved documents to GitHub, then read them back and record the commit or remote reference. Update Linear with the work ID, change summary, verification result, unresolved or unverified scope, and commit/reference; read the issue back and record the result. Do not mix application, test, dependency, or unrelated files into a recording commit. If an external write or read-back is unavailable, mark it `waiting`/`external_save_pending`; never call it complete.

Keep `PASS`, `FAIL`, and `unconfirmed` as separate outcomes. A FAIL or unconfirmed result includes its evidence, affected scope, owner for the next action, and re-verification condition. It remains unresolved until the changed version is checked again. The final report is complete only when implementation evidence, independent verification, recording evidence, and any required GitHub/Linear read-backs are linked to the same work ID.
