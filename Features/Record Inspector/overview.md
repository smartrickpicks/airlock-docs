# Record Inspector -- Overview

## Purpose

Record Inspector is the primary data-dense sub-view inside the Orchestrate panel. It surfaces every extracted field from a contract document, organized into scored sections, and cross-links those fields to their source locations in the Document Viewer. It is the default view that loads when a user opens a contract channel.

---

## Position in the Application

| Property | Value |
|---|---|
| Parent module | Contracts (heaviest module in Airlock) |
| Host panel | Orchestrate panel (center of triptych) |
| Default context | Opens automatically when a contract channel is selected |
| Companion views | Document Viewer (tab sibling), Signal panel (entity resolution), Artifact Focus (split layout) |

---

## Source Mapping

Record Inspector ports functionality from two major backend systems in OrcestrateOS:

| OrcestrateOS Source | Scope | What It Provides |
|---|---|---|
| `server/extraction/` | 7 extractors, 152 fields | Field extraction logic, confidence scoring, anchor matching |
| `server/preflight_engine.py` | 3,895 lines | Section scoring, tier classification, pass/fail determination |
| `dossier-demo.html` | 106 KB | Reference layout -- most complex data-dense page in the system |

The demo file (`dossier-demo.html`) lives in this feature folder and serves as the canonical reference for layout density, information hierarchy, and interaction patterns.

---

## Sub-Views and Companion Specs

| Spec File | Responsibility |
|---|---|
| `field-cards.md` | Field card design, expandable drawers, section grouping, confidence badges, status dots |
| `document-viewer.md` | PDF rendering, annotation overlays, scroll sync, field linking, side-by-side split |
| `entity-resolution.md` | Inline disambiguation in Signal panel, candidate matching, alias management |
| `heatmap-mode.md` | Confidence color overlay across all field cards, legend bar, sort-by-confidence |
| `spotlight-mode.md` | Focus on single field/section, blur everything else, auto-trigger Artifact Focus |

---

## Key Design Principles

- **Density without overwhelm.** The Record Inspector must present up to 152 fields without making the user scroll endlessly or lose context. Tier-based expansion and section grouping are the primary mechanisms.
- **Evidence is one click away.** Every extracted value links back to its source evidence. Clicking a field card opens a drawer; clicking further scrolls the Document Viewer to the exact region.
- **Cross-panel coordination.** Record Inspector does not live in isolation. It emits events to the Document Viewer (scroll-to-region), receives events from the Signal panel (entity confirmed), and participates in Artifact Focus layout splits.
- **Confidence is always visible.** Every field carries a confidence badge and status dot. The heatmap toggle provides a macro-level confidence overview without requiring field-by-field inspection.

---

## Section Model

The preflight engine scores contracts across five weighted sections. Record Inspector groups field cards by these sections.

| Section | Weight | Description |
|---|---|---|
| Entity Resolution | 0.20 | Legal entity identification, counterparty matching, alias resolution |
| Opportunities | 0.25 | Revenue signals, deal terms, pricing structures |
| Schedule | 0.15 | Dates, durations, renewal terms, milestones |
| Financials | 0.25 | Dollar amounts, payment terms, rate structures, fee schedules |
| Addons | 0.15 | Supplementary terms, riders, amendments, non-standard clauses |

---

## Field Tier Model

| Tier | Behavior | Rationale |
|---|---|---|
| Tier 0 (Hinge) | Always visible, teal accent, **HINGE** badge | Gate-all-downstream fields. Processing cannot advance until these are resolved. |
| Tier 1 | Expanded by default | High-priority fields the user needs to see immediately on load |
| Tier 2 | Collapsed by default | Supporting fields available on demand to reduce initial visual load |

### Hinge Fields (Tier 0)

Hinge fields sit at the top of the Record Inspector, above section grouping. They gate all downstream extraction and scoring — if a hinge field is unresolved, the preflight engine cannot classify the document.

| Hinge Field | Check Code | Why It's Hinge |
|---|---|---|
| Contract Type | `OPP_CONTRACT_TYPE` | Determines which extraction rules, clause triggers, and schedule expectations apply |
| Contract Subtype | `OPP_CONTRACT_SUBTYPE` | Refines extraction scope (e.g., CMA vs MSA vs Service Agreement) |

**Visual treatment:**
- Teal-accented cards (distinct from section cards)
- **HINGE** badge on left edge
- If unresolved: **START HERE** badge appears, guiding the analyst to resolve hinge fields first
- If resolved: green checkmark replaces badges

Tier assignment is determined by the preflight engine and is not user-configurable in v1.

---

## Interaction Flow (Happy Path)

1. User opens a contract channel in Airlock.
2. Orchestrate panel loads with Record Inspector as the default tab.
3. Field cards render grouped by section, Tier 1 expanded.
4. User clicks a field card -- drawer opens with evidence, anchor info, extractor source, confidence score.
5. User clicks evidence context -- Document Viewer scrolls to the source page/region.
6. If entity is ambiguous, Signal panel shows inline disambiguation (see `entity-resolution.md`).
7. User confirms entity -- resolution event propagates, field cards update.

