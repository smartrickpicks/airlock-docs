# Batch Processing State Machine — Design Spec

> **Status:** SPECCED
> **Source:** BatchProcessing overview.md, OrcestrateOS `server/batch_processor.py` (558 lines) + `server/contract_lifecycle.py` (232 lines), async pipeline patterns

## Summary

This spec defines the two interlocking state machines that govern batch processing: the batch-level state machine (lifecycle of the entire upload) and the per-vault state machine (lifecycle of each contract within the batch). It also specifies the auto-pass gate criteria, content hash deduplication algorithm, pause/resume behavior, error handling strategies, WebSocket progress events, and concurrency controls.

---

## 1. Batch-Level State Machine

```
                         ┌──────────────────────────┐
                         │                          │
                         ▼                          │
  ┌─────────┐    ┌────────────┐    ┌─────────────┐  │   ┌───────────┐
  │ created  │───▶│ uploading  │───▶│ processing  │──┼──▶│ completed │
  └─────────┘    └────────────┘    └──────┬──┬────┘  │   └───────────┘
                                         │  │       │
                                         │  │  ┌────────┐
                                         │  └─▶│ paused │
                                         │     └───┬────┘
                                         │         │ (resume)
                                         │         └───┘
                                         ▼
                                    ┌────────┐
                                    │ failed │
                                    └────────┘

  Cancel from any active state ──▶  ┌───────────┐
                                    │ cancelled │
                                    └───────────┘
```

### Valid Transitions

| From | To | Trigger | Side Effects |
|---|---|---|---|
| `created` | `uploading` | Admin initiates batch upload | Fire `batch_created` event, allocate batch vault |
| `uploading` | `processing` | All files received and stored | Fire `batch_upload_complete`, begin per-vault pipeline |
| `processing` | `completed` | All contracts reach a terminal state (`ready` or `failed`) | Fire `batch_complete`, send notification to batch owner |
| `processing` | `failed` | Batch-level infrastructure failure (S3 unavailable, DB down) | Fire `batch_failed`, notify admin, schedule retry |
| `processing` | `paused` | Admin clicks pause OR circuit breaker trips (>50% failure rate) | Fire `batch_paused`, in-progress vaults finish current step |
| `paused` | `processing` | Admin clicks resume | Fire `batch_resumed`, queued vaults resume pipeline |
| `processing` | `cancelled` | Admin clicks cancel | Fire `batch_cancelled`, in-progress vaults complete current step, remaining marked `skipped` |
| `paused` | `cancelled` | Admin clicks cancel while paused | Fire `batch_cancelled`, remaining queued vaults marked `skipped` |

**Invariant:** A batch can only be in one state at a time. State transitions are atomic and recorded in the append-only audit log.

---

## 2. Per-Vault State Machine

Each contract within a batch is tracked as an item vault with its own lifecycle.

```
                              retry (max 3)
                           ┌────────────────┐
                           │                │
                           ▼                │
  ┌────────┐    ┌──────────────┐    ┌───────┴───┐
  │ queued │───▶│ downloading  │───▶│  failed   │
  └────────┘    └──────┬───────┘    └───────────┘
                       │                  ▲
                       ▼                  │
                ┌──────────────┐          │
                │  extracting  │──────────┤
                └──────┬───────┘          │
                       │                  │
                       ▼                  │
                ┌──────────────┐          │
                │  preflight   │──────────┘
                └──────┬───────┘
                       │
                       ▼
                  ┌─────────┐
                  │  ready  │
                  └─────────┘
```

### Valid Transitions

| From | To | Trigger | Timeout | Events Fired |
|---|---|---|---|---|
| `queued` | `downloading` | Semaphore acquired, vault enters pipeline | — | `vault:step_started {step: "downloading"}` |
| `downloading` | `extracting` | File downloaded and stored successfully | 30s | `vault:step_complete {step: "downloading"}` |
| `downloading` | `failed` | Download fails after 3 retries or timeout | 30s | `vault:step_failed {step: "downloading", error}` |
| `extracting` | `preflight` | All extractors complete successfully | 60s | `vault:step_complete {step: "extracting"}` |
| `extracting` | `failed` | Extraction fails after 3 retries or timeout | 60s | `vault:step_failed {step: "extracting", error}` |
| `preflight` | `ready` | Preflight gates evaluated (pass or fail) | 30s | `vault:step_complete {step: "preflight"}` |
| `preflight` | `failed` | Preflight engine crashes after 3 retries | 30s | `vault:step_failed {step: "preflight", error}` |
| `failed` | `queued` | Admin triggers manual retry | — | `vault:retry {attempt: N}` |

### Retry Policy

