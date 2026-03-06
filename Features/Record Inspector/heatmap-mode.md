# Record Inspector -- Heatmap Mode

## Summary

Heatmap mode is a toggle overlay that color-codes every field card by extraction confidence, providing a macro-level quality overview without requiring field-by-field inspection. It answers the question: "Where should I focus my review time?"

When active, each field card receives a background tint mapped to its confidence score, section headers display aggregate confidence, and a legend bar at the top of the Record Inspector summarizes distribution across confidence bands. Heatmap mode is purely visual -- it does not alter card layout, drawer content, or default sort order.

---

## Activation

| Property | Value |
|---|---|
| Toggle location | Record Inspector header bar (right side, alongside existing toolbar controls) |
| Icon | Thermometer (Lucide `Thermometer`) |
| Keyboard shortcut | `Cmd+E` |
| Default state | Off |
| Persistence | Per session (not per vault -- toggling in one vault applies to all vaults until session ends) |

### Eligible Triptych States

| Triptych State | Heatmap Available | Notes |
|---|---|---|
| Standard (Overview) | Yes | Primary use case -- scanning all fields at a glance |
| Artifact Focus (Inspect) | Yes | Full-width field cards with heatmap colors in expanded layout |
| Action Focus (Edit) | No | Editing workflow does not display field cards in the standard grid |
| Gate Lock (Approve) | Yes | Useful for reviewers assessing extraction quality before approval |

---

## Confidence Color Scale

Heatmap mode maps each field's numeric confidence score (0.0 -- 1.0) to one of four visual bands.

| Band | Confidence Range | Color Hex | Opacity | Card Background | Label |
|---|---|---|---|---|---|
| HIGH | 75% -- 100% (0.75 -- 1.0) | `#22C55E` | 15% | Green tint | Confident |
| MEDIUM | 40% -- 74% (0.40 -- 0.74) | `#F59E0B` | 15% | Amber tint | Needs Review |
| LOW | 0% -- 39% (0.00 -- 0.39) | `#EF4444` | 15% | Red tint | Low Confidence |
| MISSING | Not extracted | `#64748B` | 10% | Gray tint | Missing |

### Band Edge Rules

| Boundary | Resolution |
|---|---|
| Exactly 75% | Classified as HIGH |
| Exactly 40% | Classified as MEDIUM |
| Confidence unavailable (N/A) | Treated as MISSING |

---

## Legend Bar

When heatmap mode is active, a legend bar appears at the top of the Record Inspector, above all section headers.

### Layout

```
+------------------------------------------------------------------------+
| [Green dot] HIGH: 87  | [Amber dot] MED: 34  | [Red dot] LOW: 18  | [Gray dot] MISSING: 13 |
+------------------------------------------------------------------------+
```

