# Record Inspector -- Field Cards

## Design Decision

**Cards with expandable drawers.**

Rejected alternatives:
- Dense table rows -- too clinical, poor touch targets, hard to scan mixed-type data.
- Grouped cards with inline evidence -- too tall per card, evidence competes with the primary value for attention.

The expandable drawer pattern keeps the collapsed state compact (one line of essential info) while making full evidence available in a single click.

---

## Section Grouping

Field cards are organized into collapsible sections matching the preflight engine's five scored categories.

| Section | Weight | Sort Order |
|---|---|---|
| Entity Resolution | 0.20 | 1 |
| Opportunities | 0.25 | 2 |
| Schedule | 0.15 | 3 |
| Financials | 0.25 | 4 |
| Addons | 0.15 | 5 |

Each section header displays:
- Section name
- Section weight (as percentage label, e.g., "25%")
- Aggregate status summary (count of pass / review / fail / missing fields in that section)
- Collapse/expand toggle

---

## Tier System

| Tier | Default State | Criteria |
|---|---|---|
| Tier 1 | Expanded (card visible, drawer closed) | High-priority fields determined by preflight engine |
| Tier 2 | Collapsed (card hidden until section is expanded) | Supporting fields, lower priority |

Within each section:
- Tier 1 fields render first.
- A divider labeled "N more fields" separates Tier 1 from Tier 2.
- Clicking the divider reveals Tier 2 fields.

---

## Card Anatomy (Collapsed State)

Each field card in its collapsed state displays a single row with the following elements:

| Element | Position | Description |
|---|---|---|
| Status dot | Left edge | Colored dot indicating field status (see table below) |
| Field name | Left of center | Human-readable label for the extracted field |
| Extracted value | Right of center | The value pulled from the document (truncated if long) |
| Confidence badge | Right edge | Labeled badge showing confidence tier |

### Status Dot Colors

| Status | Color | Meaning |
|---|---|---|
| Pass | Green | Field extracted and validated |
| Review | Amber | Field needs human verification |
| Fail | Red | Field failed validation |
| Missing | Gray | Expected field not found in document |
| Suggested | Blue | AI-suggested value, not confirmed |

### Confidence Badge

| Tier | Threshold | Badge Label | Badge Color |
|---|---|---|---|
| HIGH | >= 75% | HIGH | Green background |
| MED | >= 40% and < 75% | MED | Amber background |
| LOW | < 40% | LOW | Red background |

---

## Drawer (Expanded State)

Clicking a collapsed field card opens its drawer below the card row. The drawer contains four information blocks.

### 1. Evidence Context

- 150-character window around the matched text in the source document.
- The matched substring is highlighted (bold or background color).
- Clicking the evidence context triggers cross-linking to the Document Viewer (scrolls to source page/region).

### 2. Anchor Info

| Property | Value |
|---|---|
| Anchor matched | The specific anchor string or pattern that triggered extraction |
| Anchor tier | Tier 1 or Tier 2 |

### 3. Extractor Source

Identifies which of the seven extractors produced this field.

| Extractor Type | Description |
|---|---|
| Boolean | True/false determination |
| Split | Value split from a compound field |
| Picklist | Value matched against a predefined option set |
| Date | Date extraction and normalization |
| Pattern | Regex or pattern-based extraction |
| Text | Free-text extraction |
| *(7th extractor from pipeline)* | As defined in `server/extraction/` |

### 4. Match Confidence Score

- Numeric score displayed as a value between 0.0 and 1.0.
- Accompanied by a horizontal bar visualization (filled proportionally).
- Color of the bar matches the confidence badge tier (green / amber / red).

---

## Heatmap Toggle

A toggle control in the Record Inspector toolbar activates heatmap mode.

| State | Behavior |
|---|---|
| Off (default) | Field cards use standard neutral background |
| On | Field card backgrounds tinted with confidence colors |

**Heatmap Colors**

| Confidence Tier | Background Tint |
|---|---|
| HIGH (>= 75%) | Light green |
| MED (>= 40%) | Light amber |
| LOW (< 40%) | Light red |

Heatmap mode is purely visual overlay -- it does not change card layout, drawer content, or sort order.

---

## Spotlight Mode

Clicking a field card while holding a modifier key (or via a "Spotlight" button in the drawer) activates spotlight mode.

| Aspect | Behavior |
|---|---|
| Focused field | Full opacity, slightly elevated (shadow) |
| All other fields | Blurred and dimmed (reduced opacity) |
| Triptych layout | Automatically switches to Artifact Focus if not already active |
| Document Viewer | If side-by-side is active, scrolls to the spotlighted field's source region |
| Exit | Click anywhere outside the focused card, or press Escape |

Spotlight mode is designed for review workflows where the user needs to evaluate one field at a time without distraction.

---

## Interaction Summary

| User Action | Result |
|---|---|
| Click field card | Drawer expands/collapses |
| Click evidence context in drawer | Document Viewer scrolls to source region |
| Toggle heatmap | Confidence color overlay applied to all cards |
| Activate spotlight | All cards blur except selected; Artifact Focus triggered |
| Click "N more fields" divider | Tier 2 fields revealed in that section |
| Click section header toggle | Entire section collapses/expands |

---

## Empty and Edge States

| Condition | Display |
|---|---|
| No fields extracted for a section | Section header visible with "(0 fields)" label; no cards rendered |
| Field value is empty string | Card shows extracted value as "--" with Missing status |
| Confidence score unavailable | Badge shows "N/A" with gray background |
| Extremely long extracted value | Value truncated with ellipsis; full value visible in drawer |
| Extraction in progress | Skeleton card placeholders with shimmer animation |
