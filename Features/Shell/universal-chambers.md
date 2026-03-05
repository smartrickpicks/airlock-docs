# Universal Chamber Architecture

> **Status:** FOUNDATIONAL — This spec defines the core navigation and workflow pattern for every module in Airlock.

> **Key Insight:** The Four Chambers (Discover, Build, Review, Ship) aren't just a data lifecycle — they're the **UI navigation structure** inside every module. Every domain follows the same template. What changes per module is the tools and views inside each chamber.

> **Terminology:** See [Naming & Hierarchy](./naming-and-hierarchy.md) for the canonical vocabulary. Key terms: **Vault** (workflow instance), **Chamber** (lifecycle stage), **Gate** (checkpoint), **View** (screen within a chamber).

## The Pattern

Every piece of work in any domain follows the same lifecycle:

```
DISCOVER it  →  BUILD it  →  REVIEW it  →  SHIP it
```

This is true for contracts, creative assets, marketing campaigns, code releases, customer onboarding, hiring pipelines — any workflow. Airlock makes this universal.

## How It Maps to the UI

### Level 1: Module Bar (Far Left, ~56-72px)

Thin icon strip. Like Discord's server icons. Each icon = a domain/module.

```
+--------+
|  [A]   |  <-- Airlock icon --> Home (inbox + workspace)
|--------|
|  [C]   |  <-- Contracts
|  [R]   |  <-- CRM
|  [T]   |  <-- Tasks
|  [D]   |  <-- Documents
|  [K]   |  <-- Calendar
|        |
|  ....  |
|--------|  <-- Divider
|  [*]   |  <-- Favorites / Pinned
|--------|
|  [av]  |  <-- User avatar --> Personal Settings overlay
+--------+
```

Clicking a module icon opens its **sub-panel** and loads the **last-visited view** in the main content area. Admin is accessed from the user avatar, not the module bar.

### Level 2: Module Sub-Panel (~240px)

The sub-panel is organized by **chambers**. Each chamber section contains the views relevant to that stage of the lifecycle.

```
+------------------------+
| CONTRACTS              |
| > Mini Dashboard       |  <-- Item counts per chamber
|   15 Discover          |
|   28 Build             |
|   14 Review            |
|    6 Ship              |
|------------------------|
| > DISCOVER             |
|   o Triage Board       |  <-- What's arrived, what needs attention
|   o Incoming Queue     |  <-- New uploads, unprocessed items
|                        |
| > BUILD                |
|   o Extraction Results |  <-- Processing status, field coverage
|   o Preflight Reports  |  <-- Gate results, blockers
|   o Contract Generator |  <-- Draft new contracts
|                        |
| > REVIEW               |
|   o Review Queue       |  <-- Vaults awaiting gatekeeper review
|   o Record Inspector   |  <-- Deep-dive into a single record
|   o Patch Approvals    |  <-- Pending approval chain items
|                        |
| > SHIP                 |
|   o Export Center      |  <-- Approved vaults ready for export
|   o Sync Status        |  <-- External system sync (Salesforce, etc.)
|   o Published Records  |  <-- Completed, baseline records
|------------------------|
| ACTIVE VAULTS          |  <-- Your in-progress workflow instances
|   # henderson-msa    o |  <-- Gate-colored dot
|   # sony-dist-2024   o |
|   # warner-amendment  o |
+------------------------+
```

### Level 3: Triptych (Main Content Area)

When you click a chamber view (e.g., "Triage Board"), the main content loads that view (full-width dashboard). When you click a vault (e.g., "henderson-msa"), the triptych loads that vault's context-aware Signal | Orchestrate | Control workspace.

## Vaults

A **vault** is Airlock's fundamental unit of work. It is the interactive, living record of how something moves from discovery to release — the source material + annotation layer + metadata + audit trail + collaboration context, all in one secure container.

**When a contract is ingested:**
1. A vault is created (e.g., `henderson-msa`)
2. Stakeholders are auto-assigned based on rules (who uploaded, their supervisor, the assigned gatekeeper, relevant owners)
3. The vault appears in each stakeholder's module sub-panel under "Active Vaults"
4. **Each person sees a different screen** based on:
   - Their **role** (Builder sees builder tools, Gatekeeper sees review tools)
   - The vault's **current gate** (if it's in Review, the Gatekeeper sees the approval interface)
   - Their **agentic role** (personality-optimized layout and tool prominence)

**Vault lifecycle:**
```
Vault created (Discover)
  -> Extraction runs, preflight gates evaluated
  -> Vault moves to Build (or stays in Discover if blocked)
  -> Builder completes work, submits for review
  -> Vault moves to Review
  -> Gatekeeper reviews, approves/rejects
  -> Vault moves to Ship (or back to Build)
  -> Owner promotes, exports, syncs
  -> Vault archived (completed)
```

## Role-Based Views of the Same Vault

The same vault (`henderson-msa`) shows different content depending on who's looking:

### Builder viewing `henderson-msa` at Build gate:
```
Signal               | Orchestrate              | Control
---------------------+--------------------------+-------------------
- Extraction events  | Record Inspector          | Health Score: 72%
- Preflight results  | (field cards, evidence)   | Gate: Build
- AI suggestions     |                           | SLA: 4h 12m
- Triage items       | [Start Patch] button      | Audit trail
```

