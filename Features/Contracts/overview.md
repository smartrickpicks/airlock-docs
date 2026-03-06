# Contracts Module -- Design Spec

> **Status:** SPECCED
> **Source:** OrcestrateOS contracts pipeline + Airlock feature specs

## Summary

The Contracts module is the primary workflow engine in Airlock. Every contract is a **Vault** (level-4 item vault) that moves through the four **Chambers** -- Discover, Build, Review, Ship -- accumulating extraction data, quality scores, amendments, and approvals along the way. The module composes five sub-features (Triage Dashboard, Contract Generator, Record Inspector, Patch Workflow, Review Queue) into a single cohesive lifecycle that takes a raw document from ingestion to final export and external system sync.

---

## Module Purpose

Contracts is the heaviest module in Airlock. It is the first icon in the module bar and the default landing experience for analysts. The module manages the full contract lifecycle:

1. **Ingest** raw documents (single upload or batch)
2. **Extract** structured data from unstructured contracts (7 extractors, 152 fields)
3. **Triage** quality issues, entity ambiguities, and extraction gaps
4. **Build** new contracts from templates or amend existing ones
5. **Review** data corrections, approval chains, and cross-vault governance
6. **Ship** finalized contracts via export and external system sync

Every action within this lifecycle is recorded as an immutable event in the vault's Signal feed, producing a complete audit trail from first upload to final delivery.

---

## Sub-Feature Composition

The Contracts module is assembled from five independently specced features, each owning a distinct responsibility within the lifecycle.

| Sub-Feature | Spec Location | Chamber | Role | Responsibility |
|---|---|---|---|---|
| **Triage Dashboard** | `/Triage/overview.md` | Discover | Builder | Intake queue. Analysts review incoming vaults, inspect extraction results, resolve entity ambiguities, dismiss false positives. Module-level notification aggregator. |
| **Contract Generator** | `/ContractGenerator/overview.md` | Build | Builder | Template-driven contract creation. Two-pane builder with live preview. Unified v2 clause library (188 clauses, 24 contract types, 5 verticals). Trigger-based clause selection. |
| **Record Inspector** | `/Record Inspector/overview.md` | Cross-chamber | Builder, Gatekeeper | Default vault view in Orchestrate panel. Field cards with confidence scores across 5 weighted sections (Entity Resolution, Opportunities, Schedule, Financials, Addons). Heatmap and Spotlight modes. |
| **Patch Workflow** | `/PatchWorkflow/overview.md` | Review | Gatekeeper | Data correction proposals. 12 states, 20 transitions, structured evidence packs (when/then/because), self-approval prevention, optimistic locking. Action Focus editor in the Triptych. |
| **Review Queue** | `/OrgOverview/overview.md` | Review | Gatekeeper | Cross-vault governance view. Parent vault cards with aggregate health, handoff signals (RFIs, corrections, anomalies), filterable activity feed. |

---

## Chamber Mapping

Each chamber represents a lifecycle stage with specific views, roles, and gate conditions.

| Chamber | Color | Primary Views | Primary Role | Gate Condition to Advance |
|---------|-------|--------------|--------------|---------------------------|
| **Discover** | Blue | Triage Dashboard, Batch Upload, Record Inspector | Builder | All hinge fields (Tier 0) resolved, no blocker-severity triage items, entity resolution confidence >= 0.80 |
| **Build** | Green | Contract Generator, Record Inspector, Document Viewer | Builder | All required sections scored, health score >= 60%, no RED preflight gates |
| **Review** | Purple | Patch Editor (Action Focus), Review Queue, Record Inspector | Gatekeeper | All patches in `Applied` or `Rejected` state, Gatekeeper approval recorded, no open RFIs |
| **Ship** | Teal | Export Center, Sync Status, Archive | Owner | Final Owner sign-off, export completed or waived |

### Chamber Navigation in the Sub-Panel

```
+------------------------+
| CONTRACTS              |
| > Mini Dashboard       |
|   142 Vaults           |
|   23 in Discover       |
|   48 in Build          |
|   31 in Review         |
|   40 in Ship           |
|------------------------+
| > DISCOVER             |
|   o Triage Dashboard   |  <- pinned, singleton
|   o Batch Upload       |
|                        |
| > BUILD                |
|   o Generator          |  <- pinned, singleton
|   o Templates          |
|                        |
| > REVIEW               |
|   o Review Queue       |  <- pinned, singleton
|   o Patch Approvals    |
|                        |
| > SHIP                 |
|   o Export Center      |  <- pinned, singleton
|   o Sync Status        |
|------------------------+
| ACTIVE VAULTS          |
|   # sony-amend-01    o |
|   # getdough-lic     o |
|   # atlantic-mgmt    o |
+------------------------+
```