---

## Orchestrate Panel Tab Bar

Record Inspector is one tab in the Orchestrate panel. The tab bar at the top of Orchestrate lets users switch between views:

| Tab | Label | Content |
|---|---|---|
| **Record Inspector** (default) | "Fields" | Field cards grouped by section |
| **Document Viewer** | "Document" | PDF viewer with annotation overlay |
| **Patch History** | "Patches" | Timeline of all patches for this vault |
| **Extraction Detail** | "Extraction" | Raw extraction results per extractor |
| **Preflight Report** | "Preflight" | Full preflight sections breakdown |

The active tab is highlighted with a cyan underline. Tab state persists per vault per session.

---

## Health Score Integration

The Record Inspector header (above the tab bar) displays the vault's aggregate health score:

| Element | Display |
|---|---|
| Health score | Large number (e.g., "78%") with colored band badge (VERY_HIGH / HEALTHY / NEEDS_REVIEW) |
| Section scores | Mini bar chart showing per-section weighted scores |
| Gate status | Colored dot + gate name (Discovery / Build / Review / Ship) |
| Chamber indicator | Color bar matching the current chamber |

Clicking the health score expands a detailed breakdown panel showing how each section contributes to the aggregate.

---

## Cross-Panel Event Flow

Record Inspector participates in a bidirectional event flow across the triptych:

```
Signal Panel (left)              Orchestrate (center)              Control Panel (right)
     |                                |                                  |
     |  entity.resolved ────────────> |  Update entity field cards        |
     |  ai.suggestion ─────────────> |  Highlight suggested field         |
     |                                |                                  |
     |                                |  field.clicked ──────────────>   |  Show field in audit trail
     |                                |  spotlight.activated ─────────>  |  Collapse to icon-only
     |                                |                                  |
     |  <──── field.evidence_opened   |                                  |
     |         (scroll to event)      |  document.scroll_to ──────>     |  (Document Viewer sync)
```

### Event Types Emitted

| Event | Trigger | Consumer |
|---|---|---|
| `field.clicked` | User clicks a field card | Control panel (audit context) |
| `field.drawer_opened` | User expands field drawer | Document Viewer (scroll to evidence) |
| `spotlight.activated` | User double-clicks field | Shell (trigger Artifact Focus) |
| `heatmap.toggled` | User toggles heatmap | Document Viewer (update annotation colors) |

### Events Consumed

| Event | Source | Effect |
|---|---|---|
| `entity.resolved` | Signal panel | Update entity field cards with confirmed match |
| `entity.disambiguation_needed` | Signal panel | Highlight ambiguous fields with amber border |
| `ai.suggestion` | Signal panel | Show cyan glow on suggested field + AI tooltip |
| `extraction.updated` | Backend (WebSocket) | Refresh field values and confidence scores |
| `patch.applied` | Backend (WebSocket) | Update affected field values with new data |

---

## Loading States

| State | Display |
|---|---|
| **Initial load** | Skeleton cards (animated pulse) in section layout, 3 per section |
| **Extraction in progress** | Progress bar in header + "Extracting..." label. Fields appear incrementally as extractors complete. |
| **Extraction complete** | Full field cards render. Health score appears. Heatmap toggle enables. |
| **Error state** | Error banner with retry button. Show any partial results. |
| **Empty state** | "No extraction results" with explanation and "Upload document" CTA |

---

## Feature Control Plane Integration

| Capability | Parameter |
|---|---|
| Toggle | `record_inspector.enabled` (disabling shows "Feature disabled" placeholder) |
| Heatmap toggle | `record_inspector.heatmap.enabled` |
| Spotlight toggle | `record_inspector.spotlight.enabled` |
| Default tier expansion | `record_inspector.default_tier` (1 or 2) |
| Max fields displayed | `record_inspector.max_fields` (default: 152) |

---

## Amendment & Clause Library v2 Integration

The Record Inspector renders amendment-type contracts differently, leveraging the unified clause library v2.

### Amendment-Specific Behavior

| Behavior | Standard Contract | Amendment Contract |
|---|---|---|
| Schedule section | Required for GREEN gate | Relaxed — status "review" instead of "fail" if absent |
| Hinge field display | Contract Type + Subtype | Contract Type + Subtype + **What Are We Amending** (multi-select) |
| Addons section | Generic add-on signals | Amendment-specific signals: rider, supplemental, addendum |
| Template keywords | Standard extraction | Additional patterns: "amendment", "addendum", "modification agreement" |

The `What_are_we_amending__c` field (`OPP_WHAT_ARE_WE_AMENDING`) is a multi-select that defines what aspects of the original contract are being changed. When the contract type is "amendment", this field appears prominently in the Opportunities section with individual option tags for each amended aspect.

### Clause Library Bridge

Each extracted field links back to its source clause definition in the v2 clause library (188 clauses). Field card drawers display:

