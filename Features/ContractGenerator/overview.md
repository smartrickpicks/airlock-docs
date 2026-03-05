# Contract Generator -- Design Spec

> **Status:** SPECCED (v2 unified clause library)

## Purpose

The Contract Generator is a **singleton view** pinned at the top of the Contracts module sub-panel. It provides a guided builder for creating new contracts from pre-approved templates, a unified clause library (188 clauses), and trigger-based clause selection across 24 contract types.

---

## Source Mapping (OrcestrateOS)

| OrcestrateOS File | Size | Airlock Target |
|---|---|---|
| `server/generation/engine.py` | 453 lines | Generation pipeline (clause index, trigger eval, interpolation) |
| `server/generation/fake_data.py` | 800+ lines | Deterministic test data generator (24 types) |
| `server/generation/variation_engine.py` | 500 lines | Section reorder, heading styles, paragraph density |
| `server/generation/pdf_renderer.py` | 322 lines | PDF output (972 style combinations) |
| `rules/generation/clause_library_v2.json` | 388KB, 188 clauses | Unified clause library (generation + extraction) |
| `rules/generation/templates/*.json` | 14 files | Per-type template definitions |

---

## Layout: Two-Pane

The Contract Generator uses a two-pane layout, distinct from the standard triptych.

| Pane | Position | Purpose |
|---|---|---|
| **Builder** | Left | Contract type selector, section-by-section form wizard, clause picker |
| **Preview** | Right | Live rendered output of the contract being built |

---

## Builder Panel

### Contract Type Selector

The first step selects the contract type, which determines the template and available clauses. **24 types across 5 verticals**, displayed as collapsible groups:

| Vertical | Types |
|----------|-------|
| **Music Core** | Distribution, License, Recording, Publishing |
| **Music Ancillary** | Producer, Sync License, Master Use, Management, Co-Publishing, Sample Clearance, Merchandise |
| **Film** | Film Distribution, Talent (Actor), Director, Screenplay Option, Co-Production |
| **TV** | TV Distribution, TV Talent, TV Development, TV Licensing |
| **Cross-Entertainment** | NDA, Work for Hire, Assignment, Termination |

**UX Decision:** Grouped by vertical with collapsible sections. Groups expand on click. Most recent/frequent types promoted to top.

### Section-by-Section Form Wizard

After selecting a contract type, the builder presents a step-by-step wizard:

- Each section of the template is presented as a separate form step.
- Fields within each section use `{{CHECK_CODE}}` variable names mapped to canonical keys.
- Required fields are marked; optional fields have sensible defaults.
- Users proceed section by section, with the preview updating after each step.

### Clause Picker

- Each section offers a clause picker drawn from the **unified clause library**.
- **Primary filters:** Section + contract type (automatic, based on current step).
- **Secondary filter:** `clause_type` dropdown for power users (101 functional types like PARTIES_IDENTIFICATION, TERM_DURATION, ROYALTY_RATE). Hidden by default, shown via "Advanced filters" toggle.
- Selected clauses replace the auto-selected clause for that section.
- Clause risk level is displayed as a pill badge: `standard` (no indicator), `elevated` (amber), `critical` (red), `high` (red, pulsing).
- **Extraction bridge indicator:** Each clause card shows a subtle "Extraction: N fields" badge, expandable to reveal which fields this clause also configures extraction for. Useful for admins; unobtrusive for analysts.

### Field Autocomplete

- Field inputs offer autocomplete suggestions from the field registry (442 fields).
- Previously used values are suggested first.
- Entity fields (party names, addresses) pull from the Identity Resolver's known entities.

---

## Preview Panel

The preview panel renders the contract in real time as the builder is filled out.

| Feature | Description |
|---|---|
| **Live markdown rendering** | Contract renders as formatted markdown, updating as fields change |
| **Section headers with clause IDs** | Each section shows its header and the clause ID in use (e.g., `GEN-RECITALS-DIST-V1`) |
| **Highlighted variables** | Unfilled fields appear as `[TO_BE_DEFINED]` (the generation_config fallback) |
| **Extraction badge** | Subtle indicator showing how many extraction fields this contract covers |
| **Export options** | PDF, DOCX, and Markdown export buttons at the top of the preview |

---

## Generation Pipeline

The v2 generation engine follows a unified pipeline where clause selection and interpolation happen from a single data source.

