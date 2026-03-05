# Template Compiler — Admin Tool Brainstorm

> **Status:** BRAINSTORM — Admin panel for auto-generating extraction configs from sample documents.

> **Key Insight:** Onboarding a new customer currently requires manually creating extraction anchors, picklist synonyms, and field mappings. The Template Compiler automates this: feed it a batch of sample contracts, a local LLM analyzes them, and out comes a ready-to-use extraction config.

---

## The Problem

Setting up extraction for a new customer or contract type is manual and slow:
1. An architect reviews sample contracts to understand the document structure
2. They manually create anchor patterns for each field (primary, secondary, negative)
3. They configure proximity windows, confidence thresholds, and value patterns
4. They add picklist synonyms for terminology the customer uses
5. They test, iterate, adjust — rinse and repeat for each contract type

This process takes hours to days per contract type. The Template Compiler reduces it to minutes.

---

## How It Works

### Input
- **Document source**: S3 bucket URL, Google Drive folder, or direct upload (batch of 5-50 sample documents)
- **Document type hint** (optional): "distribution agreement", "license agreement", "service contract", etc.
- **Target fields** (optional): Which of the 152 master schema fields to look for, or "auto-detect all"

### Process

```
Sample Documents (S3 / Drive / Upload)
    |
    v
[1] Format Detection + Text Extraction
    |  (multi-format pipeline: PDF, DOCX, XLSX, etc.)
    |
    v
[2] Document Structure Analysis (Local LLM)
    |  "What sections does this contract have?"
    |  "Where are dates, amounts, parties mentioned?"
    |  "What terminology/jargon is used?"
    |
    v
[3] Field Discovery + Anchor Generation
    |  For each target field:
    |    - Find anchor phrases that reliably precede the value
    |    - Identify negative anchors (phrases that cause false matches)
    |    - Calculate optimal proximity window
    |    - Detect value patterns (date formats, currency formats, etc.)
    |
    v
[4] Synonym Detection
    |  "This customer says 'Advance' instead of 'Payment'"
    |  "They use 'Territory' and 'Region' interchangeably"
    |  Build picklist synonym mappings
    |
    v
[5] Config Generation
    |  Output:
    |    - extraction_anchors.json (per-field anchors + patterns)
    |    - picklist_synonyms.json (customer-specific terminology)
    |    - boolean_signals.json (yes/no indicators)
    |    - contract_type_rules.json (document classification rules)
    |
    v
[6] Validation Run
    |  Run the generated config against the sample docs
    |  Show extraction results vs expected (human review)
    |
    v
[7] Deploy Config
    Deploy to workspace as a versioned config bundle
    Feature Control Plane tracks the active config version
```

### Output

A complete extraction config bundle:

| File | Content | Generated How |
|------|---------|--------------|
| `extraction_anchors.json` | Per-field primary/secondary/negative anchors, proximity_chars, extraction_type | LLM identifies reliable anchor patterns across documents |
| `picklist_synonyms.json` | Canonical values + customer-specific synonyms | LLM maps customer terminology to master schema |
| `boolean_signals.json` | Yes/no signal phrases for toggle fields | LLM identifies affirmative/negative language patterns |
| `contract_type_rules.json` | Keywords, expected schedules, subtypes per contract category | LLM classifies document types and their characteristics |
| `field_mapping.json` | Which master schema fields are relevant for this customer | LLM + coverage analysis determines applicable fields |

---

## Admin UI

### Location

Workspace Admin > Connectors > Template Compiler (or a new "Extraction Config" section)

### Workflow

