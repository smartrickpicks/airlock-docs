# Record Inspector -- Spotlight Mode

## Summary

Spotlight mode focuses attention on a single field or section by visually dimming everything else. When activated, all non-focused field cards blur and fade, creating a "spotlight" effect that isolates the field of interest. Spotlight mode automatically triggers Artifact Focus (Inspect state) in the triptych, expanding the Orchestrate panel to give maximum space to the spotlighted content.

Spotlight mode is designed for review workflows where the user needs to evaluate one field at a time without distraction, or compare a small group of related fields against their source evidence.

---

## Activation

| Method | Trigger | Context |
|---|---|---|
| Double-click | Double-click a field card | Works in both collapsed and expanded card states |
| Context menu | Right-click a field card, select "Focus on this field" | Available in all card states |
| Keyboard | With a field card focused via Tab navigation, press `Enter` | Toggles spotlight on/off for the focused card |
| Drawer button | Click "Spotlight" button inside an open field card drawer | Secondary activation path for users already inspecting a field |

### Exit

| Method | Behavior |
|---|---|
| `Escape` | Exits spotlight mode and returns to Standard (Overview) triptych state |
| Click "Clear spotlight" button | Button appears in the Record Inspector toolbar when spotlight is active; clears all spotlighted fields |
| `Cmd+1` | Force return to Standard state (also exits spotlight) |
| Click empty area outside all field cards | Exits spotlight mode |

---

## Visual Effect

### Focused Field

| Property | Value |
|---|---|
| Opacity | 1.0 (full) |
| Transform | `scale(1.02)` -- subtle enlargement to lift the card visually |
| Box shadow | `0 4px 24px rgba(0, 0, 0, 0.5)` -- elevated shadow |
| Left border | 2px solid `--accent-primary` (cyan) |
| Background | Standard card background (`--airlock-card`) or heatmap tint if heatmap is active |
| Z-index | Elevated above dimmed cards |

### Everything Else (Non-Focused)

| Property | Value |
|---|---|
| Filter | `blur(2px)` |
| Opacity | `0.3` |
| Pointer events | Active (user can still click dimmed cards to shift spotlight) |
| Transform | None |

### Scope

The spotlight effect applies within the Orchestrate panel only. Signal and Control panels are not blurred or dimmed -- they follow standard Artifact Focus behavior (collapse to icon-only mode).

---

## Transition Timing

| Transition | Duration | Easing |
|---|---|---|
| Enter spotlight (focus applied) | 200ms | ease-in |
| Exit spotlight (focus removed) | 150ms | ease-out |
| Triptych state change to Artifact Focus | 300ms | ease-in-out (per `Shell/view-states.md`) |
| Card scale-up | 200ms | ease-in |
| Card scale-down on exit | 150ms | ease-out |
| Blur/dim applied to non-focused cards | 200ms | ease-in |
| Blur/dim removed from non-focused cards | 150ms | ease-out |

The triptych state transition and the spotlight visual effect run concurrently. The 300ms panel width animation overlaps with the 200ms spotlight fade-in, so the user sees a single fluid motion.

---

## Triptych State Integration

Entering spotlight mode automatically triggers Artifact Focus (Inspect state) if not already active.

### Panel Layout in Spotlight

| Panel | Width | Content |
|---|---|---|
| Signal | ~80px | Collapsed to icon-only vertical strip (per Artifact Focus behavior) |
| Orchestrate | ~90% of triptych | Expanded. Record Inspector shows with spotlight effect applied. |
| Control | ~80px | Collapsed to icon-only vertical strip (per Artifact Focus behavior) |

### Spotlight vs. Generic Artifact Focus

| Property | Generic Artifact Focus | Spotlight Active |
|---|---|---|
| Orchestrate border | 1px amber glow (`#F59E0B` at 30% opacity) | 1px cyan glow (`--accent-primary` at 30% opacity) |
| Orchestrate inset shadow | `0 0 20px rgba(245, 158, 11, 0.1)` | `0 0 20px rgba(6, 182, 212, 0.1)` |
| Toolbar indicator | None | "Spotlight" label with cyan dot in Record Inspector toolbar |

The cyan glow visually differentiates spotlight from a generic Artifact Focus triggered by other means (e.g., opening a document for detailed viewing).

### State Transitions

| From | To | Trigger |
|---|---|---|
| Standard | Spotlight (Artifact Focus) | User double-clicks a field card |
| Artifact Focus (generic) | Spotlight (Artifact Focus) | User double-clicks a field card while already in Artifact Focus |
| Spotlight | Standard | User presses Escape or clicks "Clear spotlight" |
| Spotlight | Spotlight (updated) | User Shift+clicks another field (adds to group) |
| Spotlight | Standard | User clicks empty area outside all cards |

