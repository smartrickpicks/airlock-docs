# CRM Data Model -- Design Spec

> **Status:** SPECCED
> **Source:** [crm-views-design-spec.md](./crm-views-design-spec.md), [overview.md](./overview.md), [crm-comms-brainstorm.md](./crm-comms-brainstorm.md), [VaultHierarchy/overview.md](../VaultHierarchy/overview.md), [TaskSystem/overview.md](../TaskSystem/overview.md)

## Summary

This document defines the complete data model for the Airlock CRM module. The vault hierarchy IS the CRM -- there are no separate CRM tables for accounts, divisions, or counterparties. Instead, the `vaults` table (levels 1-4) stores all relationship data, and the CRM module provides a lens over that tree. This spec covers: (1) how each vault level maps to CRM entities, (2) supporting tables for contacts, communications, and activity events, (3) junction tables for relationship mappings, (4) computed fields and aggregation logic, (5) index strategy for every CRM view, and (6) encryption boundaries per the Airlock security architecture.

---

## 1. Core Entities

The CRM does not create its own entity tables. All four core CRM entities are rows in the shared `vaults` table, differentiated by `vault_level`. CRM-specific fields live in the `metadata` JSONB column and in dedicated supporting tables where query performance or encryption requirements demand it.

### 1.1 Account (Vault Level 1 -- Parent)

An Account is a top-level legal entity or corporate group. It is the root of a vault subtree.

**Storage:** `vaults` table row with `vault_level = 1`.

| Field | Column / Path | Type | Required | Default | Validation | Description |
|-------|--------------|------|----------|---------|------------|-------------|
| id | `vaults.id` | UUID | Yes | `gen_random_uuid()` | PK | Unique account identifier |
| workspace_id | `vaults.workspace_id` | UUID | Yes | -- | FK to `workspaces.id` | Tenant isolation key |
| name | `vaults.name` | TEXT | Yes | -- | Non-empty, max 255 chars | Legal entity name (e.g., "Acme Inc") |
| slug | `vaults.slug` | TEXT | Yes | -- | Unique per workspace, URL-safe, max 100 chars | URL-friendly identifier (e.g., "acme-inc") |
| vault_level | `vaults.vault_level` | INT | Yes | 1 | Must be `1` | Identifies this as an Account |
| vault_type | `vaults.vault_type` | TEXT | Yes | `'entity'` | One of: `entity` | Vault type discriminator |
| module_type | `vaults.module_type` | TEXT | No | `NULL` | -- | NULL for org-level vaults (visible across modules) |
| industry | `metadata.industry` | TEXT | No | `NULL` | Free text, max 100 chars | Industry vertical (e.g., "Music", "Film", "Technology") |
| segment | `metadata.segment` | ENUM (TEXT) | No | `NULL` | One of: `Enterprise`, `Mid-Market`, `SMB` | Market segment classification |
| health_score | `vaults.health_score` | FLOAT | No | `NULL` | Range 0-100, computed | Weighted aggregate health of all child vaults (see Section 4) |
| smart_line_number | `metadata.smart_line_number` | TEXT | No | `NULL` | E.164 format (e.g., `+13105550142`) | Dedicated Smart Line phone number for this account |
| smart_line_status | `metadata.smart_line_status` | ENUM (TEXT) | No | `'not_provisioned'` | One of: `active`, `suspended`, `not_provisioned` | Current status of the Smart Line |
| address | `metadata.address` | JSON | No | `NULL` | Object: `{street, city, state, zip, country}` | **Encrypted at rest** -- physical/mailing address |
| website | `metadata.website` | TEXT | No | `NULL` | Valid URL format | Company website |
| logo_url | `metadata.logo_url` | TEXT | No | `NULL` | Valid URL format | Path to company logo image |
| owner_id | `metadata.owner_id` | UUID | No | `NULL` | FK to `users.id` | Assigned account owner (relationship manager) |
| annual_revenue | `metadata.annual_revenue` | DECIMAL | No | `NULL` | >= 0 | Estimated annual revenue for segmentation |
| employee_count | `metadata.employee_count` | INT | No | `NULL` | >= 0 | Number of employees for firmographic scoring |
| timezone | `metadata.timezone` | TEXT | No | `NULL` | IANA timezone string (e.g., `America/Los_Angeles`) | Account's primary timezone for SLA and scheduling |
| source | `metadata.source` | ENUM (TEXT) | No | `NULL` | One of: `Manual`, `Entity_Resolution`, `Contract_Import`, `Web_Form`, `Smart_Line`, `Referral`, `API` | How this account entered the system |
| renewal_date | `metadata.renewal_date` | DATE | No | `NULL` | Valid date | Next renewal date (aggregated or manually set) |
| notes | `metadata.notes` | TEXT | No | `NULL` | Max 10,000 chars | Internal notes about the account |
| tags | `metadata.tags` | TEXT[] | No | `[]` | Array of strings, each max 50 chars | Freeform tags (e.g., `['Enterprise', 'Music', 'Priority']`) |
| created_at | `vaults.created_at` | TIMESTAMPTZ | Yes | `now()` | -- | Account creation timestamp |
| updated_at | `vaults.updated_at` | TIMESTAMPTZ | Yes | `now()` | Auto-updated on change | Last modification timestamp |
| archived_at | `vaults.archived_at` | TIMESTAMPTZ | No | `NULL` | -- | Soft-delete timestamp (NULL = active) |
| metadata | `vaults.metadata` | JSONB | Yes | `'{}'` | Valid JSON | Extensible bag for fields not enumerated above |

**Notes:**
- `health_score` is computed, never set directly. See Section 4 for the formula.
- `owner_id` in metadata is separate from `vault_members` -- it designates the primary relationship owner for CRM views. The owner must also exist in `vault_members` with role `'owner'` or `'gatekeeper'`.
- `address` is encrypted at rest because it contains PII. The encryption middleware decrypts it at the application layer before serving to the frontend. See Section 6.

---

### 1.2 Division (Vault Level 2)

A Division is an acquired company, subsidiary, business unit, label, or regional office nested under an Account. Optional level -- many accounts skip directly from Account to Counterparty.

**Storage:** `vaults` table row with `vault_level = 2`.

| Field | Column / Path | Type | Required | Default | Validation | Description |
|-------|--------------|------|----------|---------|------------|-------------|
| id | `vaults.id` | UUID | Yes | `gen_random_uuid()` | PK | Unique division identifier |
| workspace_id | `vaults.workspace_id` | UUID | Yes | -- | FK to `workspaces.id` | Tenant isolation key |
| parent_vault_id | `vaults.parent_vault_id` | UUID | Yes | -- | FK to `vaults.id`, must reference a vault_level=1 | Parent Account |
| name | `vaults.name` | TEXT | Yes | -- | Non-empty, max 255 chars | Division name (e.g., "Big Booty Inc") |
| slug | `vaults.slug` | TEXT | Yes | -- | Unique per workspace, URL-safe | URL-friendly identifier |
| vault_level | `vaults.vault_level` | INT | Yes | 2 | Must be `2` | Identifies this as a Division |
| vault_type | `vaults.vault_type` | TEXT | Yes | `'division'` | One of: `division` | Vault type discriminator |
| division_type | `metadata.division_type` | ENUM (TEXT) | No | `'Custom'` | One of: `Region`, `Department`, `Brand`, `Label`, `Subsidiary`, `Custom` | Type of organizational subdivision |
| health_score | `vaults.health_score` | FLOAT | No | `NULL` | Range 0-100, computed | Weighted aggregate of child vaults |
| address | `metadata.address` | JSON | No | `NULL` | Object: `{street, city, state, zip, country}` | **Encrypted at rest** -- division office address |
| head_contact_id | `metadata.head_contact_id` | UUID | No | `NULL` | FK to `vault_contacts.id` | Primary point of contact for this division |
| owner_id | `metadata.owner_id` | UUID | No | `NULL` | FK to `users.id` | Relationship owner (inherits from parent if NULL) |
| tags | `metadata.tags` | TEXT[] | No | `[]` | Array of strings | Freeform tags |
| created_at | `vaults.created_at` | TIMESTAMPTZ | Yes | `now()` | -- | Creation timestamp |
| updated_at | `vaults.updated_at` | TIMESTAMPTZ | Yes | `now()` | -- | Last modification timestamp |
| archived_at | `vaults.archived_at` | TIMESTAMPTZ | No | `NULL` | -- | Soft-delete timestamp |
| metadata | `vaults.metadata` | JSONB | Yes | `'{}'` | Valid JSON | Extensible metadata |

**Notes:**
- Divisions are optional. A flat organization can go directly from Account (L1) to Counterparty (L3) by setting the Counterparty's `parent_vault_id` to the Account's `id`.
- When an Account is acquired by another Account, the acquired Account is re-leveled to a Division (vault_level changes from 1 to 2, parent_vault_id set to acquirer). All descendants move with it. This operation is logged in the audit trail.

