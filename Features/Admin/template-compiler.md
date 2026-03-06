# Template Compiler — Admin Tool

> **Status:** SPECCED — Admin panel for auto-generating extraction configs from sample documents.

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

## LLM Prompt Templates (Per Pipeline Step)

Each pipeline step uses a structured prompt optimized for local LLMs (8B parameter range). Prompts use XML-delimited sections for reliable parsing.

### Step 2: Document Structure Analysis

```
<system>
You are a contract analysis assistant. Your job is to identify the structural
sections, terminology patterns, and data fields present in legal documents.
Respond ONLY with valid JSON matching the schema below. No commentary.
</system>

<task>
Analyze this contract document and identify:
1. Major sections/headings and their approximate character positions
2. Parties to the agreement (legal entities)
3. Key terminology patterns unique to this document
4. Date formats used (MM/DD/YYYY, Month DD, YYYY, etc.)
5. Currency/number formats used
6. Contract type classification

Output schema:
{
  "sections": [{"heading": str, "start_char": int, "end_char": int}],
  "parties": [{"name": str, "role": "licensor"|"licensee"|"vendor"|"client"|"other"}],
  "terminology": [{"term": str, "canonical": str, "frequency": int}],
  "date_formats": [str],
  "number_formats": [str],
  "contract_type": str,
  "contract_subtype": str,
  "confidence": float
}
</task>

<document>
{document_text}
</document>
```

**Chunking strategy:** Documents longer than 6,000 tokens are split into overlapping chunks (4,000 tokens with 500-token overlap). Section boundaries are merged across chunks using heading text matching.

### Step 3: Field Discovery + Anchor Generation

Run once per target field, batched in groups of 10 fields per prompt to reduce LLM calls:

```
<system>
You are an extraction anchor generator. Given contract text and a list of
target fields, identify reliable anchor phrases that appear near each field's
value. Respond ONLY with valid JSON.
</system>

<task>
For each field below, find anchor patterns in the contract text:

Fields to find:
{fields_json}
  // e.g. [{"check_code": "territory", "label": "Territory", "data_type": "text"},
  //       {"check_code": "effective_date", "label": "Effective Date", "data_type": "date"}]

For each field, provide:
- primary_anchors: phrases that reliably precede or contain the value (2-5 anchors)
- secondary_anchors: weaker but still useful phrases (0-3 anchors)
- negative_anchors: phrases that cause false positives (0-3 anchors)
- proximity_chars: typical distance between anchor and value (50-500)
- value_pattern: regex-like description of the value format
- found_values: actual values found in this document
- confidence: 0.0-1.0 how reliable these anchors are

Output schema:
{
  "fields": [{
    "check_code": str,
    "primary_anchors": [str],
    "secondary_anchors": [str],
    "negative_anchors": [str],
    "proximity_chars": int,
    "value_pattern": str,
    "found_values": [str],
    "confidence": float
  }]
}
</task>

<document>
{document_text}
</document>
```

**Multi-document consensus:** After running against all sample documents, anchors are scored by hit rate. An anchor that works in 18/20 documents scores 0.90. Only anchors with hit rate >= 0.60 are included in the final config.

### Step 4: Synonym Detection

```
<system>
You are a legal terminology mapper. Given contract text and a list of
canonical field names with their standard values, identify any synonyms,
abbreviations, or alternative phrasings the customer uses.
Respond ONLY with valid JSON.
</system>

<task>
Compare this document's terminology against the canonical vocabulary:

Canonical vocabulary:
{canonical_terms_json}
  // e.g. {"territory": ["Worldwide", "North America", "Europe", ...],
  //       "payment_terms": ["Net 30", "Net 60", "Upon Receipt", ...]}

Identify:
- Terms used in this document that mean the same as canonical values
- Abbreviations or shorthand (e.g., "NA" for "North America")
- Industry-specific jargon the customer uses
- Phrases that should map to boolean yes/no signals

Output schema:
{
  "synonym_groups": [{
    "check_code": str,
    "canonical_value": str,
    "customer_synonyms": [str],
    "context_example": str
  }],
  "boolean_signals": [{
    "check_code": str,
    "yes_signals": [str],
    "no_signals": [str]
  }],
  "unmapped_terms": [str]
}
</task>

<document>
{document_text}
</document>
```