```
+------------------------------------------------------------------+
| TEMPLATE COMPILER                                                  |
|------------------------------------------------------------------|
|                                                                    |
| Step 1: SOURCE                                                     |
| [S3 Bucket URL    ] [Google Drive Link] [Upload Files]            |
| Detected: 23 files (18 PDF, 3 DOCX, 2 XLSX)                      |
|                                                                    |
| Step 2: ANALYZE                                                    |
| Contract type: [Auto-detect ▼] or [Distribution Agreement ▼]      |
| Target fields: [All 152 fields ▼] or [Select specific fields]     |
| LLM Model: [otto-local (Llama 3.1) ▼]  ← Uses local model        |
|                                                                    |
| [Run Analysis]                                                     |
|                                                                    |
| Step 3: REVIEW RESULTS                                             |
| ┌─────────────────────────────────────────────────────────────┐   |
| │ Fields Detected: 87/152                                      │   |
| │ Anchor Confidence: 78% avg                                   │   |
| │ Synonym Groups: 14 found                                     │   |
| │ Contract Types: 3 detected (distribution, license, service)  │   |
| │                                                              │   |
| │ [Preview Config] [Edit Config] [Test Against Samples]        │   |
| └─────────────────────────────────────────────────────────────┘   |
|                                                                    |
| Step 4: DEPLOY                                                     |
| Config name: [Acme Corp Distribution v1    ]                       |
| [Deploy to Workspace] [Export as ZIP] [Save as Draft]             |
|                                                                    |
+------------------------------------------------------------------+
```

### Config Preview Panel

Shows the generated JSON with syntax highlighting. Each section is collapsible. Editable inline — the admin can tweak anchors, add synonyms, adjust thresholds before deploying.

### Test Results Panel

After running the generated config against the sample documents:

```
+------------------------------------------------------------------+
| TEST RESULTS (23 documents)                                        |
|------------------------------------------------------------------|
| Overall Coverage: 74%                                              |
| Avg Confidence: 0.72                                               |
|                                                                    |
| By Field:                                                          |
| ✓ territory        23/23  avg: 0.89  anchors working well         |
| ✓ effective_date   22/23  avg: 0.85  1 doc uses non-standard fmt  |
| ~ counterparty     18/23  avg: 0.64  needs more synonyms          |
| ✗ advance_amount    8/23  avg: 0.41  anchor too generic           |
|                                                                    |
| [Re-run with Adjustments] [Accept & Deploy]                        |
+------------------------------------------------------------------+
```

---

## Local LLM Requirement

The Template Compiler deliberately uses a **local LLM** (via LiteLLM → Ollama):
- **Privacy**: Customer contracts may be sensitive. Processing stays on-prem.
- **Cost**: Analyzing 50 documents with a cloud LLM would cost $10-50. Local is free per-query.
- **Speed**: Local inference for structured extraction tasks is fast enough (5-15s per doc).
- **Model**: `otto-local` alias (Ollama/Llama 3.1 or similar). Configured in LiteLLM gateway.

The analysis pipeline sends structured prompts like:
```
Given this contract text, identify:
1. All anchor phrases that appear near [field_name] values
2. The typical proximity (in characters) between anchor and value
3. Any terminology unique to this customer for [field_name]
4. The value format/pattern for [field_name]

Contract text:
[...document text...]
```

---

## Versioning and Rollback

- Each generated config is versioned (v1, v2, v3...)
- Workspace can have multiple configs for different contract types
- Feature Control Plane tracks active config version per workspace
- Rollback: one-click revert to previous config version
- A/B testing: run new config on 10% of incoming documents, compare results

---

## Integration Points

| System | Integration |
|--------|------------|
| **S3** | Read files from customer-provided bucket (presigned URLs or IAM role) |
| **Google Drive** | OAuth connection via Connectors panel, read files from shared folder |
| **LiteLLM Gateway** | Route analysis prompts to otto-local (Ollama) |
| **Feature Control Plane** | Track config versions, toggle per-workspace, audit config changes |
| **Extraction Pipeline** | Deploy generated configs as the active extraction rules |
| **Batch Processor** | Run validation against sample docs using generated config |

---

## Capacity Planning

| Metric | Value | Notes |
|--------|-------|-------|
| Analysis time per document | 5-15s (local LLM) | Depends on document length and model |
| Full batch analysis (50 docs) | 5-12 minutes | Parallelized with semaphore(5) |
| Generated config size | 50-200KB | Depends on field count and synonym depth |
| Local LLM memory requirement | 8-16GB VRAM | Llama 3.1 8B quantized |

---

## Related Specs

- **[Feature Control Plane](../FeatureControlPlane/overview.md)** — Config versioning and toggles
- **[LiteLLM Gateway](../AIAgent/litellm-gateway.md)** — Local model routing
- **[Multi-Format Support](../DocumentSuite/multi-format-support.md)** — Document format detection and conversion
- **[Admin / Settings](./overview.md)** — Admin overlay where this tool lives
