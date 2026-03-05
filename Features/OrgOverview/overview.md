# Review Queue — Design Spec

> **Status:** SPECCED — Review Queue is the Gatekeeper's cross-vault work surface within Contracts > Review.

> **Key Insight:** What was "Org Overview" in OrcestrateOS becomes the **Review Queue** in Airlock. It's a Gatekeeper-centric view that aggregates all vaults needing review across the Contracts module, grouped by parent entity. This is NOT a standalone module — it lives at Contracts > Review > Review Queue.

> **Terminology:** See [Naming & Hierarchy](../Shell/naming-and-hierarchy.md). This is a **view** within the **Review** chamber of the Contracts module.

---

## Position in Airlock

| Attribute | Value |
|---|---|
| Module | Contracts |
| Chamber | Review (purple) |
| View name | Review Queue |
| Sub-panel location | Contracts > Review > Review Queue |
| Layout | Full-width dashboard (no triptych — this is a chamber view, not a vault view) |
| Primary role | Gatekeeper (Verifier) |
| Route | `/contracts/review-queue` |

---

## Source Mapping (OrcestrateOS)

| OrcestrateOS File | Size | Airlock Target |
|---|---|---|
| `org-overview-demo.html` | 52 KB | Review Queue UI (entity cards, alert panels, activity feed) |
| `server/routes/admin_reporting.py` | 420 lines | Entity aggregation queries, team metrics |
| `server/routes/triage_items.py` | 343 lines | Alert/triage item data (shared with Triage Board) |

---

## What Changed from OrcestrateOS

| OrcestrateOS (Org Overview) | Airlock (Review Queue) |
|---|---|
| Standalone page with its own sidebar | View within Contracts module — sub-panel provides nav |
| "Org Overview" title | "Review Queue" — describes its function |
| Sidebar with team overview, progress, role | All moved to sub-panel mini dashboard and Admin overlay |
| Otto FAB chat widget | AI Agent in Control panel (triptych) — available when viewing a vault |
| Three nav items (Org Overview, Triage, Verifier Review) | Three views in Review chamber: Review Queue, Record Inspector, Patch Approvals |
| Entity-centric card grouping | Vault hierarchy grouping — parent vaults are entities |

---

## Layout

Full-width dashboard with four stacked sections:

```
+----------------------------------------------------------+
| REVIEW QUEUE                          [Group ▾] [Filter]  |
|----------------------------------------------------------|
|                                                           |
| 1. PARENT VAULT CARDS (Entity Strip)                     |
| +--Ostereo--+  +--Get Dough--+  +--Broke Records--+     |
| | 62 vaults  |  | 9 vaults    |  | 15 vaults       |    |
| | Health: 41 |  | Health: 52  |  | Health: 49       |    |
| |  [expand]  |  |             |  |  [expand]        |    |
| +------------+  +-------------+  +------------------+    |
|                                                           |
| 2. HANDOFF SIGNALS (Alert Panels)                        |
| +-----RFIs------+  +--Corrections--+  +--Anomalies--+   |
| |  3 pending    |  |  8 pending    |  |  3 flagged   |   |
| |  details...   |  |  details...   |  |  details...  |   |
| +---------------+  +---------------+  +--------------+   |
|                                                           |
| 3. FILTER BAR                                            |
| [All] [Patches] [RFIs] [Corrections] [Anomalies]        |
| Analyst: [▾ All]  Entity: [▾ All]                        |
|                                                           |
| 4. ACTIVITY FEED                                         |
| +-- Today --+                                            |
| | o Correction: SF Account Name changed (Daniele Leoni)  |
| | o Anomaly: Account name not in document                |
| | o Patch #12 submitted (Daniele Leoni)                  |
| +-- Yesterday --+                                        |
| | o Activity: Ana Chen daily summary                     |
| | ...                                                    |
+----------------------------------------------------------+
```

---

## Section 1: Parent Vault Cards (Entity Strip)

Horizontal row of cards, one per **parent vault** (Level 1 in the [vault hierarchy](../VaultHierarchy/overview.md)). These are the top-level accounts/entities.

### Card Anatomy