### Step 5: Config Generation (Deterministic — No LLM)

Config generation is **not** an LLM step. It's deterministic aggregation:

```python
def aggregate_configs(analyses: list[dict], field_results: list[dict],
                      synonyms: list[dict]) -> ConfigBundle:
    """Merge per-document LLM outputs into final extraction config."""

    # 1. Anchors: keep only those with cross-document hit rate >= 0.60
    # 2. Proximity: use median proximity_chars across documents
    # 3. Synonyms: union all customer synonyms, deduplicate
    # 4. Boolean signals: union yes/no signals, remove contradictions
    # 5. Contract type rules: majority-vote classification across documents
    # 6. Value patterns: most common pattern per field

    return ConfigBundle(
        extraction_anchors=merged_anchors,
        picklist_synonyms=merged_synonyms,
        boolean_signals=merged_signals,
        contract_type_rules=merged_rules,
        field_mapping=coverage_map
    )
```

---

## Accuracy Evaluation Framework

### Confidence Scoring

Each generated config field receives a confidence score based on cross-document consistency:

| Metric | Formula | Weight |
|--------|---------|--------|
| **Anchor hit rate** | docs_with_anchor / total_docs | 0.35 |
| **Value consistency** | unique_values / expected_unique (inverted for enums) | 0.25 |
| **Proximity stability** | 1 - (std_dev / mean) of proximity_chars | 0.20 |
| **Cross-field interference** | 1 - false_positive_rate | 0.20 |

**Per-field confidence = weighted sum of the 4 metrics.**

### Quality Thresholds

| Confidence | Label | Action |
|-----------|-------|--------|
| >= 0.80 | HIGH | Auto-include in config. Green indicator in UI. |
| 0.60-0.79 | MEDIUM | Include with warning. Amber indicator. Human review recommended. |
| 0.40-0.59 | LOW | Include as draft only. Red indicator. Requires manual anchor editing. |
| < 0.40 | SKIP | Exclude from config. Gray indicator. Admin must add manually. |

### Minimum Viability Threshold

A generated config bundle is considered **viable** when:

| Criterion | Threshold | Rationale |
|-----------|-----------|-----------|
| Fields with HIGH confidence | >= 40% of target fields | Enough coverage to be useful |
| Fields with any confidence >= MEDIUM | >= 60% of target fields | Majority of fields detectable |
| Zero fields detected | FAIL — reject entirely | No point deploying an empty config |
| Average confidence across all fields | >= 0.55 | Overall quality floor |
| At least 1 primary anchor per included field | 100% | No anchor = no extraction |

If the config fails viability, the UI shows:

```
⚠ Config below minimum quality threshold

  Fields detected: 34/152 (22%) — need >= 60%
  Average confidence: 0.48 — need >= 0.55

  Possible causes:
  • Sample documents may be too diverse (mix of contract types)
  • Document quality is low (scanned/OCR'd with errors)
  • Target fields aren't present in this contract type

  Actions:
  [Upload More Samples] [Narrow Target Fields] [Select Specific Contract Type]
```

### Validation Run Scoring

Step 6 (validation run) processes every sample document through the generated config and compares results:

| Metric | Calculation |
|--------|-------------|
| **True Positive Rate** | Fields correctly extracted with correct value / total extractable |
| **False Positive Rate** | Fields extracted with wrong value / total attempted |
| **Coverage** | Fields with any extraction / total target fields |
| **Precision** | Correct extractions / total extractions |
| **Recall** | Correct extractions / total extractable values |

The validation panel shows per-field breakdown (already designed in the Test Results UI wireframe above) plus aggregate F1 score.

---

