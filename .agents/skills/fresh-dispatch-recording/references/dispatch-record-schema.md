# Dispatch record schema

Use this compact schema in the ledger or handoff artifact. Adapt field names to the repository's existing system without creating a second source of truth.

## Handoff card

```text
Work ID / role / normalized instruction SHA256:
Objective / expected result / out of scope:
Current state: planned | not_sent | sending | sent | received | working | verifying | returned | waiting | external_save_pending | complete
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
