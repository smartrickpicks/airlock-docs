# Vault Hierarchy & Entity Model

> **Status:** FOUNDATIONAL — This spec defines how vaults relate to each other and how the CRM emerges from the vault system.

> **Key Insight:** The vault hierarchy IS the CRM. There is no separate CRM data model — the tree of vaults (parent companies, divisions, counterparties, items) is the relationship graph. The "CRM module" is a lens over this tree, not a separate database.

> **Terminology:** See [Naming & Hierarchy](../Shell/naming-and-hierarchy.md) for canonical vocabulary. A **vault** is Airlock's fundamental unit of work — source material + annotation layer + metadata + audit trail + collaboration context.

---

## The Problem

When a company acquires another company, inherits contracts, or works with the same counterparty across many deals, a flat list of vaults doesn't capture the real-world relationships. Airlock needs a tree structure where:

- A parent company vault can contain division vaults
- A division vault can contain counterparty/asset vaults
- A counterparty vault can contain individual item vaults (contracts, tasks, documents)
- Any level can be the "entry point" depending on the user's work context

---

## Hierarchy Model

```
WORKSPACE (tenant boundary)
  |
  +-- PARENT VAULT (top-level entity)
  |     |
  |     +-- DIVISION VAULT (acquired/subsidiary entity)
  |     |     |
  |     |     +-- COUNTERPARTY VAULT (business relationship)
  |     |     |     |
  |     |     |     +-- ITEM VAULT (contract, task, document)
  |     |     |     +-- ITEM VAULT
  |     |     |
  |     |     +-- COUNTERPARTY VAULT
  |     |           |
  |     |           +-- ITEM VAULT
  |     |
  |     +-- COUNTERPARTY VAULT (direct, no division)
  |           |
  |           +-- ITEM VAULT
  |
  +-- PARENT VAULT
        |
        +-- COUNTERPARTY VAULT
              |
              +-- ITEM VAULT
```

### Vault Levels

| Level | Name | Description | Example |
|-------|------|-------------|---------|
| 0 | **Workspace** | Tenant boundary. Not a vault — the organizational root. | "Ostereo Music Group" |
| 1 | **Parent Vault** | Top-level legal entity or corporate group. | "Acme Inc" |
| 2 | **Division Vault** | Acquired company, subsidiary, business unit, or label. | "Big Booty Inc" (acquired by Acme) |
| 3 | **Counterparty Vault** | External party that does business with the division/parent. | "Sony Music" (counterparty to Big Booty Inc) |
| 4 | **Item Vault** | Individual work artifact processed through chambers. | "Sony-BigBooty Distribution Agreement Q1-2026" |

### Rules

1. **Depth is flexible.** Not every vault needs all 4 levels. A simple workspace might only have Parent → Item (no divisions, no counterparties). The hierarchy adapts to organizational complexity.

2. **Levels 1-3 are organizational.** They don't move through chambers themselves — they exist to group and organize. They have metadata, audit trails, and health aggregates, but no gate progression.

3. **Level 4 (Item Vault) is the work unit.** Only item vaults progress through Discover > Build > Review > Ship. They have gate colors, SLA timers, approval chains.

4. **Parent-child is always explicit.** Every vault has a `parent_vault_id` (nullable at level 1). No implicit grouping — the tree is the source of truth.

5. **A vault can exist at any level without children.** Not all parents have divisions. Not all divisions have counterparties. The tree grows as work enters the system.

---

## How Vaults Get Created

### Scenario 1: New Customer (No Existing Vault)

```
1. User uploads contract PDF
2. Extraction identifies legal entity: "Big Booty Inc"
3. Entity resolution: no match found (confidence < 0.40)
4. Signal panel shows "New customer detected" card
5. User clicks "Create in CRM"
6. System creates:
   - Parent Vault: "Big Booty Inc" (level 1)
   - Counterparty Vault: extracted counterparty (level 3, child of Big Booty Inc)
   - Item Vault: the contract (level 4, child of counterparty)
7. Item vault enters Discover chamber, begins processing
```