## Edge Case Handling

### Sparse Documents

**Problem:** Some documents have very few of the 152 target fields (e.g., an amendment that only changes effective date and territory).

**Solution:**
- Auto-detect document "density" — count of unique field matches / total target fields
- If density < 0.15, classify as "sparse document" and exclude from anchor consensus calculations for missing fields
- Don't penalize anchor confidence for fields that legitimately don't appear in sparse docs
- UI shows: "12 of 23 documents are sparse (< 15% field coverage). Anchor confidence is based on the 11 dense documents."

### Conflicting Anchors

**Problem:** The same anchor phrase leads to different fields in different documents (e.g., "Amount" could mean advance_amount or royalty_rate).

**Solution:**
- Detect conflicts: if anchor A appears in both field X and field Y results with hit rate > 0.30 for each
- Resolution order:
  1. Check anchor proximity — shorter proximity wins (anchor is closer to the field it belongs to)
  2. Check value pattern — if the value after the anchor is always a date for field X but always a currency for field Y, the pattern disambiguates
  3. If still ambiguous — mark both fields as LOW confidence, flag conflict in UI for human resolution

### Inconsistent Document Structure

**Problem:** Sample documents don't follow the same template (mix of contract types, or same type with different layouts).

**Solution:**
- Step 2 classifies each document by type. If > 2 types detected, prompt the admin:
  ```
  ⚠ Multiple contract types detected in sample batch:
  • Distribution Agreement (14 documents)
  • License Agreement (6 documents)
  • Service Contract (3 documents)

  [Generate separate configs per type (Recommended)]
  [Generate one combined config]
  [Remove outliers and re-analyze]
  ```
- If combined: anchors must hit in >= 60% of documents of their classified type, not all documents

### OCR Quality Issues

**Problem:** Scanned documents produce noisy text that breaks anchor matching.

**Solution:**
- Pre-screen all documents: check OCR confidence if available (from multi-format pipeline)
- Documents with avg OCR confidence < 60%: exclude from anchor training, include in validation only
- If > 50% of sample batch is low-quality OCR: warn admin that results will be unreliable
- UI shows per-document quality badge: GREEN (clean text), AMBER (minor OCR issues), RED (major OCR issues)

### Field Not Present in Any Document

**Problem:** Admin selected "All 152 fields" but the contract type simply doesn't have fields like `addon_type` or `schedule_presence`.

**Solution:**
- After analysis, report fields as "Not Applicable" (distinct from "Not Found")
- A field is "Not Applicable" if 0 of the sample documents contain any anchor or value match
- UI groups results: Detected (87) | Not Found (12) | Not Applicable (53)
- "Not Applicable" fields are excluded from the config entirely (no stub entry)
- "Not Found" fields get LOW confidence entries — they might exist but anchors weren't discovered

---

## Extraction Pipeline Integration

### Config File Hierarchy

The extraction pipeline loads configs in priority order (later overrides earlier):

| Priority | Config Source | Scope | Managed By |
|----------|-------------|-------|-----------|
| 1 (base) | `rules/extraction_anchors.json` | Global default | Code repo |
| 2 | Workspace config bundle (Template Compiler output) | Per-workspace | Admin via Template Compiler |
| 3 | Manual field overrides (admin inline edits) | Per-workspace per-field | Admin via Extraction Config panel |

**Merge behavior:**
- **Anchors:** Workspace config adds to (not replaces) global anchors. If a workspace anchor conflicts with a global one, workspace wins.
- **Synonyms:** Workspace synonyms are additive. Customer-specific terms layer on top of global synonyms.
- **Boolean signals:** Workspace signals are additive. Same merge logic as synonyms.
- **Contract type rules:** Workspace rules can add new contract types but cannot remove global ones.
- **Field mappings:** Workspace can disable fields (`enabled: false`) but cannot add fields outside the master schema.

### Config Deployment Process