| Property | Value |
|---|---|
| Position | Sticky, top of Record Inspector scrollable area (below header bar, above first section) |
| Height | 36px |
| Background | `--airlock-surface` (#0F1219) |
| Border | 1px bottom border in `--airlock-border` |
| Typography | Fira Sans, 12px, `--airlock-muted` for labels, `--airlock-text` for counts |
| Dot size | 8px diameter, filled with band color at 100% opacity |
| Spacing | 16px between band groups |

### Legend Bar Behavior

| Condition | Behavior |
|---|---|
| Click a band in the legend | Scrolls to the first field in that confidence band (within current sort order) |
| Hover a band in the legend | Tooltip shows percentage of total fields (e.g., "57% of extracted fields") |
| All fields in one band | Other bands still display with count of 0 |
| Heatmap toggled off | Legend bar animates out (150ms fade + 100ms slide up) |

---

## Visual Application to Field Cards

### Card Background

When heatmap is active, each field card's background changes from the default neutral (`--airlock-card`) to the band-appropriate tint color.

| Element | Heatmap Off | Heatmap On |
|---|---|---|
| Card background | `--airlock-card` (#151923) | Band color at specified opacity, blended over `--airlock-card` |
| Card border | `--airlock-border` | `--airlock-border` (unchanged) |
| Status dot | Standard colors (green/amber/red/gray/blue) | Standard colors (unchanged -- status dots are independent of heatmap) |
| Confidence badge | Standard badge | Standard badge (unchanged) |
| Text color | `--airlock-text` | `--airlock-text` (unchanged) |

### Drawer State

The heatmap tint applies to the collapsed card only. When a drawer is expanded, the drawer interior retains the standard `--airlock-surface` background to preserve readability of evidence text and anchor info.

---

## Visual Application to Section Headers

### Aggregate Confidence

Each section header computes a weighted average of its child fields' confidence scores.

| Calculation | Formula |
|---|---|
| Aggregate confidence | Sum of (field confidence * 1) for all fields in section / count of fields with a confidence score |
| Missing fields | Excluded from the average (they have no numeric confidence) |
| Section with all fields missing | Aggregate displays "N/A" and uses MISSING band color |

### Section Header Display

| Element | Heatmap Off | Heatmap On |
|---|---|---|
| Header background | `--airlock-surface` | Band color at specified opacity, based on aggregate confidence |
| Aggregate label | Not shown | Appears to the right of the section weight label: "Avg: 72%" |
| Status summary | Count of pass/review/fail/missing | Count of pass/review/fail/missing (unchanged) |

---

## Heatmap in Artifact Focus

When heatmap is active and the user enters Artifact Focus (Inspect state), the expanded Orchestrate panel shows full-width field cards with heatmap colors applied.

### Record Inspector (Expanded)

| Property | Behavior |
|---|---|
| Card width | Full width of expanded Orchestrate panel |
| Heatmap tint | Applied at the same opacity values as Standard state |
| Legend bar | Still visible, sticky at top |
| Tier 2 fields | Heatmap tint applies to both Tier 1 and Tier 2 cards equally |

### Document Viewer (Side-by-Side Split)

When side-by-side mode is active in Artifact Focus, heatmap colors extend to the Document Viewer's annotation overlays.

| Annotation Overlay | Heatmap Off | Heatmap On |
|---|---|---|
| Pass annotation | Green highlight | Replaced with field's confidence band color |
| Review annotation | Amber highlight | Replaced with field's confidence band color |
| Fail annotation | Red highlight | Replaced with field's confidence band color |
| AI-detected annotation | Cyan highlight | Replaced with field's confidence band color |

| Property | Value |
|---|---|
| Annotation opacity (heatmap on) | 20% (slightly higher than card tint to be visible on PDF surface) |
| Annotation border | 1px solid at 40% opacity of the band color |
| Transition | Annotation colors crossfade over 200ms when heatmap is toggled |

---

## Sorting in Heatmap Mode

### Confidence Sort

When heatmap mode is active, an additional sort control appears in the Record Inspector toolbar.

| Property | Value |
|---|---|
| Button label | "Sort by confidence" |
| Button location | Record Inspector toolbar, to the right of the heatmap toggle |
| Icon | Lucide `ArrowUpDown` |
| Visibility | Only visible when heatmap is active |

### Sort Behavior

| Sort State | Field Order |
|---|---|
| Default (schema order) | Fields appear in the order defined by the extraction schema, within each section |
| Confidence ascending | Fields sorted lowest confidence first within each section (surfaces problems at the top) |
| Confidence descending | Fields sorted highest confidence first within each section |

### Sort Cycling

Clicking the sort button cycles through three states:

| Click # | State | Icon Indicator |
|---|---|---|
| 1st click | Ascending (lowest first) | Arrow pointing up |
| 2nd click | Descending (highest first) | Arrow pointing down |
| 3rd click | Default (schema order) | Neutral arrows |

### Sort Independence

| Scenario | Behavior |
|---|---|
| Heatmap off, sort was active | Sort order persists -- fields remain in confidence order even without heatmap colors |
| Heatmap on, sort is default | No sort reordering -- only heatmap colors applied |
| Sort active across sections | Each section sorts independently (Entity Resolution fields sorted among themselves, Opportunities among themselves, etc.) |
| Missing fields in sort | Missing fields (no confidence score) always appear at the bottom of their section regardless of sort direction |

---

## Interaction

| User Action | Result |
|---|---|
| Toggle heatmap on | All field cards receive confidence band background tint; legend bar appears; section headers show aggregate |
| Toggle heatmap off | Card backgrounds return to neutral; legend bar disappears; section aggregate labels hidden |
| Hover field card (heatmap on) | Tooltip appears showing exact confidence: "Confidence: 67.3%" |
| Hover field card (heatmap off) | Standard tooltip behavior (no confidence tooltip) |
| Click field card (heatmap on) | Drawer expands normally -- heatmap is visual only, does not change interaction |
| Click legend bar band | Scrolls to first field in that confidence band |
| Press `Cmd+E` | Toggles heatmap on/off |

---

## Animation

| Transition | Duration | Easing | Description |
|---|---|---|---|
| Heatmap toggle on | 150ms | ease-in | Card backgrounds fade from neutral to band color |
| Heatmap toggle off | 150ms | ease-out | Card backgrounds fade from band color to neutral |
| Legend bar appear | 150ms fade + 100ms slide down | ease-in | Legend fades in and slides into position from above |
| Legend bar disappear | 150ms fade + 100ms slide up | ease-out | Legend fades out and slides up out of view |
| Sort reorder | 200ms | ease-in-out | Cards animate to new positions within their sections (layout shift) |
| Document Viewer annotation crossfade | 200ms | ease | Annotation colors transition between standard and heatmap colors |

---

## Edge Cases

| Condition | Behavior |
|---|---|
| All fields have HIGH confidence | Heatmap still applies (uniform green tint); legend shows HIGH: 152, all others: 0 |
| Extraction in progress (skeleton cards) | Skeleton cards do not receive heatmap tint; they retain shimmer animation |
| Field confidence updates while heatmap is active | Card tint transitions to new band color over 150ms |
| Zero extracted fields | Legend bar shows all counts as 0; no visible tint changes |
| Heatmap + spotlight mode | Both can be active simultaneously (see `spotlight-mode.md` for interaction rules) |
| Session ends | Heatmap state resets to off on next session |

---

## State Management

| State | Stored Where | Persistence |
|---|---|---|
| Heatmap toggle (on/off) | Session-level store (Zustand) | Per session, not per vault |
| Confidence sort state | Session-level store (Zustand) | Per session, survives heatmap toggle |
| Legend bar scroll position | Local component state | Resets on toggle |

---

## Related Specs

| Spec | Relevance |
|---|---|
| `overview.md` | Section model, tier model, confidence scoring context |
| `field-cards.md` | Card anatomy, confidence badge, status dots, drawer behavior |
| `document-viewer.md` | Annotation overlay colors, side-by-side split behavior |
| `spotlight-mode.md` | Co-activation with heatmap, visual interaction rules |
| `Shell/view-states.md` | Triptych state definitions (Standard, Artifact Focus, Action Focus, Gate Lock) |