| Element | Content |
|---|---|
| Clause source badge | Clause ID (e.g., `GEN-AMENDMENT-V1`) if the field maps to a known clause |
| Risk level indicator | Standard (none), Elevated (amber), Critical (red) |
| Extraction config | Anchor patterns, preferred zones, confidence floor — sourced from the clause's embedded `variables[].extraction` block |
| Generation link | "Used in: Distribution, License, Recording..." showing which contract templates reference this clause |

This bridge means that improving a clause in the library automatically improves extraction coverage in the Record Inspector, and vice versa.

---

## Context Menus

Record Inspector provides rich right-click menus for field-level and text-level actions, ported from OrcestrateOS's dossier view.

### Field Context Menu (Right-Click on Field Card)

| Action | Shortcut | Description |
|---|---|---|
| Verify | V | Mark field as human-verified |
| Remap Value | M | Change the extracted value manually |
| Request Info | R | Create an RFI for this field |
| Blacklist | B | Exclude this field from scoring |
| Reset to TODO | T | Reset field to unresolved state |
| View in Document | D | Scroll Document Viewer to extraction source |
| Ask Otto about field | K | Open Otto with field context pre-loaded |
| Jump to Section | J | Scroll to this field's parent section |
| Copy Value | P | Copy extracted value to clipboard |
| Add Comment | N | Attach a note to this field |

### Text Selection Context Menu (Right-Click on Selected Text in Document Viewer)

| Action | Shortcut | Description |
|---|---|---|
| Ask Otto | K | Ask Otto about the selected text |
| Create Alias | A | Create an entity alias from selected text |
| Copy | C | Copy to clipboard |
| Evidence Mark | E | Mark selected text as evidence for a field |
| Search Glossary | G | Look up selected term in domain glossary |
| Explain Clause | X | Request Otto explanation of the clause containing this text |

---

## Progress Tracking Model

The Record Inspector header tracks section-level progress with a "N/5 sections" model:

### Section Progress Pills

Each section displays a progress pill showing its readiness state:

| State | Display | Color |
|---|---|---|
| Complete | Section name + checkmark | Green |
| In Review | Section name + "In Review" | Amber |
| Pending | Section name + "Pending" | Gray |
| Failed | Section name + "Failed" | Red |

**Aggregate readiness** is shown as a percentage (e.g., "3/5 sections · 72% ready") in the Record Inspector header, next to the health score.

### Section Canonical Codes

Each readiness section maps to canonical section codes used for internal routing:

| Section | Canonical Codes |
|---|---|
| Entity Resolution | `SEC_PREAMBLE`, `SEC_ENTITY` |
| Opportunities | `SEC_SCOPE`, `SEC_RIGHTS`, `SEC_TERRITORY`, `SEC_TERM`, `SEC_TERMINATION`, `SEC_RESTRICTIONS` |
| Schedule | `SEC_SCHEDULE` |
| Financials | `SEC_COMPENSATION` |
| Addons | `SEC_ADDONS` |

---

## Track-Listing Stripping

For music-industry contracts, the preflight engine strips "track listing" and "song list" sections before entity resolution to prevent music metadata (artist names, song titles) from false-matching against account names. The Record Inspector shows a subtle indicator when stripping was applied:

- Banner: "Track listing section detected and excluded from entity matching"
- Affected fields show a `stripped_context` evidence chip
- The original text is still viewable in the Document Viewer (not removed, only excluded from matching)

---

## Counterparty Resolution Form

When entity resolution identifies a **new customer** (no existing account match), the Signal panel shows a structured onboarding form. This form is part of the entity resolution flow (see `entity-resolution.md`) but renders alongside Record Inspector field cards.

### Form Fields

| Field | Source | Confidence | Required |
|---|---|---|---|
| Legal Entity | Extracted | Varies | Yes |
| Counterparty Name | Extracted | ~96% typical | Yes |
| Street Address | Extracted | ~96% | No |
| City | Extracted | ~94% | No |
| State | Extracted | ~89% | Yes |
| Postal Code | Extracted | ~94% | Yes |
| Country | Extracted | ~96% | No |
| Contact Name | Manual entry | — | No |
| Email | Manual entry | — | No |
| Phone | Manual entry | — | No |

**Actions:**
- "Search Existing" — search for an existing account to link
- "Create New Counterparty" — create a new CRM entry (cross-module action)

Extracted fields show confidence badges. Manual fields are blank by default.

---

## Dependencies

| Dependency | Direction | Description |
|---|---|---|
| Document Viewer | Bidirectional | Field click scrolls viewer; annotation click opens field drawer |
| Signal Panel | Inbound | Entity resolution confirmation updates field cards |
| Artifact Focus | Outbound | Spotlight mode triggers Artifact Focus layout |
| Heatmap Mode | Internal | Color overlay on field cards by confidence band |
| Spotlight Mode | Internal | Focus/blur effect on individual fields or sections |
| Preflight Engine | Backend | Provides section scores, tier assignments, confidence values |
| Extraction Pipeline | Backend | Provides field values, anchors, evidence windows |
| Patch Workflow | Backend | Applied patches update field values in Record Inspector |
| Clause Library v2 | Backend | Provides clause source links, risk levels, extraction config per field |
| Track-Listing Stripper | Backend | Excludes music metadata from entity resolution |