```python
def deploy_config(workspace_id: str, bundle: ConfigBundle,
                  deployed_by: str) -> ConfigVersion:
    """Deploy a Template Compiler output to a workspace."""

    # 1. Validate bundle against master schema
    validate_field_codes(bundle, master_schema)

    # 2. Create versioned snapshot
    version = create_config_version(
        workspace_id=workspace_id,
        bundle=bundle.to_json(),
        parent_version=get_active_version(workspace_id),
        created_by=deployed_by,
        source='template_compiler'
    )

    # 3. Activate (swap pointer, old config remains for rollback)
    activate_config_version(workspace_id, version.id)

    # 4. Emit audit event
    emit_event('config.deployed', {
        'workspace_id': workspace_id,
        'version': version.id,
        'field_count': len(bundle.fields),
        'avg_confidence': bundle.avg_confidence,
        'deployed_by': deployed_by
    })

    # 5. Invalidate extraction cache for this workspace
    invalidate_extraction_cache(workspace_id)

    return version
```

### Config Version Table

```sql
CREATE TABLE extraction_config_versions (
    id TEXT PRIMARY KEY,                    -- prefix: ecv_
    workspace_id UUID NOT NULL REFERENCES workspaces(id),
    version_number INT NOT NULL,
    bundle JSONB NOT NULL,                  -- full config bundle
    source TEXT NOT NULL,                   -- 'template_compiler' | 'manual' | 'import'
    parent_version_id TEXT REFERENCES extraction_config_versions(id),
    is_active BOOLEAN NOT NULL DEFAULT false,
    field_count INT NOT NULL,
    avg_confidence FLOAT,
    validation_results JSONB,              -- F1, precision, recall from validation run
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (workspace_id, version_number)
);

CREATE INDEX idx_ecv_active ON extraction_config_versions(workspace_id)
    WHERE is_active = true;
```

### Rollback Mechanism

```
Active Config: Acme Corp Distribution v3 (deployed Mar 1)

Version History:
  v3 ● Active   — 87 fields, 0.78 avg confidence   [Rollback to v2]
  v2 ○ Previous — 82 fields, 0.74 avg confidence   [Activate]
  v1 ○ Initial  — 71 fields, 0.69 avg confidence   [Activate]

  [Compare v2 ↔ v3] — shows diff of changed anchors/synonyms
```

Rollback is instant — swaps the `is_active` flag. No data loss, no re-processing needed. The extraction pipeline reads the active version on each run.

---

## VRAM Requirements and Model Selection

| Model | Parameters | Quantization | VRAM Required | Speed (tokens/s) | Quality |
|-------|-----------|-------------|--------------|-------------------|---------|
| Llama 3.1 8B | 8B | Q4_K_M | 6 GB | 30-50 | Good for structured extraction |
| Llama 3.1 8B | 8B | Q8_0 | 10 GB | 20-35 | Better accuracy, higher VRAM |
| Mistral 7B | 7B | Q4_K_M | 5 GB | 35-55 | Fast, good for anchor detection |
| Qwen 2.5 7B | 7B | Q4_K_M | 5 GB | 30-45 | Strong on structured JSON output |
| Llama 3.1 70B | 70B | Q4_K_M | 42 GB | 5-10 | Best quality, requires serious GPU |

**Default recommendation:** Llama 3.1 8B at Q4_K_M quantization. Fits in 6 GB VRAM (runs on most modern GPUs including RTX 3060 and above). For CPU-only deployments, same model runs at ~5 tokens/s — usable but slow for 50-document batches.

**Docker deployment:**
```yaml
ollama:
  image: ollama/ollama:latest
  ports:
    - "11434:11434"
  volumes:
    - ollama_data:/root/.ollama
  deploy:
    resources:
      reservations:
        devices:
          - driver: nvidia
            count: 1
            capabilities: [gpu]
```

**CPU fallback:** If no GPU detected, the Template Compiler shows a warning: "Running on CPU — analysis will take approximately 3-5x longer. Consider using a GPU-equipped machine for large batches."

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