---

### 1.3 Counterparty (Vault Level 3)

A Counterparty represents an external business relationship -- a customer, prospect, partner, or vendor. Created automatically by entity resolution during contract ingestion or manually through the CRM.

**Storage:** `vaults` table row with `vault_level = 3`.

| Field | Column / Path | Type | Required | Default | Validation | Description |
|-------|--------------|------|----------|---------|------------|-------------|
| id | `vaults.id` | UUID | Yes | `gen_random_uuid()` | PK | Unique counterparty identifier |
| workspace_id | `vaults.workspace_id` | UUID | Yes | -- | FK to `workspaces.id` | Tenant isolation key |
| parent_vault_id | `vaults.parent_vault_id` | UUID | Yes | -- | FK to `vaults.id`, must reference vault_level IN (1,2) | Parent Account or Division |
| name | `vaults.name` | TEXT | Yes | -- | Non-empty, max 255 chars | Counterparty legal name (e.g., "Sony Music") |
| slug | `vaults.slug` | TEXT | Yes | -- | Unique per workspace, URL-safe | URL-friendly identifier |
| vault_level | `vaults.vault_level` | INT | Yes | 3 | Must be `3` | Identifies this as a Counterparty |
| vault_type | `vaults.vault_type` | TEXT | Yes | `'counterparty'` | One of: `counterparty` | Vault type discriminator |
| entity_resolution_confidence | `metadata.entity_resolution_confidence` | FLOAT | No | `NULL` | Range 0.0 - 1.0 | Confidence score from entity resolution. Below 0.40 = unqualified, 0.40-0.80 = ambiguous, above 0.80 = high confidence, 1.0 = exact match |
| primary_contact_id | `metadata.primary_contact_id` | UUID | No | `NULL` | FK to `vault_contacts.id` | Primary point of contact for this counterparty |
| relationship_type | `metadata.relationship_type` | ENUM (TEXT) | No | `'Prospect'` | One of: `Customer`, `Prospect`, `Partner`, `Vendor`, `Competitor` | Nature of the business relationship |
| lifecycle_stage | `metadata.lifecycle_stage` | ENUM (TEXT) | No | `'Lead'` | One of: `Lead`, `MQL`, `SAL`, `SQL`, `Customer`, `Churned`, `Evangelist` | Current lifecycle stage of the relationship |
| first_interaction | `metadata.first_interaction` | TIMESTAMPTZ | No | `NULL` | -- | Timestamp of the first recorded interaction |
| last_interaction | `metadata.last_interaction` | TIMESTAMPTZ | No | `NULL` | -- | Timestamp of the most recent interaction (auto-updated) |
| health_score | `vaults.health_score` | FLOAT | No | `NULL` | Range 0-100, computed | Aggregate health of child item vaults |
| deal_count | -- (computed) | INT | -- | -- | -- | COUNT of active child item vaults (not stored, computed at query time) |
| total_value | -- (computed) | DECIMAL | -- | -- | -- | SUM of child deal values (not stored, computed at query time) |
| lead_source | `metadata.lead_source` | ENUM (TEXT) | No | `NULL` | One of: `Manual`, `Web_Form`, `Smart_Line`, `Referral`, `LinkedIn`, `Cold_Outbound`, `Contract_Import`, `Entity_Resolution` | How this counterparty was first identified |
| lead_score | `metadata.lead_score` | INT | No | `NULL` | Range 0-100 | Qualification score (only for prospects, NULL once converted to Customer) |
| lead_score_breakdown | `metadata.lead_score_breakdown` | JSON | No | `NULL` | Object: `{engagement: int, fit: int, behavior: int}` | Component scores of lead qualification |
| qualification_notes | `metadata.qualification_notes` | TEXT | No | `NULL` | Max 5,000 chars | Notes from qualification process |
| tags | `metadata.tags` | TEXT[] | No | `[]` | Array of strings | Freeform tags |
| created_at | `vaults.created_at` | TIMESTAMPTZ | Yes | `now()` | -- | Creation timestamp |
| updated_at | `vaults.updated_at` | TIMESTAMPTZ | Yes | `now()` | -- | Last modification timestamp |
| archived_at | `vaults.archived_at` | TIMESTAMPTZ | No | `NULL` | -- | Soft-delete timestamp |
| metadata | `vaults.metadata` | JSONB | Yes | `'{}'` | Valid JSON | Extensible metadata |

**Notes:**
- `lifecycle_stage` tracks the lead-to-customer lifecycle. In Airlock, there is no hard "lead conversion" boundary like Salesforce -- the same vault row simply changes its `lifecycle_stage` from `SQL` to `Customer`. No object migration, no history fragmentation.
- `entity_resolution_confidence` below 0.40 causes the counterparty to appear in the CRM's "Unqualified" view under the Discover chamber. See [overview.md](./overview.md) for the Discover > Unqualified view spec.
- `last_interaction` is auto-updated by a database trigger whenever a new `communications` or `activity_events` row is inserted referencing this counterparty's ID (via `account_id` lookup through the vault hierarchy).

---

### 1.4 Item / Deal (Vault Level 4)

An Item vault is the work unit -- a specific contract, deal, or document. Only item vaults progress through the chamber lifecycle (Discover > Build > Review > Ship). In the CRM context, an Item vault represents a **deal** in the pipeline.

**Storage:** `vaults` table row with `vault_level = 4`.

| Field | Column / Path | Type | Required | Default | Validation | Description |
|-------|--------------|------|----------|---------|------------|-------------|
| id | `vaults.id` | UUID | Yes | `gen_random_uuid()` | PK | Unique deal identifier |
| workspace_id | `vaults.workspace_id` | UUID | Yes | -- | FK to `workspaces.id` | Tenant isolation key |
| parent_vault_id | `vaults.parent_vault_id` | UUID | Yes | -- | FK to `vaults.id`, must reference vault_level=3 (or 1/2 if no counterparty) | Parent Counterparty (or Account/Division) |
| name | `vaults.name` | TEXT | Yes | -- | Non-empty, max 255 chars | Deal name (e.g., "Sony-BigBooty Distribution Agreement Q1-2026") |
| slug | `vaults.slug` | TEXT | Yes | -- | Unique per workspace, URL-safe | URL-friendly identifier |
| vault_level | `vaults.vault_level` | INT | Yes | 4 | Must be `4` | Identifies this as an Item/Deal |
| vault_type | `vaults.vault_type` | TEXT | Yes | -- | One of: `contract`, `task`, `document` | What type of item this vault holds |
| module_type | `vaults.module_type` | TEXT | No | `NULL` | One of: `contracts`, `crm`, `tasks`, `documents` | Which module owns this vault |
| chamber | `vaults.chamber` | TEXT | Yes | `'discover'` | One of: `discover`, `build`, `review`, `ship` | Current lifecycle chamber |
| gate | `vaults.gate` | TEXT | No | `NULL` | Gate identifier within chamber | Current gate checkpoint |
| value | `metadata.value` | DECIMAL | No | `NULL` | >= 0, precision 2 | Deal monetary value |
| currency | `metadata.currency` | TEXT | No | `'USD'` | ISO 4217 currency code (e.g., `USD`, `EUR`, `GBP`) | Currency of the deal value |
| stage | `metadata.stage` | ENUM (TEXT) | No | `NULL` | One of: `Prospecting`, `Discovery`, `Proposal`, `Negotiation`, `Close` | Pipeline stage (CRM-specific; maps to but is distinct from `chamber`) |
| probability | `metadata.probability` | INT | No | `NULL` | Range 0-100, auto-set by stage | Win probability percentage |
| close_date | `metadata.close_date` | DATE | No | `NULL` | Valid date, must be >= created_at date | Expected close date |
| actual_close_date | `metadata.actual_close_date` | DATE | No | `NULL` | Valid date | Date the deal actually closed (set on Won or Lost) |
| owner_id | `metadata.owner_id` | UUID | No | `NULL` | FK to `users.id` | Deal owner / assigned sales rep |
| created_by | `metadata.created_by` | UUID | No | `NULL` | FK to `users.id` | User who created this deal |
| source | `metadata.source` | ENUM (TEXT) | No | `'Manual'` | One of: `Manual`, `Web_Form`, `Smart_Line`, `Referral`, `LinkedIn`, `Cold_Outbound`, `Contract_Import` | How this deal entered the system |
| lifecycle_status | `metadata.lifecycle_status` | ENUM (TEXT) | Yes | `'Active'` | One of: `Active`, `Won`, `Lost`, `Abandoned` | Current deal outcome status |
| loss_reason | `metadata.loss_reason` | TEXT | No | `NULL` | Max 500 chars; only valid when lifecycle_status = `Lost` | Why the deal was lost |
| loss_category | `metadata.loss_category` | ENUM (TEXT) | No | `NULL` | One of: `Price`, `Competitor`, `No_Decision`, `Timing`, `Fit`, `Internal`, `Other`; only valid when lifecycle_status = `Lost` | Categorized loss reason for analytics |
| gate_blocked | `metadata.gate_blocked` | BOOLEAN | No | `false` | -- | Whether the deal is blocked at a gate checkpoint |
| gate_message | `metadata.gate_message` | TEXT | No | `NULL` | Max 500 chars; only relevant when gate_blocked = true | Human-readable reason the gate is blocked |
| last_stage_change_at | `metadata.last_stage_change_at` | TIMESTAMPTZ | No | `NULL` | -- | When the deal last changed pipeline stage (used to compute `days_in_stage`) |
| days_in_stage | -- (computed) | INT | -- | -- | -- | `EXTRACT(DAY FROM now() - last_stage_change_at)`. Not stored; computed at query time |
| task_count | -- (computed) | INT | -- | -- | -- | COUNT of linked tasks. Not stored; computed via `deal_tasks` join |
| task_progress | -- (computed) | FLOAT | -- | -- | -- | `completed_tasks / total_tasks * 100`. Computed at query time |
| health_score | `vaults.health_score` | FLOAT | No | `NULL` | Range 0-100 | Direct health score for this deal (not aggregated) |
| tags | `metadata.tags` | TEXT[] | No | `[]` | Array of strings | Freeform tags |
| created_at | `vaults.created_at` | TIMESTAMPTZ | Yes | `now()` | -- | Deal creation timestamp |
| updated_at | `vaults.updated_at` | TIMESTAMPTZ | Yes | `now()` | -- | Last modification timestamp |
| archived_at | `vaults.archived_at` | TIMESTAMPTZ | No | `NULL` | -- | Soft-delete timestamp |
| metadata | `vaults.metadata` | JSONB | Yes | `'{}'` | Valid JSON | Extensible metadata |