| Step | Description |
|---|---|
| **1. Load clause library** | Load `clause_library_v2.json` (LRU cached). 188 clauses with embedded generation + extraction config. |
| **2. Build clause index** | Index clauses by `(section, clause_type, contract_type)` for fast lookup. |
| **3. Load template** | Fetch the template for the selected contract type (14 explicit + auto-generated fallbacks). |
| **4. For each section** | Iterate through template sections in order. |
| **4a. Select clause** | Evaluate each candidate clause's `triggers[]` against form values. Highest `generation_config.priority` match wins per `(section, clause_type)` slot. |
| **4b. Interpolate variables** | Replace `{{CHECK_CODE}}` placeholders with user-provided values. Unfilled variables use `generation_config.fallback` text (default: `[TO_BE_DEFINED]`). |
| **5. Assemble** | Concatenate all sections into the final contract document. |

### Trigger Evaluation

Triggers are evaluated as AND logic -- all conditions in a clause's `triggers[]` array must match for the clause to be selected.

| Condition | Description | Example |
|---|---|---|
| `equals` | Case-insensitive string match | `OPP_CONTRACT_TYPE equals "Distribution"` |
| `in` | CSV list, any match wins | `OPP_PAYMENT_TERMS in "Net 30, Net 60"` |
| `not_equals` | Inverse of equals | `OPP_AUTO_RENEW not_equals "Yes"` |
| `pattern` | Regex match | `OPP_ADVANCE_AMOUNT pattern "^[\d,.]+$"` |
| `exists` | Field has a truthy value | `OPP_ADVANCE_AMOUNT exists` |
| `not_exists` | Field is empty/null | `OPP_ADVANCE_AMOUNT not_exists` |
| `contains` | Substring match | `OPP_TERRITORY contains "Europe"` |
| `default` | Always true (catch-all fallback) | Used for generic clauses |

---

## Template Schema

Each contract type has a template defining its section structure:

```json
{
  "template_id": "distribution-standard-v1",
  "contract_type": "distribution",
  "display_name": "Distribution Agreement",
  "sections": [
    {
      "id": "recitals",
      "title": "RECITALS",
      "order": 1,
      "type": "boilerplate"
    },
    {
      "id": "scope",
      "title": "1. SCOPE OF AGREEMENT",
      "order": 2,
      "type": "conditional"
    }
  ],
  "required_fields": ["OPP_CONTRACT_TYPE", "OPP_TERRITORY", "OPP_TERM_LENGTH"]
}
```

### Section Types

| Type | Description |
|---|---|
| **Boilerplate** | Fixed text or a default clause reference. No trigger evaluation needed. |
| **Conditional** | Populated by matching clauses from the unified library based on trigger evaluation against form values. |

---

## Unified Clause Library (v2)

The clause library is the single source of truth for BOTH generation AND extraction. Adding a clause extends both systems automatically.

### Clause Record Schema

```json
{
  "clause_id": "GEN-RECITALS-DIST-V1",
  "clause_type": "PARTIES_IDENTIFICATION",
  "version": 1,
  "contract_types": ["distribution"],
  "section": "recitals",
  "risk_level": "standard",
  "body": "This Distribution Agreement... {{OPP_EFFECTIVE_DATE}} by and between {{ACCT_LEGAL_ENTITY}}...",
  "variables": [
    {
      "field": "OPP_EFFECTIVE_DATE",
      "canonical_key": "Effective_Date__c",
      "placeholder": "{{OPP_EFFECTIVE_DATE}}",
      "default": null,
      "extraction": {
        "extractor_type": "date",
        "anchor_patterns": ["entered into as of", "effective as of"],
        "negative_anchors": ["expiration"],
        "preferred_zones": ["recitals", "preamble"],
        "proximity_chars": 200,
        "confidence_floor": 0.7
      }
    }
  ],
  "triggers": [
    {"field": "OPP_CONTRACT_TYPE", "condition": "equals", "value": "Distribution"}
  ],
  "generation_config": {"priority": 10, "fallback": "[TO_BE_DEFINED]"},
  "extraction_config": {
    "tier": 1,
    "preflight_weight": 1.0,
    "section_anchors": {
      "heading_patterns": ["RECITALS"],
      "proximity_keywords": ["entered into"]
    }
  },
  "cuad_labels": ["Parties", "Agreement Date"],
  "source_contracts": [],
  "approved_by": null,
  "last_reviewed": null
}
```

### Field Reference

