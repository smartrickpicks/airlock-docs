# Universal Task & Triage System

> **Status:** FOUNDATIONAL — This spec defines the universal task system that underlies every module's triage view and the Home workspace widgets.

> **Key Insight:** Think of it like a SQL database. There is ONE master task table. Every module's triage board is a **filtered view** (like a SQL VIEW) off that master table. The Home workspace shows summary widgets — small table previews of each module's triage, all querying the same underlying data.

> **Terminology:** "Task" here means any actionable item — triage items, review requests, approval needs, SLA warnings, extraction issues, entity disambiguation. Not just the Tasks module's task cards.

---

## Architecture

```
+---------------------------------------------------------------+
|                    MASTER TASK TABLE                           |
|  (universal schema: who, what, where, when, severity, status) |
+---------------------------------------------------------------+
        |              |              |              |
        v              v              v              v
  +-----------+  +-----------+  +-----------+  +-----------+
  | Contracts |  |    CRM    |  |   Tasks   |  | Calendar  |
  |  Triage   |  |  Triage   |  |  Triage   |  |  Triage   |
  |  (VIEW)   |  |  (VIEW)   |  |  (VIEW)   |  |  (VIEW)   |
  +-----------+  +-----------+  +-----------+  +-----------+
        |              |              |              |
        +------+-------+------+------+
               |              |
               v              v
        +-------------+  +-----------+
        |    HOME     |  |  TASKS    |
        |  Dashboard  |  |  Module   |
        |  (widgets)  |  | (all tasks)|
        +-------------+  +-----------+
```

### Data Flows

1. **System creates tasks** — extraction completes, preflight fails, SLA approaches, entity needs disambiguation, patch needs review. Each event creates a task row in the master table.

2. **Module triage = filtered view** — When a user opens Contracts > Discover > Triage Board, they see tasks WHERE `module_type = 'contracts'`. Same data, different filter.

3. **Home widgets = summary view** — Home workspace shows a compact table widget per module, each showing the top N urgent tasks for that module.

4. **Tasks module = the whole table** — The Tasks module is the unfiltered view of ALL tasks across all modules. It's the universal work queue.

---

## Master Task Schema

```sql
CREATE TABLE tasks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id),

    -- What
    title TEXT NOT NULL,
    description TEXT,
    task_type TEXT NOT NULL,              -- 'triage' | 'review' | 'approval' | 'action' | 'sla_warning' | 'entity_resolution' | 'manual'

    -- Where (module + vault context)
    module_type TEXT NOT NULL,            -- 'contracts' | 'crm' | 'tasks' | 'calendar' | 'documents'
    vault_id UUID REFERENCES vaults(id), -- which vault this task belongs to (nullable for workspace-level tasks)
    field_code TEXT,                      -- specific field (for triage items from extraction)

    -- Priority & Status
    severity TEXT NOT NULL DEFAULT 'info', -- 'info' | 'warning' | 'blocker' | 'urgent'
    status TEXT NOT NULL DEFAULT 'open',   -- 'open' | 'in_progress' | 'in_review' | 'resolved' | 'dismissed'
    priority INT NOT NULL DEFAULT 0,       -- numeric sort weight (higher = more urgent)

    -- People
    created_by UUID REFERENCES users(id),
    assigned_to UUID REFERENCES users(id),
    source TEXT NOT NULL DEFAULT 'system', -- 'system' | 'preflight' | 'qa_rule' | 'manual' | 'ai' | 'cross_module'

    -- Lifecycle
    due_at TIMESTAMPTZ,                   -- SLA deadline (nullable)
    resolved_at TIMESTAMPTZ,
    resolution_note TEXT,

    -- Metadata
    metadata JSONB NOT NULL DEFAULT '{}', -- type-specific data (evidence, context, linked events)
    version INT NOT NULL DEFAULT 1,       -- optimistic locking

    -- Timestamps
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_tasks_workspace_module ON tasks(workspace_id, module_type, status);
CREATE INDEX idx_tasks_vault ON tasks(vault_id, status);
CREATE INDEX idx_tasks_assigned ON tasks(assigned_to, status);
CREATE INDEX idx_tasks_due ON tasks(due_at) WHERE status IN ('open', 'in_progress');
CREATE INDEX idx_tasks_severity ON tasks(workspace_id, severity, status);
```

### Task Types

| Type | Source | Module | Description |
|------|--------|--------|-------------|
| `triage` | Preflight/QA rules | Contracts | Field-level quality issue (existing triage_items, migrated) |
| `review` | Gate advancement | Contracts | Vault needs human review at a gate checkpoint |
| `approval` | Patch workflow | Contracts | Patch submitted, needs verifier/admin approval |
| `action` | Manual/system | Any | Generic actionable item (clean up data, verify entity, etc.) |
| `sla_warning` | SLA engine | Any | Deadline approaching or overdue |
| `entity_resolution` | Entity resolver | Contracts/CRM | Ambiguous entity needs human disambiguation |
| `manual` | User-created | Any | User manually creates a task (slash command, button) |
| `onboarding` | Cross-module | Tasks | Auto-created when contract ships (onboarding checklist) |