### Scenario 2: Existing Customer (Vault Already Exists)

```
1. User uploads another contract for Big Booty Inc
2. Extraction identifies legal entity: "Big Booty Inc"
3. Entity resolution: exact match (confidence 1.0) → "Big Booty Inc" parent vault
4. Extraction identifies counterparty: "Warner Music"
5. Entity resolution checks for existing counterparty vault under Big Booty Inc:
   a. If found → item vault created as child of existing counterparty vault
   b. If not found → new counterparty vault created, then item vault under it
6. Item vault enters Discover chamber
```

### Scenario 3: Acquisition (Parent Adopts Division)

```
1. Acme Inc acquires Big Booty Inc
2. Admin action: "Add Division" on Acme Inc parent vault
3. Big Booty Inc vault (previously level 1) becomes a Division vault (level 2) under Acme Inc
4. All of Big Booty Inc's counterparty and item vaults move with it
5. Acme Inc's aggregate health, contract counts, and CRM view now include Big Booty Inc's data
6. Audit event logged: "Big Booty Inc → division of Acme Inc" with actor + timestamp
```

### Scenario 4: Batch Ingest (Multiple Contracts)

```
1. User uploads batch of 12 PDFs from Big Booty Inc
2. Batch processor creates a batch record (not a vault — batches are processing containers)
3. For each PDF:
   a. Extract legal entity + counterparty
   b. Resolve against existing vault hierarchy
   c. Create item vault under appropriate counterparty vault
   d. If counterparty vault doesn't exist, create it
4. All 12 item vaults appear in Big Booty Inc's tree
5. Triage dashboard shows the batch progress
```

---

## Data Model

### `vaults` Table

```sql
CREATE TABLE vaults (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id),

    -- Hierarchy
    parent_vault_id UUID REFERENCES vaults(id),
    vault_level INT NOT NULL DEFAULT 4,  -- 1=parent, 2=division, 3=counterparty, 4=item

    -- Identity
    name TEXT NOT NULL,
    slug TEXT NOT NULL,                   -- URL-friendly: "big-booty-inc"
    vault_type TEXT NOT NULL,             -- 'entity' | 'division' | 'counterparty' | 'contract' | 'task' | 'document'
    module_type TEXT,                     -- which module owns this vault (NULL for org-level vaults)

    -- State (item vaults only)
    chamber TEXT DEFAULT 'discover',      -- discover | build | review | ship (NULL for levels 1-3)
    gate TEXT,                            -- current gate within chamber (NULL for levels 1-3)

    -- Metadata
    metadata JSONB NOT NULL DEFAULT '{}', -- type-specific data (address, industry, deal size, etc.)
    health_score FLOAT,                   -- aggregate for org vaults, direct for item vaults

    -- Timestamps
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    archived_at TIMESTAMPTZ,

    -- Constraints
    UNIQUE (workspace_id, slug),
    CHECK (vault_level BETWEEN 1 AND 4)
);

CREATE INDEX idx_vaults_parent ON vaults(parent_vault_id);
CREATE INDEX idx_vaults_workspace ON vaults(workspace_id, vault_level);
CREATE INDEX idx_vaults_module ON vaults(workspace_id, module_type);
CREATE INDEX idx_vaults_slug ON vaults(workspace_id, slug);
```

### Key Relationships

```
vaults.parent_vault_id → vaults.id  (self-referential tree)
vaults.id → channel_events.channel_id  (events live on vaults, not separate channels)
vaults.id → vault_members.vault_id  (who can see/act)
vaults.id → contracts.vault_id  (item vaults link to source records)
```

### Vault Members (Access Control)

```sql
CREATE TABLE vault_members (
    vault_id UUID NOT NULL REFERENCES vaults(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id),
    role TEXT NOT NULL DEFAULT 'member',  -- 'owner' | 'gatekeeper' | 'builder' | 'viewer'
    inherited BOOLEAN NOT NULL DEFAULT false, -- true = inherited from parent vault
    PRIMARY KEY (vault_id, user_id)
);
```