**Stage-to-Probability Mapping (auto-set when stage changes):**

| Stage | Default Probability | Description |
|-------|-------------------|-------------|
| Prospecting | 10% | Initial outreach, no confirmed interest |
| Discovery | 25% | Qualified interest, exploring requirements |
| Proposal | 50% | Proposal sent, awaiting response |
| Negotiation | 75% | Active negotiation on terms |
| Close | 90% | Verbal agreement, finalizing paperwork |

Probability auto-sets when `stage` is updated but can be manually overridden. Manual overrides are tracked in the audit trail.

**Stage-to-Chamber Mapping:**

The `stage` field (CRM pipeline) and `chamber` field (Airlock lifecycle) are related but distinct. The stage is CRM-specific terminology; the chamber is the universal Airlock lifecycle. When a deal is created from the CRM, both are set:

| CRM Stage | Airlock Chamber | Rationale |
|-----------|----------------|-----------|
| Prospecting | Discover | Early exploration |
| Discovery | Discover | Still qualifying |
| Proposal | Build | Actively constructing the deal |
| Negotiation | Review | Terms under review |
| Close | Ship | Finalizing and delivering |

When a deal enters via contract import (Contracts module), the `chamber` is set by the extraction pipeline, and the `stage` is inferred from the chamber.

---

## 2. Supporting Entities

These tables are NOT part of the vault hierarchy. They are dedicated tables that store data requiring specific indexing, encryption, or query patterns that JSONB metadata cannot efficiently support.

### 2.1 Contact (`vault_contacts` table)

Contacts are people associated with accounts. They are stored in a dedicated table (not as vaults) because: (a) they need efficient text search on name/email, (b) a single contact can appear across multiple accounts, and (c) contact PII requires column-level encryption.