### Gatekeeper viewing `henderson-msa` at Review gate:
```
Signal               | Orchestrate              | Control
---------------------+--------------------------+-------------------
- Patch submitted    | Diff View                 | Approval Chain
- Evidence pack      | (original vs proposed)    | check Builder submitted
- Builder's reasoning|                           | >> YOUR REVIEW
- AI review flags    | [Approve] [Reject] [Hold] | - Owner pending
```

### Owner viewing `henderson-msa` at Ship gate:
```
Signal               | Orchestrate              | Control
---------------------+--------------------------+-------------------
- Full event history | Final Record View         | Export Options
- All approvals      | (canonical, clean)        | [CSV] [JSON] [PDF]
- Health trajectory  |                           | Sync: Salesforce ok
- Release checklist  | [Promote to Baseline]     | Version history
```

## Universal Template — Applied to Any Module

The 4-chamber pattern works for any domain. Here's how different modules fill the template:

### Contracts Module
| Chamber | Views | Primary Role |
|---------|-------|-------------|
| Discover | Triage Board, Incoming Queue | Builder |
| Build | Extraction Results, Preflight, Contract Generator | Builder |
| Review | Review Queue, Record Inspector, Patch Approvals | Gatekeeper |
| Ship | Export Center, Sync Status, Published Records | Owner |

### CRM Module
| Chamber | Views | Primary Role |
|---------|-------|-------------|
| Discover | New Leads, Unqualified Contacts | Builder (SDR) |
| Build | Lead Enrichment, Account Matching, Pipeline Stage | Builder (AE) |
| Review | Deal Review, Pricing Approval, Legal Review | Gatekeeper (Manager) |
| Ship | Won Deals, Onboarding Handoff, Revenue Reporting | Owner (VP Sales) |

### Creative/Marketing Module (hypothetical)
| Chamber | Views | Primary Role |
|---------|-------|-------------|
| Discover | New Briefs, Asset Requests, Campaign Ideas | Builder (Designer) |
| Build | Design Iteration, Brand Compliance, Asset Library | Builder (Designer) |
| Review | Creative Director Review, Stakeholder Feedback, Legal Approval | Gatekeeper (Director) |
| Ship | Publish, Asset Distribution, Performance Tracking | Owner (CMO) |

### Tasks Module
| Chamber | Views | Primary Role |
|---------|-------|-------------|
| Discover | New Tasks, Backlog, Unassigned | Builder |
| Build | In Progress, Subtask Breakdown, Dependencies | Builder |
| Review | Code Review, QA, Acceptance Testing | Gatekeeper |
| Ship | Done, Deployed, Sprint Report | Owner |

## Chamber-Specific Standard Views

Each chamber has **default view types** that render differently based on context but follow a consistent pattern:

### Discover Chamber Views
- **Triage Board** — Card-based overview of incoming items. Grouped by urgency, entity, or batch. The Builder's morning landing page.
- **Incoming Queue** — Raw feed of newly arrived items. Unprocessed, unassigned.
- **Dashboard Metrics** — Counts, trends, SLA status for Discover-stage items.

### Build Chamber Views
- **Processing Dashboard** — Status of items being processed (extraction, enrichment, normalization). Progress bars, field coverage, blockers.
- **Tool Views** — Module-specific processing tools (Record Inspector for field editing, Contract Generator for drafting, Lead Enrichment for CRM).
- **Preflight/Quality Gate** — Results of automated quality checks. Pass/fail/review per gate rule.

### Review Chamber Views
- **Review Queue** — Vaults awaiting human review. Sorted by SLA urgency. The Gatekeeper's landing page (this is what Org Overview becomes — a Gatekeeper's view of their builders' work).
- **Diff/Comparison View** — Side-by-side original vs proposed changes. Evidence packs.
- **Approval Interface** — Approve / Reject / Hold / Request Clarification. SLA countdown. Cannot approve own work.

### Ship Chamber Views
- **Release Queue** — Approved vaults ready for final action. The Owner's landing page.
- **Export/Publish** — Format selection, destination selection, batch export.
- **Sync Status** — External system integration status (Salesforce, Google Drive, etc.).
- **Archive** — Completed vaults with full audit trail.

## The Home Screen (Airlock Icon)

Clicking the Airlock icon shows your **Home** — a hybrid workspace with admin-configured template blocks and user-customizable sections.

**Home sub-panel** lists all modules (synced — clicking one auto-opens that module), favorites, and resources.

**Home main content area (Notion-style page with triage widgets):**