---

## Multi-Field Spotlight

Users can spotlight multiple fields simultaneously for comparison.

### Adding Fields

| Method | Behavior |
|---|---|
| `Shift+click` a field card | Adds the card to the spotlight group (existing spotlighted fields remain) |
| `Shift+double-click` a field card | Same as Shift+click (double-click is not required when Shift is held) |
| `Shift+Enter` on Tab-focused card | Adds the focused card to the spotlight group via keyboard |

### Multi-Field Visual

| Element | Behavior |
|---|---|
| All spotlighted fields | Full opacity, `scale(1.02)`, elevated shadow, cyan left border |
| Non-spotlighted fields | `blur(2px)`, `opacity(0.3)` |
| Fields in different sections | Both sections remain visible; non-spotlighted fields in visible sections are dimmed |

### Removing Fields from Group

| Method | Behavior |
|---|---|
| `Shift+click` a spotlighted field | Removes it from the group (toggles off) |
| Click a non-spotlighted field (without Shift) | Replaces the entire group with the clicked field |
| `Escape` | Clears all spotlighted fields, exits spotlight mode |
| "Clear spotlight" button | Clears all spotlighted fields, exits spotlight mode |

### Use Cases for Multi-Field

| Scenario | Fields |
|---|---|
| Date comparison | "Effective Date" + "Term Length" + "Expiration Date" |
| Financial verification | "Total Contract Value" + "Annual Rate" + "Payment Terms" |
| Entity cross-check | "Counterparty Legal Name" + "Counterparty Address" + "Signatory" |

---

## Document Viewer Sync

When a field is spotlighted, the Document Viewer responds if it is open (either as a tab or in side-by-side split).

### Single Field Spotlight

| Document Viewer State | Behavior |
|---|---|
| Tab mode (Record Inspector active) | No Document Viewer visible -- no sync action |
| Tab mode (Document Viewer active) | Annotations for the spotlighted field pulse; all other annotations dim |
| Side-by-side split (Artifact Focus) | Record Inspector on left with spotlight; Document Viewer on right scrolls to source location |

### Sync Behavior

| Property | Value |
|---|---|
| Scroll target | The page and region where the spotlighted field was extracted |
| Scroll animation | Smooth scroll, 300ms duration |
| Spotlighted annotation | Pulses with a cyan animation (box-shadow oscillates between 0 and `0 0 12px rgba(6, 182, 212, 0.4)` over 1.5s, repeats 3 times then holds steady) |
| Non-spotlighted annotations | Dim to 10% opacity |
| Annotation border (spotlighted) | 2px solid `--accent-primary` (cyan) |

### Multi-Field Spotlight Sync

| Condition | Behavior |
|---|---|
| Multiple fields on same page | All spotlighted annotations pulse; Document Viewer scrolls to the first one |
| Multiple fields on different pages | Document Viewer scrolls to the page of the most recently added field |
| User scrolls Document Viewer manually | Annotations remain styled (pulse/dim) regardless of scroll position |

---

## Section-Level Spotlight

Double-clicking a section header spotlights all fields in that section at once.

### Activation

| Method | Behavior |
|---|---|
| Double-click section header | All fields in the section (Tier 1 and Tier 2) become spotlighted |
| Right-click section header, "Focus on this section" | Same effect via context menu |

### Visual

| Element | Behavior |
|---|---|
| Section header | Full opacity, cyan left border |
| All fields in section | Full opacity, elevated, cyan left border (same as individual spotlight) |
| Tier 2 fields | Auto-expanded (overrides default collapsed state) |
| "N more fields" divider | Hidden (all fields visible) |
| All other sections | Headers and fields at `blur(2px)` + `opacity(0.3)` |

### Document Viewer Sync (Section Spotlight)

| Behavior | Detail |
|---|---|
| Annotations | All annotations belonging to fields in the spotlighted section pulse with cyan |
| Non-section annotations | Dim to 10% opacity |
| Scroll | Document Viewer scrolls to the first annotation in the section (by page order) |

---

## Edge Cases

### Spotlight + Heatmap

Both modes can be active simultaneously.

| Element | Behavior |
|---|---|
| Spotlighted field(s) | Heatmap band color applied at full specified opacity (15% or 10%) + cyan left border + full card opacity |
| Dimmed fields | Heatmap band color applied but reduced by the dimming: effective opacity is band opacity * 0.3 |
| Legend bar | Remains visible and interactive (not dimmed) |
| Section headers | Spotlighted section header shows heatmap aggregate tint at full opacity; other section headers show tint at reduced opacity |