```sql
CREATE TABLE vault_contacts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id),
    account_id UUID NOT NULL REFERENCES vaults(id),  -- FK to Account (L1) or Division (L2) vault
    counterparty_id UUID REFERENCES vaults(id),       -- FK to Counterparty (L3) vault, nullable

    -- Identity
    name TEXT NOT NULL,                                -- Needed for search (plaintext, but redacted before LLM)
    role TEXT,                                         -- Job title / role (e.g., "VP Business Development")
    department TEXT,                                   -- Department (e.g., "Legal", "Finance", "Operations")
    email TEXT,                                        -- **Encrypted at rest** (PII)
    phone TEXT,                                        -- **Encrypted at rest** (PII)
    linkedin_url TEXT,                                 -- LinkedIn profile URL

    -- Source & Classification
    source TEXT NOT NULL DEFAULT 'Manual',             -- 'Manual' | 'Inbound' | 'Entity_Resolution' | 'Import' | 'Smart_Line' | 'Web_Form'
    lifecycle_stage TEXT NOT NULL DEFAULT 'Other',     -- 'Subscriber' | 'Lead' | 'MQL' | 'SAL' | 'SQL' | 'Customer' | 'Evangelist' | 'Other'
    is_primary BOOLEAN NOT NULL DEFAULT false,         -- Primary contact for the account/counterparty
    is_decision_maker BOOLEAN NOT NULL DEFAULT false,  -- Identified as key decision maker

    -- Lead Scoring (only for leads, NULL for established contacts)
    lead_score INT,                                    -- Range 0-100, NULL if not a lead
    lead_score_breakdown JSONB,                        -- {engagement: int, fit: int, behavior: int}

    -- Interaction Tracking
    interaction_count INT NOT NULL DEFAULT 0,          -- COUNT of communications (auto-incremented)
    last_interaction TIMESTAMPTZ,                      -- Timestamp of most recent communication

    -- Tags & Notes
    tags TEXT[] NOT NULL DEFAULT '{}',                 -- e.g., ['Enterprise', 'Renewal', 'Technical']
    notes TEXT,                                        -- Internal notes about this contact

    -- Metadata
    metadata JSONB NOT NULL DEFAULT '{}',              -- Extensible (social profiles, preferences, etc.)

    -- Timestamps
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Full Field Reference:**

| Field | Type | Required | Default | Validation | Description |
|-------|------|----------|---------|------------|-------------|
| id | UUID | Yes | `gen_random_uuid()` | PK | Unique contact identifier |
| workspace_id | UUID | Yes | -- | FK to `workspaces.id` | Tenant isolation key |
| account_id | UUID | Yes | -- | FK to `vaults.id` (L1 or L2) | Which account this contact belongs to |
| counterparty_id | UUID | No | `NULL` | FK to `vaults.id` (L3) | Specific counterparty relationship (nullable) |
| name | TEXT | Yes | -- | Non-empty, max 255 chars | Full name |
| role | TEXT | No | `NULL` | Max 100 chars | Job title or role |
| department | TEXT | No | `NULL` | Max 100 chars | Department name |
| email | TEXT | No | `NULL` | Valid email format, **encrypted at rest** | Email address (PII) |
| phone | TEXT | No | `NULL` | E.164 format, **encrypted at rest** | Phone number (PII) |
| linkedin_url | TEXT | No | `NULL` | Valid URL | LinkedIn profile |
| source | ENUM (TEXT) | Yes | `'Manual'` | One of: `Manual`, `Inbound`, `Entity_Resolution`, `Import`, `Smart_Line`, `Web_Form` | How this contact was discovered |
| lifecycle_stage | ENUM (TEXT) | Yes | `'Other'` | One of: `Subscriber`, `Lead`, `MQL`, `SAL`, `SQL`, `Customer`, `Evangelist`, `Other` | Contact's position in the sales funnel |
| is_primary | BOOLEAN | Yes | `false` | -- | Designates the primary contact for the parent account |
| is_decision_maker | BOOLEAN | Yes | `false` | -- | Identified as a key decision maker |
| lead_score | INT | No | `NULL` | Range 0-100, only valid when lifecycle_stage IN (`Lead`, `MQL`, `SAL`, `SQL`) | Qualification score |
| lead_score_breakdown | JSONB | No | `NULL` | `{engagement: int, fit: int, behavior: int}`, each 0-100 | Score breakdown by component |
| interaction_count | INT | Yes | `0` | >= 0 | Auto-incremented COUNT of communications |
| last_interaction | TIMESTAMPTZ | No | `NULL` | -- | Auto-updated on new communication |
| tags | TEXT[] | Yes | `'{}'` | Each tag max 50 chars | Freeform tags for filtering |
| notes | TEXT | No | `NULL` | Max 10,000 chars | Internal notes |
| metadata | JSONB | Yes | `'{}'` | Valid JSON | Extensible metadata |
| created_at | TIMESTAMPTZ | Yes | `now()` | -- | Creation timestamp |
| updated_at | TIMESTAMPTZ | Yes | `now()` | -- | Last modification timestamp |

**Indexes:**

```sql
CREATE INDEX idx_contacts_workspace ON vault_contacts(workspace_id);
CREATE INDEX idx_contacts_account ON vault_contacts(account_id);
CREATE INDEX idx_contacts_counterparty ON vault_contacts(counterparty_id) WHERE counterparty_id IS NOT NULL;
CREATE INDEX idx_contacts_name_search ON vault_contacts USING gin(to_tsvector('english', name));
CREATE INDEX idx_contacts_lifecycle ON vault_contacts(workspace_id, lifecycle_stage);
CREATE INDEX idx_contacts_lead_score ON vault_contacts(workspace_id, lead_score DESC) WHERE lead_score IS NOT NULL;
CREATE INDEX idx_contacts_primary ON vault_contacts(account_id) WHERE is_primary = true;
```

---

### 2.2 Communication (`communications` table)

Communications store all message interactions -- iMessage, SMS, email, calls, web forms, and meetings. The Smart Line architecture (see [crm-comms-brainstorm.md](./crm-comms-brainstorm.md)) feeds directly into this table.

```sql
CREATE TABLE communications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id),

    -- Relationships
    contact_id UUID REFERENCES vault_contacts(id),     -- Who sent/received (nullable for unknown contacts)
    deal_id UUID REFERENCES vaults(id),                -- Linked deal/item vault (nullable)
    account_id UUID NOT NULL REFERENCES vaults(id),    -- Parent account vault (always required)

    -- Message Identity
    channel TEXT NOT NULL,                              -- 'imessage' | 'sms' | 'email' | 'call' | 'web_form' | 'meeting'
    direction TEXT NOT NULL,                            -- 'inbound' | 'outbound'

    -- Content
    content_preview TEXT,                               -- Truncated to 200 chars, plaintext (for list views)
    content_encrypted TEXT NOT NULL,                    -- **Always encrypted** -- full message body as ciphertext
    subject TEXT,                                       -- Email subject line (NULL for non-email channels)

    -- AI Classification
    ai_intent TEXT,                                     -- e.g., "deal_discussion", "billing_inquiry", "renewal_request", "support_request"
    ai_intent_confidence FLOAT,                         -- Range 0.0 - 1.0
    ai_summary TEXT,                                    -- AI-generated one-line summary (for activity feed)

    -- Status
    read BOOLEAN NOT NULL DEFAULT false,                -- Has this been read by the assigned user
    replied BOOLEAN NOT NULL DEFAULT false,             -- Has a reply been sent
    archived BOOLEAN NOT NULL DEFAULT false,            -- Removed from inbox but not deleted

    -- Routing
    smart_line_id TEXT,                                 -- Which Smart Line number handled this
    thread_id TEXT,                                     -- Conversation thread identifier (groups related messages)
    assigned_to UUID REFERENCES users(id),             -- User assigned to handle this communication

    -- Call-Specific
    call_duration_seconds INT,                          -- Duration for call channel (NULL for non-calls)
    call_recording_url TEXT,                            -- Path to encrypted call recording (NULL for non-calls)
    voicemail_transcription TEXT,                       -- Transcription of voicemail (NULL if no voicemail)

    -- Metadata
    metadata JSONB NOT NULL DEFAULT '{}',              -- Channel-specific data (email headers, form fields, etc.)

    -- Timestamps
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()       -- When this communication occurred
);
```

**Full Field Reference:**

| Field | Type | Required | Default | Validation | Description |
|-------|------|----------|---------|------------|-------------|
| id | UUID | Yes | `gen_random_uuid()` | PK | Unique communication identifier |
| workspace_id | UUID | Yes | -- | FK to `workspaces.id` | Tenant isolation key |
| contact_id | UUID | No | `NULL` | FK to `vault_contacts.id` | Linked contact (NULL for unknown senders) |
| deal_id | UUID | No | `NULL` | FK to `vaults.id` (L4) | Linked deal, if communication is deal-specific |
| account_id | UUID | Yes | -- | FK to `vaults.id` (L1) | Parent account (required for inbox filtering) |
| channel | ENUM (TEXT) | Yes | -- | One of: `imessage`, `sms`, `email`, `call`, `web_form`, `meeting` | Communication channel |
| direction | ENUM (TEXT) | Yes | -- | One of: `inbound`, `outbound` | Message direction relative to the workspace |
| content_preview | TEXT | No | `NULL` | Max 200 chars, plaintext | Truncated preview for list views (not encrypted -- contains no full PII) |
| content_encrypted | TEXT | Yes | -- | Ciphertext | **Always encrypted at rest** -- full message body |
| subject | TEXT | No | `NULL` | Max 500 chars | Email subject line (NULL for non-email) |
| ai_intent | TEXT | No | `NULL` | Max 100 chars | AI-classified intent category |
| ai_intent_confidence | FLOAT | No | `NULL` | Range 0.0 - 1.0 | Confidence of intent classification |
| ai_summary | TEXT | No | `NULL` | Max 200 chars | AI-generated one-line summary |
| read | BOOLEAN | Yes | `false` | -- | Read status |
| replied | BOOLEAN | Yes | `false` | -- | Whether a reply has been sent |
| archived | BOOLEAN | Yes | `false` | -- | Archived (removed from inbox, not deleted) |
| smart_line_id | TEXT | No | `NULL` | E.164 format | Which Smart Line handled this |
| thread_id | TEXT | No | `NULL` | Max 100 chars | Groups messages into conversation threads |
| assigned_to | UUID | No | `NULL` | FK to `users.id` | User responsible for handling this |
| call_duration_seconds | INT | No | `NULL` | >= 0, only valid for channel = `call` | Call duration |
| call_recording_url | TEXT | No | `NULL` | Valid path, only valid for channel = `call` | Encrypted recording path |
| voicemail_transcription | TEXT | No | `NULL` | Only valid for channel = `call` | Voicemail text |
| metadata | JSONB | Yes | `'{}'` | Valid JSON | Channel-specific data |
| created_at | TIMESTAMPTZ | Yes | `now()` | -- | Communication timestamp |

**Indexes:**

```sql
CREATE INDEX idx_comms_workspace ON communications(workspace_id, created_at DESC);
CREATE INDEX idx_comms_account ON communications(account_id, created_at DESC);
CREATE INDEX idx_comms_contact ON communications(contact_id, created_at DESC) WHERE contact_id IS NOT NULL;
CREATE INDEX idx_comms_deal ON communications(deal_id, created_at DESC) WHERE deal_id IS NOT NULL;
CREATE INDEX idx_comms_inbox ON communications(workspace_id, channel, read, created_at DESC) WHERE archived = false;
CREATE INDEX idx_comms_thread ON communications(thread_id, created_at ASC) WHERE thread_id IS NOT NULL;
CREATE INDEX idx_comms_unread ON communications(workspace_id, read, created_at DESC) WHERE read = false AND archived = false;
CREATE INDEX idx_comms_smart_line ON communications(smart_line_id, created_at DESC) WHERE smart_line_id IS NOT NULL;
```

---

### 2.3 Task (from Universal Task System)

Tasks are stored in the shared `tasks` table defined in [TaskSystem/overview.md](../TaskSystem/overview.md). CRM tasks are rows where `module_type = 'crm'`. This spec does not redefine the task schema -- it documents how CRM uses it.

**CRM-relevant task fields (from the master tasks table):**

| Field | Type | CRM Usage |
|-------|------|-----------|
| id | UUID | Task identifier |
| workspace_id | UUID | Tenant key |
| title | TEXT | Task description (e.g., "Follow up with Jack Chen about Henderson renewal") |
| description | TEXT | Extended context |
| task_type | TEXT | CRM uses: `action`, `entity_resolution`, `manual`, `onboarding`, `sla_warning` |
| module_type | TEXT | Always `'crm'` for CRM-originated tasks |
| vault_id | UUID | FK to the Account, Counterparty, or Deal vault |
| field_code | TEXT | NULL for CRM tasks (used by Contracts triage) |
| severity | TEXT | `info`, `warning`, `blocker`, `urgent` |
| status | TEXT | `open`, `in_progress`, `in_review`, `resolved`, `dismissed` |
| priority | INT | Numeric sort weight |
| created_by | UUID | Who/what created the task |
| assigned_to | UUID | Assigned user |
| source | TEXT | `system`, `manual`, `ai`, `cross_module`, `workflow`, `stage_gate`, `onboarding` |
| due_at | TIMESTAMPTZ | SLA deadline |
| resolved_at | TIMESTAMPTZ | When resolved |
| resolution_note | TEXT | Resolution details |
| metadata | JSONB | CRM-specific: `{source_communication_id, source_deal_id, stage_gate_name, onboarding_step}` |
| created_at | TIMESTAMPTZ | Creation time |
| updated_at | TIMESTAMPTZ | Last update |

**CRM Task Sources:**

| Source | Trigger | Task Type | Example |
|--------|---------|-----------|---------|
| Manual | User clicks "Create Task" in CRM | `manual` | "Call Jack about renewal terms" |
| Workflow | Workflow engine automation | `action` | "Send welcome packet to new customer" |
| Stage Gate | Deal advances past a gated stage | `action` | "Complete 2 required tasks before advancing to Proposal" |
| Onboarding | Contract ships, cross-module event | `onboarding` | "Assign team members for Acme onboarding" |
| AI Suggestion | Otto suggests a follow-up | `action` | "Health score dropping -- suggest check-in with Summit Media" |
| Smart Line | Inbound message creates follow-up | `action` | "Follow up with +1-555-0199 (unknown contact)" |

---

### 2.4 Activity Event (`activity_events` table)

Activity events provide the chronological record for the CRM Activity Feed view. They are denormalized for fast reads -- each event is a self-contained record that does not require joins to render.

```sql
CREATE TABLE activity_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id),

    -- Relationships (all nullable -- events can be account-level, deal-level, or contact-level)
    account_id UUID NOT NULL REFERENCES vaults(id),    -- Always set (every event belongs to an account)
    deal_id UUID REFERENCES vaults(id),                -- Set when event relates to a specific deal
    contact_id UUID REFERENCES vault_contacts(id),     -- Set when event involves a specific contact

    -- Event
    event_type TEXT NOT NULL,                           -- See enum below
    description TEXT NOT NULL,                          -- Human-readable event description
    detail TEXT,                                        -- Extended detail (optional)

    -- Metadata
    metadata JSONB NOT NULL DEFAULT '{}',              -- Event-specific structured data
    actor_id UUID,                                     -- FK to users.id or NULL for system events
    actor_type TEXT NOT NULL DEFAULT 'user',            -- 'user' | 'system' | 'ai' | 'workflow'

    -- Timestamps
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

