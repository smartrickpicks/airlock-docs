# CRM Module

> **Status:** SPECCED — CRM is a lens over the vault hierarchy, not a separate data store.

> **Key Insight:** The vault tree IS the CRM. Parent vaults are accounts, division vaults are subsidiaries, counterparty vaults are contacts/relationships, and item vaults are deals. The CRM module provides views, workflows, and enrichment tools over this shared tree.

> **Tech Stack:** [react-admin](https://github.com/marmelab/react-admin) + [Atomic CRM](https://github.com/marmelab/atomic-crm) patterns. See [Tech Stack](../TechStack/overview.md).

---

## Module Identity

| Property | Value |
|----------|-------|
| Module bar icon | Users (Lucide) |
| Module name | CRM |
| Module color | Teal/cyan accent |
| Primary data | Vault hierarchy (levels 1-3) |
| Chamber lifecycle | Discover > Build > Review > Ship (applied to relationships, not individual contracts) |

---

## How CRM Relates to the Vault Tree

See [Vault Hierarchy](../VaultHierarchy/overview.md) for the full data model.

| Vault Level | CRM Concept | CRM View |
|-------------|-------------|----------|
| Level 1 (Parent) | **Account** | Account list, account detail page |
| Level 2 (Division) | **Subsidiary / Label** | Nested under parent account |
| Level 3 (Counterparty) | **Contact / Relationship** | Contact list, relationship detail |
| Level 4 (Item) | **Deal / Contract** | Shown inline on relationship detail (links to Contracts module) |

The CRM module **never creates its own data tables** for accounts/contacts. It queries the `vaults` table with `vault_level IN (1, 2, 3)`.

---

## Sub-Panel (When CRM Module Selected)

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
|   o New Leads          |
|   o Unqualified        |
|                        |
| > BUILD                |
|   o Pipeline           |
|   o Account Enrichment |
|                        |
| > REVIEW               |
|   o Deal Review        |
|   o Health Alerts      |
|                        |
| > SHIP                 |
|   o Won Deals          |
|   o Onboarding         |
|------------------------+
| ACTIVE VAULTS          |
|   # Acme Inc         o |
|   # Big Booty Inc    o |
|   # Sony Music       o |
+------------------------+
```

---

## Chamber Views

### Discover Chamber

**New Leads** — Parent vaults created from entity resolution where `entity.new_customer` event was emitted. These are companies/entities that were encountered for the first time during contract ingestion. No manual data entry required — the extraction pipeline feeds the CRM.

**Unqualified** — Vaults at levels 1-3 where entity resolution confidence is below threshold. Need human review to confirm identity, merge with existing account, or create new relationship.

### Build Chamber

**Pipeline** — Counterparty vaults (level 3) grouped by their aggregate chamber distribution. A counterparty whose item vaults are mostly in Discover/Build is "early pipeline." One whose items are in Review/Ship is "late pipeline." Kanban board layout using @hello-pangea/dnd.

Pipeline columns:
- **Lead** — counterparties with 0 items past Discover
- **Active** — counterparties with items in Build
- **Review** — counterparties with items in Review
- **Closed** — counterparties with all items in Ship/archived

**Account Enrichment** — Parent/division vaults that need metadata (industry, address, key contacts, deal size). Task list from universal task system filtered to `module_type = 'crm' AND task_type = 'action'`.

### Review Chamber

**Deal Review** — Counterparty vaults with item vaults in the Review chamber. Gatekeeper's view showing aggregate health scores, pending approvals, and SLA status across all deals for a relationship.

**Health Alerts** — Vaults (any level) where health score dropped below threshold or trend is worsening. Sorted by severity.

### Ship Chamber

**Won Deals** — Counterparty vaults where all item vaults have shipped. Celebration state — show total value, timeline, key milestones.

**Onboarding** — Newly created relationships that need operational setup. Tasks auto-created by cross-module event bus when contracts ship.

---

## Vault Triptych in CRM

When clicking a vault in the CRM module:

### Account Vault (Level 1 or 2)

| Panel | Content |
|-------|---------|
| **Signal** | Aggregated events from all child vaults: new contracts, health changes, entity resolutions, approvals. Filtered to relationship-level events. |
| **Orchestrate** | Dashboard: divisions list (if any), counterparty table with deal counts and health, chamber distribution chart, aggregate financials, timeline of key events. |
| **Control** | Account metadata: industry, address, contacts, ownership. Aggregate health score. Structural controls (add division, merge, archive). Audit trail. |

### Counterparty Vault (Level 3)

| Panel | Content |
|-------|---------|
| **Signal** | Events from this counterparty's deals: extraction results, patches, approvals, health changes. |
| **Orchestrate** | Item vault table: each contract/deal with gate color, health %, chamber, last activity. Click to drill into Contracts module. Relationship notes (TipTap). |
| **Control** | Relationship metadata: primary contact, deal value, entity resolution confidence, first/last interaction. Version history. |

---

## Cross-Module Integration

### CRM ← Contracts (Automatic)

| Contract Event | CRM Action |
|---------------|------------|
| `entity.new_customer` | Create parent vault (level 1) + counterparty vault (level 3) in CRM |
| `entity.resolved` | Link item vault to existing counterparty vault |
| `gate.advanced(ship)` | Update counterparty vault aggregate metrics |
| `health.updated` | Recalculate counterparty + parent vault aggregate health |

### CRM → Contracts (Navigation)

Clicking an item vault row in CRM Orchestrate panel → switches to Contracts module → opens that vault's triptych.

### CRM ← Tasks (Cross-Module)

Tasks created in the CRM module (account enrichment, follow-up actions) appear in the universal task table and render in both:
- CRM module's triage views
- Tasks module's universal view
- Home workspace's CRM triage widget

---

## react-admin Integration

The CRM module uses react-admin patterns for its list/detail views:

| react-admin Component | CRM Usage |
|----------------------|-----------|
| `<List>` + `<Datagrid>` | Account list, counterparty list, pipeline table |
| `<Show>` | Account detail, counterparty detail |
| `<Edit>` | Account metadata editing, relationship notes |
| `<ReferenceField>` | Parent → division → counterparty navigation |
| `<FilterList>` | Chamber filtering, health score range, industry |
| `<BulkActionButtons>` | Bulk assign, bulk merge, bulk archive |

**Data Provider:** Custom data provider that queries the Airlock vault API (`/api/vaults?vault_level=1&module_type=crm`).

---

## Related Specs

- **[Vault Hierarchy](../VaultHierarchy/overview.md)** — The data model CRM views
- **[Entity Resolution](../Record%20Inspector/entity-resolution.md)** — How new accounts get created
- **[Tech Stack](../TechStack/overview.md)** — react-admin + Atomic CRM
- **[Universal Task System](../TaskSystem/overview.md)** — CRM triage backed by master task table
- **[Notifications](../Notifications/overview.md)** — CRM health alerts, new lead notifications
