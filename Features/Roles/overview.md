# Role Architecture — Multi-Tier Permission Model

> Airlock uses a Discord-like multi-tier role system where permissions are scoped per-module, not globally.

## Philosophy

From the OrcestrateOS governance framework:
- **"No Self-Grading"** — The person writing a check shouldn't be the one signing it
- **"Speed belongs to discovery, authority belongs to promotion"** — AI/Analysts move fast, Gatekeepers/Owners hold the line
- **Separation of Duties (SoD)** — Code enforces role boundaries, not just UI suggestions
- **Decision Lineage** — Every action has an auditable chain: who proposed, who verified, who promoted

## Three Layers of Identity

### Layer 1: Organization Role (Org-Level)

Your baseline identity across the entire platform. Determines which modules you can see and your default access level.

| Org Role | Access | Description |
|----------|--------|-------------|
| **Member** | View-only across org + assigned modules | Standard employee, contributor |
| **Lead** | Manage within assigned modules | Department lead, team manager |
| **Director** | Full access to assigned modules + cross-module visibility | Department head |
| **Executive** | Read access everywhere + Admin module | C-suite, stakeholder view |

### Layer 2: Module Role (Per-Module)

Your role within a specific module. Determines what tools you see and what actions you can take. A person can have different module roles in different modules.

| Module Role | Permission Level | What They Do | NotebookLM Archetype |
|-------------|-----------------|--------------|---------------------|
| **Builder** | Draft, observe, assemble | Discovers issues, drafts fixes, creates records, assembles evidence | "The Doer" — legislative aide |
| **Gatekeeper** | Review, approve/reject, replay | Evaluates evidence, approves/rejects, enforces "show me proof" | "The Referee" — quality control |
| **Owner** | Promote, configure, publish | Makes final decisions, promotes truth to baseline, manages module config | "The Publisher" — ultimate custodian |
| **Designer** | Schema, calibrate, sandbox | Builds schemas, configures rules, operates in sandbox/draft mode | "The Architect" — system designer |
| **Viewer** | Read-only | Sees data and events but cannot take action | "The Stakeholder" — observer |

**Example:** Ana Chen is a **Member** at the org level. In the **Contracts** module she's an **Owner** (she runs contract adjudication). In the **CRM** module she's a **Viewer** (she can see customer data but not edit it). In the **Tasks** module she's a **Builder** (she creates and completes tasks).

### Layer 3: Agentic Role (Personality-Optimized)

The 16 functional roles from the Sovereign Workplace framework. These are assigned based on what a person is naturally good at and determine which tool configurations and dashboard layouts they see.

#### Numerator Contributors (Growth & Momentum)
| # | Role | Function | PI Type | AI Ceiling |
|---|------|----------|---------|------------|
| 1 | **Truth Keeper** | Define "correct", establish baselines | Values-driven | 40% |
| 2 | **System Architect** | Build rules, schemas, structures | Individualist/Strategist | 50% |
| 3 | **Momentum Builder** | Accelerate execution, build workflows | Venturer/Captain | 60% |
| 4 | **Evidence Curator** | Assemble proof for governance | Specialist/Analyzer | 70% |
| 5 | **Fast Path Executor** | Push low-risk items through safely | Persuader | 65% |
| 6 | **Maverick Innovator** | Challenge status quo, prototype | Maverick | 30% |

#### Denominator Managers (Entropy & Friction Control)
| # | Role | Function | PI Type | AI Ceiling |
|---|------|----------|---------|------------|
| 7 | **Verifier** | Eliminate "trust me" scenarios | Analyzer | 50% |
| 8 | **Friction Taxonomist** | Categorize chaos, understand exceptions | Empathy-driven | 55% |
| 9 | **Authority Validator** | Enforce permissions, audit access | Controller | 60% |
| 10 | **Cold Route Guardian** | High-risk go/no-go gatekeeper | Guardian | 45% |
| 11 | **Semantic Sheriff** | Prevent semantic drift | Controller | 55% |
| 12 | **Process Facilitator** | Optimize flow, coordinate stakeholders | Operator | 40% |
| 13 | **Compliance Analyst** | Validate proof against controls | Specialist | 50% |