**Event Types:**

| event_type | Icon | Color | Description | metadata keys |
|------------|------|-------|-------------|---------------|
| `message_received` | Chat bubble (inbound) | Blue | Inbound iMessage/SMS/email received | `{communication_id, channel, sender_name}` |
| `message_sent` | Chat bubble (outbound) | Teal | Outbound message sent | `{communication_id, channel, recipient_name}` |
| `email_received` | Envelope (inbound) | Gray | Inbound email received | `{communication_id, subject, sender_email}` |
| `email_sent` | Envelope (outbound) | Gray | Outbound email sent | `{communication_id, subject, recipient_email}` |
| `call_completed` | Phone | Purple | Call ended (inbound or outbound) | `{communication_id, duration_seconds, direction, outcome}` |
| `call_missed` | Phone (x) | Red | Missed inbound call | `{communication_id, caller_phone, has_voicemail}` |
| `deal_stage_changed` | Arrow right | Teal | Deal moved to a new pipeline stage | `{deal_id, from_stage, to_stage, changed_by}` |
| `deal_created` | Plus circle | Blue | New deal created | `{deal_id, value, source}` |
| `deal_won` | Trophy | Green | Deal closed-won | `{deal_id, value, days_to_close}` |
| `deal_lost` | X circle | Red | Deal closed-lost | `{deal_id, value, loss_reason, loss_category}` |
| `task_created` | Plus | Blue | New task created | `{task_id, title, assigned_to, source}` |
| `task_completed` | Checkmark | Green | Task resolved | `{task_id, title, resolved_by}` |
| `lead_created` | Person+ | Amber | New lead entered the system | `{contact_id, source, lead_score}` |
| `lead_converted` | Arrow up | Green | Lead converted to deal | `{contact_id, deal_id, from_stage}` |
| `health_alert` | Warning triangle | Red | Health score dropped below threshold | `{previous_score, current_score, risk_factors: [...]}` |
| `health_recovered` | Heart | Green | Health score recovered above threshold | `{previous_score, current_score}` |
| `note_added` | Pencil | Gray | User added a note to the account/deal | `{note_preview, added_by}` |
| `meeting_completed` | Calendar | Purple | Meeting took place | `{communication_id, attendees: [...], duration_minutes}` |
| `contact_added` | Person+ | Blue | New contact added to account | `{contact_id, name, source}` |
| `entity_resolved` | Link | Teal | Entity resolution matched a document to this account | `{item_vault_id, confidence, resolver_type}` |
| `document_uploaded` | File | Gray | Document uploaded to this account | `{item_vault_id, filename, file_type}` |
| `stage_gate_blocked` | Lock | Amber | Deal blocked at a gate checkpoint | `{deal_id, gate_name, blocking_tasks: [...]}` |

**Full Field Reference:**

| Field | Type | Required | Default | Validation | Description |
|-------|------|----------|---------|------------|-------------|
| id | UUID | Yes | `gen_random_uuid()` | PK | Event identifier |
| workspace_id | UUID | Yes | -- | FK to `workspaces.id` | Tenant key |
| account_id | UUID | Yes | -- | FK to `vaults.id` (L1) | Parent account |
| deal_id | UUID | No | `NULL` | FK to `vaults.id` (L4) | Related deal |
| contact_id | UUID | No | `NULL` | FK to `vault_contacts.id` | Related contact |
| event_type | ENUM (TEXT) | Yes | -- | One of the 21 event types above | Event category |
| description | TEXT | Yes | -- | Max 500 chars | Human-readable description |
| detail | TEXT | No | `NULL` | Max 2,000 chars | Extended detail |
| metadata | JSONB | Yes | `'{}'` | Valid JSON | Event-specific structured data |
| actor_id | UUID | No | `NULL` | FK to `users.id` | Who or what triggered this event |
| actor_type | ENUM (TEXT) | Yes | `'user'` | One of: `user`, `system`, `ai`, `workflow` | Actor category |
| created_at | TIMESTAMPTZ | Yes | `now()` | -- | Event timestamp |

**Indexes:**

```sql
CREATE INDEX idx_events_workspace ON activity_events(workspace_id, created_at DESC);
CREATE INDEX idx_events_account ON activity_events(account_id, created_at DESC);
CREATE INDEX idx_events_deal ON activity_events(deal_id, created_at DESC) WHERE deal_id IS NOT NULL;
CREATE INDEX idx_events_contact ON activity_events(contact_id, created_at DESC) WHERE contact_id IS NOT NULL;
CREATE INDEX idx_events_type ON activity_events(workspace_id, event_type, created_at DESC);
```

---

## 3. Relationship Mappings (Junction Tables)

### 3.1 `deal_contacts` -- Contacts Linked to Deals

Maps contacts to specific deals with their role in that deal. A contact can be linked to multiple deals, and a deal can have multiple contacts.

```sql
CREATE TABLE deal_contacts (
    deal_id UUID NOT NULL REFERENCES vaults(id) ON DELETE CASCADE,
    contact_id UUID NOT NULL REFERENCES vault_contacts(id) ON DELETE CASCADE,
    role_in_deal TEXT NOT NULL DEFAULT 'End_User',
    added_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    added_by UUID REFERENCES users(id),
    notes TEXT,
    PRIMARY KEY (deal_id, contact_id)
);
```

| Field | Type | Required | Default | Validation | Description |
|-------|------|----------|---------|------------|-------------|
| deal_id | UUID | Yes | -- | FK to `vaults.id` (L4), ON DELETE CASCADE | The deal |
| contact_id | UUID | Yes | -- | FK to `vault_contacts.id`, ON DELETE CASCADE | The contact |
| role_in_deal | ENUM (TEXT) | Yes | `'End_User'` | One of: `Decision_Maker`, `Influencer`, `Champion`, `Blocker`, `End_User`, `Legal`, `Finance`, `Technical` | Contact's role in this specific deal |
| added_at | TIMESTAMPTZ | Yes | `now()` | -- | When the contact was linked |
| added_by | UUID | No | `NULL` | FK to `users.id` | Who linked them |
| notes | TEXT | No | `NULL` | Max 1,000 chars | Notes about this contact's involvement |

```sql
CREATE INDEX idx_deal_contacts_contact ON deal_contacts(contact_id);
CREATE INDEX idx_deal_contacts_role ON deal_contacts(deal_id, role_in_deal);
```

### 3.2 `deal_tasks` -- Tasks Linked to Deals

Maps tasks from the universal task table to specific deals. Provides the task list shown on deal cards in the Pipeline Kanban view.

