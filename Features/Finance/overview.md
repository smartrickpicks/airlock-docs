# Finance -- Design Spec

> **Status:** BRAINSTORM
> **Source:** Clause library v2 financial variables, Hiperweb ERP conversation, open-source billing research, entertainment royalty accounting standards

## Summary

Finance is Airlock's intelligence layer for turning contractual financial terms into live obligations, calculations, statements, and invoices -- without becoming an accounting system. It bridges the gap between what the clause library promises (royalty rates, advance amounts, payment frequencies) and what artists, labels, and producers actually get paid. All money movement, tax withholding, general ledger entries, and bank account management live in an external ERP (Hiperweb, QuickBooks, NetSuite, Xero). Airlock calculates, generates, triggers, and records.

---

## 1. Core Architecture: Intelligence Layer, Not Accounting

### The Boundary

The single most important design decision in this module: **Airlock is the intelligence layer, not the accounting system.** This boundary comes from a conversation with the owner of Hiperweb ERP and reflects a hard lesson from enterprise software -- building accounting into a data-operations platform creates a compliance nightmare that consumes the entire engineering team.

| Airlock Does | External ERP Does |
|---|---|
| **CALCULATES** what is owed (obligation engine) | Holds chart of accounts, GL entries |
| **GENERATES** statements and invoices (document pipeline) | Manages bank accounts, payment rails |
| **TRIGGERS** payments in external systems (integration layer) | Withholds taxes (federal, state, international) |
| **RECORDS** that it happened (append-only audit trail) | Handles multi-currency conversion at settlement |
| **TRACKS** recoupment balances and royalty waterfalls | Runs payroll, 1099s, W-8BEN processing |
| **ALERTS** on overdue obligations and SLA breaches | Manages AR/AP aging, collections |

### Why This Boundary Matters

1. **SOX and audit exposure.** The moment Airlock touches actual money movement, it becomes subject to SOX compliance (if customers are public companies) and dramatically increases audit scope. By staying as the calculation layer, Airlock avoids becoming a "system of record" for financial transactions.

2. **Tax complexity.** Entertainment royalties cross international borders constantly. Withholding rates depend on tax treaties, artist residency, corporate structure, and payment type. This is a full-time job for specialized accounting software.

3. **Bank integration risk.** PCI DSS, ACH compliance, wire transfer regulations -- none of this should live in a data-operations platform.

4. **Customer flexibility.** Labels already have accounting systems. Forcing them to use Airlock for payments would be a dealbreaker. Instead, Airlock integrates with whatever they already use.

---

## 2. Clause-to-Obligation Pipeline

### How It Works

When a contract vault reaches the Ship chamber (fully approved and executed), an event bus job extracts financial obligations from the contract's clause library data. Each financial clause produces one or more `Obligation` records.

```
Contract Signed (Ship chamber gate passed)
    |
    v
EventBus: CONTRACT_EXECUTED event
    |
    v
Obligation Extraction Job (BullMQ worker)
    |
    +-- Scans clause_library matches for financial clause_types
    |   - ROYALTY_RATE -> royalty obligation
    |   - ADVANCE_AMOUNT -> advance obligation
    |   - FLAT_FEE -> flat_fee obligation
    |   - MINIMUM_GUARANTEE -> minimum_guarantee obligation
    |   - MECHANICAL_ROYALTY -> mechanical_royalty obligation
    |   - SYNC_FEE -> sync_fee obligation
    |   - PRODUCER_POINTS -> producer_points obligation
    |   - PAYMENT_FREQUENCY -> sets schedule on parent obligation
    |
    v
Obligation records created, linked to vault + clause
    |
    v
Payment schedule generated from frequency + term dates
    |
    v
Signal panel event: "7 obligations extracted from Henderson MSA"
```

### Variable-to-Obligation Mapping

| Clause Variable | Canonical Key | Obligation Type | Obligation Field |
|---|---|---|---|
| `OPP_ROYALTY_RATE` | `Royalty_Rate__c` | `royalty` | `rate` (percentage) |
| `OPP_ADVANCE_AMOUNT` | `Advance_Amount__c` | `advance` | `amount` (fixed) |
| `OPP_PAYMENT_FREQUENCY` | `Payment_Frequency__c` | (modifier) | Sets `frequency` on parent obligation |
| `OPP_TERRITORY` | `Territory__c` | (modifier) | Sets `territory` scope on obligation |
| `OPP_EFFECTIVE_DATE` | `Effective_Date__c` | (modifier) | Sets `start_date` on obligation |
| `OPP_TERM_END_DATE` | `Term_End_Date__c` | (modifier) | Sets `end_date` on obligation |
| `OPP_MINIMUM_GUARANTEE` | `Minimum_Guarantee__c` | `minimum_guarantee` | `amount` (fixed) |
| `OPP_SYNC_FEE` | `Sync_Fee__c` | `sync_fee` | `amount` (per-placement) |
| `OPP_PRODUCER_POINTS` | `Producer_Points__c` | `producer_points` | `rate` (points) |
| `OPP_MECHANICAL_RATE` | `Mechanical_Rate__c` | `mechanical_royalty` | `rate` (per-unit) |

### Obligation Types

| Type | Rate/Amount Model | Recoupable? | Schedule |
|---|---|---|---|
| `royalty` | Percentage of net receipts | Yes (against advances) | Per-period (quarterly typical) |
| `advance` | Fixed amount | Source of recoupment balance | Paid on execution or milestone |
| `flat_fee` | Fixed amount | No | One-time or milestone-based |
| `minimum_guarantee` | Fixed minimum per period | Offsets against royalties | Per-period |
| `mechanical_royalty` | Statutory per-unit rate | Sometimes | Per-period |
| `sync_fee` | Fixed per-placement | No | Per-placement invoice |
| `producer_points` | Points on retail/wholesale | Yes (after recoupment) | Per-period |