#### Consciousness Amplifiers (Trust Multipliers)
| # | Role | Function | PI Type | AI Ceiling |
|---|------|----------|---------|------------|
| 14 | **Observer** | System-wide metrics and monitoring | Specialist/Analyst | 75% |
| 15 | **Drift Detective** | Detect behavioral shifts, silent degradation | Analytical | 70% |
| 16 | **Team Orchestrator** | Balance system, prevent burnout | Altruist/Collaborator | 25% |

### How the Layers Connect

```
Organization Role    → Which modules can I see?
Module Role          → What can I do in this module?
Agentic Role         → How is my interface optimized for how I think?
```

A **Cold Route Guardian** (agentic role) with **Gatekeeper** (module role) permissions in the Contracts module would see:
- A risk-focused dashboard (Guardian personality: careful, rule-following)
- Enhanced evidence replay tools (Gatekeeper permission: verify before approve)
- SLA timers prominently displayed (high-risk focus)
- No draft/creation tools (Gatekeeper doesn't create, only validates)

A **Momentum Builder** (agentic role) with **Builder** (module role) permissions in the same Contracts module would see:
- A speed-focused dashboard (Venturer personality: rapid deployment)
- Batch operations, templates, quick-fill tools
- Automation suggestions from AI
- No approval buttons (Builder can't approve their own work)

## Permission Model

### Computed Permissions (Discord-Style)

Permissions are **additive** with three override layers, computed top-down:

```
base_permissions   = default_permissions(module_role)     // Builder defaults
+ user_overrides   = admin_granted_adhoc(user, module)    // Per-user additions
+ channel_overrides = per_channel_grants_or_denies(user, channel)
─────────────────────────────────────────────────────────
= effective_permissions(user, module, channel)

ceiling            = org_role_ceiling(org_role)            // Can never exceed
```

**How it works:**

1. **Base role permissions** — Each module role (Builder, Gatekeeper, etc.) has a default permission set. These are the starting point.
2. **Ad-hoc user overrides** — Admins can grant or remove specific permissions on a per-user basis, on top of their base role. A Builder with an ad-hoc "approve_low_risk" permission can approve low-risk items even though Builders normally can't approve.
3. **Channel overrides** — Per-channel grants or denies for specific users or roles. Lock a sensitive contract channel to only certain Gatekeepers.
4. **Org role ceiling** — The org-level role sets an absolute ceiling. A Member with ad-hoc Owner permissions in one module still can't access Admin-only features. The ceiling prevents runaway escalation.

### Ad-Hoc Permission Overrides (Admin-Managed)

Inspired by Discord's Members Page — admins can fully customize any user's permissions beyond their base role.

**How admins use it:**
- Open Admin module → Members tab → click a user
- See their base role per module (Builder in Contracts, Viewer in CRM, etc.)
- Add or remove individual permissions on top of the base role
- Changes are immediate, audited, and reversible

**Ad-hoc override types:**

| Override Type | Example | Use Case |
|--------------|---------|----------|
| **Grant** | Builder + `approve_low_risk` | Senior analyst who can approve simple items |
| **Grant** | Viewer + `export_data` | Executive who needs export but not edit |
| **Deny** | Gatekeeper - `approve_high_risk` | Junior gatekeeper restricted to low/medium risk |
| **Grant** | Builder + `configure_rules` | Analyst who also maintains extraction rules |
| **Temporary** | Builder + `approve_*` (expires 2026-03-10) | Covering for a Gatekeeper on PTO |

**Rules:**
- Ad-hoc overrides are **explicit** — they're stored as individual permission records, not role promotions
- Overrides are **audited** — every grant/deny is logged with who granted it, when, and why
- Overrides **never bypass SoD** — self-approval prevention is hardcoded and cannot be overridden (see below)
- Overrides have optional **expiry dates** — for temporary escalation without permanent role changes
- Only org-level Owners or module Owners can create overrides (depending on scope)

### Channel-Level Overrides

Like Discord's channel permission overrides — fine-grained control at the channel level.

**Per-role overrides:** All Builders in the Contracts module can see channel #henderson-msa, but only Gatekeepers can see channel #confidential-merger.

**Per-user overrides:** User "Alex" is a Builder but has explicit access to #confidential-merger because they're the assigned analyst for that deal.

**Deny overrides:** User "Sam" is a Gatekeeper but is explicitly denied access to #competitor-contract because of a conflict of interest.

**Computation order (Discord model):**
1. Start with base module role permissions
2. Apply ad-hoc user overrides (grants and denies)
3. Apply channel-level role overrides
4. Apply channel-level user overrides (most specific wins)
5. Enforce org role ceiling (absolute maximum)
6. Enforce SoD rules (absolute, non-overridable)

### Linked Roles (Automated Assignment)

Inspired by Discord's Linked Roles — roles assigned automatically based on external metadata or system conditions.

**Examples:**
- User connects their Salesforce account → auto-granted Builder role in CRM module
- User completes 50 contract reviews → auto-granted "Senior Reviewer" badge (agentic role suggestion)
- User's PI assessment results available → auto-suggested agentic role mapping
- External HR system marks user as "Department Lead" → auto-granted Lead org role

**Requirements for linked roles:**
- Connection metadata verified by the platform (not self-reported)
- Admin must approve the linked role mapping (which connection → which role)
- Users see which connections are granting them roles (transparency)
- Disconnecting a linked account revokes the auto-granted role

### Self-Approval Prevention

Hardcoded at every level:
- A Builder cannot approve their own patch
- A Gatekeeper cannot approve work they drafted
- An Owner cannot promote changes they authored without Gatekeeper sign-off
- Code enforces this — not UI suggestions

### Permission Escalation

As risk increases, approval requirements scale:
- **Low risk:** 1 Gatekeeper approval
- **Medium risk:** 1 Gatekeeper + 1 Owner approval
- **High risk:** 2 Gatekeepers + 1 Owner approval (multi-approver threshold)
- **Critical:** All of above + time-locked cooling period

## Discord Parallels

| Discord Concept | Airlock Equivalent |
|----------------|-------------------|
| Server | Module (Contracts, CRM, Tasks, etc.) |
| Server Directory | Organization (the main workspace) |
| Server Roles | Module Roles (Builder, Gatekeeper, Owner, Designer, Viewer) |
| Channel Permissions | Channel-level overrides within a module |
| @everyone | Member (org-level default) |
| Server Owner | Module Owner |
| Members Page | Admin → Members tab (per-user role + override management) |
| Linked Roles | Auto-assigned roles from external connections (Salesforce, HR, PI assessment) |
| Role Permission Editor | Admin → Roles tab (edit base permissions per module role) |
| Channel Override (per-role) | Channel settings → role-level access grants/denies |
| Channel Override (per-user) | Channel settings → user-level access grants/denies |
| User Settings | Profile, theme, notifications, keybindings |
| Server Settings | Module config (available to module Owners) |

## UI Implications

1. **Module Bar** (far left) shows modules you have access to — not all modules exist for all users
2. **Channel Sidebar** shows channels within the active module, filtered by your module role
3. **Tool availability** changes per module based on your module role + agentic role
4. **Dashboard layout** adapts to your PI personality type (optional, can be overridden)
5. **User Settings** shows your org role + per-module role assignments
6. **Admin module** (org-level Owners only) shows the full role matrix: who has what in which module

## Implementation Notes

### Database Schema

```sql
-- Module-level role assignments (extends user_workspace_roles)
CREATE TABLE user_module_roles (
    user_id UUID NOT NULL REFERENCES users(id),
    workspace_id UUID NOT NULL REFERENCES workspaces(id),
    module_id TEXT NOT NULL,
    module_role TEXT NOT NULL DEFAULT 'viewer',  -- builder | gatekeeper | owner | designer | viewer
    agentic_role TEXT,                            -- optional: truth_keeper, evidence_curator, etc.
    pi_profile TEXT,                              -- optional: analyzer, guardian, maverick, etc.
    assigned_by UUID REFERENCES users(id),
    assigned_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, workspace_id, module_id)
);

-- Ad-hoc permission overrides (per-user, on top of base role)
CREATE TABLE permission_overrides (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    workspace_id UUID NOT NULL REFERENCES workspaces(id),
    scope_type TEXT NOT NULL,               -- 'module' | 'channel'
    scope_id TEXT NOT NULL,                 -- module_id or channel_id
    permission TEXT NOT NULL,               -- 'approve_low_risk', 'export_data', etc.
    effect TEXT NOT NULL DEFAULT 'grant',   -- 'grant' | 'deny'
    expires_at TIMESTAMPTZ,                 -- NULL = permanent, or expiry date
    reason TEXT,                            -- why this override was granted
    granted_by UUID NOT NULL REFERENCES users(id),
    granted_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    revoked_at TIMESTAMPTZ,                -- NULL = active, set when revoked
    revoked_by UUID REFERENCES users(id)
);

CREATE INDEX idx_overrides_user ON permission_overrides(user_id, workspace_id)
    WHERE revoked_at IS NULL;
CREATE INDEX idx_overrides_scope ON permission_overrides(scope_type, scope_id)
    WHERE revoked_at IS NULL;

-- Linked role connections (external metadata → auto-role assignment)
CREATE TABLE linked_role_connections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    workspace_id UUID NOT NULL REFERENCES workspaces(id),
    provider TEXT NOT NULL,                 -- 'salesforce', 'hr_system', 'pi_assessment'
    provider_user_id TEXT,                  -- external ID
    metadata JSONB NOT NULL DEFAULT '{}',   -- provider-specific data
    connected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Linked role mappings (admin-configured: connection → role grant)
CREATE TABLE linked_role_mappings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id),
    provider TEXT NOT NULL,
    condition JSONB NOT NULL,               -- {"field": "department", "equals": "Legal"}
    target_module TEXT NOT NULL,
    target_role TEXT NOT NULL,              -- module role to auto-assign
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### API Permission Check

```python
def compute_effective_permissions(user_id, workspace_id, module_id, channel_id=None):
    """Compute effective permissions with ad-hoc overrides (Discord-style)."""

    # 1. Start with base module role permissions
    module_role = get_module_role(user_id, workspace_id, module_id)
    permissions = set(DEFAULT_PERMISSIONS[module_role])

    # 2. Apply ad-hoc module-level overrides
    overrides = get_active_overrides(user_id, workspace_id, 'module', module_id)
    for ov in overrides:
        if ov.effect == 'grant':
            permissions.add(ov.permission)
        elif ov.effect == 'deny':
            permissions.discard(ov.permission)

    # 3. Apply channel-level overrides (if checking a specific channel)
    if channel_id:
        channel_overrides = get_active_overrides(user_id, workspace_id, 'channel', channel_id)
        for ov in channel_overrides:
            if ov.effect == 'grant':
                permissions.add(ov.permission)
            elif ov.effect == 'deny':
                permissions.discard(ov.permission)

    # 4. Enforce org role ceiling
    org_role = get_org_role(user_id, workspace_id)
    permissions &= ORG_ROLE_CEILING[org_role]

    # 5. Enforce SoD (non-overridable)
    # Self-approval rules checked at action time, not permission level

    return permissions


def check_permission(user_id, workspace_id, module_id, permission, channel_id=None):
    """Check if user has a specific permission."""
    effective = compute_effective_permissions(user_id, workspace_id, module_id, channel_id)
    return permission in effective
```

### Admin UI: Members Page

The Members tab in the Admin module shows the full role matrix with inline override management:

```
┌─────────────────────────────────────────────────────────────────────┐
│ Members                                          [Invite] [Search] │
├──────────┬──────────┬───────────┬────────┬───────┬─────────────────┤
│ User     │ Org Role │ Contracts │ CRM    │ Tasks │ Overrides       │
├──────────┼──────────┼───────────┼────────┼───────┼─────────────────┤
│ Ana Chen │ Member   │ Owner     │ Viewer │ Builder│ 2 active        │
│ Dev Patel│ Lead     │ Gatekeeper│ Builder│ Owner │ 1 temporary     │
│ Sam Liu  │ Member   │ Builder   │ —      │ Builder│ 0               │
│ Jo Kim   │ Executive│ Viewer    │ Viewer │ Viewer│ 0               │
└──────────┴──────────┴───────────┴────────┴───────┴─────────────────┘

Click user → User Permission Panel:
┌──────────────────────────────────────────┐
│ Ana Chen                                 │
│ Org: Member | Agentic: Evidence Curator  │
├──────────────────────────────────────────┤
│ Contracts — Owner (base)                 │
│   [+] approve_all         ✓ (from role)  │
│   [+] promote_to_baseline ✓ (from role)  │
│   [+] configure_module    ✓ (from role)  │
│   [+] view_audit_log      ✓ (ad-hoc)    │ ← admin-granted
│                                          │
│ CRM — Viewer (base)                      │
│   [+] export_data         ✓ (ad-hoc)    │ ← admin-granted
│   [ ] edit_records        ✗              │
│                                          │
│ [Add Override] [View History]            │
└──────────────────────────────────────────┘
```

## Open Questions

- [ ] Should agentic roles be self-selected or admin-assigned?
- [ ] Should PI personality assessments be required or optional?
- [ ] How do cross-module actions work? (Contract release triggers CRM update — which role does the system act as?)
- [x] ~~How do we handle role transitions (promotion, temporary escalation)?~~ → Ad-hoc overrides with expiry dates
- [x] ~~Should there be a "delegation" mechanism?~~ → Ad-hoc temporary grants with expiry dates

---

## Custom Role Creation — Brainstorm

> **Added:** 2026-03-04 — Discord-style custom role creation with granular permission assignment.

### The Problem

The current spec has 5 fixed module roles (Builder, Gatekeeper, Owner, Designer, Viewer). In practice, organizations need more nuance:
- A "Senior Analyst" who can do everything a Builder does plus approve low-risk items
- A "Compliance Officer" who has read-only access everywhere but can flag issues
- A "External Auditor" who sees audit logs and reports but nothing else
- A "Trainee" who can view but only edit in a sandbox
- A "Department Head" who owns one module but views others

Fixed roles force workarounds via ad-hoc overrides. Custom roles make these first-class.

### Discord's Model (Our Reference)

Discord lets server owners:
1. Create unlimited custom roles with any name and color
2. Toggle individual permissions on/off per role
3. Drag roles to set hierarchy (higher = more authority)
4. Assign multiple roles to a single user (permissions are additive)
5. Set role-specific channel overrides

### Airlock Custom Roles

#### Creating a Custom Role

Admin > Roles tab > "Create Role" button:

```
+------------------------------------------------------------------+
| CREATE ROLE                                                        |
|------------------------------------------------------------------|
| Role Name: [Senior Analyst              ]                          |
| Role Color: [#00D1FF ▼]  ← for badges and UI indicators          |
| Description: [Builder with approval privileges for low-risk items] |
|                                                                    |
| Base Template: [Builder ▼]  ← start from an existing role         |
|   (or start from scratch)                                          |
|                                                                    |
| PERMISSIONS                                                        |
| ┌──────────────────────────────────────────────────────────────┐  |
| │ VAULT OPERATIONS                                              │  |
| │   ☑ view_vaults          View vault data and events           │  |
| │   ☑ create_vaults         Create new vaults                   │  |
| │   ☑ edit_vault_metadata   Edit vault metadata fields          │  |
| │   ☐ archive_vaults        Archive/soft-delete vaults          │  |
| │   ☐ delete_vaults         Permanently delete vaults           │  |
| │                                                                │  |
| │ EXTRACTION & ANALYSIS                                          │  |
| │   ☑ view_extraction       View extraction results              │  |
| │   ☑ run_extraction        Trigger extraction on documents      │  |
| │   ☐ configure_extraction  Edit extraction anchors/config       │  |
| │                                                                │  |
| │ PATCH WORKFLOW                                                  │  |
| │   ☑ create_patches        Draft data corrections               │  |
| │   ☑ submit_patches        Submit patches for review            │  |
| │   ☑ approve_low_risk     Approve low-risk patches              │  |
| │   ☐ approve_medium_risk   Approve medium-risk patches          │  |
| │   ☐ approve_high_risk     Approve high-risk patches            │  |
| │   ☐ apply_patches         Apply approved patches to baseline   │  |
| │                                                                │  |
| │ TRIAGE & TASKS                                                  │  |
| │   ☑ view_triage           View triage items                    │  |
| │   ☑ create_triage         Create triage items                  │  |
| │   ☑ resolve_triage        Resolve/dismiss triage items         │  |
| │   ☑ manage_tasks          Create, assign, complete tasks       │  |
| │                                                                │  |
| │ AI AGENT                                                        │  |
| │   ☑ chat_with_otto        Use Otto AI agent                    │  |
| │   ☐ configure_otto        Configure Otto roles/prompts        │  |
| │                                                                │  |
| │ EXPORT & INTEGRATION                                            │  |
| │   ☑ export_csv            Export vault data as CSV             │  |
| │   ☑ export_pdf            Export vault data as PDF             │  |
| │   ☐ export_json           Export vault data as JSON            │  |
| │   ☐ manage_connectors     Configure external integrations     │  |
| │                                                                │  |
| │ ADMINISTRATION                                                  │  |
| │   ☐ manage_members        Invite/remove workspace members     │  |
| │   ☐ manage_roles          Create/edit roles (this screen)     │  |
| │   ☐ toggle_features       Toggle feature flags                │  |
| │   ☐ calibrate_thresholds  Adjust calibration parameters       │  |
| │   ☐ view_audit_log        View audit trail                    │  |
| │   ☐ system_health         View system health dashboard        │  |
| └──────────────────────────────────────────────────────────────┘  |
|                                                                    |
| [Save Role] [Cancel]                                               |
+------------------------------------------------------------------+
```

#### Role Hierarchy (Drag to Reorder)

```
+----------------------------------+
| ROLE HIERARCHY (drag to reorder) |
|----------------------------------|
| 1. Architect          (system)   |  ← Cannot be reordered or deleted
| 2. Owner              (system)   |
| 3. Department Head    (custom)   |  ← Custom role, can be repositioned
| 4. Senior Analyst     (custom)   |
| 5. Gatekeeper         (system)   |
| 6. Builder            (system)   |
| 7. Compliance Officer (custom)   |
| 8. Trainee            (custom)   |
| 9. Viewer             (system)   |  ← Cannot be reordered below this
|----------------------------------|
| Higher = more authority.         |
| A role can only manage roles     |
| below it in the hierarchy.       |
+----------------------------------+
```

#### Multiple Roles Per User

Like Discord, a user can have multiple roles. Permissions are **additive** (union of all role permissions):
- Ana has "Builder" + "Senior Analyst" → gets all Builder permissions plus `approve_low_risk`
- Dev has "Gatekeeper" + "Compliance Officer" → gets Gatekeeper permissions plus audit log access

**Exception:** Deny overrides always win. If any role explicitly denies a permission, it's denied regardless of other roles granting it.

### Permission Catalog

The full list of granular permissions that can be assigned to custom roles:

| Category | Permission | Description | Default Roles That Have It |
|----------|-----------|-------------|---------------------------|
| **Vaults** | `view_vaults` | View vault data, events, health | All except none |
| | `create_vaults` | Create new vaults | Builder+ |
| | `edit_vault_metadata` | Edit vault name, type, metadata | Builder+ |
| | `archive_vaults` | Soft-archive vaults | Owner+ |
| | `delete_vaults` | Permanent delete | Architect only |
| **Extraction** | `view_extraction` | View extraction results | All |
| | `run_extraction` | Trigger extraction | Builder+ |
| | `configure_extraction` | Edit anchors, synonyms, configs | Designer+ |
| **Patches** | `create_patches` | Draft patches | Builder+ |
| | `submit_patches` | Submit for review | Builder+ |
| | `approve_low_risk` | Approve low-risk patches | Gatekeeper+ |
| | `approve_medium_risk` | Approve medium-risk | Gatekeeper+ |
| | `approve_high_risk` | Approve high-risk | Owner+ |
| | `apply_patches` | Apply to baseline | Owner+ |
| **Triage** | `view_triage` | View triage items | All |
| | `create_triage` | Create triage items | Builder+ |
| | `resolve_triage` | Resolve/dismiss | Gatekeeper+ |
| **Tasks** | `view_tasks` | View tasks | All |
| | `manage_tasks` | Create, assign, complete | Builder+ |
| **AI** | `chat_with_otto` | Use Otto AI | Builder+ |
| | `configure_otto` | Configure agent roles/prompts | Owner+ |
| **Export** | `export_csv` | CSV export | Builder+ |
| | `export_pdf` | PDF export | Builder+ |
| | `export_json` | JSON export | Owner+ |
| | `export_docx` | DOCX export | Builder+ |
| **Admin** | `manage_members` | User management | Owner+ |
| | `manage_roles` | Role creation/editing | Owner+ |
| | `toggle_features` | Feature flag control | Owner+ |
| | `calibrate_thresholds` | Calibration parameters | Owner+ |
| | `view_audit_log` | Audit trail access | Gatekeeper+ |
| | `system_health` | Health dashboard | Owner+ |
| | `manage_connectors` | External integrations | Owner+ |

### Database Schema Addition

```sql
-- Custom role definitions
CREATE TABLE custom_roles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id),
    name TEXT NOT NULL,
    color TEXT DEFAULT '#64748B',           -- Hex color for badges
    description TEXT,
    base_template TEXT,                     -- 'builder', 'gatekeeper', etc. or NULL
    permissions TEXT[] NOT NULL DEFAULT '{}', -- Array of permission strings
    hierarchy_position INT NOT NULL,        -- Higher = more authority
    is_system BOOLEAN DEFAULT false,        -- true for the 5 base roles
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ DEFAULT now(),
    updated_at TIMESTAMPTZ DEFAULT now(),
    UNIQUE(workspace_id, name)
);

-- Users can have multiple roles (junction table)
CREATE TABLE user_custom_roles (
    user_id UUID NOT NULL REFERENCES users(id),
    workspace_id UUID NOT NULL REFERENCES workspaces(id),
    role_id UUID NOT NULL REFERENCES custom_roles(id),
    assigned_by UUID REFERENCES users(id),
    assigned_at TIMESTAMPTZ DEFAULT now(),
    PRIMARY KEY (user_id, workspace_id, role_id)
);
```

### Updated Permission Computation

```python
def compute_effective_permissions(user_id, workspace_id, module_id, channel_id=None):
    """Compute permissions from multiple custom roles (additive, Discord-style)."""

    # 1. Get all roles for this user
    roles = get_user_roles(user_id, workspace_id)

    # 2. Union all permissions from all roles
    permissions = set()
    for role in roles:
        permissions |= set(role.permissions)

    # 3. Apply module-level overrides (ad-hoc grants/denies)
    overrides = get_active_overrides(user_id, workspace_id, 'module', module_id)
    for ov in overrides:
        if ov.effect == 'grant':
            permissions.add(ov.permission)
        elif ov.effect == 'deny':
            permissions.discard(ov.permission)

    # 4. Apply channel-level overrides
    if channel_id:
        channel_overrides = get_active_overrides(user_id, workspace_id, 'channel', channel_id)
        for ov in channel_overrides:
            if ov.effect == 'grant':
                permissions.add(ov.permission)
            elif ov.effect == 'deny':
                permissions.discard(ov.permission)

    # 5. Enforce SoD (non-overridable, hardcoded)
    # Self-approval checked at action time

    return permissions
```

### Migration Path

The existing 5 base roles become **system roles** (is_system = true) with pre-configured permissions. They can't be deleted but their permissions can be viewed as a template. Custom roles extend on top of them.

### Audit

Every role creation, modification, assignment, and removal is logged:
- `role.created` — new custom role defined
- `role.modified` — permissions changed on existing role
- `role.assigned` — role assigned to user
- `role.revoked` — role removed from user
- `role.deleted` — custom role removed (system roles can't be deleted)