```sql
CREATE TABLE deal_tasks (
    deal_id UUID NOT NULL REFERENCES vaults(id) ON DELETE CASCADE,
    task_id UUID NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
    stage_gate TEXT,                                    -- Which pipeline stage this task is for
    is_gate_requirement BOOLEAN NOT NULL DEFAULT false, -- Must complete before advancing past stage_gate
    display_order INT NOT NULL DEFAULT 0,              -- Sort order within the deal's task list
    PRIMARY KEY (deal_id, task_id)
);
```

| Field | Type | Required | Default | Validation | Description |
|-------|------|----------|---------|------------|-------------|
| deal_id | UUID | Yes | -- | FK to `vaults.id` (L4), ON DELETE CASCADE | The deal |
| task_id | UUID | Yes | -- | FK to `tasks.id`, ON DELETE CASCADE | The task |
| stage_gate | TEXT | No | `NULL` | One of: `Prospecting`, `Discovery`, `Proposal`, `Negotiation`, `Close` | Which stage this task blocks |
| is_gate_requirement | BOOLEAN | Yes | `false` | -- | If true, this task must be completed before the deal can advance past `stage_gate` |
| display_order | INT | Yes | `0` | >= 0 | Sort order in the deal's task list |

```sql
CREATE INDEX idx_deal_tasks_task ON deal_tasks(task_id);
CREATE INDEX idx_deal_tasks_gate ON deal_tasks(deal_id, stage_gate) WHERE is_gate_requirement = true;
```

### 3.3 `account_tags` -- Tags on Accounts

While tags are also stored in `metadata.tags` (JSONB array), this junction table enables efficient tag-based filtering and tag analytics across accounts.

```sql
CREATE TABLE account_tags (
    account_id UUID NOT NULL REFERENCES vaults(id) ON DELETE CASCADE,
    tag TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (account_id, tag)
);
```

| Field | Type | Required | Default | Validation | Description |
|-------|------|----------|---------|------------|-------------|
| account_id | UUID | Yes | -- | FK to `vaults.id` (L1), ON DELETE CASCADE | The account |
| tag | TEXT | Yes | -- | Max 50 chars, lowercase, trimmed | Tag value |
| created_at | TIMESTAMPTZ | Yes | `now()` | -- | When the tag was applied |

```sql
CREATE INDEX idx_account_tags_tag ON account_tags(tag);
```

### 3.4 `contact_tags` -- Tags on Contacts

Same pattern as `account_tags` but for contacts.

```sql
CREATE TABLE contact_tags (
    contact_id UUID NOT NULL REFERENCES vault_contacts(id) ON DELETE CASCADE,
    tag TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (contact_id, tag)
);
```

| Field | Type | Required | Default | Validation | Description |
|-------|------|----------|---------|------------|-------------|
| contact_id | UUID | Yes | -- | FK to `vault_contacts.id`, ON DELETE CASCADE | The contact |
| tag | TEXT | Yes | -- | Max 50 chars, lowercase, trimmed | Tag value |
| created_at | TIMESTAMPTZ | Yes | `now()` | -- | When the tag was applied |

```sql
CREATE INDEX idx_contact_tags_tag ON contact_tags(tag);
```

---

## 4. Computed Fields & Aggregations

These fields are never stored directly. They are computed at query time (or via materialized views for expensive aggregations).

### 4.1 Account Health Score

```
Account.health_score = weighted average of:
  contact_responsiveness (30%)  -- avg response time to inbound messages
  deal_progress          (25%)  -- % of active deals advancing on schedule
  task_completion        (20%)  -- resolved_tasks / total_tasks for this account
  interaction_recency    (15%)  -- inverse of days since last interaction
  renewal_proximity      (10%)  -- how close the nearest renewal date is (closer = lower)
```

**Component calculations:**

| Component | Weight | Formula | Range |
|-----------|--------|---------|-------|
| contact_responsiveness | 30% | `100 - MIN(avg_response_hours * 2, 100)`. Average hours to reply to inbound messages. Lower response time = higher score. | 0-100 |
| deal_progress | 25% | `(deals_on_schedule / active_deals) * 100`. A deal is "on schedule" if `days_in_stage < stage_sla_days`. | 0-100 |
| task_completion | 20% | `(resolved_tasks / total_tasks) * 100`. Across all CRM tasks for this account. Returns 100 if no tasks exist. | 0-100 |
| interaction_recency | 15% | `MAX(0, 100 - (days_since_last_interaction * 3))`. Decays 3 points per day of silence. | 0-100 |
| renewal_proximity | 10% | `100` if no renewal in next 90 days; `MAX(0, (days_until_renewal / 90) * 100)` if renewal exists. Closer renewal = lower score (higher urgency). | 0-100 |

**Refresh strategy:** Computed on-demand when viewing the account, cached in `vaults.health_score` column, refreshed every 15 minutes by a background BullMQ job or on any event that affects a component (new message, task resolved, deal stage change).

### 4.2 Other Computed Fields

| Entity | Computed Field | Formula | Notes |
|--------|---------------|---------|-------|
| Account | `deal_count` | `SELECT COUNT(*) FROM vaults WHERE parent_vault_id IN (child_tree) AND vault_level = 4 AND metadata->>'lifecycle_status' = 'Active'` | Recursive through division/counterparty tree |
| Account | `total_value` | `SELECT SUM((metadata->>'value')::decimal) FROM vaults WHERE ... AND vault_level = 4 AND metadata->>'lifecycle_status' = 'Active'` | Same tree traversal as deal_count |
| Account | `last_activity` | `SELECT MAX(created_at) FROM activity_events WHERE account_id = :id` | Indexed query |
| Contact | `interaction_count` | `SELECT COUNT(*) FROM communications WHERE contact_id = :id` | Auto-incremented via trigger for performance |
| Contact | `last_interaction` | `SELECT MAX(created_at) FROM communications WHERE contact_id = :id` | Auto-updated via trigger |
| Deal | `days_in_stage` | `EXTRACT(DAY FROM now() - (metadata->>'last_stage_change_at')::timestamptz)` | Computed at query time |
| Deal | `task_progress` | `SELECT (COUNT(*) FILTER (WHERE status = 'resolved') * 100.0 / NULLIF(COUNT(*), 0)) FROM deal_tasks dt JOIN tasks t ON dt.task_id = t.id WHERE dt.deal_id = :id` | Percentage of linked tasks completed |
| Deal | `task_count` | `SELECT COUNT(*) FROM deal_tasks WHERE deal_id = :id` | Total linked tasks |
| Deal | `completed_task_count` | `SELECT COUNT(*) FROM deal_tasks dt JOIN tasks t ON dt.task_id = t.id WHERE dt.deal_id = :id AND t.status = 'resolved'` | Completed linked tasks |
| Counterparty | `deal_count` | `SELECT COUNT(*) FROM vaults WHERE parent_vault_id = :id AND vault_level = 4 AND metadata->>'lifecycle_status' = 'Active'` | Direct children only |
| Counterparty | `total_value` | `SELECT SUM((metadata->>'value')::decimal) FROM vaults WHERE parent_vault_id = :id AND vault_level = 4 AND metadata->>'lifecycle_status' = 'Active'` | Direct children only |

### 4.3 Materialized Views (for Dashboard Performance)

For the CRM sub-panel Quick Stats and Health Monitor views, use materialized views refreshed on a schedule:

```sql
-- Pipeline summary by workspace
CREATE MATERIALIZED VIEW crm_pipeline_summary AS
SELECT
    v.workspace_id,
    v.metadata->>'stage' AS stage,
    COUNT(*) AS deal_count,
    SUM((v.metadata->>'value')::decimal) AS total_value,
    AVG((v.metadata->>'probability')::int) AS avg_probability
FROM vaults v
WHERE v.vault_level = 4
  AND v.vault_type = 'contract'
  AND v.metadata->>'lifecycle_status' = 'Active'
  AND v.archived_at IS NULL
GROUP BY v.workspace_id, v.metadata->>'stage';

-- Refresh every 5 minutes
-- Triggered by: deal stage changes, new deals, deal closures

-- Account health rankings
CREATE MATERIALIZED VIEW crm_account_health AS
SELECT
    v.id AS account_id,
    v.workspace_id,
    v.name,
    v.health_score,
    LAG(v.health_score) OVER (PARTITION BY v.id ORDER BY v.updated_at) AS previous_health_score,
    v.health_score - LAG(v.health_score) OVER (PARTITION BY v.id ORDER BY v.updated_at) AS health_trend
FROM vaults v
WHERE v.vault_level = 1
  AND v.archived_at IS NULL
ORDER BY v.health_score ASC;

-- Refresh every 15 minutes
```

---

## 5. Indexes & Query Patterns

Each CRM view requires specific query patterns. This section maps views to their queries and the indexes that support them.

### 5.1 CRM Inbox

**View:** DISCOVER > Inbox
**Purpose:** Unread communications across all channels, newest first.