---

## Contract Vault Lifecycle

A contract vault progresses through the following stages. Each transition is gated -- the system evaluates gate conditions and blocks advancement until all criteria are met.

```
Upload/Create
     |
     v
[DISCOVER] ---- extract --> triage --> resolve entities
     |
     | (all hinge fields resolved, no blockers, entity confidence >= 0.80)
     v
[BUILD] ------- amend / generate --> preflight scoring
     |
     | (health >= 60%, no RED gates, all required sections scored)
     v
[REVIEW] ------ patch proposals --> approval chain --> gatekeeper sign-off
     |
     | (all patches resolved, gatekeeper approved, no open RFIs)
     v
[SHIP] -------- export --> sync --> archive
     |
     | (owner sign-off, export completed or waived)
     v
  ARCHIVED
```

### Auto-Advance (System Pass)

When a vault clears all gate conditions without any triage items, entity ambiguities, or manual flags, the Batch Processing pipeline can auto-advance it from Discover to Build via `system_pass`. This is the fast path for clean documents. See [Batch Processing](../BatchProcessing/overview.md).

### Reverse Transitions

Vaults can move backward under controlled conditions:

| From | To | Trigger | Who |
|---|---|---|---|
| Build | Discover | New extraction issue surfaced | System or Builder |
| Review | Build | Patch rejected, requires re-work | Gatekeeper |
| Ship | Review | Post-export issue detected | Owner |

Reverse transitions emit a `chamber.reversed` audit event with reason and actor.

---

## Vault Data Model

Contract vaults use the shared `vaults` table defined in [Vault Hierarchy](../VaultHierarchy/overview.md). Contract-specific behavior is driven by the following fields:

| Field | Value | Notes |
|---|---|---|
| `vault_level` | `4` (Item) | Only item vaults progress through chambers |
| `vault_type` | `'contract'` | Distinguishes from task, document vault types |
| `module_type` | `'contracts'` | Scopes this vault to the Contracts module |
| `chamber` | `'discover'` / `'build'` / `'review'` / `'ship'` | Current lifecycle stage |
| `gate` | Gate identifier within current chamber | Tracks checkpoint progress |
| `health_score` | `0.0` -- `100.0` | Computed by preflight engine across 5 weighted sections |
| `metadata` | JSONB | Contract-specific: `contract_type`, `territory`, `effective_date`, `counterparty`, `deal_value`, `vertical` |
| `parent_vault_id` | FK to counterparty vault (level 3) | Positions this contract in the vault hierarchy |

### Contract-Specific Metadata Schema

```json
{
  "contract_type": "Distribution",
  "vertical": "music_core",
  "territory": "Worldwide",
  "effective_date": "2026-03-01",
  "expiration_date": "2029-02-28",
  "counterparty_name": "Sony Music",
  "deal_value": 250000,
  "currency": "USD",
  "source_file": "sony-dist-2026.pdf",
  "page_count": 24,
  "extraction_version": "v2",
  "batch_id": "batch-20260301-001"
}
```

---

## Contract Export

The Export Center is a Ship-chamber view for packaging finalized contracts into deliverable formats. Demonstrated in `contracts-export-demo.html`.

### Layout

Three stacked sections:

| Section | Content |
|---|---|
| **Ready to Ship** | Horizontal scroll of vault cards that have cleared all gates. Each card shows vault name, health score, approval date, approver, and a one-click Export button. |
| **Export Queue** | Table of in-progress and queued exports: vault name, format badges, requester, timestamp, status (Processing / Queued), progress bar. |
| **Recent Exports** | Table of completed exports: vault name, format badges, exporter, date, file size, download action. |

### Export Formats

| Format | Extension | Use Case |
|---|---|---|
| **PDF** | `.pdf` | Final executed version, human-readable |
| **DOCX** | `.docx` | Editable version for legal markup |
| **JSON** | `.json` | Machine-readable extraction data for downstream systems |

### Export Package Contents

Each export bundles:
- Generated or amended contract document
- Extraction summary (all 152 fields with confidence scores)
- Audit trail (chamber transitions, patch history, approval chain)
- Health score report (section-by-section breakdown)

### Batch Export

Multiple vaults can be selected and exported together. The Export Queue processes them sequentially, respecting the same `asyncio.Semaphore(5)` concurrency limit as batch ingestion.

### Export Triggers

| Trigger | Behavior |
|---|---|
| Manual button | Builder or Owner clicks Export in the Ship chamber |
| Auto-export on gate clearance | When the final Ship gate clears, auto-queue export if `ENABLE_AUTO_EXPORT` flag is on |
| Scheduled export | Cron-based batch export for all newly shipped vaults (configurable interval) |

