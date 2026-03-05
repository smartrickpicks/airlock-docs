# Tasks Module

> **Status:** SPECCED — The Tasks module is the universal view of the master task table.

> **Key Insight:** Every other module generates tasks (contract triage, CRM enrichment, calendar deadlines). The Tasks module shows ALL of them in one place. It's the unfiltered master work queue — the productivity hub across all modules.

> **Tech Stack:** @hello-pangea/dnd for Kanban. See [Tech Stack](../TechStack/overview.md).

---

## Module Identity

| Property | Value |
|----------|-------|
| Module bar icon | CheckSquare (Lucide) |
| Module name | Tasks |
| Module color | Blue accent |
| Primary data | Universal task table (all modules) |
| Chamber lifecycle | Discover > Build > Review > Ship (maps to task status progression) |

---

## How Tasks Differs From Other Modules

| Other Modules | Tasks Module |
|--------------|-------------|
| Filter to `module_type = 'contracts'` (or 'crm', etc.) | Shows ALL tasks, no module filter by default |
| Triage is one view inside Discover | The entire module IS task management |
| Tasks are supporting items for vaults | Tasks are the primary content |

The Tasks module queries the same `tasks` table as every module's triage board. See [Universal Task System](../TaskSystem/overview.md) for the data model.

---

## Sub-Panel

```
+------------------------+
| TASKS                  |
| > Mini Dashboard       |
|   24 Open              |
|   8 In Progress        |
|   5 In Review          |
|   12 Done Today        |
|------------------------+
| > DISCOVER             |
|   o Inbox              |   <-- All new/unassigned tasks
|   o My Tasks           |   <-- Assigned to current user
|                        |
| > BUILD                |
|   o Kanban Board       |   <-- Drag-and-drop columns by status
|   o Focus Mode         |   <-- Single task deep-dive
|                        |
| > REVIEW               |
|   o Pending Review     |   <-- Tasks in in_review status
|   o Completed          |   <-- Recently resolved
|                        |
| > SHIP                 |
|   o Done               |   <-- Resolved/dismissed tasks
|   o Reports            |   <-- Productivity metrics
|------------------------+
| ACTIVE VAULTS          |
|   (Tasks don't have    |
|    vaults — shows      |
|    pinned task groups) |
+------------------------+
```

---

## Chamber Views

### Discover: Inbox

All tasks with `status = 'open'` across every module. The universal catch-all.

**Layout:** Sortable table (default) or card list.

| Column | Description |
|--------|-------------|
| Title | Task title (links to source vault if applicable) |
| Module | Source module badge (Contracts, CRM, Calendar, etc.) |
| Vault | Source vault name (if task is vault-scoped) |
| Type | Task type badge (triage, review, approval, manual, etc.) |
| Severity | Color-coded badge |
| Assigned To | Avatar + name (or "Unassigned") |
| Due | Countdown timer or date |
| Created | Relative timestamp |

**Filters:** Module, severity, task type, assignee, vault, date range.
**Quick filters:** "My Tasks", "Unassigned", "Overdue", "Blockers".

### Discover: My Tasks

Same as Inbox but filtered to `assigned_to = current_user`.

### Build: Kanban Board

Tasks organized into columns by status. Drag cards between columns to update status.

```
+----------+  +----------+  +----------+  +----------+
|   OPEN   |  |IN PROGRESS|  |IN REVIEW |  | RESOLVED |
+----------+  +----------+  +----------+  +----------+
| [card]   |  | [card]   |  | [card]   |  | [card]   |
| [card]   |  | [card]   |  |          |  |          |
| [card]   |  |          |  |          |  |          |
| [card]   |  |          |  |          |  |          |
+----------+  +----------+  +----------+  +----------+
```

**Card anatomy:**
```
+--------------------------------+
| [Severity dot] Task Title      |
| Module: Contracts              |
| Vault: henderson-msa           |
| Due: 2h 15m [amber countdown]  |
| [Avatar] Ana Chen              |
+--------------------------------+
```

**Kanban rules:**
- Drag from Open → In Progress: sets `assigned_to` if unassigned
- Drag from In Progress → In Review: emits review event
- Drag from In Review → Resolved: requires reviewer confirmation
- Drag backward (e.g., Resolved → Open): requires reason text
- Module-generated tasks (triage items) follow their module's lifecycle rules — can't arbitrarily move a preflight blocker to "resolved" without actual resolution

**Grouping options (dropdown):**
- By status (default — the Kanban columns)
- By module (one swimlane per module)
- By assignee (one swimlane per person)
- By vault (one swimlane per source vault)
- By due date (overdue, today, this week, later)

### Build: Focus Mode

Single-task deep-dive. Select a task from Inbox or Kanban → Focus Mode shows:

| Panel | Content |
|-------|---------|
| **Signal** | Task history: status changes, comments, linked events from source vault |
| **Orchestrate** | Task detail: title, description, evidence, linked vault context. Action buttons. |
| **Control** | Metadata: assignee, severity, due date, source module, created by. Edit controls. |

Focus Mode uses the triptych layout just like vault views.

### Review: Pending Review

Tasks with `status = 'in_review'`. The Gatekeeper's view of submitted work across all modules.

### Ship: Done + Reports

**Done:** Resolved/dismissed tasks (last 30 days). Searchable archive.

**Reports:** Productivity metrics:
- Tasks resolved per day/week/month
- Average resolution time by module
- Tasks by severity distribution
- Assignee workload comparison
- Overdue rate trending

---

## Task Creation in the Tasks Module

Users can create tasks manually from the Tasks module:

**Slash command:** `/task [title]` in TipTap blocks
**Button:** "New Task" button in Inbox toolbar
**Quick add:** Inline "Add task" row at bottom of Kanban columns

Manual task fields:
- Title (required)
- Description (optional, TipTap rich text)
- Module (defaults to 'tasks', can be set to any module)
- Vault (optional — link to an existing vault)
- Severity (default: info)
- Assignee (optional)
- Due date (optional)

---

## Cross-Module Task Flow

```
Contract Shipped (Contracts)
    → BullMQ → Create "Onboarding" tasks in Tasks module
    → Tasks appear in: Tasks Inbox, Home Tasks widget, Contracts Signal panel

CRM Lead Created (CRM)
    → BullMQ → Create "Enrich account" task
    → Task appears in: Tasks Inbox, Home CRM widget, CRM Discover

SLA Warning (any module)
    → BullMQ → Create "SLA warning" task
    → Task appears in: Tasks Inbox, source module triage, Home widget
    → Novu → Push notification to assigned user
```

---

## Related Specs

- **[Universal Task System](../TaskSystem/overview.md)** — Master task table and lifecycle
- **[Tech Stack](../TechStack/overview.md)** — @hello-pangea/dnd for Kanban
- **[Shell Overview](../Shell/overview.md)** — Home workspace task widgets
- **[Notifications](../Notifications/overview.md)** — Task assignments trigger notifications