```sql
-- Primary inbox query
SELECT c.*, vc.name AS contact_name, v.name AS account_name
FROM communications c
LEFT JOIN vault_contacts vc ON c.contact_id = vc.id
JOIN vaults v ON c.account_id = v.id
WHERE c.workspace_id = :workspace_id
  AND c.archived = false
  AND c.read = false
ORDER BY c.created_at DESC
LIMIT 50;

-- Channel-filtered inbox (e.g., Texts only)
SELECT c.*, vc.name AS contact_name, v.name AS account_name
FROM communications c
LEFT JOIN vault_contacts vc ON c.contact_id = vc.id
JOIN vaults v ON c.account_id = v.id
WHERE c.workspace_id = :workspace_id
  AND c.channel IN ('imessage', 'sms')
  AND c.archived = false
  AND c.read = false
ORDER BY c.created_at DESC
LIMIT 50;
```

**Supporting index:** `idx_comms_unread`, `idx_comms_inbox`

### 5.2 New Leads

**View:** DISCOVER > New Leads
**Purpose:** Contacts in lead/MQL stages, sorted by lead score.

```sql
SELECT vc.*, v.name AS account_name
FROM vault_contacts vc
JOIN vaults v ON vc.account_id = v.id
WHERE vc.workspace_id = :workspace_id
  AND vc.lifecycle_stage IN ('Lead', 'MQL')
ORDER BY vc.lead_score DESC NULLS LAST, vc.created_at DESC
LIMIT 50;
```

**Supporting index:** `idx_contacts_lifecycle`, `idx_contacts_lead_score`

### 5.3 Pipeline (Deal Kanban)

**View:** DEALS > Pipeline
**Purpose:** Active deals grouped by stage, with deal value totals per column.

```sql
SELECT
    v.id,
    v.name,
    v.metadata->>'stage' AS stage,
    (v.metadata->>'value')::decimal AS value,
    (v.metadata->>'probability')::int AS probability,
    v.metadata->>'owner_id' AS owner_id,
    v.metadata->>'last_stage_change_at' AS last_stage_change_at,
    EXTRACT(DAY FROM now() - (v.metadata->>'last_stage_change_at')::timestamptz) AS days_in_stage,
    parent.name AS counterparty_name
FROM vaults v
LEFT JOIN vaults parent ON v.parent_vault_id = parent.id
WHERE v.workspace_id = :workspace_id
  AND v.vault_level = 4
  AND v.metadata->>'lifecycle_status' = 'Active'
  AND v.archived_at IS NULL
ORDER BY
    CASE v.metadata->>'stage'
        WHEN 'Prospecting' THEN 0
        WHEN 'Discovery' THEN 1
        WHEN 'Proposal' THEN 2
        WHEN 'Negotiation' THEN 3
        WHEN 'Close' THEN 4
    END,
    (v.metadata->>'value')::decimal DESC;
```

**Supporting index:** `idx_vaults_workspace` (workspace_id, vault_level)

**Additional GIN index for JSONB stage filtering:**

```sql
CREATE INDEX idx_vaults_metadata_stage ON vaults USING gin((metadata->'stage')) WHERE vault_level = 4;
CREATE INDEX idx_vaults_metadata_lifecycle ON vaults USING gin((metadata->'lifecycle_status')) WHERE vault_level = 4;
```

### 5.4 Health Monitor

**View:** CUSTOMERS > Health Monitor
**Purpose:** Accounts sorted by health score (worst first), with trend indicators.

```sql
SELECT
    v.id,
    v.name,
    v.health_score,
    v.metadata->>'segment' AS segment,
    (SELECT MAX(ae.created_at) FROM activity_events ae WHERE ae.account_id = v.id) AS last_activity,
    (SELECT COUNT(*) FROM vaults child WHERE child.parent_vault_id = v.id AND child.vault_level = 4 AND child.metadata->>'lifecycle_status' = 'Active') AS active_deals
FROM vaults v
WHERE v.workspace_id = :workspace_id
  AND v.vault_level = 1
  AND v.archived_at IS NULL
  AND v.health_score IS NOT NULL
ORDER BY v.health_score ASC
LIMIT 50;
```

**Supporting index:**

```sql
CREATE INDEX idx_vaults_health ON vaults(workspace_id, health_score ASC) WHERE vault_level = 1 AND archived_at IS NULL;
```

### 5.5 Renewals

**View:** CUSTOMERS > Renewals
**Purpose:** Deals with upcoming renewal dates within a configurable window.

```sql
SELECT
    v.id,
    v.name,
    (v.metadata->>'value')::decimal AS value,
    (v.metadata->>'renewal_date')::date AS renewal_date,
    ((v.metadata->>'renewal_date')::date - CURRENT_DATE) AS days_until_renewal,
    parent.name AS counterparty_name,
    grandparent.name AS account_name
FROM vaults v
LEFT JOIN vaults parent ON v.parent_vault_id = parent.id
LEFT JOIN vaults grandparent ON parent.parent_vault_id = grandparent.id
WHERE v.workspace_id = :workspace_id
  AND v.vault_level = 4
  AND v.metadata->>'renewal_date' IS NOT NULL
  AND (v.metadata->>'renewal_date')::date <= CURRENT_DATE + interval '90 days'
  AND v.metadata->>'lifecycle_status' = 'Active'
  AND v.archived_at IS NULL
ORDER BY (v.metadata->>'renewal_date')::date ASC;
```

**Supporting index:**

```sql
CREATE INDEX idx_vaults_renewal ON vaults((metadata->>'renewal_date'))
    WHERE vault_level = 4 AND archived_at IS NULL AND metadata->>'renewal_date' IS NOT NULL;
```

### 5.6 Activity Feed

**View:** ACCOUNTS > Activity Feed
**Purpose:** Chronological event stream across all accounts or filtered to one account.

```sql
-- Cross-account activity feed (newest first)
SELECT ae.*, v.name AS account_name, vc.name AS contact_name
FROM activity_events ae
JOIN vaults v ON ae.account_id = v.id
LEFT JOIN vault_contacts vc ON ae.contact_id = vc.id
WHERE ae.workspace_id = :workspace_id
ORDER BY ae.created_at DESC
LIMIT 50;

-- Single-account activity feed
SELECT ae.*, vc.name AS contact_name
FROM activity_events ae
LEFT JOIN vault_contacts vc ON ae.contact_id = vc.id
WHERE ae.account_id = :account_id
ORDER BY ae.created_at DESC
LIMIT 50;

-- Type-filtered feed (e.g., messages only)
SELECT ae.*, v.name AS account_name, vc.name AS contact_name
FROM activity_events ae
JOIN vaults v ON ae.account_id = v.id
LEFT JOIN vault_contacts vc ON ae.contact_id = vc.id
WHERE ae.workspace_id = :workspace_id
  AND ae.event_type IN ('message_received', 'message_sent', 'email_received', 'email_sent')
ORDER BY ae.created_at DESC
LIMIT 50;
```

**Supporting index:** `idx_events_workspace`, `idx_events_account`, `idx_events_type`

### 5.7 Accounts List

**View:** ACCOUNTS > Accounts
**Purpose:** All accounts with key metrics.

```sql
SELECT
    v.id,
    v.name,
    v.health_score,
    v.metadata->>'segment' AS segment,
    v.metadata->>'industry' AS industry,
    v.metadata->>'owner_id' AS owner_id,
    v.metadata->>'smart_line_status' AS smart_line_status,
    (SELECT COUNT(*) FROM vaults child
     WHERE child.parent_vault_id = v.id
       AND child.vault_level = 4
       AND child.metadata->>'lifecycle_status' = 'Active') AS deal_count,
    (SELECT MAX(ae.created_at) FROM activity_events ae WHERE ae.account_id = v.id) AS last_activity
FROM vaults v
WHERE v.workspace_id = :workspace_id
  AND v.vault_level = 1
  AND v.archived_at IS NULL
ORDER BY v.name ASC;
```

**Supporting index:** `idx_vaults_workspace`

### 5.8 Contacts Directory

**View:** ACCOUNTS > Contacts
**Purpose:** All contacts grouped by account.

```sql
SELECT
    vc.*,
    v.name AS account_name,
    cp.name AS counterparty_name
FROM vault_contacts vc
JOIN vaults v ON vc.account_id = v.id
LEFT JOIN vaults cp ON vc.counterparty_id = cp.id
WHERE vc.workspace_id = :workspace_id
ORDER BY v.name ASC, vc.name ASC;
```

**Supporting index:** `idx_contacts_workspace`, `idx_contacts_account`

---

## 6. Encryption Boundaries

Per the Airlock security architecture (see [Security/overview.md](../Security/overview.md)), all sensitive data is ciphered at rest. The application layer (FastAPI middleware) is the sole trust boundary where encryption/decryption occurs.