---

## 3. Data Model (PostgreSQL)

All tables follow existing Airlock patterns: `workspace_id` for tenant isolation (RLS), `id` as UUID primary key, `created_at`/`updated_at` timestamps, encrypted at rest per cipher model.

### obligation

The core record linking a contractual promise to a calculable financial event.

```sql
CREATE TABLE obligation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspace(id),
    vault_id        UUID NOT NULL REFERENCES vault(id),
    clause_id       TEXT NOT NULL,                          -- e.g., 'MUSIC-COMP-DIST-V1'
    obligation_type TEXT NOT NULL CHECK (obligation_type IN (
        'royalty', 'advance', 'flat_fee', 'minimum_guarantee',
        'mechanical_royalty', 'sync_fee', 'producer_points'
    )),
    rate            NUMERIC(10, 6),                         -- percentage (0.150000 = 15%) or per-unit
    amount          NUMERIC(14, 2),                         -- fixed dollar amount
    currency        TEXT NOT NULL DEFAULT 'USD',
    frequency       TEXT CHECK (frequency IN (
        'one_time', 'monthly', 'quarterly', 'semi_annual', 'annual', 'per_event'
    )),
    territory       TEXT,                                   -- ISO 3166 or 'worldwide'
    start_date      DATE NOT NULL,
    end_date        DATE,                                   -- NULL = perpetual
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN (
        'active', 'paused', 'completed', 'terminated', 'pending_execution'
    )),
    conditions_json JSONB DEFAULT '{}',                     -- escalation tiers, format splits, etc.
    parent_obligation_id UUID REFERENCES obligation(id),   -- for cross-collateralization pools
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    version         INTEGER NOT NULL DEFAULT 1              -- optimistic locking
);

-- RLS policy
ALTER TABLE obligation ENABLE ROW LEVEL SECURITY;
CREATE POLICY obligation_workspace ON obligation
    USING (workspace_id = current_setting('app.workspace_id')::UUID);
```

### payment_schedule

Generated from obligation frequency + term dates. One row per payment period.

