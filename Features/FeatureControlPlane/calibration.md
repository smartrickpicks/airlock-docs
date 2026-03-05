# Calibration -- Design Spec

## Purpose

Expose thresholds, weights, and confidence scores as **admin-editable parameters** stored in the database per workspace. No hardcoded magic numbers -- every tunable value is surfaced as a slider or input in the Admin UI with full audit history.

---

## Calibration Record Schema

Each calibration parameter is stored as a record:

| Field | Type | Description |
|---|---|---|
| `calibration_key` | string | Unique identifier (e.g., `preflight.mojibake_threshold`) |
| `display_name` | string | Human-readable label for Admin UI |
| `feature` | string | Parent feature (e.g., `preflight`, `identity_resolver`) |
| `value` | number | Current calibrated value |
| `default_value` | number | Factory default for reset |
| `min` | number | Minimum allowed value |
| `max` | number | Maximum allowed value |
| `step` | number | Slider increment (e.g., 0.01, 1) |
| `unit` | string | Display unit (e.g., `%`, `score`, `count`, `seconds`) |
| `workspace_id` | string | Workspace this calibration applies to |
| `last_calibrated_at` | timestamp | When this value was last changed |
| `calibrated_by` | string | User ID of last calibrator |

---

## Calibration UI Requirements

- Each calibration parameter renders as a **slider with numeric input** in the Admin UI.
- The slider respects `min`, `max`, and `step` from the record.
- Below each parameter, display: **"Last calibrated: {timestamp} by {user}"**.
- A "Reset to Default" button restores `default_value`.
- Every change emits a calibration audit event before persisting.
- Changes take effect immediately (no restart).
- **Impact description** (required): Each parameter shows a plain-English explanation of what changing this value does. Displayed as muted text below the slider. Must answer: "What happens if I raise this? What happens if I lower it?"
- **Live preview** (where possible): For thresholds, show how many current documents would be affected by the change (e.g., "At 0.80: 142 fields pass. At 0.70: 167 fields pass (+25)").

---

## Features with Calibration Parameters

### Extractors

| Parameter | Key | Default | Range | Unit |
|---|---|---|---|---|
| Tier-1 confidence threshold | `extractor.tier1.confidence` | 0.80 | 0.50 - 1.00 | score |
| Tier-2 confidence threshold | `extractor.tier2.confidence` | 0.65 | 0.30 - 1.00 | score |
| Anchor pattern match sensitivity | `extractor.anchor_sensitivity` | 0.75 | 0.50 - 1.00 | score |

#### Impact Descriptions

| Parameter | What It Controls | Raise It | Lower It |
|---|---|---|---|
| **Tier-1 confidence** | Minimum confidence score for a Tier-1 field (critical fields like territory, term, effective date) to be marked as "pass". Tier-1 fields are high-value — these anchor the contract's identity. | Stricter: more fields flagged for human review. Fewer false positives, but more manual work. | Looser: more fields auto-pass. Faster throughput, but risk accepting low-confidence extractions. |
| **Tier-2 confidence** | Minimum confidence for Tier-2 fields (secondary fields like schedule type, addon indicators). These support the core fields but aren't deal-breakers alone. | Stricter: secondary fields get more scrutiny. Good for high-stakes contracts. | Looser: secondary fields auto-pass more easily. Good for high-volume, lower-risk workflows. |
| **Anchor sensitivity** | How closely an extracted value must match its anchor pattern (the text phrase that signals where a value should be). Higher = requires nearly exact anchor match. | Requires precise anchor matches. Fewer false matches, but may miss values when document phrasing varies. | Accepts looser anchor matches. Catches more values, but may match wrong sections of the document. |

### Preflight

#### Gate Thresholds

| Parameter | Key | Default | Range | Unit |
|---|---|---|---|---|
| Mojibake RED threshold | `preflight.mojibake_red` | 5.0 | 0.0 - 20.0 | % |
| Control character RED threshold | `preflight.control_red` | 3.0 | 0.0 - 10.0 | % |