---

## Module Triage Views

Each module's sub-panel includes a Triage Board (or equivalent) under the Discover chamber. Each is a filtered query against the master task table.

### Contracts Triage

```sql
-- Contracts > Discover > Triage Board
SELECT * FROM tasks
WHERE workspace_id = :ws
  AND module_type = 'contracts'
  AND status IN ('open', 'in_progress', 'in_review')
ORDER BY
  CASE severity WHEN 'urgent' THEN 0 WHEN 'blocker' THEN 1 WHEN 'warning' THEN 2 ELSE 3 END,
  due_at NULLS LAST,
  created_at DESC
```

**Columns:** Vault name, Field, Issue Type, Severity, Source, Status, Assigned To, Due, Updated

This is the existing triage dashboard (see [Triage spec](../Triage/overview.md)), now backed by the universal table.

### CRM Triage

```sql
-- CRM > Discover > New Leads
SELECT * FROM tasks
WHERE workspace_id = :ws
  AND module_type = 'crm'
  AND task_type IN ('entity_resolution', 'action', 'manual')
  AND status IN ('open', 'in_progress')
ORDER BY priority DESC, created_at DESC
```

**Columns:** Account/Vault name, Issue, Severity, Status, Assigned To, Updated

**CRM-specific tasks:** Unqualified leads (low entity confidence), account enrichment needed, duplicate accounts detected, relationship health alerts.

### Tasks Module (Universal View)

```sql
-- Tasks module: ALL tasks across ALL modules
SELECT * FROM tasks
WHERE workspace_id = :ws
  AND status IN ('open', 'in_progress', 'in_review')
ORDER BY priority DESC, due_at NULLS LAST, created_at DESC
```

The Tasks module is the **unfiltered master view**. It shows everything — contracts triage items, CRM leads, calendar deadlines, manual tasks. Users can filter by module, severity, assignee, vault, or date range.

**Layout options (user switchable):**
- **Table view** — sortable columns, bulk actions (default)
- **Kanban view** — columns by status (open | in_progress | in_review | resolved), drag cards between columns
- **Timeline view** — tasks plotted on a timeline by due_at, color-coded by module

### Calendar "Triage"

Calendar doesn't have a traditional triage board. Instead, it surfaces tasks with due dates:

```sql
-- Calendar module shows date-bound tasks
SELECT * FROM tasks
WHERE workspace_id = :ws
  AND due_at IS NOT NULL
  AND due_at BETWEEN :start AND :end
ORDER BY due_at ASC
```

Calendar events are rendered on Schedule-X, color-coded by source module.

---

## Home Workspace Triage Widgets

> **Key Concept:** The Home workspace is a Notion-style page. Each module's triage appears as an **inline table widget** — a compact, read-only preview of that module's top tasks.

### Widget Layout on Home

```
+----------------------------------------------------------+
| Good morning, Ana                                         |
| You have 7 items needing attention                        |
|----------------------------------------------------------|
|                                                           |
| CONTRACTS TRIAGE                              [View All →]|
| +------+----------+---------+--------+--------+          |
| | Vault| Issue     |Severity |Status  |Due     |          |
| +------+----------+---------+--------+--------+          |
| | hend.| Missing $ | blocker | open   | 2h 15m |          |
| | sony | Low conf  | warning | open   | —      |          |
| | warn.| Entity    | warning | review | 4h 30m |          |
| +------+----------+---------+--------+--------+          |
|                                                           |
| CRM TRIAGE                                    [View All →]|
| +------+----------+---------+--------+--------+          |
| | Vault| Issue     |Severity |Status  |Due     |          |
| +------+----------+---------+--------+--------+          |
| | Acme | New lead  | info    | open   | —      |          |
| | Sony | Dup acct  | warning | open   | —      |          |
| +------+----------+---------+--------+--------+          |
|                                                           |
| TASKS                                         [View All →]|
| +------+----------+---------+--------+--------+          |
| | Task | Context   |Priority |Status  |Due     |          |
| +------+----------+---------+--------+--------+          |
| | Onb. | henderson | high    | open   | 1d     |          |
| | Rev. | sony Q1   | medium  | prog   | 3d     |          |
| +------+----------+---------+--------+--------+          |
|                                                           |
| [Module Status Cards]                                     |
| Contracts: 15/28/14/6  |  CRM: 42  |  Tasks: 18          |
|                                                           |
| [User's custom blocks below]                              |
+----------------------------------------------------------+
```

### Widget Behavior

| Aspect | Behavior |
|--------|----------|
| **Data source** | `SELECT * FROM tasks WHERE module_type = :module AND status IN ('open', 'in_progress') ORDER BY priority DESC LIMIT 5` |
| **Rows shown** | Top 3-5 tasks by priority (configurable per widget) |
| **Columns** | Condensed: Vault name (truncated), Issue title, Severity badge, Status badge, Due countdown |
| **Click row** | Navigates to vault in that module (switches module, opens vault) |
| **[View All]** | Navigates to module's triage board (full view) |
| **Auto-refresh** | Widgets update via WebSocket subscription to task events |
| **Empty state** | "No open items" with green checkmark |
| **Collapsed** | Widget can be collapsed to just header + count badge |

