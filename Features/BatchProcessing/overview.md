# Batch Processing — Design Spec

> **Status:** SPECCED — Batch Processing is the async ingestion pipeline for bulk contract uploads.

> **Terminology:** See [Naming & Hierarchy](../Shell/naming-and-hierarchy.md). A batch creates a **batch vault** within the Contracts module.

## Purpose

Batch Processing handles the async ingestion of bulk contract uploads. Each batch upload creates a **batch vault** within the Contracts module, one vault per batch. Individual contracts within the batch each get their own **item vault**. The pipeline downloads, stores, deduplicates, extracts, preflights, and auto-passes contracts with no manual intervention required for clean documents.

---

## Source Mapping (OrcestrateOS)

| OrcestrateOS File | Size | Airlock Target |
|---|---|---|
| `server/batch_processor.py` | 558 lines | Batch pipeline orchestration, concurrency, retry logic |
| `server/contract_lifecycle.py` | 232 lines | State machine, valid transitions, auto-system-pass gates |

---

## Async Pipeline

Contracts flow through the pipeline in strict sequential order. Each step must succeed before the next begins.

| Step | Description | Failure Behavior |
|---|---|---|
| **Download** | Fetch the contract file from the source URL | Retry up to 3 times, then mark `failed` |
| **Store** | Persist the file to object storage with metadata | Retry up to 3 times, then mark `failed` |
| **Dedup** | Check for duplicate contracts by content hash | Skip extraction if duplicate found |
| **Extract** | Run all enabled extractors against the document | Retry up to 3 times, then mark `failed` |
| **Preflight** | Evaluate preflight gates (quality, structure, coverage) | Mark `preflight_ready` regardless of gate result |
| **Auto-Pass** | If all gates GREEN, no triage items, all entities resolved: auto-advance to `system_pass` | Stay at `preflight_ready` for manual review |

---

## Concurrency

| Parameter | Default | Configurable Via |
|---|---|---|
| Concurrent contracts | 5 | `batch.concurrency_limit` (calibration) |
| Implementation | `asyncio.Semaphore(N)` | -- |
| Retry count per step | 3 | `batch.retry_count` (calibration) |

The semaphore gates how many contracts can be actively processing at any time within a single batch. Each contract acquires the semaphore before entering the pipeline and releases it upon completion or failure.

---

## Batch Vault Structure (Triptych)

Each batch vault uses the standard Airlock triptych layout:

| Pane | Name | Content |
|---|---|---|
| **Left** | Signal | Batch health summary -- progress bar, pass/fail/pending counts, error rate |
| **Center** | Orchestrate | Contract table -- one row per contract, columns: name, status, gate result, health score |
| **Right** | Control | Batch metadata -- upload source, upload time, total count, processing duration, batch ID |

### Signal Pane Details

- **Progress bar**: `(completed + failed) / total` contracts
- **Pass count**: Contracts that reached `system_pass` or beyond
- **Fail count**: Contracts in `failed` state
- **Pending count**: Contracts still in pipeline (`queued` through `preflight_running`)
- **Error rate**: `failed / (completed + failed)` as a percentage
- Auto-refreshes every 5 seconds while the batch is processing

### Orchestrate Pane Details

- Sortable, filterable contract table
- Clicking a contract row opens that vault's Record Inspector
- Status column uses color-coded badges matching lifecycle states
- Bulk actions: retry failed, cancel pending, export results

### Control Pane Details

- Read-only batch metadata
- Batch-level actions: pause batch, resume batch, cancel batch
- Processing timeline with timestamps for each pipeline step

---

## Per-Contract Lifecycle States

Each contract within a batch tracks its own lifecycle state. See [Lifecycle](./lifecycle.md) for the full state machine.

| State | Description |
|---|---|
| `queued` | Waiting to enter the pipeline |
| `downloading` | File download in progress |
| `extracting` | Extractors running against the document |
| `preflight_running` | Preflight gates being evaluated |
| `preflight_ready` | Preflight complete; awaiting review or auto-pass |
| `failed` | Pipeline failed after exhausting retries |

---

## Vault Hierarchy Integration

When a batch is uploaded, the following vault hierarchy actions occur:

| Step | Vault Action |
|---|---|
| Batch created | Create a **batch vault** (special type, not in the 4-level hierarchy) |
| Contract downloaded | Create an **item vault** (Level 4) — auto-linked to batch vault |
| Entity extracted | Entity resolution runs → link item vault to existing counterparty vault (Level 3), or create new parent (Level 1) + counterparty (Level 3) |
| All contracts processed | Batch vault metadata updated with final counts and health |

See [Vault Hierarchy](../VaultHierarchy/overview.md) for the 4-level tree structure.

---

## Cross-Module Events

When batch processing reaches milestones, events propagate via the event bus:

| Trigger | Target | Event |
|---------|--------|-------|
| New customer entity detected | CRM module | Create parent vault + counterparty vault |
| Contract reaches `system_pass` | Tasks module | Create onboarding task |
| Contract date extracted | Calendar module | Create renewal/expiry date event |
| Batch error rate > threshold | Admin module | Circuit breaker alert in System Health |
| Batch completes | Notifications | Workspace-level notification to batch owner |

---

## Feature Control Plane Integration

| Control | Parameter |
|---|---|
| Toggle | `batch_processor.enabled` |
| Concurrency calibration | `batch.concurrency_limit` (default: 5) |
| Retry calibration | `batch.retry_count` (default: 3) |
| Timeout calibration | `batch.step_timeout_seconds` (default: 120) |
| Audit log | All state transitions, retries, and failures logged |
| Circuit breaker | Trips if batch error rate exceeds `batch.circuit_breaker_threshold` (default: 50%) |
| Health reporting | Status: healthy/degraded/erroring/disabled visible in Admin > System Health |

---

## Related Specs

- **[Lifecycle](./lifecycle.md)** — Full state machine for individual contracts
- **[Vault Hierarchy](../VaultHierarchy/overview.md)** — How batch vaults relate to the vault tree
- **[Feature Control Plane](../FeatureControlPlane/overview.md)** — Toggle, calibrate, audit, failure flag
- **[Triage Board](../Triage/overview.md)** — Triage items created during preflight appear here
- **[Review Queue](../OrgOverview/overview.md)** — Batch progress visible in entity cards
