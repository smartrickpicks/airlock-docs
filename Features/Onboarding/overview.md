# Onboarding & First-Run Experience — Design Spec

> **Status:** SPECCED — Defines what happens when a user or workspace is new to Airlock.

> **Key Insight:** Airlock is a complex multi-module platform. Without onboarding, users face an empty shell with 5 modules, 4 chambers each, and no data. The first-run experience must make the platform immediately useful and progressively reveal complexity.

---

## Two Onboarding Paths

| Path | Triggered by | Actor | Goal |
|------|-------------|-------|------|
| **Workspace Setup** | First admin creates workspace | Architect/Owner | Configure workspace: enable modules, invite team, connect data sources |
| **User Onboarding** | User accepts invite and logs in | Any role | Understand the platform, find their work, start contributing |

---

## Path 1: Workspace Setup (Architect/Owner)

### Step 1: Create Workspace

After OAuth login (Google), the first user creates a workspace:

```
+------------------------------------------+
|          Welcome to Airlock              |
|                                          |
|  Create your workspace                   |
|                                          |
|  Workspace name: [________________]     |
|  Industry:       [Music/Legal/Other ▾]  |
|                                          |
|              [Create Workspace]          |
+------------------------------------------+
```

This creates:
- `workspaces` row
- Default `workspace_modules` entries (all 5 core modules enabled)
- Assigns creator as Architect role
- Default feature flags (all enabled)
- Default calibration values (factory defaults)

### Step 2: Module Configuration

Guided wizard with progressive disclosure:

```
+------------------------------------------+
| WORKSPACE SETUP                    2 / 5 |
|------------------------------------------|
|                                          |
| Which modules do you need?               |
|                                          |
| [x] Contracts  — Contract lifecycle      |
| [x] CRM        — Customer relationships  |
| [x] Tasks      — Work tracking           |
| [ ] Calendar   — Scheduling              |
| [ ] Documents  — Document library        |
|                                          |
| You can always enable more later.        |
|                                          |
|         [Back]          [Continue]        |
+------------------------------------------+
```

### Step 3: Invite Team

```
+------------------------------------------+
| WORKSPACE SETUP                    3 / 5 |
|------------------------------------------|
|                                          |
| Invite your team                         |
|                                          |
| [email@example.com     ] [Builder    ▾]  |
| [email2@example.com    ] [Gatekeeper ▾]  |
| [+ Add another]                          |
|                                          |
| Role guide:                              |
| Builder — Creates and edits data         |
| Gatekeeper — Reviews and approves        |
| Owner — Manages team and settings        |
|                                          |
|         [Back]     [Skip]  [Invite]      |
+------------------------------------------+
```

### Step 4: Connect Data Source (Optional)

```
+------------------------------------------+
| WORKSPACE SETUP                    4 / 5 |
|------------------------------------------|
|                                          |
| Connect a data source                    |
|                                          |
| [Google Drive]  Connect folder of PDFs   |
| [Upload]        Upload files manually    |
| [API]           Configure API ingest     |
|                                          |
| Or skip and add data later.              |
|                                          |
|         [Back]     [Skip]  [Connect]     |
+------------------------------------------+
```

### Step 5: Ready

```
+------------------------------------------+
| WORKSPACE SETUP                    5 / 5 |
|------------------------------------------|
|                                          |
|  Your workspace is ready!                |
|                                          |
|  3 modules enabled                       |
|  2 team members invited                  |
|  Google Drive connected                  |
|                                          |
|  What's next:                            |
|  1. Upload your first contract batch     |
|  2. Watch extraction + preflight run     |
|  3. Review results in Triage Board       |
|                                          |
|              [Go to Workspace]           |
+------------------------------------------+
```

---

## Path 2: User Onboarding (Any Role)

### First Login

When a user accepts an invite and logs in for the first time:

1. **Welcome modal** — brief (3 slides max), role-specific:

| Role | Slides |
|------|--------|
| **Builder** | "Your job: find issues, fix data, submit patches" → "Start in Triage Board" → "Explore your assigned vaults" |
| **Gatekeeper** | "Your job: review work, approve or reject" → "Start in Review Queue" → "Check Patch Approvals" |
| **Owner** | "Your job: manage team and quality" → "Start in Admin" → "Monitor System Health" |

2. **Contextual tooltips** — first-time hints on key UI elements:

