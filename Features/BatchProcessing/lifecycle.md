# Contract Lifecycle -- Design Spec

## Purpose

Define the complete state machine governing a contract's lifecycle from initial queue entry through final sync. Every state transition is validated, enforced, and audited.

---

## Lifecycle States

| # | State | Description |
|---|---|---|
| 1 | `queued` | Contract is waiting to enter the processing pipeline |
| 2 | `downloading` | File is being fetched from the source |
| 3 | `extracting` | Extractors are running against the document content |
| 4 | `preflight_running` | Preflight gates are being evaluated |
| 5 | `preflight_ready` | Preflight complete; results available for review |
| 6 | `needs_review` | Manual review required (failed auto-pass gates) |
| 7 | `system_pass` | All automated checks passed; triage items created for any warnings |
| 8 | `sent_to_autospark` | Contract submitted to AutoSpark for downstream processing |
| 9 | `autospark_running` | AutoSpark is actively processing the contract |
| 10 | `autospark_review` | AutoSpark results require human review |
| 11 | `autospark_completed` | AutoSpark processing finished successfully |
| 12 | `salesforce_synced` | Contract data synced to Salesforce |
| 13 | `failed` | Processing failed after exhausting retries |
| 14 | `cancelled` | Processing cancelled by user or system |

---

## Valid Transitions

| From State | Allowed To States |
|---|---|
| `queued` | `downloading`, `cancelled`, `failed` |
| `downloading` | `extracting`, `failed` |
| `extracting` | `preflight_running`, `failed` |
| `preflight_running` | `preflight_ready`, `failed` |
| `preflight_ready` | `needs_review`, `system_pass`, `failed` |
| `needs_review` | `system_pass`, `failed`, `cancelled` |
| `system_pass` | `sent_to_autospark`, `failed` |
| `sent_to_autospark` | `autospark_running`, `failed` |
| `autospark_running` | `autospark_review`, `autospark_completed`, `failed` |
| `autospark_review` | `autospark_completed`, `failed` |
| `autospark_completed` | `salesforce_synced`, `failed` |
| `salesforce_synced` | -- (terminal state) |
| `failed` | `queued` (retry) |
| `cancelled` | `queued` (re-queue) |

---

## Auto-System-Pass Gates

A contract automatically transitions from `preflight_ready` to `system_pass` when **all three conditions** are met:

| Gate | Condition |
|---|---|
| **GREEN gate** | Preflight overall result is GREEN (no RED or YELLOW gates) |
| **No triage items** | Zero checks returned `fail` or `review` status |
| **All entities resolved** | Every extracted entity has been matched by the Identity Resolver (no unresolved entities) |

If any gate fails, the contract moves to `needs_review` instead.

---

## Triage Item Creation on System Pass

When a contract reaches `system_pass`, the system creates triage items for downstream human review:

- A triage item is created for **every check that returned `fail` or `review` status** during preflight.
- Each triage item references the specific check, the extracted value, and the gate that flagged it.
- Triage items appear in the Triage module for verifier-level users.
- Even though the contract passed system checks, triage items ensure nothing is silently ignored.

---

## State Transition Enforcement

Every state transition follows a strict protocol:

| Step | Description |
|---|---|
| **1. Validate** | Check that the requested transition exists in `VALID_TRANSITIONS` for the current state |
| **2. Reject or proceed** | If the transition is not valid, reject with an error (no state change occurs) |
| **3. Update state** | Write the new state to the database |
| **4. Emit audit event** | Record the transition with full context |
| **5. Transactional** | Steps 3 and 4 execute within a single database transaction -- both succeed or both roll back |

### Audit Event Fields

| Field | Description |
|---|---|
| `event_type` | `lifecycle.transition` |
| `contract_id` | The contract that transitioned |
| `batch_id` | The batch this contract belongs to |
| `from_state` | Previous state |
| `to_state` | New state |
| `trigger` | What caused the transition (`auto`, `manual`, `retry`, `cancel`) |
| `actor` | User ID (for manual transitions) or `system` (for automated transitions) |
| `timestamp` | When the transition occurred |
| `metadata` | Additional context (error message for `failed`, gate results for `system_pass`) |

---

## State Diagram (Text)

```
queued --> downloading --> extracting --> preflight_running --> preflight_ready
  |                                                              |          |
  v                                                              v          v
cancelled                                                   needs_review  system_pass
  |                                                              |          |
  v                                                              v          v
queued (re-queue)                                           system_pass  sent_to_autospark
                                                                            |
                                                                            v
                                                                    autospark_running
                                                                      |           |
                                                                      v           v
                                                              autospark_review  autospark_completed
                                                                      |           |
                                                                      v           v
                                                              autospark_completed  salesforce_synced
                                                                      |
                                                                      v
                                                              salesforce_synced

Any state (except terminals) --> failed --> queued (retry)
```

---

## Retry Behavior

| Parameter | Value |
|---|---|
| Max retries per step | 3 (configurable via `batch.retry_count`) |
| Retry from `failed` | Transitions back to `queued` for full re-processing |
| Retry from `cancelled` | Transitions back to `queued` for full re-processing |
| Retry delay | Exponential backoff: 1s, 2s, 4s |
