# Enrichment Sources

Nine enrichment sources are pre-injected into the agent context before every agent call. Enrichment is **fire-and-forget** -- non-blocking, with graceful degradation. If any source fails, the agent call proceeds without it.

Each source can be **individually toggled** in the Feature Control Plane.

---

## Source Catalog

| # | Source | Description | Key data points |
|---|---|---|---|
| 1 | Gate State | Contract's preflight gate status | Gate color, health score, processing status |
| 2 | Field Summary | Aggregate pass/fail counts from preflight checks | Pass count, fail count, review count, skip count |
| 3 | Contract Health | Calibrated health assessment | Health score (numeric), gate color, total checks run |
| 4 | Domain Rules | Available classification categories for the contract | Contract classification categories defined by the org |
| 5 | Preflight Sections | Per-section status of preflight processing | Section name + status for: accounts, catalog, pricing, terms, etc. |
| 6 | Corpus Search | Full-text search in the contract's extracted text | Up to 10 matching lines from the document corpus |
| 7 | Extraction Meta | Metadata about the document extraction process | Page count, encoding type, mojibake count |
| 8 | Patch Summary | Triage items grouped by status | Counts for: open, in_review, resolved, dismissed |
| 9 | Deal Fields | Structured deal-level attributes | Territory, term length, contract type, deal type, counterparty type, legal entity |

---

## Detailed Source Descriptions

### 1. Gate State

Provides the contract's current position in the preflight pipeline.

| Field | Type | Example |
|---|---|---|
| Gate color | enum | `green` / `yellow` / `red` |
| Health score | float | `0.87` |
| Processing status | enum | `pending` / `in_progress` / `complete` / `failed` |

### 2. Field Summary

Aggregated results from all preflight field-level checks.

| Field | Type | Description |
|---|---|---|
| Pass | int | Fields that passed validation |
| Fail | int | Fields that failed validation |
| Review | int | Fields flagged for human review |
| Skip | int | Fields excluded from checks |

### 3. Contract Health

A higher-order health metric that combines gate state and check results.

| Field | Type | Description |
|---|---|---|
| Health score | float | Calibrated score (0.0 -- 1.0) |
| Gate color | enum | Derived gate color |
| Total checks | int | Number of checks executed |

### 4. Domain Rules

The classification taxonomy available for this contract's domain.

| Field | Type | Description |
|---|---|---|
| Categories | list[string] | Available contract classification categories |

### 5. Preflight Sections

Status of each logical section in the preflight pipeline.

| Field | Type | Description |
|---|---|---|
| Section name | string | e.g., `accounts`, `catalog`, `pricing`, `terms` |
| Status | enum | `pending` / `running` / `passed` / `failed` / `skipped` |

### 6. Corpus Search

Full-text search across the contract's extracted document text. Returns contextual snippets.

| Field | Type | Description |
|---|---|---|
| Query | string | Search term (derived from agent context) |
| Matches | list | Up to 10 matching lines with surrounding context |

### 7. Extraction Meta

Metadata about how the contract document was processed.

| Field | Type | Description |
|---|---|---|
| Page count | int | Number of pages in the source document |
| Encoding type | string | Detected text encoding |
| Mojibake count | int | Number of encoding-error characters detected |

### 8. Patch Summary

Triage items for this contract, grouped by current status.

| Status | Type | Description |
|---|---|---|
| Open | int | Unresolved triage items |
| In review | int | Items currently being reviewed |
| Resolved | int | Items that have been resolved |
| Dismissed | int | Items dismissed by an analyst |

### 9. Deal Fields

Structured business-level attributes of the deal.

| Field | Type | Description |
|---|---|---|
| Territory | string | Geographic region or market |
| Term length | string | Duration of the contract |
| Contract type | string | e.g., `new`, `renewal`, `amendment` |
| Deal type | string | Business classification of the deal |
| Counterparty type | string | Classification of the other party |
| Legal entity | string | The legal entity on the org's side |

---

## Behavioral Properties

| Property | Detail |
|---|---|
| Execution model | Fire-and-forget (non-blocking) |
| Failure handling | Graceful degradation -- failed sources are omitted, not retried |
| Toggleability | Each source independently toggleable via Feature Control Plane |
| Visibility | AI responses include an "enrichment used" indicator showing which sources were available |