| Element | Tooltip | Dismissable |
|---------|---------|-------------|
| Module Bar | "Switch between modules here. Each is a different workspace." | Yes, one-time |
| Sub-panel chambers | "These are lifecycle stages. Work flows left to right." | Yes, one-time |
| Active Vaults | "Your assigned vaults appear here. Click to open." | Yes, one-time |
| Triptych panels | "Signal (events), Orchestrate (work area), Control (health & approvals)" | Yes, one-time |
| Cmd+K hint | "Press Cmd+K to search anything" | Yes, one-time |

3. **Empty state guidance** — when a view has no data, show actionable empty states:

| View | Empty State Message | Action |
|------|-------------------|--------|
| Home | "Welcome! Your triage widgets will populate as work arrives." | "Upload your first contract" button |
| Triage Board | "No triage items yet. Upload contracts to start." | "Upload" button |
| Review Queue | "Nothing to review. Your Builders haven't submitted work yet." | "View team status" link |
| CRM | "No accounts yet. They'll appear when contracts are processed." | "How the CRM works" link |
| Tasks | "No tasks assigned. Tasks are created automatically as work progresses." | "Create a task" button |

---

## Demo Data Option

For evaluation/trial purposes, offer a pre-loaded demo workspace:

```
+------------------------------------------+
|  Try Airlock with sample data?           |
|                                          |
|  Load a demo workspace with:             |
|  - 3 accounts (86 contracts)             |
|  - Pre-extracted fields                  |
|  - Sample patches and approvals          |
|  - Populated CRM and task board          |
|                                          |
|  [Load Demo Data]  [Start Empty]         |
+------------------------------------------+
```

Demo data includes:
- 3 parent vaults (Ostereo, Get Dough, Broke Records)
- 86 item vaults at various lifecycle stages
- Pre-populated triage items, patches, events
- 3 demo users (Builder, Gatekeeper, Owner)

This mirrors the data from `org-overview-demo.html`.

---

## Progressive Complexity

Airlock reveals features gradually based on usage:

| Phase | What's visible | What's hidden | Trigger to reveal |
|-------|---------------|---------------|-------------------|
| **Day 1** | Home, one module, triage, basic vault view | Multi-module nav, command palette, keyboard shortcuts | Default |
| **Week 1** | All enabled modules, full triage, vault triptych | Advanced filters, bulk ops, calibration | User visits 10+ vaults |
| **Week 2+** | Everything | Nothing hidden | Full unlock after 2 weeks |

This is NOT feature gating — all features are accessible via Cmd+K or direct URL. The sub-panel just starts simplified and expands.

**Implementation:** Client-side flag `onboarding_phase` in user preferences. No server enforcement.

---

## Onboarding Checklist (Home Widget)

New users see a checklist widget on their Home page that tracks their onboarding progress:

```
+------------------------------------------+
| GETTING STARTED                   3 / 7  |
|------------------------------------------|
| [x] Log in to Airlock                    |
| [x] View your Home page                 |
| [x] Open the Contracts module            |
| [ ] View a vault in the Record Inspector |
| [ ] Submit your first patch              |
| [ ] Use Cmd+K to search                 |
| [ ] Customize your Home page             |
+------------------------------------------+
```

- Checklist items are tracked client-side (localStorage)
- Auto-completes as user performs actions
- Dismissable — user can hide the widget permanently
- After all items complete: celebration state, then auto-dismiss after 3 days

---

## Workspace Admin Onboarding

For Owners/Architects, show an admin-specific checklist in the Workspace Admin overlay:

```
+------------------------------------------+
| WORKSPACE SETUP PROGRESS          4 / 8  |
|------------------------------------------|
| [x] Create workspace                     |
| [x] Enable modules                       |
| [x] Invite team members                  |
| [x] Connect data source                  |
| [ ] Upload first batch                   |
| [ ] Configure feature flags              |
| [ ] Set calibration thresholds           |
| [ ] Review first batch results           |
+------------------------------------------+
```

---

## Re-Onboarding

If a user hasn't logged in for 30+ days, show a "What's New" modal on next login:

| Content | Source |
|---------|--------|
| New features added | Manually curated changelog |
| Changes to their vaults | System-generated from events |
| Pending tasks | From Universal Task System |

---

## Related Specs

- **[Shell / Naming](../Shell/naming-and-hierarchy.md)** — Module Bar, Sub-panel, Triptych — the UI elements being introduced.
- **[Roles & Permissions](../Roles/overview.md)** — Role-specific onboarding content.
- **[Search / Cmd+K](../Search/overview.md)** — Introduced during onboarding.
- **[Universal Task System](../TaskSystem/overview.md)** — Onboarding checklist is a task-like widget.
