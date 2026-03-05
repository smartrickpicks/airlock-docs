# Triage Dashboard

> **Decision:** Triage is the analyst's home -- they start their day here, then drill into contract channels.

## Position in Airlock

| Attribute | Value |
|---|---|
| Module | Contracts |
| Sidebar behavior | Singleton channel, pinned at top of sidebar |
| Badge | Open issue count (real-time via subscription) |
| Layout | Full-width dashboard (no triptych -- this is not a contract view) |

## OrcestrateOS Lineage

| Backend artifact | Detail |
|---|---|
| Route file | `server/routes/triage_items.py` (343 lines) |
| Pagination | Cursor-based, limit 1--200, default 50 |
| Concurrency | Optimistic locking via `version` field on each triage item |

---

## Data Model

### Triage Item Fields

| Field | Type | Values |
|---|---|---|
| Severity | enum | `info` / `warning` / `blocker` |
| Source | enum | `qa_rule` / `preflight` / `system_pass` / `manual` |
| Status | enum | `open` / `in_review` / `resolved` / `dismissed` |
| Contract | FK | Link to parent contract |
| Field | string | The specific contract field in question |
| Assigned To | FK | Analyst user |
| Created | datetime | |
| Updated | datetime | |

---

## Dashboard Sections

The dashboard is a single scrollable page composed of five stacked sections.

### 1. Metrics Strip

A horizontal row of summary cards at the top of the page.

| Metric | Description |
|---|---|
| Open Issues | Count of items with status `open` |
| In Review | Count of items with status `in_review` |
| Resolved Today | Count of items resolved since midnight (analyst's local time) |
| Avg Resolution Time | Mean time from `open` to `resolved` across visible items |

### 2. Filter Bar

Persistent filter bar below the metrics strip. All filters combine with AND logic.

| Filter | Control type | Notes |
|---|---|---|
| Severity | Multi-select chips | `info` / `warning` / `blocker` |
| Source | Multi-select chips | `qa_rule` / `preflight` / `system_pass` / `manual` |
| Entity | Searchable dropdown | Filter by counterparty or legal entity |
| Batch | Searchable dropdown | Filter by processing batch |
| Date range | Date picker (range) | Applies to `created` timestamp |
| Assignee | Searchable dropdown | Analyst users |

**Quick-filter presets** (toggle buttons alongside the filter bar):

- **My Issues** -- filters to current user's assignments
- **Blockers Only** -- filters to severity `blocker`
- **Needs Verification** -- filters to status `in_review`

### 3. Triage Table

The primary content area. Sortable, paginated table.

| Column | Sortable | Notes |
|---|---|---|
| (checkbox) | -- | Row selection for bulk ops |
| Contract | yes | Contract name, links to contract channel |
| Field | yes | Specific field flagged |
| Issue Type | yes | Human-readable issue label |
| Severity | yes | Color-coded badge: blue/amber/red |
| Source | yes | Chip label |
| Status | yes | Color-coded badge |
| Assigned To | yes | Avatar + name |
| Created | yes | Relative timestamp |
| Updated | yes | Relative timestamp |

### 4. Batch Progress Cards

Horizontal scrollable row of cards, one per active batch.

- Batch name and ID
- Progress bar (processed / total contracts)
- Counts by triage status (open / in_review / resolved / dismissed)
- Timestamp of last activity

### 5. Entity Health Cards

Horizontal scrollable row of cards, one per entity with open items.

- Entity name
- Aggregate health score
- Open issue count by severity
- Trend indicator (improving / worsening / stable)

---

## Actions

### Row Click

Clicking a triage table row navigates to the parent contract's channel, scrolled to the relevant triage item in the Signal feed.

### Bulk Operations

1. Select rows via checkboxes (individual or "select all on page").
2. A **bulk actions bar** appears at the top of the table when one or more rows are selected.
3. Available bulk actions:

| Action | Detail |
|---|---|
| Assign | Reassign selected items to a chosen analyst |
| Change Severity | Set a new severity for all selected items |
| Dismiss | Dismiss selected items; requires a reason (free-text modal) |

4. All bulk actions surface a **confirmation modal** before executing.

---

## Module-Level Notification Aggregator

> **Decision:** The Triage Dashboard also serves as the module-level notification aggregator for the Contracts module.

### "Recent Events" Tab

- Secondary tab alongside the default triage view.
- Shows **all contract events across all contracts** in reverse-chronological order.
- Event types include: status changes, preflight completions, agent messages, approval actions, triage item updates.
- Same cursor-based pagination as the triage table.

---

## Pagination

| Parameter | Value |
|---|---|
| Strategy | Cursor-based (keyset pagination) |
| Min limit | 1 |
| Max limit | 200 |
| Default limit | 50 |
| Concurrency control | Optimistic locking via `version` field |