### 6.1 Fields Encrypted at Rest (Ciphertext in Storage)

These fields contain PII or sensitive content. They are encrypted before writing to PostgreSQL and decrypted at the application layer before serving to the frontend. They CANNOT be indexed or queried by value -- only by their associated plaintext identifiers.

| Table | Field | Reason | Encryption Method |
|-------|-------|--------|-------------------|
| `communications` | `content_encrypted` | Full message body -- always encrypted, no exceptions | AES-256-GCM via workspace DEK |
| `communications` | `call_recording_url` | Points to encrypted audio file | AES-256-GCM via workspace DEK |
| `communications` | `voicemail_transcription` | Transcribed speech content | AES-256-GCM via workspace DEK |
| `vault_contacts` | `email` | PII -- email address | AES-256-GCM via workspace DEK |
| `vault_contacts` | `phone` | PII -- phone number | AES-256-GCM via workspace DEK |
| `vaults` (L1/L2) | `metadata.address` | PII -- physical address | Encrypted within JSONB via application-layer field encryption |
| Any document content | File blobs | Document content | AES-256-GCM via workspace DEK, stored encrypted at file system level |

**Encryption implementation notes:**
- Each workspace has its own Data Encryption Key (DEK), which is itself encrypted by a Key Encryption Key (KEK) -- envelope encryption.
- `content_preview` in `communications` is a truncated, non-PII preview (first 200 chars of the message with names/numbers redacted). It is stored in plaintext to support inbox list rendering without decryption overhead.
- The `content_encrypted` column stores base64-encoded ciphertext with a prepended IV/nonce.

### 6.2 Fields Stored in Plaintext (Queryable/Indexable)

These fields must remain in plaintext because they are used for filtering, sorting, aggregation, and indexing. They do NOT contain PII or sensitive content.

| Category | Fields | Reason Plaintext |
|----------|--------|-----------------|
| **Identifiers** | All `id`, `workspace_id`, `parent_vault_id` (UUIDs) | Foreign keys, joins, lookups |
| **Timestamps** | All `created_at`, `updated_at`, `archived_at`, `due_at`, `resolved_at` | Sorting, filtering, SLA calculations |
| **Enums/Status** | `channel`, `direction`, `read`, `replied`, `status`, `severity`, `lifecycle_stage`, `relationship_type`, `segment`, `stage`, `lifecycle_status` | Filtering, grouping, Kanban columns |
| **Scores/Counts** | `health_score`, `lead_score`, `interaction_count`, `probability`, `priority` | Sorting, aggregation, dashboard stats |
| **Names** | `vault.name`, `vault_contacts.name` | Search, display, sorting. **Note:** contact names are redacted before being sent to LLM providers |
| **Slugs** | `vault.slug` | URL routing |
| **Tags** | `tags` arrays | Filtering |
| **Deal Financials** | `value`, `currency` | Aggregation, pipeline totals |
| **Metadata (non-PII)** | `industry`, `website`, `logo_url`, `division_type`, `source`, `notes` | Filtering, display |
| **AI Classifications** | `ai_intent`, `ai_intent_confidence`, `ai_summary` | Filtering, display |
| **Content Preview** | `communications.content_preview` | Inbox rendering (truncated, redacted) |

### 6.3 LLM Redaction Rules

Before any CRM data reaches an LLM provider (via Otto's enrichment pipeline or AI intent classification), the following redaction rules apply:

| Data Type | Redaction Rule | Example |
|-----------|---------------|---------|
| Contact email | Replace with `[EMAIL_REDACTED]` | `jack@nova.com` -> `[EMAIL_REDACTED]` |
| Contact phone | Replace with `[PHONE_REDACTED]` | `+13105550142` -> `[PHONE_REDACTED]` |
| Physical address | Replace with `[ADDRESS_REDACTED]` | `123 Main St, LA, CA 90001` -> `[ADDRESS_REDACTED]` |
| Contact name | Allowed (needed for context) but flagged as PII in the prompt wrapper | `Jack Chen` stays as-is |
| Deal value | Allowed (needed for analysis) | `$120,000` stays as-is |
| Message content | Full content passed only to intent classifier with PII redaction | Names, emails, phones stripped from message body before LLM |

---

## 7. Database Triggers & Auto-Update Rules

These triggers maintain data consistency without requiring application-layer intervention.

| Trigger | Table | Event | Action |
|---------|-------|-------|--------|
| `trg_contact_interaction_count` | `communications` | INSERT | Increment `vault_contacts.interaction_count` and update `vault_contacts.last_interaction` for the matching `contact_id` |
| `trg_counterparty_last_interaction` | `communications` | INSERT | Update `metadata.last_interaction` on the counterparty vault (L3) by traversing `account_id` -> counterparty hierarchy |
| `trg_deal_stage_change` | `vaults` | UPDATE of `metadata.stage` | Set `metadata.last_stage_change_at = now()`, auto-set `metadata.probability` based on stage, create `activity_events` row with `event_type = 'deal_stage_changed'` |
| `trg_deal_won` | `vaults` | UPDATE of `metadata.lifecycle_status` to `'Won'` | Set `metadata.actual_close_date = CURRENT_DATE`, create `activity_events` row with `event_type = 'deal_won'` |
| `trg_deal_lost` | `vaults` | UPDATE of `metadata.lifecycle_status` to `'Lost'` | Set `metadata.actual_close_date = CURRENT_DATE`, create `activity_events` row with `event_type = 'deal_lost'` |
| `trg_activity_event_on_comm` | `communications` | INSERT | Create `activity_events` row with appropriate `event_type` based on `channel` and `direction` |
| `trg_updated_at` | `vaults`, `vault_contacts` | UPDATE | Set `updated_at = now()` |
| `trg_health_refresh` | `activity_events` | INSERT | Enqueue a BullMQ job to recompute `health_score` for the affected `account_id` |

---

## 8. Entity Relationship Diagram (Textual)

```
workspaces
    |
    +--< vaults (L1: Account)
    |       |
    |       +--< vaults (L2: Division)
    |       |       |
    |       |       +--< vaults (L3: Counterparty)
    |       |               |
    |       |               +--< vaults (L4: Item/Deal)
    |       |               |       |
    |       |               |       +--< deal_contacts >--< vault_contacts
    |       |               |       |
    |       |               |       +--< deal_tasks >--< tasks
    |       |               |
    |       |               +--< vault_contacts
    |       |
    |       +--< vaults (L3: Counterparty, direct -- no division)
    |       |       |
    |       |       +--< vaults (L4: Item/Deal)
    |       |
    |       +--< vault_contacts
    |       |
    |       +--< communications
    |       |
    |       +--< activity_events
    |       |
    |       +--< account_tags
    |
    +--< tasks (module_type = 'crm')
    |
    +--< contact_tags >--< vault_contacts
```

**Key relationships:**
- `vaults` self-references via `parent_vault_id` (tree structure)
- `vault_contacts` belongs to an Account (L1/L2) and optionally a Counterparty (L3)
- `communications` belongs to an Account and optionally a Contact and Deal
- `activity_events` belongs to an Account and optionally a Deal and Contact
- `deal_contacts` is a many-to-many junction between Deals (L4) and Contacts
- `deal_tasks` is a many-to-many junction between Deals (L4) and Tasks
- `tasks` references `vaults` via `vault_id` (the universal task system)
- `account_tags` and `contact_tags` are tag junction tables

---

## Related Specs

- **[overview.md](./overview.md)** -- CRM module identity, chamber views, cross-module integration
- **[crm-views-design-spec.md](./crm-views-design-spec.md)** -- UI specs for all 11 CRM views (Inbox, New Leads, Pipeline, etc.)
- **[crm-comms-brainstorm.md](./crm-comms-brainstorm.md)** -- Smart Line architecture, iMessage bridge, communications routing
- **[web-to-imessage-handoff.md](./web-to-imessage-handoff.md)** -- Web widget, QR bridge, lead intake workflow
- **[VaultHierarchy/overview.md](../VaultHierarchy/overview.md)** -- Vault tree data model, `vaults` table schema
- **[TaskSystem/overview.md](../TaskSystem/overview.md)** -- Universal task table schema, task lifecycle
- **[TaskSystem/task-lifecycle-deep-dive.md](../TaskSystem/task-lifecycle-deep-dive.md)** -- Builder/Modifier/Signer roles
- **[Security/overview.md](../Security/overview.md)** -- Encryption architecture, threat model, PII handling
- **[Roles/overview.md](../Roles/overview.md)** -- Role-based access at each vault level
- **[Record Inspector/entity-resolution.md](../Record%20Inspector/entity-resolution.md)** -- Entity resolution confidence scoring