**Inheritance rule:** When a user is a member of a parent vault, they automatically inherit membership at all child vaults (with `inherited = true`). Direct assignments override inherited roles.

---

## CRM as Vault Hierarchy

The "CRM module" is not a separate data store. It's a **view layer** over the vault tree filtered to levels 1-3 (Parent, Division, Counterparty).

### What the CRM Module Shows

| CRM View | Vault Level | What's Displayed |
|----------|-------------|-----------------|
| **Accounts** | Level 1 (Parent) | List of parent companies with aggregate stats |
| **Divisions** | Level 2 (Division) | Subsidiaries/labels under a parent, with their own aggregates |
| **Contacts** | Level 3 (Counterparty) | Business relationships with deal history |
| **Pipeline** | Level 3-4 | Counterparty vaults + their item vaults, grouped by chamber |

### CRM Aggregate Metrics (Computed from Children)

| Metric | Source | Computation |
|--------|--------|-------------|
| Total Contracts | Count of level-4 item vaults (type='contract') | Recursive count through tree |
| Health Score | Average of child item vault health scores | Weighted by recency or deal size |
| Active Deals | Count of item vaults NOT in Ship chamber | Filter by chamber != 'ship' |
| Revenue | Sum of extracted financial values from item vaults | Aggregate from extraction metadata |
| Last Activity | Most recent event across all child vaults | MAX(channel_events.created_at) |
| Chamber Distribution | Count of item vaults per chamber | GROUP BY chamber |

### CRM Sub-Panel (When CRM Module Selected)

```
+------------------------+
| CRM                    |
| > Mini Dashboard       |
|   42 Accounts          |
|   8 New Leads          |
|   12 Active Deals      |
|   3 At Risk            |
|------------------------+
| > DISCOVER             |
|   o New Leads          |   <-- Parent vaults with no item vaults yet
|   o Unqualified        |   <-- Low-confidence entity matches
|                        |
| > BUILD                |
|   o Pipeline           |   <-- Counterparties with active item vaults
|   o Account Enrichment |   <-- Parent vaults needing metadata
|                        |
| > REVIEW               |
|   o Deal Review        |   <-- Counterparty vaults with items in Review
|   o Health Alerts      |   <-- Vaults with health score below threshold
|                        |
| > SHIP                 |
|   o Won Deals          |   <-- Counterparty vaults with all items shipped
|   o Onboarding         |   <-- New relationships being activated
|------------------------+
| ACTIVE VAULTS          |
|   # Acme Inc         o |
|   # Big Booty Inc    o |
|   # Sony Music       o |
+------------------------+
```

### Navigation: CRM → Contracts

When a user is in the CRM module viewing "Acme Inc" and clicks a contract under it, the system:

1. Switches to the Contracts module (module bar highlights Contracts)
2. Opens that item vault in the triptych
3. The sub-panel shows the Contracts module's chamber views
4. Breadcrumb in the main content header shows: "Acme Inc > Big Booty Inc > Sony Music > sony-dist-2026"

The vault hierarchy provides the breadcrumb trail across modules.

---

## Vault Hierarchy in the Triptych

When viewing an **organizational vault** (levels 1-3), the triptych adapts:

### Parent/Division Vault (Levels 1-2)

| Panel | Content |
|-------|---------|
| **Signal** | Aggregated events from all child vaults. Health alerts. New entity detections. |
| **Orchestrate** | Dashboard: child vault table, chamber distribution chart, recent activity, top issues. |
| **Control** | Aggregate health score. Division tree. Ownership. Audit trail of structural changes (acquisitions, reorganizations). |

### Counterparty Vault (Level 3)

| Panel | Content |
|-------|---------|
| **Signal** | Events from this counterparty's item vaults. Relationship health alerts. |
| **Orchestrate** | Item vault list: each contract/task with gate color, health %, last activity. Click to drill into item. |
| **Control** | Relationship metadata: contact info, deal history, total value, avg health. Entity resolution confidence. |

### Item Vault (Level 4) — Standard Behavior