### Spotlight + Search

| Scenario | Behavior |
|---|---|
| User activates search while spotlight is active | Spotlight clears immediately; search results take over as the filtering mechanism |
| Search is active, user double-clicks a result | Search clears; spotlight activates on the clicked field |

### Spotlight on Collapsed Tier 2 Field

| Step | Behavior |
|---|---|
| 1 | User double-clicks the "N more fields" divider or a reference to a Tier 2 field |
| 2 | Tier 2 fields in that section auto-expand |
| 3 | The target field receives spotlight treatment |
| 4 | On exit, Tier 2 fields return to collapsed state |

### Spotlight on Missing Field

| Condition | Behavior |
|---|---|
| Field has no extracted value (Missing status) | Spotlight applies normally -- card shows at full opacity with gray status dot |
| Document Viewer sync | No scroll action (no source region exists for a missing field) |
| Tooltip on missing spotlighted field | "No extraction found for this field" |

### Spotlight During Extraction in Progress

| Condition | Behavior |
|---|---|
| Skeleton cards visible | Spotlight is disabled -- double-click has no effect on skeleton cards |
| Extraction completes while spotlight is active | Newly populated cards respect current spotlight state (dimmed if not in group) |

### Spotlight + Gate Lock

| Condition | Behavior |
|---|---|
| Gate Lock state active, user spotlights a field | Spotlight triggers Artifact Focus visuals but Governance Bar remains visible (sticky above spotlighted content) |
| Governance Bar interaction | Approve/Reject/Clarify/Hold buttons remain accessible; clicking them exits both Gate Lock and spotlight |

---

## Toolbar Indicator

When spotlight mode is active, the Record Inspector toolbar shows an indicator.

| Property | Value |
|---|---|
| Label | "Spotlight" (or "Spotlight: N fields" if multi-field) |
| Icon | Cyan dot (8px, `--accent-primary`) to the left of the label |
| "Clear spotlight" button | Appears to the right of the label; text button with `--airlock-muted` color |
| Position | Record Inspector toolbar, right side, to the left of the heatmap toggle |

---

## Keyboard Navigation in Spotlight

| Key | Behavior |
|---|---|
| `Tab` | Cycles focus among spotlighted fields only (skips dimmed fields) |
| `Shift+Tab` | Reverse cycle among spotlighted fields |
| `Arrow Down` | Moves to the next spotlighted field in document order |
| `Arrow Up` | Moves to the previous spotlighted field in document order |
| `Enter` | Opens/closes the drawer of the currently focused spotlighted field |
| `Escape` | Exits spotlight mode, returns to Standard triptych state |
| `Shift+Enter` | Adds the currently Tab-focused (non-spotlighted) field to the spotlight group |

Note: When spotlight is not active, Tab navigation moves through all visible field cards normally.

---

## State Management

| State | Stored Where | Persistence |
|---|---|---|
| Spotlight active (on/off) | Local component state | Resets on vault change or navigation |
| Spotlighted field IDs | Local component state (Set) | Resets on spotlight exit |
| Previous triptych state | Session store (Zustand) | Used to restore state on spotlight exit |
| Tier 2 auto-expanded by spotlight | Local component state | Reverts to collapsed on spotlight exit |

---

## Accessibility

| Requirement | Implementation |
|---|---|
| Screen reader announcement | On spotlight activation: "Spotlight mode active. Focused on [field name]." |
| Multi-field announcement | "Added [field name] to spotlight. N fields focused." |
| Dimmed content | `aria-hidden="false"` on dimmed cards (still navigable but announced as "dimmed") |
| Exit announcement | "Spotlight mode exited. Returned to standard view." |
| Reduced motion preference | If `prefers-reduced-motion` is set, skip scale transform and blur; use opacity change only |

---

## Related Specs

| Spec | Relevance |
|---|---|
| `overview.md` | Section model, tier model, cross-panel coordination, dependency map |
| `field-cards.md` | Card anatomy, drawer behavior, spotlight mode brief reference, tier system |
| `heatmap-mode.md` | Co-activation rules, visual interaction when both modes are active |
| `document-viewer.md` | Annotation overlays, scroll sync, side-by-side split, cross-linking protocol |
| `entity-resolution.md` | Signal panel behavior during Artifact Focus (icon-only collapse) |
| `Shell/view-states.md` | Triptych state definitions, Artifact Focus layout, transition timing, keyboard overrides |
