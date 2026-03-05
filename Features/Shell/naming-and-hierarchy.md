# Naming Conventions & Navigation Hierarchy

> **Status:** FOUNDATIONAL — This spec defines the vocabulary and navigation structure for all of Airlock. Every other spec references this.

> **Rule:** One term per concept. No synonyms. If this doc says "Vault," never say "channel," "workstream," or "item" in the UI.

## Core Vocabulary

### Vault

The **vault** is Airlock's fundamental unit of work. It is the interactive, living record of how something moves from discovery to release.

**What a vault contains:**
- The **source material** (a contract PDF, a deal, a task, a creative brief)
- The **annotation layer** (extracted fields, AI enrichments, human corrections)
- The **metadata** (health score, gate status, entity resolution, ownership)
- The **audit trail** (every action, by whom, when, why)
- The **collaboration context** (assigned stakeholders, role-based views, approvals)

**What a vault is NOT:**
- Not just a file (it's the intelligence wrapped around the file)
- Not a chat room (collaboration happens through structured events, not free-form chat)
- Not a task (a task is one action; a vault is the entire lifecycle)

**A vault is a secure container where data enters raw, moves through governed chambers, and exits trustworthy.**

**How vaults work:**
1. A piece of work enters the system (document upload, form submission, API ingest)
2. A vault is created automatically
3. Stakeholders are auto-assigned based on rules (who uploaded, their supervisor, the assigned gatekeeper)
4. The vault appears in each stakeholder's module sidebar under "Active Vaults"
5. Each person sees a different view based on their **role** and the vault's **current gate**
6. As work progresses through chambers, the vault accumulates events, approvals, corrections
7. When the vault reaches the Ship chamber and passes all gates, it's released and archived

**Examples across modules:**

| Module | What creates a vault | Vault name example | Source material |
|--------|---------------------|-------------------|-----------------|
| Contracts | PDF upload | `Henderson MSA` | Contract PDF |
| CRM | New deal created | `Acme Corp Q1 Renewal` | Deal record |
| Tasks | Task assigned | `Migrate Auth to OAuth2` | Task brief |
| Creative | Brief submitted | `Q1 Campaign - Social` | Creative brief |
| Documents | Document uploaded | `2024 Annual Report` | PDF/DOCX |

**Route pattern:** `/contracts/henderson-msa/record-inspector`
- Segment 1: Module ID
- Segment 2: Vault slug (derived from vault name)
- Segment 3: Active view name

### Module

A **module** is a top-level domain — the equivalent of a Discord server. Each module represents a major functional area of the organization.

**Core modules (v1):**

| Module | Icon | Description |
|--------|------|-------------|
| Contracts | FileText | Contract lifecycle management |
| CRM | Users | Customer and entity management |
| Tasks | CheckSquare | Task tracking and assignment |
| Calendar | Calendar | Scheduling and deadline views |
| Documents | FolderOpen | Document library and templates |

**Admin is NOT a module.** It's a system overlay (see below).

**Navigation:** Clicking a module icon in the Module Bar opens that module's sub-panel and loads the last-visited view in the main content area.

### Chamber

A **chamber** is one of the four universal lifecycle stages that every vault passes through. Chambers are the primary navigation sections in each module's sub-panel.

**The four chambers:**

| Chamber | Verb | Color | Primary Role | What happens here |
|---------|------|-------|-------------|-------------------|
| **Discover** | Find it | Red | Builder | Intake, triage, assignment |
| **Build** | Shape it | Yellow | Builder | Extraction, standardization, enrichment |
| **Review** | Verify it | Purple | Gatekeeper | Quality review, approval chain |
| **Ship** | Release it | Green | Owner | Export, sync, publish |

**Previous names:** Discovery, Standardize, Verify, Release. Updated to verb-first short forms for clarity across all modules.

**Chambers contain views** — the specific screens and tools available at each stage. See "View" below.

### Gate

A **gate** is a checkpoint within a chamber where human action is required. Gates determine what view renders and what actions are available. A single chamber can have multiple gates.

```
DISCOVER
  |-- gate_ingest      (document received, processing started)
  |-- gate_triage      (human triages: assign, prioritize, flag)

BUILD
  |-- gate_extract     (extraction complete, fields available)
  |-- gate_preflight   (automated quality checks run)
  |-- gate_enrich      (entity resolution, external data matched)

REVIEW
  |-- gate_builder     (builder self-review before submission)
  |-- gate_gatekeeper  (gatekeeper evaluates evidence)
  |-- gate_owner       (owner final approval)

SHIP
  |-- gate_export      (format selected, export generated)
  |-- gate_sync        (external system sync confirmed)
```

### View

A **view** is a specific screen or tool within a chamber. Views are the clickable items in the module sub-panel under each chamber section.

**Naming convention:** Just the name, no suffix. The chamber section header provides context.

**Examples (Contracts module):**

| Chamber | Views |
|---------|-------|
| Discover | Triage Board, Incoming Queue |
| Build | Extraction Results, Preflight Reports, Contract Generator |
| Review | Review Queue, Record Inspector, Patch Approvals |
| Ship | Export Center, Sync Status, Published Records |

**Views are module-specific.** Each module defines its own views per chamber. But the chamber structure is universal.

### Triptych

The **triptych** is the three-panel workspace that renders when viewing a vault. The three panels are:

| Panel | Position | Purpose | Width |
|-------|----------|---------|-------|
| **Signal** | Left | Event feed, flags, triage items, AI suggestions | ~280px |
| **Orchestrate** | Center | Main work area (record inspector, document viewer, editor) | flex-1 |
| **Control** | Right | Health, SLA, approval chain, audit trail, AI agent | ~300px |

The triptych has **four states** that determine panel sizing and behavior:

| State | Internal Name | What's happening | Trigger |
|-------|--------------|------------------|---------|
| **Overview** | Default three-panel view | All three panels visible | Opening a vault |
| **Inspect** | Orchestrate expanded, sides narrow | Deep-diving into a document or record | Clicking a field/document to examine |
| **Edit** | Orchestrate splits into editor + preview | Creating or modifying a patch | Clicking "New Patch" or edit action |
| **Approve** | Governance bar prominent, SLA countdown | Reviewing and approving/rejecting | Gate requires reviewer action |

**View state switching is automatic** based on user actions. Power users can override with keyboard shortcuts (Cmd+1/2/3/4).

---

## Navigation Hierarchy

### Level 1: Module Bar (Far Left, 56-72px)

Thin icon strip. Always visible. Each icon = one module.

```
+--------+
|  [A]   |  <-- Airlock icon --> Home (always first)
|--------|
|  [C]   |  <-- Contracts
|  [R]   |  <-- CRM
|  [T]   |  <-- Tasks
|  [K]   |  <-- Calendar
|  [D]   |  <-- Documents
|  ....  |
|--------|  <-- Divider
|  [*]   |  <-- Pinned vault or view (user-customized)
|  [#]   |  <-- Favorites folder
|--------|
|  [av]  |  <-- User avatar --> Personal Settings overlay
+--------+
```

**Behavior:**
- Clicking a module icon opens its sub-panel and loads the **last-visited view** in the main content area
- Active module has a left pip indicator (like Discord)
- Hover shows tooltip with module name + unread count
- Notification badges appear on modules with urgent items
- Admin/Settings is NOT in the module bar — it's accessed from the user avatar or a gear icon

### Level 2: Module Sub-Panel (Left, ~240px)

The sub-panel is organized by **chambers**. It changes entirely when you switch modules.

```
+------------------------+
| CONTRACTS              |
| > Mini Dashboard       |  <-- Item counts per chamber
|   15 Discover          |
|   28 Build             |
|   14 Review            |
|    6 Ship              |
|------------------------|
| > DISCOVER             |  <-- Red label
|   o Triage Board       |
|   o Incoming Queue     |
|                        |
| > BUILD                |  <-- Yellow label
|   o Extraction Results |
|   o Preflight Reports  |
|   o Contract Generator |
|                        |
| > REVIEW               |  <-- Purple label
|   o Review Queue       |
|   o Record Inspector   |
|   o Patch Approvals    |
|                        |
| > SHIP                 |  <-- Green label
|   o Export Center      |
|   o Sync Status        |
|   o Published Records  |
|------------------------|
| ACTIVE VAULTS          |
|   # henderson-msa    o |  <-- Gate-colored dot
|   # sony-dist-2024   o |
|   # warner-amendment  o |
+------------------------+
```

### Level 3: Main Content Area

What renders here depends on what's selected in the sub-panel:

| Selected | Main content renders |
|----------|---------------------|
| Chamber view (e.g., "Triage Board") | Module-level dashboard/view (full-width, no triptych) |
| Active vault (e.g., "henderson-msa") | Triptych: Signal / Orchestrate / Control |

### Home (Airlock Icon)

Clicking the Airlock icon is special. It does NOT follow the chamber pattern.

**Home sub-panel:**
```
+------------------------+
| HOME                   |
|------------------------|
| > MODULES              |  <-- Synced: clicking opens that module
|   o Contracts     (60) |
|   o CRM          (24) |
|   o Tasks         (18) |
|   o Calendar       (5) |
|   o Documents     (42) |
|------------------------|
| > FAVORITES            |
|   o [user's pinned items] |
|------------------------|
| > RESOURCES            |
|   o Company Wiki       |
|   o Templates          |
|   o Help Center        |
+------------------------+
```

**Home main content area:**
Notion-style page with admin-configured template blocks + user-customizable blocks.

**Admin-configured template (base):**
- Inbox feed (priority-ordered: urgent > action > info)
- Module triage widgets (one per enabled module — compact inline table showing top 3-5 open tasks)
- Module status cards (chamber distribution counts per module)

**Module triage widgets** (the key Home feature):
Each widget is a small table pulling from the [Universal Task System](../TaskSystem/overview.md):
- Shows top tasks for that module, sorted by priority
- Columns: Vault name, Issue, Severity badge, Status, Due countdown
- Click a row → navigates to that vault in the source module
- [View All] link → navigates to the module's triage board
- Auto-refreshes via WebSocket
- Collapsible, reorderable, filterable per user

**User-customizable blocks (added below template):**
- Calendar widget (upcoming deadlines)
- Pinned vaults
- Quick actions ("/upload-contract", "/new-task")
- Rich text notes (TipTap editor with slash commands)
- Custom embeds

### Admin / Settings

**Admin is a system overlay**, not a module. Accessed from the user avatar icon at the bottom of the module bar, or a dedicated gear icon.

**Two separate overlays:**

| Overlay | Access | Contains |
|---------|--------|----------|
| **Workspace Admin** | Gear icon (admin role required) | Members, Roles & Permissions, Module Config, Feature Control Plane, Audit Log |
| **Personal Settings** | User avatar click | Profile, Notifications, Theme, Keybindings, Connected Accounts |

**Why not a module?** Admin doesn't follow the chamber pattern. There's no Discover > Build > Review > Ship lifecycle for settings. It's configuration, not workflow.

---

## Route Tree

```
/                                    --> Redirect to /home
/home                                --> Home (inbox + module cards)

/contracts                           --> Contracts module, last-visited view
/contracts/triage-board              --> Contracts > Discover > Triage Board
/contracts/extraction-results        --> Contracts > Build > Extraction Results
/contracts/review-queue              --> Contracts > Review > Review Queue
/contracts/henderson-msa             --> Vault: Henderson MSA (triptych, default view)
/contracts/henderson-msa/record-inspector  --> Vault: specific view active
/contracts/henderson-msa/document    --> Vault: document viewer active

/crm                                 --> CRM module, last-visited view
/crm/pipeline                        --> CRM > Discover > Pipeline View
/crm/acme-corp                       --> Vault: Acme Corp customer record

/tasks                               --> Tasks module, last-visited view
/tasks/board                         --> Tasks > Build > Kanban Board
/tasks/migrate-auth                  --> Vault: specific task

/calendar                            --> Calendar module, last-visited view
/calendar/month                      --> Calendar > month view

/documents                           --> Documents module, last-visited view
/documents/q1-reports                --> Vault: document folder/collection

/settings                            --> Personal Settings overlay
/admin                               --> Workspace Admin overlay
/admin/members                       --> Admin > Members page
/admin/roles                         --> Admin > Roles & Permissions
/admin/features                      --> Admin > Feature Control Plane
/admin/audit                         --> Admin > Audit Log
```

**Rules:**
- No chamber names in URLs (chamber is derived from the view or vault state)
- Vault slugs are human-readable, derived from vault name
- View names in URLs are kebab-case
- Module landing (`/contracts`) loads the last-visited view (persisted in client state)
- All routes render inside the shell (module bar + sub-panel always visible)

---

## Naming Cheat Sheet

| Concept | Term | NOT this |
|---------|------|----------|
| Top-level domain | **Module** | Server, workspace, app |
| Workflow instance | **Vault** | Channel, workstream, room, item, record |
| Lifecycle stage | **Chamber** | Phase, step, stage, category |
| Chamber checkpoint | **Gate** | Checkpoint, milestone, barrier |
| Screen within chamber | **View** | Page, panel, station, screen |
| Three-panel workspace | **Triptych** | Layout, split, panels |
| Triptych left panel | **Signal** | Feed, events, sidebar |
| Triptych center panel | **Orchestrate** | Main, workspace, content |
| Triptych right panel | **Control** | Context, sidebar, info |
| Triptych layout mode | **State** (Overview/Inspect/Edit/Approve) | Mode, view state |
| Far-left icon strip | **Module Bar** | Server bar, nav bar |
| Module's left panel | **Sub-panel** | Sidebar, channel list |
| Vault list in sub-panel | **Active Vaults** | Recent channels, items |
| Settings UI | **Overlay** | Module, page, modal |
| Rich text home editor | **Workspace** (Home workspace) | Dashboard, homepage |

---

## Cross-References

- **[Universal Chambers](./universal-chambers.md)** — How chambers structure every module (update pending to use new names)
- **[Module Bar](./module-bar.md)** — Icon strip spec
- **[Triptych](./triptych.md)** — Three-panel workspace
- **[View States](./view-states.md)** — Overview / Inspect / Edit / Approve
- **[Roles](../Roles/overview.md)** — Role determines which chamber views are prominent
