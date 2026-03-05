# Clause Library v1 → v2 Migration Brief

> **Status:** RESEARCH
> **Source:** OrcestrateOS-sync clause_library_v2.json (188 clauses, 388KB) + generation pipeline rebuild (completed)
> **Priority:** BLOCKING — must be resolved before any ContractGenerator code is written

## Summary

The ContractGenerator specs across this repo describe a **v1 architecture** that no longer exists. The actual clause library and generation engine were rebuilt from the ground up into a **unified v2 system** with fundamentally different data structures, naming conventions, and a merged extraction+generation model. Every spec file that references the old system needs updating before we hand off to implementation, or the engineers will build against a dead architecture.

This document explains what changed, why, and exactly which files need modification.

---

## What Changed and Why

### The Core Shift: Two Systems → One

**v1 (what the specs currently describe):**
- Clause library = standalone text templates with `[PLACEHOLDER]` markers
- Generation pairs = separate mapping file (93 rules) connecting fields to clauses
- Extraction = completely separate system with its own config files
- Result: three separate data sources to maintain, no guarantee they stay in sync

**v2 (what actually exists):**
- Unified clause library = every clause carries BOTH generation config AND extraction config in a single record
- No generation pairs file — the mapping logic is embedded in each clause via `triggers[]` and `generation_config`
- Each clause variable has an `extraction` block that tells the extraction pipeline how to find that value in real contracts
- Result: add one clause → generation AND extraction both extend automatically. One source of truth.

**Why this matters for Airlock:** The generation pairs concept, the `[PLACEHOLDER]` syntax, the 8-type limitation, and the simple clause schema described in the current specs don't exist anymore. Building from those specs would create code that can't consume the actual data.

---

## Architectural Differences (Complete Reference)

### 1. Contract Types: 8 → 24

**v1 spec says:** 8 types (Distribution, License, Recording, Publishing, Management, Service, Amendment, Termination)

**v2 actual:** 24 types across 5 verticals:

| Vertical | Types |
|----------|-------|
| Music Core | distribution, license, recording, publishing |
| Music Ancillary | producer, sync_license, master_use, management, co_publishing, sample_clearance, merchandise |
| Film | film_distribution, talent_actor, director, screenplay_option, co_production |
| TV | tv_distribution, tv_talent, tv_development, tv_licensing |
| Cross-Entertainment | nda, work_for_hire, assignment, termination |

The Contract Type Selector in the Builder Panel needs to reflect all 24 types, ideally grouped by vertical.

### 2. Generation Pairs → Eliminated

**v1 spec says:** 93 generation pair rules with schema: `pair_id`, `field`, `clause_id`, `when`, `priority`, `section`

**v2 actual:** No generation pairs file. The mapping logic moved INTO each clause record:
- `triggers[]` = the old `when` conditions (equals, in, not_equals, exists, contains)
- `generation_config.priority` = the old `priority` field
- `section` = still exists on the clause itself

The generation pipeline step "Load generation pairs" no longer exists. Instead: load clause library → build index by (section, clause_type, contract_type) → `select_clauses()` evaluates triggers against form values → highest priority match wins per (section, clause_type) slot.

### 3. Placeholder Syntax: `[FIELD]` → `{{CHECK_CODE}}`

**v1 spec says:** `[PLACEHOLDER]` markers like `[TERRITORY]`, `[EFFECTIVE_DATE]`

**v2 actual:** Double-brace with check codes: `{{OPP_TERRITORY}}`, `{{OPP_EFFECTIVE_DATE}}`, `{{ACCT_LEGAL_ENTITY}}`

The check code maps directly to the SF-compatible field key: strip prefix, lowercase, add `__c`. So `OPP_TERRITORY` → `territory__c`. This naming convention runs through the entire system — extraction, generation, field registry, SF export.

### 4. Clause Schema: Expanded

**v1 spec (10 fields):**
```
clause_id, title, body, contract_types, section, risk_level,
variables, triggers, approved_by, last_reviewed
```