**Mojibake RED threshold:** Percentage of garbled/corrupted characters in the document text. If the document exceeds this percentage, preflight triggers a RED hard stop — the document is too corrupted for reliable extraction. *Raise it* to tolerate more corruption (e.g., for scanned PDFs with OCR noise). *Lower it* for stricter quality requirements.

**Control character RED threshold:** Percentage of non-printable control characters (tab, null, bell, etc.) in the text. High control char ratios indicate binary content or corrupted encoding. *Raise it* to be more lenient with messy documents. *Lower it* to block documents with encoding issues early.

#### Section Weights

Section weights must sum to 1.00. The UI enforces this constraint by rebalancing when one weight is adjusted.

| Section | Key | Default Weight |
|---|---|---|
| Document Quality | `preflight.weight.doc_quality` | 0.25 |
| Structural Integrity | `preflight.weight.structural` | 0.15 |
| Extraction Coverage | `preflight.weight.extraction` | 0.20 |
| Entity Resolution | `preflight.weight.entity` | 0.25 |
| Consistency | `preflight.weight.consistency` | 0.15 |

**How section weights work:** These weights determine how much each quality dimension contributes to the overall health score. All 5 weights must sum to 1.00. The UI enforces this — adjusting one weight automatically rebalances the others.

| Section | What It Measures | When to Increase Weight |
|---|---|---|
| **Document Quality** | Text readability: mojibake, control chars, OCR quality, page char density | When processing many scanned/OCR'd documents with quality issues |
| **Structural Integrity** | Document structure: sections present, expected schedule types found, page count reasonable | When contract structure matters more than individual field accuracy |
| **Extraction Coverage** | How many target fields were successfully extracted with acceptable confidence | When field completeness is the primary quality indicator |
| **Entity Resolution** | Counterparty identification confidence, legal entity match score, agreement type detection | When customer identity is critical (e.g., regulated industries) |
| **Consistency** | Cross-field validation: do dates make sense together? Do amounts add up? | When logical consistency between fields matters most |

### Identity Resolver

#### Match Thresholds

| Parameter | Key | Default | Range | Unit |
|---|---|---|---|---|
| Fuzzy match threshold | `identity.fuzzy_threshold` | 0.55 | 0.30 - 1.00 | score |
| Ambiguous match threshold | `identity.ambiguous_threshold` | 0.80 | 0.50 - 1.00 | score |

#### Scoring Weights

| Component | Key | Default Weight | Range |
|---|---|---|---|
| Name similarity | `identity.weight.name` | 0.85 | 0.00 - 1.00 |
| Address similarity | `identity.weight.address` | 0.00 | 0.00 - 1.00 |
| Context similarity | `identity.weight.context` | 0.20 | 0.00 - 1.00 |

**How entity resolution scoring works:** The composite score formula is `(name_score * name_weight) + (address_score * address_weight) + (context_score * context_weight) - service_penalty`. This determines whether an extracted entity (counterparty, legal name) matches an existing account in the system.

| Component | What It Measures | Impact |
|---|---|---|
| **Fuzzy match threshold** | Minimum composite score to consider a match valid. Below this = "not found". | *Raise*: fewer false matches, more "unknown" entities needing manual review. *Lower*: more auto-matches, risk of matching wrong entities. |
| **Ambiguous match threshold** | Score above which a match is considered definitive (no human review needed). Between fuzzy and ambiguous = "review needed". | *Raise*: more matches flagged for human confirmation. *Lower*: more matches auto-confirmed. |
| **Name similarity weight** | How much the entity name match contributes. Default 0.85 = name is the dominant signal. | *Raise*: name match matters more. *Lower*: other signals (address, context) gain influence. |
| **Address similarity weight** | How much address/location match contributes. Default 0.00 = not used (many contracts lack addresses). | *Raise*: address becomes a factor. Good when entities have known addresses in the system. |
| **Context similarity weight** | How much surrounding text context contributes (account numbers, prior references, deal history). | *Raise*: contextual clues matter more. Good for repeat customers with established patterns. |