- **Max retries per step:** 3 (configurable via `batch.retry_count` in Feature Control Plane)
- **Backoff strategy:** Exponential — 1s, 2s, 4s
- **Retry scope:** Only the failed step is retried, not the entire pipeline
- **Retry counter:** Tracked per step, per vault. Resets if admin manually re-queues.
- **Terminal failure:** After 3 retries at any step, the vault moves to `failed` with the error detail preserved

---

## 3. Auto-Pass Gate Criteria

A contract auto-passes from the Discover chamber to the Build chamber when **ALL** of the following conditions are met:

| # | Condition | Threshold | Source |
|---|---|---|---|
| 1 | Extraction confidence >= threshold | Default 80%, configurable via Feature Control Plane | Extraction engine aggregate score |
| 2 | All Tier 0 (hinge) fields extracted with high confidence | >= 0.9 per field | Field-level confidence from extractors |
| 3 | No entity resolution conflicts | Single match or manually resolved | Entity resolution chain |
| 4 | Preflight quality score >= threshold | Default 70%, configurable via Feature Control Plane | Preflight engine composite score |
| 5 | No critical flags present | Zero critical flags | Document structural analysis |

**Critical flags** (any one blocks auto-pass):
- Missing signature page
- Empty document (zero extractable text)
- Corrupt PDF (parser failure)
- Document language not in supported set
- File size exceeds maximum (configurable, default 50MB)

**When ANY condition fails:** The contract stays in the Discover chamber with `gate: needs_review`. A triage item is created on the Triage Board with the specific failing conditions listed.

**Configurable thresholds** are managed through the Feature Control Plane calibration interface at `batch.auto_pass_confidence` and `batch.auto_pass_quality`.

---

## 4. Content Hash Dedup Algorithm

### Process

1. **Normalize the document text:**
   - Strip all leading/trailing whitespace from each line
   - Collapse consecutive whitespace to a single space
   - Convert to lowercase
   - Remove headers and footers (detected via repeated text on first/last 3 lines of each page)
   - Remove page numbers

2. **Compute SHA-256** of the normalized text output

3. **Query the `document_hash` index** in PostgreSQL:
   ```sql
   SELECT vault_id FROM document_hashes
   WHERE content_hash = $1
     AND workspace_id = $2;
   ```

4. **On exact match:**
   - Mark the new vault as `duplicate`
   - Link the new vault to the original vault via `duplicate_of_vault_id`
   - Skip extraction entirely (no extractor resources consumed)
   - Fire event: `vault:duplicate_detected {original_vault_id, duplicate_vault_id}`

5. **On no match:**
   - Insert the hash into the index: `INSERT INTO document_hashes (vault_id, workspace_id, content_hash)`
   - Continue pipeline normally

### Scope and Constraints

- Dedup applies **per-workspace only** — no cross-workspace hash matching (workspace isolation principle)
- The hash index has a composite unique constraint on `(workspace_id, content_hash)`
- Near-duplicate detection (future enhancement): SimHash with Hamming distance < 3, flagged for manual review rather than auto-skipped

---

## 5. Batch Pause / Resume / Cancel Behavior

### Pause

1. Admin clicks **Pause** in the batch vault Control pane (or circuit breaker trips automatically)
2. Batch status transitions to `paused`
3. All `queued` vaults remain in `queued` — they do not enter the pipeline
4. Vaults currently in `downloading`, `extracting`, or `preflight` **finish their current step**, then stop
5. Fire event: `batch_paused {batch_id, reason: "admin" | "circuit_breaker"}`
6. Notification sent to batch owner

### Resume

1. Admin clicks **Resume** in the batch vault Control pane
2. Batch status transitions to `processing`
3. Vaults in `queued` begin entering the pipeline (subject to semaphore)
4. Vaults that were mid-pipeline when paused continue from where they stopped
5. Fire event: `batch_resumed {batch_id}`

### Cancel

1. Admin clicks **Cancel** in the batch vault Control pane
2. Batch status transitions to `cancelled`
3. Vaults currently in-progress **complete their current step** (no mid-step interruption)
4. All remaining `queued` vaults are marked `skipped`
5. Fire event: `batch_cancelled {batch_id, skipped_count}`
6. Notification sent to batch owner with final counts

---

## 6. Error Handling

### Per-Vault Failure

- Log the error with full stack trace and step context to the append-only audit log
- Mark the vault as `failed` with structured error detail: `{step, error_type, message, attempt}`
- **Continue processing remaining vaults** — one failure does not block the batch
- Fire `batch:contract_failed` WebSocket event for real-time UI update

### Batch-Level Failure