**v2 actual (15+ fields):**
```json
{
  "clause_id": "GEN-RECITALS-DIST-V1",
  "clause_type": "PARTIES_IDENTIFICATION",
  "version": 1,
  "contract_types": ["distribution"],
  "section": "recitals",
  "risk_level": "standard",
  "body": "This Distribution Agreement (\"Agreement\") is entered into as of {{OPP_EFFECTIVE_DATE}} by and between {{ACCT_LEGAL_ENTITY}} (\"Label\") and {{ACCT_COUNTERPARTY}} (\"Artist\").",
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

New fields that matter for Airlock:
- `clause_type` — groups clauses by function (PARTIES_IDENTIFICATION, TERM_DURATION, etc.) — 101 types
- `version` — clause versioning (supports V1, V2, etc. per clause)
- `variables[].extraction` — the unified bridge (extraction config lives ON the clause)
- `variables[].canonical_key` — SF-compatible field key (e.g., `Effective_Date__c`)
- `generation_config` — priority + fallback text
- `extraction_config` — tier, preflight weight, section anchors
- `cuad_labels` — CUAD taxonomy tags (79 labels) for benchmarking

### 5. Clause ID Convention: `DIST-001` → `GEN-RECITALS-DIST-V1`

**v1 spec says:** Type prefix: `DIST-001`, `LIC-001`, `REC-001`

**v2 actual:** Category-description-version: `GEN-RECITALS-DIST-V1`, `RISK-CONFIDENTIALITY-V1`, `CUAD-TERMINATION-CONVENIENCE-V1`

Categories: `GEN` (generation), `RISK` (risk clauses), `CUAD` (CUAD-aligned), `MUSIC`, `FILM`, `TV`, `BOILER` (boilerplate)

### 6. Section Types: 4 → 2 (simpler)

**v1 spec says:** 4 types: Boilerplate, Field-Driven, Conditional, Composite

**v2 actual:** 2 types in templates: `boilerplate` (fixed text, may reference a default clause) and `conditional` (populated by matching clauses from the library based on trigger evaluation)

The old "Field-Driven" and "Composite" distinctions are handled naturally by the trigger system — a clause with one trigger is effectively field-driven, a clause with multiple triggers is effectively composite. No need for separate type labels.

### 7. New Concept: Extraction Bridge

**v1:** Extraction was not part of the ContractGenerator spec at all.

**v2:** The unified library means the ContractGenerator's clause data IS also the extraction system's configuration. The `enrich_from_clause_library()` function reads every variable's `extraction` block and synthesizes extraction config entries. This is a key selling point — when a legal team approves a new clause in the Generator, extraction coverage for those fields automatically improves.

This should be reflected in the Feature Control Plane and Admin specs.

---

## Files That Need Updating

### Priority 1: Direct v1 → v2 Rewrites

#### `Features/ContractGenerator/overview.md`
The most impacted file. Needs these changes:
- **Contract Type Selector** (line 35-44): Expand from 8 types to 24, grouped by vertical
- **Generation Pipeline** (lines 86-96): Remove "Load generation pairs" step. Replace with: Load clause library → Build clause index → Select clauses via trigger evaluation → Interpolate `{{variables}}` → Assemble sections
- **Section Types** (lines 99-109): Simplify from 4 types to 2 (`boilerplate`, `conditional`)
- **Clause Library schema** (lines 117-130): Replace with v2 schema (add clause_type, version, extraction block in variables, generation_config, extraction_config, cuad_labels)
- **Generation Pairs section** (lines 141-165): Remove entirely. Replace with explanation of trigger-based clause selection
- **Placeholder syntax**: All `[PLACEHOLDER]` references → `{{CHECK_CODE}}` syntax
- **Source Mapping table** (lines 11-14): Update engine.py from 138 lines to 453 lines. Add clause_library_v2.json reference. Remove generation_pairs.json reference.
- **Feature Control Plane section** (lines 169-177): Remove "Generation pair rules" admin UI. Add clause library admin with extraction preview.

#### `claude.md` — Clause Library Agent Instructions (lines 167-262)
- **Contract types** (line 198): 8 → 24 types with vertical groupings
- **Generation pairs count** (line 202): Remove "93 generation pair rules" reference
- **Clause schema table** (lines 185-196): Replace with v2 schema fields
- **Placeholder syntax** (line 257): `[PLACEHOLDER]` → `{{CHECK_CODE}}`
- **Clause ID convention** (line 261): `DIST-001` → `GEN-RECITALS-DIST-V1` format
- **"What To Write" template** (lines 206-252): Remove Generation Pairs section. Add extraction config documentation. Update clause table columns.
- **Repo structure** (line 24): Update ContractGenerator description to remove "generation pairs"
- **Source directories** (line 96): Update to reference clause_library_v2.json specifically

### Priority 2: Cross-Reference Updates

#### `Features/start.md` (line 10)
- Change: `8 contract types, clause library` → `24 contract types (5 verticals), unified clause library v2`

#### `Features/DocumentSuite/overview.md` (line 60)
- Clause Block description references "clause library" — still valid but should note v2 schema and `{{variable}}` placeholder syntax for rendered clause previews

### Priority 3: Demo HTML (visual mockups — lower priority)

#### `demo-shell/index.html` (line 3134)
- References "8 contract types" — update to 24

#### `Features/DocumentSuite/document-suite-demo.html` (lines 1084, 1109, 1161)
- Uses `CL-DIST-001`, `CL-FIN-003`, `CL-TERM-002` clause ID format — update to v2 convention if these demos will be used as implementation reference

---

## Reference Documents

The full v2 architecture is documented in these files (in the Ostereo_PDFs docs repo):

| Document | Location | What It Covers |
|----------|----------|----------------|
| BUILD_PLAN.md | `/Ostereo_PDFs/BUILD_PLAN.md` | Complete rebuild plan, file reference map, build sequence |
| CLAUSE_LIBRARY.md | `/Ostereo_PDFs/CLAUSE_LIBRARY.md` | v2 clause schema, coverage tables, naming conventions, trigger system |
| GENERATION_PIPELINE.md | `/Ostereo_PDFs/GENERATION_PIPELINE.md` | All 7 generation modules, template schema, variation engine |
| EXTRACTION_PIPELINE.md | `/Ostereo_PDFs/EXTRACTION_PIPELINE.md` | 6 extractors, anchor-proximity pattern, clause library bridge |
| FIELD_REGISTRY.md | `/Ostereo_PDFs/FIELD_REGISTRY.md` | 442 field definitions, 9 data sheets, check code naming |

The source data files live in:
- `OrcestrateOS-sync/rules/generation/clause_library_v2.json` — 188 clauses, 388KB
- `OrcestrateOS-sync/rules/generation/templates/*.json` — 14 explicit templates
- `OrcestrateOS-sync/rules/rules_bundle/field_meta.json` — 442 field definitions

---

## What I Need Back

This brief gives you the full context of what changed and where it affects the Airlock specs. What I need returned:

1. **Updated versions of the affected files** (or precise edit instructions) — especially `overview.md` and the `claude.md` Clause Library Agent Instructions section
2. **Any cascading impacts** you find in other feature specs that I may have missed (e.g., does BatchProcessing reference the old clause system? Does the AI Agent spec assume generation pairs?)
3. **Decisions on open questions:**
   - Should the Contract Type Selector show all 24 types flat, or grouped by vertical?
   - Should the v2 extraction bridge be surfaced in the Generator UI (e.g., "this clause also configures extraction for these fields")?
   - Should `clause_type` (101 types) be exposed in the clause picker, or is filtering by section + contract_type sufficient?

---

## Related Specs

- [overview.md](./overview.md) -- Parent feature spec (needs v2 rewrite)
- [/Ostereo_PDFs/CLAUSE_LIBRARY.md](/Users/zacharyholwerda/Desktop/New%20Folder%20With%20Items/Ostereo_PDFs/CLAUSE_LIBRARY.md) -- Full v2 clause library reference
- [/Ostereo_PDFs/GENERATION_PIPELINE.md](/Users/zacharyholwerda/Desktop/New%20Folder%20With%20Items/Ostereo_PDFs/GENERATION_PIPELINE.md) -- v2 generation pipeline reference