### Health Scoring

| Parameter | Key | Default | Range | Unit |
|---|---|---|---|---|
| Green band threshold | `health.green_threshold` | 0.95 | 0.80 - 1.00 | score |
| Yellow band threshold | `health.yellow_threshold` | 0.80 | 0.50 - 0.95 | score |
| Calibration model type | `health.calibration_model` | identity | -- | enum |

#### Calibration Model Options

| Model | Description |
|---|---|
| `identity` | Raw scores passed through without transformation |
| `platt` | Platt scaling -- logistic regression on raw scores |
| `isotonic` | Isotonic regression -- non-parametric monotonic fit |

**How health scoring works:** The raw health score is a weighted sum of section scores (see preflight weights above), minus gate penalties (RED: -0.40, YELLOW: -0.10, GREEN: 0.00). The calibration model then transforms this raw score before band classification.

| Parameter | What It Controls | Impact |
|---|---|---|
| **Green band threshold** | Score above which a vault is classified as VERY_HIGH health (green badge, auto-pass eligible). | *Raise*: fewer vaults get the green badge. Stricter quality bar for auto-processing. *Lower*: more vaults qualify as healthy. |
| **Yellow band threshold** | Score above which a vault is classified as HEALTHY_REVIEW (amber badge, needs human review). Below this = NEEDS_REVIEW (red badge). | *Raise*: more vaults flagged as needing review. *Lower*: fewer vaults flagged, only truly poor quality gets red. |
| **Calibration model** | Mathematical transformation applied to raw scores before band classification. `identity` = no transformation (raw score used as-is). `platt` = sigmoid curve that compresses extreme values toward 0.5. `isotonic` = piecewise linear fit trained on historical data. | Use `identity` for new deployments. Switch to `platt` or `isotonic` after collecting enough data to train a calibration model. |

### Batch Processing

| Parameter | Key | Default | Range | Unit |
|---|---|---|---|---|
| Concurrency limit | `batch.concurrency_limit` | 5 | 1 - 20 | count |
| Retry count | `batch.retry_count` | 3 | 0 - 10 | count |
| PDF timeout | `batch.pdf_timeout` | 120 | 30 - 600 | seconds |

**Concurrency limit:** How many contracts can be processed simultaneously in a batch. The batch processor uses an `asyncio.Semaphore` with this value. *Raise* for faster throughput if your server has capacity. *Lower* to reduce load on extraction engines and external services (OCR, LLM).

**Retry count:** How many times to retry a failed contract before marking it as permanently failed. Each retry re-queues the contract from the beginning. *Raise* for unreliable environments (flaky OCR, intermittent network). *Lower* to fail fast and surface problems sooner.

**PDF timeout:** Maximum time (in seconds) to wait for a single PDF download or conversion. *Raise* for large documents or slow source servers (S3 cross-region, Google Drive). *Lower* to prevent one stuck document from blocking the pipeline.

---

## Calibration Audit Log

Every calibration change emits an audit event:

| Field | Description |
|---|---|
| `event_type` | `calibration.changed` |
| `calibration_key` | Which parameter changed |
| `previous_value` | Value before the change |
| `new_value` | Value after the change |
| `workspace_id` | Affected workspace |
| `actor` | User who made the change |
| `timestamp` | When the change occurred |
| `reason` | Optional free-text justification |

---

## Constraints and Validation

- **Preflight section weights** must sum to 1.00 (tolerance: +/- 0.01).
- **Threshold values** must stay within their defined `min`/`max` range.
- **Calibration model type** must be one of: `identity`, `platt`, `isotonic`.
- **Concurrency limit** must be a positive integer.
- **Retry count** must be a non-negative integer.
- All validation happens server-side; the UI provides client-side hints but the server is the authority.