| Field | Type | Description |
|---|---|---|
| `clause_id` | string | Unique ID: `CATEGORY-SECTION-TYPE-VERSION` (e.g., `GEN-RECITALS-DIST-V1`) |
| `clause_type` | string | Functional category (101 types, e.g., `PARTIES_IDENTIFICATION`, `ROYALTY_RATE`) |
| `version` | integer | Clause version (V1, V2, etc.) |
| `contract_types` | list | Which of the 24 contract types this clause applies to |
| `section` | string | Which template section (e.g., `recitals`, `scope`, `compensation`) |
| `risk_level` | enum | `standard`, `elevated`, `critical`, `high` |
| `body` | text | Legal prose with `{{CHECK_CODE}}` variable markers |
| `variables` | list | Variable definitions with extraction config per variable |
| `variables[].field` | string | Check code (e.g., `OPP_EFFECTIVE_DATE`) |
| `variables[].canonical_key` | string | SF-compatible field key (e.g., `Effective_Date__c`) |
| `variables[].extraction` | object | Extraction pipeline config: extractor_type, anchors, zones, confidence |
| `triggers` | list | Conditions for auto-selection (AND logic) |
| `generation_config` | object | Priority (0-100) + fallback text for unfilled variables |
| `extraction_config` | object | Tier (1-3), preflight weight, section anchors |
| `cuad_labels` | list | CUAD taxonomy tags (79 labels) for benchmarking |
| `approved_by` | string | Legal reviewer who approved the clause |
| `last_reviewed` | timestamp | Date of last legal review |

### Clause ID Convention

Format: `CATEGORY-SECTION-TYPE-VERSION`

| Category | Usage |
|---|---|
| `GEN` | Standard generation clauses |
| `RISK` | Risk-specific clauses (elevated/critical terms) |
| `CUAD` | CUAD-aligned clauses (benchmarking) |
| `MUSIC` | Music-industry-specific |
| `FILM` | Film-industry-specific |
| `TV` | TV-industry-specific |
| `BOILER` | Boilerplate/generic |

Examples: `GEN-RECITALS-DIST-V1`, `RISK-CONFIDENTIALITY-V1`, `CUAD-TERMINATION-CONVENIENCE-V1`

### Risk Levels

| Level | Count | Description | UI Treatment |
|---|---|---|---|
| `standard` | ~107 | Default, low-risk legal prose | No special indicator |
| `elevated` | ~59 | Terms that may need review | Amber pill in clause picker |
| `critical` | ~18 | Aggressive or unusual terms | Red pill; requires confirmation before selection |
| `high` | ~4 | Reserved for edge cases | Red pulsing pill; requires confirmation |

### Extraction Bridge

The unified library means the ContractGenerator's clause data IS also the extraction system's configuration. Each variable's `extraction` block defines:

- `extractor_type` -- Which extractor to use (date, text, currency, percentage, number, boolean, pattern, picklist, split)
- `anchor_patterns` -- Text phrases that appear near the value in real contracts
- `negative_anchors` -- Phrases that indicate a false positive
- `preferred_zones` -- Which contract sections to search first
- `proximity_chars` -- Search radius from anchor
- `confidence_floor` -- Minimum confidence to accept a match

**Key benefit:** When a legal team approves a new clause in the Generator, extraction coverage for those fields automatically improves. One source of truth for both systems.

### Library Statistics

| Metric | Value |
|---|---|
| Total clauses | 188 |
| Clause types | 101 functional categories |
| Contract types | 24 across 5 verticals |
| CUAD labels covered | 79 |
| Sections covered | 75 |
| Extractor types used | 9 (date, text, currency, percentage, number, boolean, pattern, picklist, split) |

---

## Feature Control Plane Integration

| Control | Description |
|---|---|
| **Toggle** | `contract_generator.enabled` -- enable/disable the generator |
| **Template management** | Admin UI for adding, editing, and versioning contract templates |
| **Clause approval workflow** | New or modified clauses require legal review before activation. Risk level `critical` or `high` requires admin sign-off. |
| **Clause library admin** | View/edit clauses with inline extraction preview. Shows extraction bridge impact. |
| **Audit log** | Every generated contract is logged with template version, selected clauses, field values, and clause IDs used |

---

## Related Specs

- [v2-migration-brief.md](./v2-migration-brief.md) -- Full v1 to v2 migration analysis
- [Feature Control Plane](../FeatureControlPlane/overview.md) -- Toggle and calibration system
- [Record Inspector](../Record%20Inspector/overview.md) -- Field cards display extracted values that come from clause-configured extraction
