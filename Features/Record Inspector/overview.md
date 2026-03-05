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
| `field-cards.md` | Field card design, expandable drawers, section grouping, heatmap toggle, spotlight mode |
| `document-viewer.md` | PDF rendering, annotation overlays, scroll sync, field linking, side-by-side split |
| `entity-resolution.md` | Inline disambiguation in Signal panel, candidate matching, alias management |

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
| Tier 1 | Expanded by default | High-priority fields the user needs to see immediately on load |
| Tier 2 | Collapsed by default | Supporting fields available on demand to reduce initial visual load |

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

## Dependencies

| Dependency | Direction | Description |
|---|---|---|
| Document Viewer | Bidirectional | Field click scrolls viewer; annotation click opens field drawer |
| Signal Panel | Inbound | Entity resolution confirmation updates field cards |
| Artifact Focus | Outbound | Spotlight mode triggers Artifact Focus layout |
| Preflight Engine | Backend | Provides section scores, tier assignments, confidence values |
| Extraction Pipeline | Backend | Provides field values, anchors, evidence windows |