---

## Contract Sync

The Sync Status view manages bidirectional data flow between Airlock and external systems. Demonstrated in `contracts-sync-demo.html`.

### Supported Connections

| System | Direction | What Syncs |
|---|---|---|
| **Salesforce** | Bidirectional | Extracted fields mapped to Salesforce custom objects (`Opportunity__c`, `Contract__c`). Push on export; pull for ingestion. |
| **Google Drive** | Pull | Source documents pulled from shared Drive folders into Discover chamber for extraction. |
| **Amazon S3** | Push | Exported packages (PDF, DOCX, JSON) pushed to configured S3 buckets with vault-slug prefixed keys. |

### Sync Status Tracking

Each connection card displays:
- Connection health (green/amber/red)
- Last sync timestamp
- Records/files synced (current / total)
- Sync direction badge (Push / Pull / Bidirectional)
- Warning text for partial failures

### Sync Activity Log

Filterable event stream (by system: All, Salesforce, Drive, S3) showing:
- Per-event status (Success / Error with retry countdown)
- Vault name and field/file counts
- Retry button for failed syncs

### Webhook Integration

| Event | Webhook Payload |
|---|---|
| `sync.completed` | `{ vault_id, system, direction, records_synced, timestamp }` |
| `sync.failed` | `{ vault_id, system, error_code, retry_at, timestamp }` |
| `sync.connection_lost` | `{ system, last_successful_sync, timestamp }` |

### Feature Flag

`ENABLE_CONTRACT_SYNC` -- toggles the entire Sync Status view and all background sync jobs. When disabled, the sub-panel hides the Sync Status entry and no sync operations execute.

---

## Triptych Layout Per View

When a user opens an individual contract vault, the main content renders the standard Triptych. Panel content varies by the active view.

| View | Signal Panel | Orchestrate Panel | Control Panel |
|---|---|---|---|
| **Triage** (Discover) | Activity feed, extraction events, entity resolution cards | Vault cards grid with severity/status badges | Quick stats, filter bar, batch progress |
| **Generator** (Build) | Template preview, clause selection log | Section wizard + clause picker (two-pane) | Live contract preview |
| **Record Inspector** (Cross-chamber) | Extraction events, entity confirmations, Otto messages | Field cards (5 sections, 3 tiers) + Document Viewer | Entity details, confidence breakdown, section scores |
| **Patch Editor** (Review) | Diff history, transition timeline | Action Focus editor (evidence pack builder) | Approval chain (vertical timeline), SLA timer |
| **Review Queue** (Review) | Handoff signals (RFIs, corrections, anomalies) | Parent vault entity cards with child tables | Activity feed, governance filters |
| **Export Center** (Ship) | Export event log | Ready-to-ship cards, export queue, recent exports | Export settings, format selection |
| **Sync Status** (Ship) | Sync event stream | Connection cards, sync activity log | Sync settings, webhook config |

---

## API Endpoints

The Contracts module exposes the following REST surface. All endpoints are workspace-scoped via `workspace_id` extracted from the JWT.

| Method | Path | Description | Auth |
|---|---|---|---|
| `GET` | `/api/v1/contracts/triage` | List vaults in Discover chamber with triage items | Builder, Gatekeeper |
| `GET` | `/api/v1/contracts/triage/{item_id}` | Get single triage item detail | Builder, Gatekeeper |
| `PATCH` | `/api/v1/contracts/triage/{item_id}` | Update triage item (assign, change severity, dismiss) | Builder |
| `POST` | `/api/v1/contracts/generate` | Create a new contract vault from a template + clause selections | Builder |
| `GET` | `/api/v1/contracts/{vault_id}/fields` | Get all extracted fields with confidence scores | Builder, Gatekeeper |
| `GET` | `/api/v1/contracts/{vault_id}/health` | Get health score breakdown by section | Builder, Gatekeeper |
| `POST` | `/api/v1/contracts/{vault_id}/patch` | Create a new patch (data correction proposal) | Builder |
| `PATCH` | `/api/v1/contracts/{vault_id}/patch/{patch_id}` | Transition a patch state (submit, approve, reject) | Builder, Gatekeeper, Owner |
| `GET` | `/api/v1/contracts/review-queue` | List vaults pending review, grouped by parent vault | Gatekeeper |
| `POST` | `/api/v1/contracts/{vault_id}/advance` | Advance vault to the next chamber (evaluates gate conditions) | Builder, Gatekeeper, Owner |
| `POST` | `/api/v1/contracts/{vault_id}/reverse` | Reverse vault to a prior chamber (with reason) | Gatekeeper, Owner |
| `POST` | `/api/v1/contracts/{vault_id}/export` | Queue an export job (specify formats: pdf, docx, json) | Builder, Owner |
| `GET` | `/api/v1/contracts/{vault_id}/export/{export_id}` | Get export status and download URL | Builder, Owner |
| `POST` | `/api/v1/contracts/{vault_id}/sync` | Trigger sync to a specific external system | Builder, Owner |
| `GET` | `/api/v1/contracts/sync/status` | Get connection health for all configured sync targets | Builder, Gatekeeper, Owner |
| `POST` | `/api/v1/contracts/batch` | Create a batch upload job | Builder |
| `GET` | `/api/v1/contracts/batch/{batch_id}` | Get batch processing status | Builder |