### Widget Ordering

| Method | Detail |
|--------|--------|
| **Default** | Ordered by urgency: module with most blockers/urgent tasks first |
| **Manual** | User can drag-reorder widgets on their Home page (persisted) |
| **Admin template** | Admin configures which widgets appear by default for new users |
| **Per-user** | Users can hide/show module widgets, add custom widgets (TipTap blocks) |

### Widget Configuration (Per User)

Each widget supports:
- **Filter preset** — user can set a default filter (e.g., "My Issues Only")
- **Row count** — 3, 5, or 10 rows visible
- **Auto-collapse** — collapse when no open items (saves space)
- **Severity filter** — only show blockers+warnings (hide info)

---

## Task Creation Triggers

### Automatic (System-Generated)

| Trigger | Task Type | Module | Severity |
|---------|-----------|--------|----------|
| Extraction completes with low-confidence fields | `triage` | contracts | warning |
| Preflight gate fails (RED) | `triage` | contracts | blocker |
| Preflight gate warns (YELLOW) | `triage` | contracts | warning |
| QA rule fires | `triage` | contracts | varies |
| Entity resolution ambiguous | `entity_resolution` | contracts | warning |
| Entity resolution no match | `entity_resolution` | crm | info |
| SLA < 1 hour remaining | `sla_warning` | (source) | urgent |
| SLA overdue | `sla_warning` | (source) | blocker |
| Patch submitted → needs review | `approval` | contracts | warning |
| Patch needs admin approval | `approval` | contracts | warning |
| Contract shipped → onboarding | `onboarding` | tasks | info |
| Contract date extracted → calendar event | `action` | calendar | info |
| Health score drops > 10% | `triage` | contracts | warning |
| Batch processing fails 3x | `triage` | contracts | blocker |

### Manual (User-Created)

Users can create tasks via:
- **Slash command** in Home workspace: `/task Create follow-up for Henderson MSA`
- **"Add Task" button** in any triage view
- **Context menu** on a vault: Right-click > "Create Task"
- **Signal panel** action: "Flag this for review"

Manual tasks set `source = 'manual'` and `created_by = current_user`.

---

## Task Lifecycle

```
               +---------+
               |  OPEN   |
               +---------+
                 |     |
        assign   |     | self-assign
                 v     v
            +-------------+
            | IN_PROGRESS |
            +-------------+
                 |     |
       submit    |     | dismiss (with reason)
                 v     |
            +-----------+     +----------+
            | IN_REVIEW |---->| DISMISSED|
            +-----------+     +----------+
                 |
        approve  |  reject (→ back to OPEN)
                 v
            +----------+
            | RESOLVED |
            +----------+
```

### Status Transitions

| From | To | Actor | Notes |
|------|----|-------|-------|
| open | in_progress | Builder/assigned | User starts working on the task |
| open | dismissed | Builder/Gatekeeper | Must provide reason; logged |
| in_progress | in_review | Builder | Submits work for review |
| in_progress | resolved | Builder | Self-resolve (for info/manual tasks) |
| in_review | resolved | Gatekeeper | Approves the resolution |
| in_review | open | Gatekeeper | Rejects, sends back with notes |
| any | dismissed | Admin | Admin override dismiss |

---

## Cross-Module Task Propagation

When events in one module create tasks in another:

```
Contract shipped (Contracts module)
    |
    +--[BullMQ job]--> Create "onboarding" task (Tasks module)
    +--[BullMQ job]--> Create "update CRM" task (CRM module)
    +--[BullMQ job]--> Create "renewal date" event (Calendar module)

Entity resolved as new customer (Contracts module)
    |
    +--[BullMQ job]--> Create "enrich account" task (CRM module)

SLA approaching on any vault (any module)
    |
    +--[BullMQ job]--> Create "sla_warning" task (source module)
    +--[Novu]--> Push notification to assigned user
```

Each cross-module task includes `metadata.source_vault_id` and `metadata.source_event_id` so users can trace back to the originating event.

---

## Migration: Existing Triage Items → Tasks

The existing `triage_items` table from OrcestrateOS maps cleanly:

| triage_items field | tasks field |
|--------------------|-------------|
| id | id |
| contract_id | vault_id |
| check_code | field_code |
| severity | severity |
| status | status |
| source | source |
| assigned_to | assigned_to |
| message | title |
| details | description |
| resolution_note | resolution_note |
| version | version |
| created_at | created_at |

Add: `module_type = 'contracts'`, `task_type = 'triage'`, compute `priority` from severity.

---

## Related Specs

- **[Triage Dashboard](../Triage/overview.md)** — Contracts-specific triage view (now backed by universal tasks)
- **[Vault Hierarchy](../VaultHierarchy/overview.md)** — Tasks reference vaults
- **[Notifications](../Notifications/overview.md)** — Tasks trigger notifications via Novu
- **[Tech Stack](../TechStack/overview.md)** — BullMQ for cross-module propagation, @hello-pangea/dnd for Kanban
- **[Shell Overview](../Shell/overview.md)** — Home workspace widget layout