- Triggered by infrastructure problems (S3 unavailable, PostgreSQL connection lost, Redis down)
- Pause the batch immediately — no new vaults enter the pipeline
- Send `batch_failed` notification to admin with infrastructure error detail
- Auto-retry after 5 minutes, up to 3 attempts
- If all 3 infrastructure retries fail, batch remains in `failed` state awaiting manual intervention

### Whole-Batch Abort (Circuit Breaker)

- If **> 50%** of processed vaults in a batch have failed → auto-pause the batch
- Threshold is configurable via Feature Control Plane: `batch.circuit_breaker_threshold` (default: 50%)
- Fire event: `batch_circuit_breaker {batch_id, failure_rate, threshold}`
- Notify admin with failure breakdown by step (e.g., "12/20 failed at extraction, 3/20 failed at download")
- Batch stays in `paused` until admin investigates and clicks Resume or Cancel

### Poison Pill Detection

- If the **same vault** fails 3 times across retries at any step → mark as `poison`
- Poison vaults are permanently skipped — they do not re-enter the pipeline on batch resume
- Fire event: `vault:poison_detected {vault_id, step, error_history}`
- Alert admin with the vault ID and failure history for investigation

---

## 7. WebSocket Integration for Progress Updates

All progress events are broadcast over the WebSocket connection on a per-batch topic.

**Topic:** `batch:{batch_id}`

| Event | Payload | When Fired |
|---|---|---|
| `batch:progress` | `{processed: 45, total: 100, failed: 2, current_step: "extracting"}` | After each vault completes or fails a step |
| `batch:contract_complete` | `{vault_id, status: "ready", health_score: 85}` | Vault reaches `ready` state |
| `batch:contract_failed` | `{vault_id, step: "extraction", error: "timeout", attempt: 3}` | Vault reaches `failed` state |
| `batch:complete` | `{total: 100, ready: 93, failed: 7, duration_seconds: 342}` | All vaults in terminal state |
| `batch:paused` | `{batch_id, reason: "admin"}` | Batch paused by admin or circuit breaker |
| `batch:resumed` | `{batch_id}` | Batch resumed by admin |
| `batch:cancelled` | `{batch_id, skipped_count: 12}` | Batch cancelled by admin |
| `batch:duplicate` | `{vault_id, original_vault_id}` | Duplicate detected during dedup step |

**Delivery guarantees:** Events are fire-and-forget over WebSocket. If the client reconnects, it requests current batch state via REST API (`GET /api/batches/{batch_id}/status`) to rehydrate. No event replay — the REST endpoint returns the authoritative snapshot.

---

## 8. Concurrency Control

### Semaphore Configuration

- **Default concurrency:** `asyncio.Semaphore(5)` — max 5 vaults processing simultaneously per batch
- **Configurable via:** Environment variable `BATCH_CONCURRENCY` (overridden by Feature Control Plane calibration at `batch.concurrency_limit`)
- **Precedence:** Feature Control Plane value > env var > hardcoded default (5)

### Queue Priority

| Batch Size | Priority | Rationale |
|---|---|---|
| Small (< 10 vaults) | High | Fast feedback for ad-hoc uploads |
| Medium (10–100 vaults) | Normal | Standard batch processing |
| Large (> 100 vaults) | Low | Background bulk ingestion |

### Starvation Prevention

Large batches yield **1 semaphore slot every 10 vaults** to allow small batch processing to proceed. Implementation:

1. After every 10 vaults processed in a large batch, release the semaphore for 500ms
2. If a small-batch vault is waiting, it acquires the slot and processes
3. If no small-batch vault is waiting, the large batch reclaims the slot immediately

This prevents a 500-vault batch from monopolizing all pipeline capacity while an admin waits for a single urgent contract to process.

### Global Concurrency (Multi-Batch)

When multiple batches run concurrently, the total semaphore pool is shared:

- **Global max:** `BATCH_GLOBAL_CONCURRENCY` (default: 10)
- Each batch gets at least 1 slot (guaranteed minimum)
- Remaining slots distributed proportionally by batch priority
- Feature Control Plane can override per-workspace via `batch.global_concurrency_limit`

---

## Related Specs

- [BatchProcessing Overview](./overview.md) — Parent spec with pipeline steps, triptych layout
- [EventBus](../EventBus/overview.md) — Batch events route through the event bus
- [RealTime](../RealTime/overview.md) — WebSocket progress update infrastructure
- [FeatureControlPlane](../FeatureControlPlane/overview.md) — Concurrency and threshold calibration
- [Record Inspector](../Record%20Inspector/overview.md) — Extraction step produces field cards for review
- [Security](../Security/overview.md) — Append-only audit log, workspace isolation constraints