| Element | Content |
|---------|---------|
| **Header** | Parent vault name + type badge (Distribution, License, etc.) + expand chevron |
| **Stats row** | Vault count, aggregate health score |
| **Assigned** | Comma-separated list of assigned Builders |
| **Progress bar** | Percentage of child vaults in Build chamber or beyond (color-coded: red <30%, amber 30-60%, green >60%) |
| **Progress label** | "X% build-ready (N/M)" |
| **Action counts** | Patches, RFIs, Corrections, Anomalies — amber/red highlights for urgent counts |

### Expanded State

Clicking a card expands it to show a **child vault table**:

| Column | Sortable | Notes |
|---|---|---|
| Counterparty | Yes | Level 3 vault name (counterparty within this parent) |
| Builder | Yes | Assigned Builder name |
| Health | Yes | Numeric health score (color-coded) |
| Status | Yes | Gate status dot (pass/review/fail/failed) + label |
| Count | No | Number of item vaults under this counterparty |

Child vaults are grouped by **category sub-headers** (collapsible). Categories come from contract type or division vault names.

Clicking a row navigates to that vault's triptych view.

### Card Interactions

| Action | Behavior |
|--------|----------|
| Click header | Toggle expand/collapse |
| Click child row | Navigate to vault: `/contracts/{vault-slug}/record-inspector` |
| Hover card | Subtle border highlight |
| Expanded border | Cyan accent border (matches active state) |

---

## Section 2: Handoff Signals (Alert Panels)

Three alert summary cards showing the Gatekeeper's immediate priorities.

| Panel | Icon | What it counts | Breakdown |
|-------|------|---------------|-----------|
| **RFIs** | ? | Open Requests for Information | N open, M responded |
| **Corrections** | Pencil | Submitted data corrections (patches) | N pending review, M approved, K rejected |
| **Anomalies** | Warning | System-detected data quality issues | N OCR misreads, M account mismatches, K processing failures |

### Panel Anatomy

| Element | Content |
|---------|---------|
| **Icon** | Category icon |
| **Title** | Category name |
| **Count** | Large monospace number (color-coded by category) |
| **Breakdown** | Status distribution in muted text |
| **Detail rows** | Top 3-5 items, showing: vault name, entity tag, analyst, timestamp, description |
| **Overflow link** | "View all N items" — opens a modal with the complete list |

### Alert Modal

Clicking "View all" opens a centered modal:

| Element | Content |
|---------|---------|
| **Header** | Category icon + "All [Category]" title + close button |
| **Body** | Scrollable list of all items in this category |
| **Row** | Vault name + entity tag + description + analyst + timestamp |
| **Click row** | Navigate to vault's Signal panel, scrolled to the relevant event |

---

## Section 3: Filter Bar

Persistent filter bar controlling the Activity Feed below. All filters combine with AND logic.

| Filter | Control | Options |
|--------|---------|---------|
| **Type** | Signal pills (toggle) | All, Patches, RFIs, Corrections, Anomalies, Activity, Escalations |
| **Builder** | Dropdown | All, [list of Builders in workspace] |
| **Entity** | Dropdown | All, [list of parent vault names] |

Active pill gets cyan accent background + border.

---

## Section 4: Activity Feed

Reverse-chronological event stream across all vaults in the Contracts module. Grouped by time period (Today, Yesterday, This Week, Earlier).

### Feed Item Anatomy

| Element | Content |
|---------|---------|
| **Dot** | Color-coded by event type (see below) |
| **Title** | Event headline (e.g., "SF Account Name changed") |
| **Badge** | Category badge (RFI, CORRECTION, ANOMALY) — only for flagged types |
| **Meta row** | Vault name (bold), entity name, Builder name, relative timestamp |
| **Detail** | Additional context — the before/after, the question, the error message |
| **Arrow** | Hover-reveal right arrow indicating clickability |

### Event Types and Colors

| Type | Dot Color | Animation | Badge |
|------|-----------|-----------|-------|
| `patch.submitted` | Red | None | None |
| `patch.approved` | Green | None | None |
| `patch.clarification_needed` | Amber | None | None |
| `rfi.created` | Amber | None | RFI |
| `correction.submitted` | Blue | None | CORRECTION |
| `anomaly.detected` | Red | Pulse | ANOMALY |
| `activity.summary` | Teal | None | None |
| `escalation.triggered` | Red | Pulse | None |