```sql
CREATE TABLE payment_schedule (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    obligation_id           UUID NOT NULL REFERENCES obligation(id),
    period_start            DATE NOT NULL,
    period_end              DATE NOT NULL,
    due_date                DATE NOT NULL,
    status                  TEXT NOT NULL DEFAULT 'pending' CHECK (status IN (
        'pending', 'calculated', 'invoiced', 'paid', 'overdue', 'waived'
    )),
    amount_calculated       NUMERIC(14, 2),                 -- output of royalty engine
    amount_paid             NUMERIC(14, 2) DEFAULT 0,
    calculation_detail_json JSONB DEFAULT '{}',              -- full waterfall breakdown
    calculated_at           TIMESTAMPTZ,
    paid_at                 TIMESTAMPTZ,
    external_payment_ref    TEXT,                            -- ERP payment ID
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### recoupment_ledger

Tracks advance recoupment over time. Each row represents one period's application of revenue against an outstanding advance balance.

```sql
CREATE TABLE recoupment_ledger (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    obligation_id           UUID NOT NULL REFERENCES obligation(id),  -- the royalty obligation
    advance_obligation_id   UUID NOT NULL REFERENCES obligation(id),  -- the advance being recouped
    period_start            DATE NOT NULL,
    period_end              DATE NOT NULL,
    gross_royalty            NUMERIC(14, 2) NOT NULL,        -- what would have been paid
    revenue_applied         NUMERIC(14, 2) NOT NULL,         -- amount applied to recoupment
    balance_remaining       NUMERIC(14, 2) NOT NULL,         -- advance balance after this period
    fully_recouped_at       TIMESTAMPTZ,                     -- set when balance hits zero
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### statement

Quarterly (or configurable) royalty statements aggregating all obligations for a vault.

```sql
CREATE TABLE statement (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id        UUID NOT NULL REFERENCES workspace(id),
    vault_id            UUID NOT NULL REFERENCES vault(id),
    period_start        DATE NOT NULL,
    period_end          DATE NOT NULL,
    statement_number    TEXT NOT NULL,                       -- workspace-aware: 'WS001-2026-Q1-0042'
    generated_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    status              TEXT NOT NULL DEFAULT 'draft' CHECK (status IN (
        'draft', 'reviewed', 'approved', 'sent', 'acknowledged', 'disputed'
    )),
    total_gross_revenue NUMERIC(14, 2) DEFAULT 0,
    total_deductions    NUMERIC(14, 2) DEFAULT 0,
    total_royalties     NUMERIC(14, 2) DEFAULT 0,
    recoupment_applied  NUMERIC(14, 2) DEFAULT 0,
    net_payable         NUMERIC(14, 2) DEFAULT 0,
    pdf_url             TEXT,                                -- encrypted file storage ref
    approved_by         UUID REFERENCES auth_user(id),
    approved_at         TIMESTAMPTZ,
    sent_at             TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    version             INTEGER NOT NULL DEFAULT 1
);
```

### invoice

Generated from approved statements. Tracks lifecycle through to payment confirmation.

```sql
CREATE TABLE invoice (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    statement_id        UUID REFERENCES statement(id),      -- NULL for non-royalty invoices (sync fees)
    workspace_id        UUID NOT NULL REFERENCES workspace(id),
    vault_id            UUID NOT NULL REFERENCES vault(id),
    invoice_number      TEXT NOT NULL UNIQUE,                -- 'INV-WS001-2026-0042'
    issued_at           TIMESTAMPTZ,
    due_date            DATE NOT NULL,
    status              TEXT NOT NULL DEFAULT 'draft' CHECK (status IN (
        'draft', 'approved', 'sent', 'paid', 'overdue', 'void', 'disputed'
    )),
    total               NUMERIC(14, 2) NOT NULL,
    currency            TEXT NOT NULL DEFAULT 'USD',
    line_items_json     JSONB NOT NULL DEFAULT '[]',         -- array of {description, quantity, rate, amount}
    external_ref        TEXT,                                -- QuickBooks/NetSuite/Xero invoice ID
    external_system     TEXT,                                -- 'quickbooks', 'netsuite', 'xero', 'hiperweb'
    payment_received_at TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    version             INTEGER NOT NULL DEFAULT 1
);
```

### revenue_source

Imported revenue data against which royalties are calculated. This is the "input" to the royalty engine.

```sql
CREATE TABLE revenue_source (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspace(id),
    vault_id        UUID NOT NULL REFERENCES vault(id),
    source_type     TEXT NOT NULL CHECK (source_type IN (
        'streaming', 'physical', 'sync', 'mechanical', 'performance',
        'digital_download', 'merchandise', 'live', 'other'
    )),
    source_name     TEXT,                                   -- 'Spotify', 'Apple Music', 'Amazon Physical'
    territory       TEXT,                                   -- ISO 3166 or 'worldwide'
    period_start    DATE NOT NULL,
    period_end      DATE NOT NULL,
    gross_revenue   NUMERIC(14, 2) NOT NULL,
    deductions      NUMERIC(14, 2) DEFAULT 0,               -- distribution fees, platform cuts
    net_revenue     NUMERIC(14, 2) NOT NULL,                 -- gross - deductions
    units           INTEGER,                                 -- streams, units sold, placements
    import_batch_id UUID,                                    -- links rows from same import
    imported_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Entity Relationship Diagram (Simplified)

```
vault (contract)
  |
  +-- obligation (1:N) ---- clause_id links to clause_library_v2
  |     |
  |     +-- payment_schedule (1:N per obligation)
  |     |
  |     +-- recoupment_ledger (1:N, links royalty obligation to advance obligation)
  |
  +-- revenue_source (1:N, imported revenue data)
  |
  +-- statement (1:N per period)
        |
        +-- invoice (1:1 per statement, or standalone for sync fees)
```

---

## 4. Royalty Calculation Engine

Entertainment royalties are among the most complex financial calculations in any industry. The engine must handle waterfalls, cross-collateralization, territory splits, format tiers, escalation clauses, and statutory rates.

### The Waterfall Model

Every royalty calculation follows a standard waterfall. The engine must implement each step and store the intermediate values for auditability.

```
Step 1: Gross Revenue
    Sum all revenue_source rows for the obligation's vault + territory + period
    |
Step 2: Deductions
    Subtract contractually agreed deductions:
    - Distribution fees (typically 15-25%)
    - Packaging deductions (physical only, typically 25% of retail)
    - Free goods allowance (physical only, typically 15%)
    - Reserves for returns (physical only, typically 20-35%)
    - Breakage allowance (legacy, rarely used now)
    |
Step 3: Net Receipts
    Gross Revenue - Deductions = Net Receipts
    |
Step 4: Royalty Rate Application
    Net Receipts * royalty_rate = Gross Royalty
    (May vary by format tier -- see below)
    |
Step 5: Artist Share
    If multiple payees (co-writers, featured artists):
    Gross Royalty * artist_share_percentage = Artist Royalty
    |
Step 6: Recoupment Check
    If unrecouped advance exists:
    - Artist Royalty is applied against advance balance
    - If balance > Artist Royalty: payable = $0, balance reduced
    - If balance < Artist Royalty: payable = Artist Royalty - balance, fully recouped
    |
Step 7: Net Payable
    Amount after recoupment = what gets invoiced to the ERP
```

### Cross-Collateralization

Some deals pool multiple projects (albums, films) into a single recoupment account. An artist might have three albums, all sharing one advance pool.

```
Deal: Acme Records + Jane Artist
  Advance Pool: $500,000

  Album 1 (vault_id: abc) -- earned $200,000 royalties
  Album 2 (vault_id: def) -- earned $150,000 royalties
  Album 3 (vault_id: ghi) -- earned $80,000 royalties

  Total earned: $430,000
  Balance remaining: $70,000
  Net payable this period: $0 (still unrecouped)
```

**Implementation:** The `obligation.parent_obligation_id` field links royalty obligations to a shared advance obligation. The recoupment engine queries all obligations in the pool when calculating balances.

### Territory Splits

Contracts frequently specify different royalty rates by territory.

```json
// obligation.conditions_json example
{
  "territory_rates": {
    "US": 0.15,
    "GB": 0.12,
    "EU": 0.12,
    "JP": 0.10,
    "ROW": 0.08
  }
}
```

The engine groups `revenue_source` rows by territory, applies the appropriate rate, and sums. Revenue with territory "worldwide" is split proportionally based on historical territory distribution or a configurable default split.

### Format Tiers

Different revenue formats often carry different royalty rates within the same contract.

```json
// obligation.conditions_json example
{
  "format_rates": {
    "streaming": 0.15,
    "physical": 0.20,
    "digital_download": 0.18,
    "sync": 0.50,
    "performance": 0.50
  }
}
```

The engine groups `revenue_source` rows by `source_type` and applies the matching format rate. If no format-specific rate exists, the base `obligation.rate` is used as fallback.

### Escalation Clauses

Rate increases after sales thresholds are common in music contracts. Example: "15% for the first 500,000 units, 17% for 500,001--1,000,000 units, 20% thereafter."

```json
// obligation.conditions_json example
{
  "escalation_tiers": [
    { "threshold": 0,       "rate": 0.15 },
    { "threshold": 500000,  "rate": 0.17 },
    { "threshold": 1000000, "rate": 0.20 }
  ],
  "escalation_metric": "cumulative_units"
}
```

The engine tracks cumulative units across periods and applies the correct tier rate to each segment of revenue.

### Mechanical Royalties

Statutory rates set by the Copyright Royalty Board (CRB). As of 2024:

| Duration | Rate |
|---|---|
| Songs <= 5 minutes | $0.12 per copy |
| Songs > 5 minutes | $0.0231 per minute (rounded to next minute) |

These are NOT negotiable -- they are statutory. The engine must track the current statutory rate and apply it. The `obligation.conditions_json` stores the rate schedule, but the engine validates against known statutory rates and flags discrepancies.

### Producer Points

"3 points on retail, retroactive to record one after recoupment" is a typical producer deal.

- **Points** = percentage points on the applicable base (retail price, wholesale price, or net receipts, depending on the contract).
- **Retroactive to record one** = once the album recoups, the producer gets paid on ALL units sold, not just post-recoupment units.
- **After recoupment** = producer points only kick in after the artist's advance is recouped.

```json
// obligation.conditions_json for producer_points
{
  "points": 3,
  "base": "retail",
  "retroactive": true,
  "recoupment_gate": true,
  "linked_advance_obligation_id": "uuid-of-artist-advance"
}
```

### Sync Fees

Sync fees are flat negotiated amounts per placement (song in a film, TV show, commercial). No royalty calculation needed -- just invoice generation per placement.

```
Placement event received (via EventBus or manual entry)
    |
    v
Lookup sync_fee obligation for vault
    |
    v
Generate invoice for placement amount
    |
    v
Push to ERP for payment
```

### Minimum Guarantees

Guaranteed minimum payment regardless of actual revenue. If royalties earned exceed the minimum, the higher amount is paid. If royalties fall short, the minimum is paid.

```
calculated_royalty = waterfall_result
minimum = obligation.amount (for minimum_guarantee type)

if calculated_royalty >= minimum:
    payable = calculated_royalty  -- minimum is satisfied
else:
    payable = minimum             -- shortfall covered by guarantee
    shortfall = minimum - calculated_royalty  -- logged for reporting
```

---

## 5. Statement Generation

### Quarterly Royalty Statements (Industry Standard)

The entertainment industry standard is quarterly royalty accounting. Statements are generated 45-90 days after the quarter ends (time for revenue data to arrive from distributors, DSPs, and sub-publishers).

### Statement Contents

| Section | Content |
|---|---|
| **Header** | Statement number, period, counterparty name, vault reference |
| **Revenue Summary** | Revenue by source type (streaming, physical, sync, etc.) and territory |
| **Deductions** | Itemized deductions (distribution fees, packaging, free goods, reserves) |
| **Royalty Calculations** | Per-obligation calculation showing rate, base, and computed amount |
| **Escalation Status** | Current tier, units toward next threshold |
| **Recoupment Status** | Opening balance, royalties earned, applied to recoupment, closing balance |
| **Producer Points** | Separate section if producer points obligations exist |
| **Net Payable** | Final amount after all calculations and recoupment |
| **Historical Summary** | Rolling 4-quarter summary for context |

### Statement Workflow

Reuses the PatchWorkflow approval pattern (adapted for financial documents):

```
Draft
  |
  v
Reviewed (financial analyst reviews calculations)
  |
  v
Approved (Owner role required -- see Roles spec)
  |
  v
Sent (delivered to counterparty via email or portal)
  |
  v
Acknowledged (counterparty confirms receipt)
  |
  [or]
  v
Disputed (counterparty raises objection -- creates triage item)
```

**Self-approval prevention:** The person who generated the statement cannot approve it. This reuses the PatchWorkflow's self-approval block (see `/Features/PatchWorkflow/overview.md`).

### PDF Generation

Statements render as PDFs using the existing Document Suite pipeline (TipTap template + PDF.js rendering). A statement template is added to the template library with sections matching the content table above.

---

## 6. Invoice Workflow

### Invoice Generation

Invoices are created from approved statements (for royalties) or from individual events (for sync fees, flat fees).

| Trigger | Invoice Type |
|---|---|
| Statement approved | Royalty invoice (aggregates all obligations for the period) |
| Sync placement recorded | Sync fee invoice (single line item) |
| Advance milestone reached | Advance payment invoice |
| Flat fee milestone reached | Flat fee invoice |

### Invoice Numbering

Workspace-aware sequential numbering:

```
INV-{workspace_code}-{year}-{sequence}
INV-WS001-2026-0001
INV-WS001-2026-0002
...
```

Sequence resets annually. The `invoice_number` column is UNIQUE and generated via a PostgreSQL sequence per workspace per year.

### Invoice Approval Chain

Follows the same pattern as PatchWorkflow but simplified for financial documents:

```
Draft --> Approved --> Sent --> Paid
  |                     |        |
  v                     v        v
Void              Overdue    (terminal)
                     |
                     v
                  Escalated (creates task for collections follow-up)
```

- **Draft -> Approved:** Owner role required (self-approval prevention applies).
- **Approved -> Sent:** System sends via integration or marks for manual delivery.
- **Sent -> Paid:** Payment confirmation from ERP webhook or manual entry.
- **Sent -> Overdue:** Automatic transition when `due_date` passes without payment. BullMQ scheduled job checks daily.
- **Overdue -> Escalated:** After configurable grace period (default: 15 days), creates a task in the Universal Task System.

### Payment Recording

Two paths for recording payment:

1. **Webhook from ERP:** External system sends payment confirmation with `external_ref`. Airlock updates invoice status to `paid` and records `payment_received_at`.

2. **Manual entry:** User marks invoice as paid in the Orchestrate panel with payment reference and date.

Both paths append to the audit trail.

---

## 7. ERP Integration Patterns

### Integration Models

Three integration patterns, configurable per workspace:

| Pattern | Direction | Use Case |
|---|---|---|
| **Push** | Airlock -> ERP | Airlock creates invoice, pushes to QuickBooks as a new bill/invoice |
| **Pull** | ERP -> Airlock | ERP requests "what does Airlock say is owed for vault X?" via API |
| **Webhook** | ERP -> Airlock | ERP notifies Airlock when a payment clears or an invoice is updated |

### Push Model (Primary)

```
Invoice approved in Airlock
    |
    v
ERP Integration Worker (BullMQ)
    |
    +-- Serialize invoice to ERP format
    +-- POST to ERP API (QuickBooks/NetSuite/Xero/Hiperweb)
    +-- Receive external_ref (ERP invoice ID)
    +-- Update Airlock invoice.external_ref
    +-- Append to audit trail: "Pushed to QuickBooks as INV-12345"
    |
    v
Signal panel event: "Invoice INV-WS001-2026-0042 synced to QuickBooks"
```

### Pull Model (Secondary)

```
GET /api/v1/finance/obligations/{vault_id}/calculate?period=2026-Q1

Response:
{
  "vault_id": "uuid",
  "period": "2026-Q1",
  "obligations": [...],
  "total_payable": 12500.00,
  "calculation_timestamp": "2026-04-15T10:30:00Z"
}
```

### Webhook Model (Payment Confirmation)

```
POST /api/v1/finance/webhooks/payment
{
  "external_system": "quickbooks",
  "external_ref": "INV-12345",
  "amount_paid": 12500.00,
  "paid_at": "2026-04-20T14:00:00Z",
  "payment_method": "ach"
}

--> Airlock looks up invoice by external_ref
--> Updates status to 'paid'
--> Updates payment_schedule status
--> Appends audit event
--> Signal panel: "Payment received: $12,500.00 for INV-WS001-2026-0042"
```

### Integration Targets

| System | API | Authentication | Data Mapping Notes |
|---|---|---|---|
| **Hiperweb** | REST API | API key + workspace token | Native integration (same owner ecosystem) |
| **QuickBooks Online** | REST v3 | OAuth 2.0 | Invoice -> QBO Invoice, Obligation -> QBO recurring transaction |
| **NetSuite** | SuiteTalk REST | Token-based auth (TBA) | Invoice -> NS VendorBill, complex field mapping |
| **Xero** | REST API v2.0 | OAuth 2.0 | Invoice -> Xero Invoice, good line-item support |
| **ERPNext** | REST API | API key + secret | Open-source option, full accounting module |

### Sync Status Tracking

Each obligation and invoice tracks its sync state with the external system:

```json
// obligation or invoice metadata
{
  "erp_sync": {
    "system": "quickbooks",
    "last_synced_at": "2026-04-15T10:30:00Z",
    "external_ref": "INV-12345",
    "sync_status": "synced",  // synced | pending | failed | not_configured
    "last_error": null,
    "retry_count": 0
  }
}
```

---

## 8. Open-Source Landscape Research

### Evaluated Projects

| Project | License | Language | What It Does Well | Why Not a Direct Fit |
|---|---|---|---|---|
| **Kill Bill** | Apache 2.0 | Java | Subscription billing, complex rating engine, invoice generation, payment retry logic. Battle-tested at scale (Verizon, others). | Designed for subscription/SaaS billing, not entertainment royalties. No concept of waterfall calculations, recoupment, or territory splits. JVM stack does not match Airlock's Python backend. |
| **Lago** | AGPL-3.0 | Ruby/Go | Usage-based billing, event-driven metering, real-time aggregation. Clean API design. | AGPL license is problematic for commercial use. No royalty calculation concept. Good event model could inspire revenue_source ingestion. |
| **ERPNext** | GPL-3.0 | Python (Frappe) | Full ERP: accounting, invoicing, payments, inventory, HR. Python-based. Active community. | Could serve as the "external system" rather than a component inside Airlock. Too heavyweight to embed but strong as an integration target for customers who lack an existing ERP. |
| **Odoo** | LGPL-3.0 | Python | Modular ERP, strong accounting module, massive ecosystem. | Even heavier than ERPNext. Better suited as enterprise customer's existing system that Airlock integrates with. |
| **Hyperswitch** | Apache 2.0 | Rust | Payment orchestration: smart routing across payment processors, retry logic, fallback chains. | Handles payment routing/execution, not calculation. Relevant only if Airlock ever touches actual money movement (which it should not). |
| **InvoiceNinja** | Elastic License 2.0 | PHP/Laravel | Invoice generation, client portal, payment tracking. | PHP stack mismatch. Good inspiration for invoice template design and client-facing statement portals. |
| **Crater** | AGPL-3.0 | PHP/Laravel | Invoicing, expense tracking, tax management. | AGPL + PHP. Similar inspiration value to InvoiceNinja. |

### Recommendation

**Build custom obligation/calculation engine in Airlock** (entertainment-specific logic that no open-source project handles), and **integrate with ERPNext or QuickBooks for GL/payments.**

- The royalty waterfall, recoupment, cross-collateralization, territory splits, format tiers, and escalation clauses are entertainment-industry-specific. No existing open-source billing system models these concepts.
- The invoice/statement generation can borrow patterns from Kill Bill (template rendering, numbering schemes) and Lago (event-driven metering for revenue ingestion).
- ERPNext is the recommended "external system" for customers who do not already have an ERP. It is Python-based, open-source, and has a strong accounting module.

---

## 9. Revenue Import

### The Input Problem

Royalties cannot be calculated without revenue data. Revenue comes from dozens of sources (Spotify, Apple Music, Amazon, physical distributors, sync agencies, performance rights organizations) in dozens of formats. This is the single most painful operational problem in entertainment accounting.

### Import Architecture

```
Revenue data (CSV, API, SFTP)
    |
    v
Batch Processing pipeline (reuse existing BullMQ pattern)
    |
    +-- Parse (format detection + normalization)
    +-- Validate (required fields, amount sanity checks)
    +-- Deduplicate (source + period + territory composite key)
    +-- Store (insert into revenue_source table)
    +-- Trigger (recalculate affected payment_schedule rows)
    |
    v
Signal panel: "Revenue import complete: 14,230 rows, 3 territories, Q1 2026"
```

### Minimum Viable Import Format (CSV)

```csv
source_type,source_name,territory,period_start,period_end,gross_revenue,deductions,net_revenue,units
streaming,Spotify,US,2026-01-01,2026-03-31,45000.00,6750.00,38250.00,15000000
streaming,Apple Music,US,2026-01-01,2026-03-31,22000.00,3300.00,18700.00,7500000
physical,Amazon,US,2026-01-01,2026-03-31,8500.00,2125.00,6375.00,12000
sync,Netflix,worldwide,2026-01-01,2026-03-31,50000.00,0.00,50000.00,1
```

### Future Import Sources

| Source | Integration Type | Priority |
|---|---|---|
| CSV upload | File upload + parser | Phase 2 (MVP) |
| Distributor API (DistroKid, TuneCore, CD Baby) | REST API polling | Phase 5+ |
| DSP reports (Spotify for Artists, Apple Music for Artists) | Manual download + upload | Phase 2 |
| Performance rights (ASCAP, BMI, SESAC, SoundExchange) | PDF statement parsing or API | Phase 5+ |
| Sub-publisher reports | CSV/XLSX upload | Phase 3 |

---

## 10. Regulatory Considerations

### SOX Compliance

If Airlock customers are publicly traded companies, financial data handled by Airlock may fall under SOX requirements:

- **Append-only audit trail** is already an Airlock principle (Security spec). No changes needed.
- **Access controls** must demonstrate who can view/edit/approve financial data. Role-based gating (Owner for approvals) satisfies this.
- **Change management** for calculation engine changes requires versioned configuration and documented testing. Feature Control Plane handles this.

### State Royalty Payment Laws

- **California Business and Professions Code 2500** -- specific timing requirements for royalty payments. Quarterly is compliant but contracts may specify more frequent.
- **New York Arts and Cultural Affairs Law** -- protections for performing artists.
- The system must enforce payment timing based on contract terms and jurisdiction.

### International Withholding

Cross-border royalty payments are subject to withholding tax. Rates depend on:
- **Tax treaties** between countries (US-UK treaty reduces withholding from 30% to 0-15%)
- **Payee type** (individual vs. corporation)
- **W-8BEN / W-8BEN-E** certification status

Airlock does NOT apply withholding -- the external ERP handles this. But Airlock must **display** the applicable withholding context in the statement so the ERP team knows what to apply.

### PCI DSS

**NOT applicable.** Airlock never touches payment card data. The intelligence-layer boundary keeps Airlock out of PCI scope entirely. This is a key benefit of the architecture decision.

### GDPR and Financial PII

Royalty rates, payment amounts, advance balances, and revenue figures are **financial PII** under GDPR:

- Encrypted at rest per cipher model (DEK/KEK envelope encryption)
- Right to erasure: crypto-shredding destroys all financial data for a vault
- Data portability: statements and invoices are exportable as PDF
- Access logging: all reads of financial data are logged

### Music-Specific Regulations

- **MLC (Mechanical Licensing Collective):** Handles blanket mechanical licenses for streaming. Airlock must track whether mechanical royalties flow through MLC or direct license.
- **SoundExchange:** Statutory digital performance royalties (non-interactive streaming, satellite radio). Separate from label royalties.
- **Harry Fox Agency (HFA):** Mechanical license administration (now part of SESAC). Legacy contracts may reference HFA rates.

### Audit Rights

Entertainment contracts almost universally grant audit rights -- the right to inspect the books. All calculations must be reproducible:

- Every `payment_schedule.calculation_detail_json` stores the full waterfall inputs and intermediate values.
- Revenue source data is immutable after import (corrections create new rows, not updates).
- Statement versions are preserved -- if a statement is revised, the original draft remains in history.

---

## 11. Security (per Cipher Model)

All financial data follows the principles in `/Features/Security/overview.md`. Additional finance-specific requirements:

### Encryption

| Data Type | Encryption | Notes |
|---|---|---|
| Obligation rates/amounts | AES-256-GCM (DEK/KEK) | Encrypted in `obligation.rate`, `obligation.amount` columns |
| Revenue figures | AES-256-GCM (DEK/KEK) | All `revenue_source` monetary columns |
| Statement totals | AES-256-GCM (DEK/KEK) | All `statement` monetary columns |
| Invoice amounts | AES-256-GCM (DEK/KEK) | All `invoice` monetary columns |
| Recoupment balances | AES-256-GCM (DEK/KEK) | `recoupment_ledger` monetary columns |
| Calculation breakdowns | AES-256-GCM (DEK/KEK) | `calculation_detail_json` encrypted as whole blob |
| Statement PDFs | Encrypted at rest in object storage | Same as Document Suite file encryption |

### Logging and Redaction

- **No plaintext financial amounts in application logs.** Log entries reference obligation/statement IDs, not dollar amounts.
- **Otto (AI Agent) financial PII redaction:** Before any financial data reaches LLM providers, monetary values are replaced with tokens: `$12,500.00` becomes `[AMOUNT_REDACTED]`, `15%` becomes `[RATE_REDACTED]`. Otto can answer "is the Henderson deal recouped?" (yes/no) but NOT "what is the Henderson advance balance?" (requires actual amount).
- **Audit trail entries** store hashed references to amounts, not plaintext. The full amounts are recoverable only through the decryption path.

### Access Control

| Action | Minimum Role | Notes |
|---|---|---|
| View obligations | Builder | Read-only, amounts visible if role permits |
| View statements | Builder | Read-only |
| Generate statement | Gatekeeper | Creates draft |
| Approve statement | Owner | Self-approval blocked |
| Approve invoice | Owner | Self-approval blocked |
| Import revenue data | Gatekeeper | Creates revenue_source rows |
| Configure ERP integration | Owner | Workspace-level setting |
| View recoupment balances | Builder | Scoped to assigned vaults |
| Override calculation | Owner | Creates audit event, requires evidence |

---

## 12. Triptych Layout

When viewing a vault with financial obligations, the Finance module renders the standard Airlock triptych:

### Signal Panel (Left, ~280px)

| Component | Content |
|---|---|
| **Obligation Events Feed** | Timeline of financial events: obligation created, calculation completed, statement generated, invoice sent, payment received |
| **Payment Notifications** | Upcoming due dates, overdue alerts, recoupment milestones |
| **ERP Sync Status** | Latest sync results per obligation (success/failure indicators) |
| **Revenue Import Alerts** | New revenue data available, import completed, discrepancies detected |

### Orchestrate Panel (Center, flex-1)

| View | Content |
|---|---|
| **Statement Viewer** | Rendered royalty statement with expandable calculation breakdowns |
| **Invoice Editor** | Invoice form with line items, due date, counterparty details |
| **Calculation Breakdown** | Interactive waterfall visualization: gross -> deductions -> net -> royalty -> recoupment -> payable |
| **Revenue Table** | Imported revenue data with filters by source, territory, period |
| **Obligation Summary** | All obligations for the vault in a compact table with status indicators |

### Control Panel (Right, ~300px)

| Component | Content |
|---|---|
| **Obligation Details** | Selected obligation card: type, rate, frequency, territory, status, clause reference |
| **Recoupment Status** | Visual progress bar: advance balance, amount recouped, percentage, projected recoup date |
| **ERP Sync Details** | Per-obligation sync status, last sync time, external reference links |
| **Escalation Tracker** | Current tier, units toward next threshold, projected escalation date |
| **Audit Trail** | Chronological list of all financial events for the selected obligation |

---

## 13. Feature Flags

All Finance capabilities are gated by Feature Control Plane flags (see `/Features/FeatureControlPlane/overview.md`):

| Flag | Default | Description | Dependencies |
|---|---|---|---|
| `ENABLE_OBLIGATION_ENGINE` | OFF | Extract obligations from contract clauses on Ship gate | Clause library, EventBus |
| `ENABLE_ROYALTY_CALCULATOR` | OFF | Run royalty waterfall calculations from imported revenue data | Obligation engine |
| `ENABLE_STATEMENT_GENERATION` | OFF | Generate quarterly royalty statements as PDFs | Royalty calculator, Document Suite |
| `ENABLE_INVOICE_GENERATION` | OFF | Create invoices from approved statements or standalone events | Statement generation (for royalty invoices) |
| `ENABLE_ERP_SYNC` | OFF | Push/pull/webhook integration with external financial systems | Invoice generation |
| `ENABLE_REVENUE_IMPORT` | OFF | Import revenue data from CSV or API sources | -- |
| `ENABLE_CROSS_COLLATERALIZATION` | OFF | Pool obligations across vaults for shared recoupment | Obligation engine |
| `ENABLE_ESCALATION_TRACKING` | OFF | Track cumulative units for royalty rate escalation | Royalty calculator |

### Calibration Parameters

| Parameter | Default | Range | Description |
|---|---|---|---|
| `finance.overdue_grace_days` | 15 | 0-90 | Days after due date before invoice escalates to task |
| `finance.statement_generation_delay_days` | 45 | 30-120 | Days after quarter end before auto-generating statements |
| `finance.recoupment_check_frequency` | quarterly | monthly/quarterly/semi_annual | How often recoupment balances are recalculated |
| `finance.revenue_import_dedup_window_days` | 90 | 30-365 | Window for checking duplicate revenue imports |
| `finance.mechanical_statutory_rate` | 0.12 | -- | Current US mechanical royalty rate (updated manually when CRB changes it) |
| `finance.erp_sync_retry_max` | 3 | 1-10 | Maximum retry attempts for failed ERP sync |

---

## 14. Implementation Phases

### Phase 1: Obligation Extraction (Read-Only)

**Goal:** When a contract reaches Ship, extract financial obligations and display them in the Control panel.

- Implement `obligation` table and RLS policies
- Build obligation extraction BullMQ job triggered by `CONTRACT_EXECUTED` event
- Map clause library financial variables to obligation records
- Display obligation cards in the Control panel (read-only)
- Feature flag: `ENABLE_OBLIGATION_ENGINE`

### Phase 2: Revenue Import + Royalty Calculation

**Goal:** Accept revenue data and compute royalty amounts.

- Implement `revenue_source` table
- Build CSV import pipeline (reuse Batch Processing patterns)
- Implement `payment_schedule` table
- Build royalty waterfall calculation engine (Steps 1-7)
- Implement `recoupment_ledger` table
- Display calculation breakdowns in Orchestrate panel
- Feature flags: `ENABLE_REVENUE_IMPORT`, `ENABLE_ROYALTY_CALCULATOR`

### Phase 3: Statement Generation + PDF Output

**Goal:** Generate quarterly royalty statements as PDFs.

- Implement `statement` table
- Build statement generation job (quarterly BullMQ scheduled job)
- Create statement TipTap template
- Implement statement approval workflow (Draft -> Reviewed -> Approved -> Sent)
- PDF rendering via Document Suite pipeline
- Feature flag: `ENABLE_STATEMENT_GENERATION`

### Phase 4: Invoice Workflow + Approval Chain

**Goal:** Create invoices from approved statements, track payment lifecycle.

- Implement `invoice` table
- Build invoice generation from approved statements
- Implement invoice numbering (workspace-aware sequences)
- Build invoice approval chain (reuse PatchWorkflow pattern)
- Implement overdue detection (daily BullMQ job)
- Task creation for escalated overdue invoices
- Feature flag: `ENABLE_INVOICE_GENERATION`

### Phase 5: ERP Integration

**Goal:** Push invoices to external systems, receive payment confirmations.

- Build ERP integration framework (adapter pattern per system)
- Implement QuickBooks Online integration (OAuth 2.0 + Invoice API)
- Implement Hiperweb integration (API key + REST)
- Build webhook receiver for payment confirmations
- Implement sync status tracking and retry logic
- Feature flag: `ENABLE_ERP_SYNC`

### Phase 6: Advanced Calculations (Post-MVP)

**Goal:** Handle complex entertainment-specific scenarios.

- Cross-collateralization engine (`ENABLE_CROSS_COLLATERALIZATION`)
- Escalation tier tracking (`ENABLE_ESCALATION_TRACKING`)
- Territory split engine with proportional allocation
- Format tier rate application
- Producer points with retroactive recoupment
- Minimum guarantee shortfall reporting

---

## 15. Open Questions

| # | Question | Options | Leaning | Impact |
|---|---|---|---|---|
| 1 | Should Finance be a standalone Module (6th icon in ModuleBar) or a View within Contracts? | **A)** Standalone Module -- own icon, own sub-panel, cross-vault financial views. **B)** View within Contracts -- financial tab on each vault, no separate module. | **A (Standalone)** -- financial views span multiple vaults (cross-collateralization, portfolio reporting), which does not fit inside a single contract vault. | High: affects Shell layout, Module Bar, sub-panel design |
| 2 | How do we handle multi-vault obligations (compilation albums, cross-collateralized deals)? | **A)** Parent obligation record that links child obligations across vaults. **B)** Separate "Deal" entity above the vault level. | **A** -- uses `parent_obligation_id` in the data model, avoids new entity types. | Medium: affects recoupment engine complexity |
| 3 | What is the minimum viable revenue import format? | **A)** CSV only. **B)** CSV + simple API. **C)** CSV + API + SFTP. | **A (CSV only)** for Phase 2. API in Phase 5. | Low: additive, does not block other features |
| 4 | Should Otto have read access to financial data? | **A)** No financial data access (full redaction). **B)** Yes/no answers about status (recouped? overdue?) but no amounts. **C)** Full read access with PII redaction in prompts. | **B** -- Otto can answer status questions but never reveal dollar amounts to LLM providers. | Medium: affects AI Agent system prompt and tool permissions |
| 5 | Should statements be sent directly from Airlock (email) or only generated for download/ERP push? | **A)** Airlock sends email with PDF attachment. **B)** PDF generated, user downloads or ERP pushes. **C)** Both, configurable. | **C** -- some customers want direct email delivery, others want ERP-only. | Low: additive feature |
| 6 | How do we handle currency conversion for international deals? | **A)** All amounts stored in contract currency, conversion at display time. **B)** Dual storage (original + USD equivalent). **C)** Delegate entirely to ERP. | **A** -- store in contract currency, display conversion is a UI concern. ERP handles settlement conversion. | Medium: affects all monetary columns |
| 7 | Should Finance have its own Chambers or reuse the Contracts module lifecycle? | **A)** Own chambers (Discover=obligations, Build=calculations, Review=statements, Ship=invoiced). **B)** No chambers -- flat view with tabs. | **A** -- chambers map naturally to the financial lifecycle. | Medium: affects sub-panel design |
| 8 | What happens when a contract is amended (PatchWorkflow) and financial terms change? | **A)** Close old obligations, create new ones from the amendment. **B)** Version obligations in place. **C)** Hybrid -- terminate old, create new, link them. | **C** -- old obligation terminated with effective date, new obligation created with amendment reference, audit trail links them. | High: affects obligation lifecycle and historical reporting |

---

## Related Specs

- [Contract Generator](../ContractGenerator/overview.md) -- Clause library with financial variables (`OPP_ROYALTY_RATE`, `OPP_ADVANCE_AMOUNT`, etc.)
- [Patch Workflow](../PatchWorkflow/overview.md) -- Approval chain pattern reused for statement/invoice approval; amendment workflow affects obligation lifecycle
- [Batch Processing](../BatchProcessing/overview.md) -- Pipeline pattern reused for revenue import; BullMQ job architecture
- [Event Bus](../EventBus/overview.md) -- `CONTRACT_EXECUTED` event triggers obligation extraction
- [Security](../Security/overview.md) -- Cipher model for financial PII encryption, append-only audit trail
- [Document Suite](../DocumentSuite/overview.md) -- PDF generation for statements and invoices (TipTap template + PDF.js rendering)
- [Feature Control Plane](../FeatureControlPlane/overview.md) -- Feature flags and calibration parameters for all Finance capabilities
- [Roles & Permissions](../Roles/overview.md) -- Owner role required for statement/invoice approval
- [Shell / Triptych](../Shell/triptych.md) -- Signal | Orchestrate | Control panel layout
- [Universal Task System](../TaskSystem/overview.md) -- Overdue invoice escalation creates tasks
- [CRM Module](../CRM/overview.md) -- Vault hierarchy provides counterparty context for statements/invoices
- [Workflow Engine](../WorkflowEngine/overview.md) -- Potential automation: auto-generate statements, auto-escalate overdue invoices
- [AI Agent](../AIAgent/overview.md) -- Otto financial data access policy (status queries only, no amounts)
