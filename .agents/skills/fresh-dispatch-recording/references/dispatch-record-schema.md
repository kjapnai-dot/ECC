# Dispatch record schema

Use this compact schema in the ledger or handoff artifact. Adapt field names to the repository's existing system without creating a second source of truth.

## Handoff card

```text
Work ID / role / normalized instruction SHA256:
Objective / expected result / out of scope:
Current state: planned | not_sent | sending | sent | received | working | verifying | returned | waiting | delivery_unknown | failed | external_save_pending | complete
Owner and chat ID:
Assigned files or systems:
Reference version: commit, fixed snapshot, or artifact hash:
Permissions and approvals:
Verification procedure, expected evidence, and save location:
Next single action / completion condition:
```

Pass only the card plus the minimum references needed to act. Add a new card for an additional condition under the same work ID; do not make a new initial chat because the prompt text changed.

## Instruction identity

To calculate the instruction hash, normalize the exact UTF-8 text by converting line endings to LF, applying Unicode NFC, removing trailing whitespace from each line, and ending with exactly one LF. Hash that normalized text with SHA-256. Store the prompt itself or an immutable path/reference alongside the hash so another owner can reproduce it.

The duplicate-prevention key is:

```text
(work_id, role, normalized_instruction_sha256)
```

Reservation, ownership, and state transitions must be serialized by a lock or transaction with a uniqueness constraint. Keep the reservation owner, chat/operation IDs, attempt count, timestamps, and evidence for each transition. A failed or ambiguous attempt remains visible; do not hide it by allocating a new key.

## State evidence

| State | Required interpretation |
| --- | --- |
| `not_sent` | Reserved or queued, with no confirmed send. The owner may proceed. |
| `sending` | Creation or send started. One owner continues and queries its operation. |
| `sent` | A concrete send-success response exists. The ledger can advance to receipt confirmation. |
| `received` | The destination owner acknowledged the card and references. |
| `delivery_unknown` | The operation timed out or has an ambiguous result. Query before any retry or new chat. |
| `failed` | Failure is confirmed and its error/evidence is stored. Stop or follow the approved recovery path. |
| `waiting` / `external_save_pending` | A required slot, approval, connection, or read-back is unavailable. This is not completion. |

Verification and recording outcomes are separate from dispatch state: use `PASS`, `FAIL`, or `unconfirmed`, and attach the evidence path, version checked, and next owner. Never infer PASS from the absence of a reported failure.


## Queue extension

Keep these fields in the existing ledger alongside the handoff card. They describe local operating state; a record alone is not a persistent scheduler or atomic global journal. Read [priority queue operations](../../execution-coordinator/references/priority-queue.md) for selection, eligibility, and release rules.

```text
queue_id / work_id / role / stage / owner:
chat_id / host_id / client_thread_id (setup only; never a substitute thread_id):
enqueued_at / base_priority / eligible_wait_seconds / effective_priority:
depends_on: [{queue_id, required_version, required_evidence}]
eligibility: ready|blocked (non-capacity gates) / wait_reason / unblock_condition / observer / next_check_event:
reference_version / reservation_key / prompt_path / normalized_instruction_sha256:
dispatch_state / outcome: PASS|FAIL|unconfirmed / evidence:
capacity_policy: total_worker_cap / role_caps / policy_version:
slot_claim: role / owner / claimed_at / released_at / release_evidence:
ack_due_at / progress_due_at / slice_due_at / last_observed_at:
selected_at / selected_reason / next_owner / retry_count:
```

Use separate fields for queue stage, dispatch state, outcome, and slot ownership. A dependency identifies the exact version and evidence required; a prior version's PASS does not satisfy it. Preserve enqueue time and send/reservation history on return. Accumulate aging time only while non-capacity gates are ready but work is unscheduled; role-slot, total-slot, and platform-capacity waits count, while dependency, approval, connection, conflict, and unknown-delivery waits do not. A waiting row records one of `dependency`, `role_slot`, `total_slot`, `platform_capacity`, `ownership_conflict`, `approval`, `connection`, `delivery_unknown`, `stop_unconfirmed`, or `external_readback`, with the condition and event that can unblock it.

A claim starts before creation/send; release requires completion or a confirmed safe stop with saved differences and no ongoing mutation. Keep unknown sends and unconfirmed stops occupied. Lower actual capacity causes fewer active owners or serial execution, not automatic extra chats. Existing-chat follow-ups need direct human authorization. Preserve `client_thread_id` for setup correlation; resolve a confirmed final `chat_id` before using threadId-only observations or sends.