### Feed Behavior

| Aspect | Behavior |
|--------|----------|
| **Click item** | Navigate to vault's Signal panel, scrolled to the originating event |
| **Filtering** | Items filter instantly based on filter bar selections |
| **Empty state** | "No matching events" centered message |
| **Auto-refresh** | WebSocket subscription updates feed in real-time |
| **Pagination** | Infinite scroll, loads in batches of 20 |

---

## Data Sources

### Parent Vault Cards

```sql
-- Top-level aggregation
SELECT
    v.id, v.name, v.vault_type,
    COUNT(DISTINCT cv.id) AS vault_count,
    AVG(cv.health_score) AS avg_health,
    SUM(CASE WHEN cv.chamber IN ('build', 'review', 'ship') THEN 1 ELSE 0 END) AS build_ready
FROM vaults v
LEFT JOIN vaults cv ON cv.parent_vault_id = v.id
WHERE v.workspace_id = :ws
  AND v.vault_level = 1
  AND v.module_type = 'contracts'
GROUP BY v.id, v.name, v.vault_type
ORDER BY vault_count DESC
```

### Alert Panels

Alerts are derived from the [Universal Task System](../TaskSystem/overview.md):

```sql
-- RFIs
SELECT * FROM tasks
WHERE workspace_id = :ws AND module_type = 'contracts'
  AND task_type = 'action' AND metadata->>'subtype' = 'rfi'
  AND status IN ('open', 'in_progress')
ORDER BY created_at DESC

-- Corrections (patches pending review)
SELECT * FROM tasks
WHERE workspace_id = :ws AND module_type = 'contracts'
  AND task_type = 'approval' AND status IN ('open', 'in_review')
ORDER BY created_at DESC

-- Anomalies
SELECT * FROM tasks
WHERE workspace_id = :ws AND module_type = 'contracts'
  AND task_type = 'triage' AND severity IN ('warning', 'blocker')
  AND metadata->>'subtype' IN ('ocr_misread', 'account_mismatch', 'processing_failure')
  AND status = 'open'
ORDER BY severity DESC, created_at DESC
```

### Activity Feed

Events from the channel event system:

```sql
SELECT e.*, c.name AS vault_name, c.category AS entity_name
FROM channel_events e
JOIN channels c ON c.id = e.channel_id
WHERE c.workspace_id = :ws
  AND c.module_type = 'contracts'
  AND e.event_type IN (
    'patch.submitted', 'patch.approved', 'patch.clarification_needed',
    'rfi.created', 'correction.submitted', 'anomaly.detected',
    'activity.summary', 'escalation.triggered'
  )
ORDER BY e.created_at DESC
LIMIT 20 OFFSET :cursor
```

---

## Grouping Options

The Review Queue supports switching the primary grouping of parent vault cards:

| Grouping | What changes | Default |
|----------|-------------|---------|
| **By entity** | Cards = parent vaults (Level 1) | Yes (default) |
| **By chamber** | Cards = chamber stages, showing vault counts per stage | No |
| **By builder** | Cards = team members, showing their assigned vault counts | No |

Dropdown at top-right of the page header.

---

## Role Visibility

| Role | What they see |
|------|--------------|
| **Builder** | Review Queue with only their assigned vaults visible in entity cards. Feed filtered to their events. |
| **Gatekeeper** | Full Review Queue — all vaults across all Builders. The primary user of this view. |
| **Owner** | Full Review Queue + additional "approval needed" indicators on cards. |
| **Architect** | Full Review Queue. |

---

## Related Specs

- **[Triage Board](../Triage/overview.md)** — Discover chamber's work queue (Builder-focused). Review Queue is the Review chamber equivalent (Gatekeeper-focused).
- **[Vault Hierarchy](../VaultHierarchy/overview.md)** — Parent vault cards map to Level 1 vaults.
- **[Universal Task System](../TaskSystem/overview.md)** — Alert panels query the master task table.
- **[Record Inspector](../Record%20Inspector/overview.md)** — Clicking a vault row drills into the Record Inspector.
- **[Patch Workflow](../PatchWorkflow/overview.md)** — Corrections and patches in the feed originate from the patch workflow.
- **[Notifications](../Notifications/overview.md)** — Escalations in the feed are triggered by the notification system.