All mutation endpoints use optimistic locking via the `version` field. Concurrent writes return `409 Conflict`.

---

## Feature Flags

Each major capability within the Contracts module is independently toggleable via the [Feature Control Plane](../FeatureControlPlane/overview.md).

| Flag | Default | Controls |
|---|---|---|
| `ENABLE_CONTRACT_GENERATOR` | `true` | Contract creation from templates. When off, the Generator view is hidden from the sub-panel. |
| `ENABLE_BATCH_UPLOAD` | `true` | Batch ingestion pipeline. When off, only single-vault uploads are accepted. |
| `ENABLE_CONTRACT_EXPORT` | `true` | Export Center and all export jobs. When off, vaults can still reach Ship but cannot be exported. |
| `ENABLE_CONTRACT_SYNC` | `false` | External system sync (Salesforce, Drive, S3). Disabled by default; requires connection configuration. |
| `ENABLE_PATCH_WORKFLOW` | `true` | Patch creation and approval chain. When off, field edits are direct (no approval flow). |
| `ENABLE_AUTO_EXPORT` | `false` | Auto-queue export when the final Ship gate clears. Requires `ENABLE_CONTRACT_EXPORT` to also be on. |
| `ENABLE_AUTO_ADVANCE` | `true` | System pass: auto-advance clean vaults from Discover to Build. |

---

## Security Considerations

Contract vaults handle sensitive legal documents. The following security principles from [Security Overview](../Security/overview.md) apply:

- **Ciphertext at rest:** All contract source files, extracted field values, and metadata are encrypted via envelope encryption (DEK wrapped by KEK).
- **Self-approval prevention:** The Patch Workflow enforces separation of duties -- no user can approve their own patch at any approval tier. The approve button is hidden (not disabled) for the patch author.
- **Append-only audit:** Every chamber transition, patch state change, export, and sync event is recorded as an immutable event. Events cannot be modified or deleted.
- **PII redaction:** Before any extracted field is sent to Otto or an external LLM, the PII redaction pipeline strips sensitive values and replaces them with synthetic tokens.
- **Workspace isolation:** Contract vaults are scoped to `workspace_id` via RLS. No cross-workspace data access is possible.

---

## Demo Prototypes

| Demo | File | What It Shows |
|---|---|---|
| **Export Center** | `contracts-export-demo.html` | Ship chamber export workflow: ready-to-ship vault cards, export queue with progress bars, recent exports with download actions, format badges (PDF/DOCX/JSON) |
| **Sync Status** | `contracts-sync-demo.html` | External system connections (Salesforce, Google Drive, S3), sync direction indicators, activity log with success/error states, retry actions |

---

## Related Specs

- [Triage Dashboard](../Triage/overview.md) -- Discover chamber intake queue
- [Contract Generator](../ContractGenerator/overview.md) -- Build chamber template-driven creation
- [Record Inspector](../Record%20Inspector/overview.md) -- Cross-chamber field card viewer
- [Patch Workflow](../PatchWorkflow/overview.md) -- Review chamber amendment workflow
- [Review Queue / Org Overview](../OrgOverview/overview.md) -- Review chamber cross-vault governance
- [Vault Hierarchy](../VaultHierarchy/overview.md) -- Parent > Division > Counterparty > Item tree
- [Batch Processing](../BatchProcessing/overview.md) -- Async ingestion pipeline
- [Feature Control Plane](../FeatureControlPlane/overview.md) -- Toggle, calibrate, audit per feature
- [Security Overview](../Security/overview.md) -- Cipher model, threat model, audit requirements
- [Shell / Master Layout](../Shell/overview.md) -- Module bar, sub-panel, Triptych layout
- [Roles & Permissions](../Roles/overview.md) -- Builder, Gatekeeper, Owner authorization
- [Document Suite](../DocumentSuite/overview.md) -- TipTap editor + PDF.js viewer