The full chamber-driven triptych as spec'd in [Universal Chambers](../Shell/universal-chambers.md). Gate progression, SLA timers, approval chains, etc.

---

## Entity Resolution → Vault Hierarchy Bridge

Entity resolution (spec'd in [Record Inspector / Entity Resolution](../Record%20Inspector/entity-resolution.md)) feeds directly into the vault hierarchy:

| Resolution Outcome | Vault Action |
|--------------------|-------------|
| Exact match (1.0) | Item vault created under matched counterparty vault |
| High confidence (>0.80) | Item vault created under matched counterparty; confirmation prompt in Signal |
| Ambiguous (0.40-0.80) | Item vault created but **unlinked** until user disambiguates; appears in "Unqualified" CRM view |
| No match (<0.40) | Item vault created as orphan; "New Lead" in CRM Discover chamber |
| New customer | User creates parent vault + counterparty vault; item vault linked |

**Unlinked vaults** (confidence too low to auto-resolve) appear in:
- Contracts module: Signal panel shows entity resolution card
- CRM module: "Unqualified" view under Discover chamber
- Home module: Triage widget shows "X vaults need entity resolution"

---

## Structural Operations

| Operation | Who Can Do It | What Happens |
|-----------|--------------|-------------|
| **Create Parent Vault** | Owner, Admin | New level-1 vault; appears in CRM Accounts |
| **Add Division** | Owner, Admin | New level-2 vault under a parent; or re-parent an existing level-1 vault |
| **Merge Vaults** | Admin | Combine two vaults at the same level; all children transfer to survivor; audit trail preserved |
| **Split Division** | Admin | Extract a division from a parent into its own level-1 vault |
| **Re-parent Item** | Builder, Gatekeeper | Move an item vault to a different counterparty (e.g., misattributed contract) |
| **Archive Vault** | Owner | Soft-delete; vault and children hidden from active views, data preserved |

All structural operations emit audit events and are reversible within 30 days.

---

## Cross-Module Behavior

The vault hierarchy is **module-agnostic at levels 1-3**. A parent vault ("Acme Inc") exists across all modules:

| Module | What It Sees Under "Acme Inc" |
|--------|------------------------------|
| **Contracts** | Item vaults of type 'contract' — with chambers, gates, extraction results |
| **CRM** | The full tree — parent, divisions, counterparties, relationship metadata |
| **Tasks** | Item vaults of type 'task' — auto-generated from contract milestones |
| **Calendar** | Date events extracted from item vaults — renewals, terminations, deadlines |
| **Documents** | Item vaults of type 'document' — uploaded files, templates, generated docs |

Each module filters the vault tree by `vault_type` and `module_type`, but the underlying hierarchy is shared. Editing "Acme Inc" metadata in CRM updates it everywhere.

---

## Key Principles

1. **The vault tree is the CRM.** No separate entity/account tables. The vault hierarchy stores all relationship data. The CRM module is a filtered view.

2. **Organizational vaults don't have chambers.** Only level-4 item vaults progress through Discover > Build > Review > Ship. Levels 1-3 aggregate their children's status.

3. **Entity resolution creates the tree.** When a document is ingested, entity resolution determines where in the tree the new item vault goes. The resolver builds the hierarchy automatically.

4. **Inheritance flows downward.** Membership, permissions, and metadata cascade from parent to child. Direct assignments override inherited values.

5. **Modules are lenses, not silos.** Every module sees the same vault tree. They filter by vault type and show module-specific tools, but the data is shared.

6. **Structural changes are audited.** Every merge, split, re-parent, and acquisition is logged with full before/after state.

---

## Related Specs

- **[Naming & Hierarchy](../Shell/naming-and-hierarchy.md)** — Canonical vocabulary (Vault, Chamber, Gate, View)
- **[Universal Chambers](../Shell/universal-chambers.md)** — Chamber lifecycle (Discover > Build > Review > Ship)
- **[Entity Resolution](../Record%20Inspector/entity-resolution.md)** — How entities get matched to the vault tree
- **[Roles](../Roles/overview.md)** — Role-based access at each vault level