```
+----------------------------------------------------------+
| Good morning, Ana                                         |
| You have 7 items needing attention                        |
|----------------------------------------------------------|
| URGENT -- SLA in 45min                                    |
| henderson-msa needs your Review approval                  |
| Contracts > Review > henderson-msa                        |
|----------------------------------------------------------|
|                                                           |
| CONTRACTS TRIAGE                              [View All →]|
| +--------+--------------+---------+--------+--------+     |
| | Vault  | Issue         |Severity |Status  |Due     |     |
| +--------+--------------+---------+--------+--------+     |
| | hend.  | Missing $     | blocker | open   | 2h 15m |     |
| | sony   | Low conf      | warning | open   | —      |     |
| | warn.  | Entity ambig  | warning | review | 4h 30m |     |
| +--------+--------------+---------+--------+--------+     |
|                                                           |
| CRM TRIAGE                                    [View All →]|
| +--------+--------------+---------+--------+--------+     |
| | Vault  | Issue         |Severity |Status  |Due     |     |
| +--------+--------------+---------+--------+--------+     |
| | Acme   | New lead      | info    | open   | —      |     |
| | Sony   | Dup account   | warning | open   | —      |     |
| +--------+--------------+---------+--------+--------+     |
|                                                           |
| [Module Status Cards]                                     |
| Contracts: 15/28/14/6  |  CRM: 8/12/3/2  |  Tasks: 24   |
|                                                           |
| [User's custom blocks below]                              |
| (TipTap editor, slash commands, pinned vaults, etc.)      |
+----------------------------------------------------------+
```

Each triage widget is a compact table from the [Universal Task System](../TaskSystem/overview.md). Click a row to navigate to that vault; click [View All] to open the module's triage board. Widgets are collapsible, reorderable, and auto-refresh via WebSocket. Inbox items link directly to the relevant module + vault.

## Gates Within Chambers

Each chamber can have **multiple gates** — checkpoints where human action is required. Gates determine what view renders and what actions are available.

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

When a vault is at a specific gate, the triptych renders the appropriate view for that gate + the viewer's role.

## Org Overview Reconsidered

Org Overview is **not a separate module**. It's the **Review chamber's Review Queue** viewed by a Gatekeeper or Manager role. It shows:

- A list of builders' work vaults in the Review stage
- Aggregate health scores and SLA compliance
- Entity-grouped views (the expandable cards from the current demo)
- Alert panels for vaults at risk

A manager in CRM would see a similar "Org Overview" but with deal reviews, pipeline health, and sales rep activity instead of contract data.

## Relationship to Existing Demos

| Current Demo | Chamber | Gate | Role Perspective |
|-------------|---------|------|-----------------|
| **Triage Dashboard** | Discover | gate_triage | Builder |
| **Record Inspector** | Build / Review | gate_extract / gate_gatekeeper | Builder / Gatekeeper |
| **Contract Generator** | Build | gate_enrich (drafting) | Builder / Designer |
| **Org Overview** | Review | gate_gatekeeper | Gatekeeper / Manager |
| **Admin Dashboard** | N/A | N/A (system overlay) | Owner / Designer |

## Favorites & Pinning (Module Bar Customization)

Users can **pin any view or vault** to the module bar for quick access, similar to Discord's server folders:

**Pin a specific view:**
- Right-click any chamber view (e.g., "Contracts > Review > Review Queue") -> "Pin to sidebar"
- It appears as its own icon on the module bar — one click to jump directly there
- Useful for Gatekeepers who live in one specific review queue all day

**Pin a vault:**
- Right-click a vault (e.g., `henderson-msa`) -> "Pin to sidebar"
- Pinned vaults appear as icons on the module bar with a gate-colored dot
- Useful for tracking a high-priority item without navigating through the module

**Favorites folder:**
- Group multiple pins into a folder on the module bar (like Discord's server folders)
- Click folder -> expands to show pinned items
- Example: "My Active Contracts" folder containing 3 pinned vaults

**Module bar anatomy:**
```
+--------+
|  [A]   |  <-- Airlock Home (always first)
|--------|
|  [C]   |  <-- Contracts module
|  [R]   |  <-- CRM module
|  [T]   |  <-- Tasks module
|  ....  |
|--------|  <-- Divider
|  [*]   |  <-- Favorites folder (user-created)
|  [#]   |  <-- Pinned vault: henderson-msa
|  [o]   |  <-- Pinned view: Review Queue
|--------|
|  [av]  |  <-- User avatar --> Settings overlay
+--------+
```

**Rules:**
- Module icons are ordered by the admin (workspace-level config)
- Pinned items / favorites are per-user
- Pins persist across sessions
- Notification badges appear on pinned items (unread events, SLA warnings)
- Admin/Settings accessed from user avatar (overlay), not module bar

## Key Principles

1. **Same vault, different view.** The platform renders the right interface for the right person at the right time. No one sees controls they can't use.

2. **Chambers are universal.** Every module implements the same 4-chamber lifecycle (Discover > Build > Review > Ship). This creates consistency — a user who learns Contracts can immediately navigate CRM or Tasks.

3. **Gates enforce governance.** You can't skip chambers. The system moves vaults forward only when gate conditions are met. Code enforces this, not UI suggestions.

4. **Roles map to chambers.** Builders live in Discover + Build. Gatekeepers live in Review. Owners live in Ship. Everyone can see the full lifecycle, but their primary workspace is their chamber.

5. **The sub-panel is the navigator.** Clicking between items in the sub-panel doesn't switch modules — it switches views within the same module. Module switching only happens via the icon bar.
