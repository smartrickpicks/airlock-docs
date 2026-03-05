# View States

## Summary

The triptych adapts its panel layout automatically based on user context. There are four view states. Users do not manually toggle between them -- state changes are triggered by specific user actions. Power users can override with keyboard shortcuts.

**Decision:** View states are automatic, context-driven. No manual toggle UI.

## State Overview

| # | State | Trigger | Signal | Orchestrate | Control |
|---|-------|---------|--------|-------------|---------|
| 1 | Standard | Default / Escape / click outside | ~280px | flex-1 | ~300px |
| 2 | Artifact Focus | Click field or document to inspect | ~80px (icon-only) | ~90% width | ~80px (icon-only) |
| 3 | Action Focus | Click "New Patch" or similar creation action | Narrowed ~80px | Split: Editor + Preview | Normal ~300px |
| 4 | Gate Lock | Gate is active and user is current reviewer | ~280px | Normal + Governance Bar | ~300px |

---

## State 1: Standard

The default resting state. All three panels visible at their normal widths.

### Layout

| Panel | Width | Content |
|-------|-------|---------|
| Signal | ~280px | Full signal feed (cards, notifications, health) |
| Orchestrate | flex-1 | Record Inspector, document, or default view |
| Control | ~300px | Full tabbed interface (Lifecycle, SLA, Approvals, Audit, AI Agent) |

### Trigger

- Application launch (default state)
- Pressing Escape from any other state
- Clicking outside of a focused artifact
- Keyboard: Cmd+1

### Visual

No special visual indicators. Standard panel borders in `--airlock-border`.

---

## State 2: Artifact Focus

Orchestrate expands to dominate the triptych. Signal and Control collapse to icon-only sidebars. Used when the user is deeply inspecting a record field, document, or detail.

### Layout

| Panel | Width | Content |
|-------|-------|---------|
| Signal | ~80px | Collapsed: icon-only vertical strip. Shows icons for each signal card type. Click icon to temporarily expand as an overlay. |
| Orchestrate | ~90% of triptych | Expanded workspace. Full-width Record Inspector or Document Viewer. |
| Control | ~80px | Collapsed: icon-only vertical strip. Shows tab icons (GitBranch, Clock, CheckCircle, ScrollText, Bot). Click icon to temporarily expand as an overlay. |

### Trigger

- User clicks a specific field in the Record Inspector to inspect/edit
- User opens a document or attachment for detailed viewing
- Any action that signals deep-dive intent on a single artifact

### Visual Treatment

| Property | Value |
|----------|-------|
| Orchestrate border | 1px amber glow (`#F59E0B` at 30% opacity) |
| Orchestrate shadow | 0 0 20px rgba(245, 158, 11, 0.1) inset |
| Collapsed panels | `--airlock-bg` background, icons centered vertically at 24px, `--airlock-muted` |
| Transition | 300ms ease for panel width changes |

### Collapsed Panel Behavior

- Icons are vertically stacked, one per signal card type (Signal) or one per tab (Control)
- Hovering an icon shows a tooltip with the section name
- Clicking an icon temporarily expands that panel as a 280px/300px overlay (floating over Orchestrate)
- Overlay dismisses when clicking outside it or pressing Escape
- Overlay has a subtle drop shadow: 0 8px 32px rgba(0,0,0,0.6)

### Exit

- Press Escape
- Click outside the focused artifact
- Keyboard: Cmd+1 (return to Standard)

---

## State 3: Action Focus

The Orchestrate panel splits into an editor and a live preview. Used for content creation workflows like New Patch.

### Layout

| Panel | Width | Content |
|-------|-------|---------|
| Signal | ~80px | Collapsed to icon-only (same as Artifact Focus) |
| Orchestrate | flex-1, split 50/50 | Left half: Structured editor. Right half: Live preview / diff view. |
| Control | ~300px | Normal width. Shows relevant tabs (Approvals, AI Agent) for the creation workflow. |

### Trigger

- User clicks "New Patch" button
- User initiates any structured creation action (new contract clause, new amendment)

### Orchestrate Split

```
+---------------------------+---------------------------+
|        Editor             |        Preview            |
|                           |                           |
|  When: [field]            |  Live rendered output     |
|  Then: [field]            |  or                       |
|  Because: [field]         |  Diff against current     |
|  Attachments: [upload]    |  version                  |
|                           |                           |
+---------------------------+---------------------------+
```

| Property | Value |
|----------|-------|
| Split divider | 4px draggable, same style as panel borders |
| Editor background | `--airlock-surface` (#0F1219) |
| Preview background | `--airlock-card` (#151923) |
| Default split | 50/50 |
| Min width | 300px each side |

### Editor Content (New Patch Example)

- **When** clause: Text input describing the condition or trigger
- **Then** clause: Text input describing the action or change
- **Because** clause: Text input describing the justification
- File upload zone: Drag-and-drop area for supporting documents
- Metadata: Target contract, effective date, priority selector

### Preview Content

- Rendered output: How the patch will appear when applied
- Diff view: Side-by-side or inline diff against the current contract version
- Toggle: Switch between rendered and diff views

### Visual Treatment

| Property | Value |
|----------|-------|
| Editor header | "New Patch" label, Fira Sans 15px semibold |
| Preview header | "Preview" or "Diff" label with toggle |
| No special border glow | (Amber glow is reserved for Artifact Focus) |

### Exit

- Submit the patch (returns to Standard with the new patch visible)
- Cancel / Escape (returns to Standard, prompts to save draft if edits exist)
- Keyboard: Cmd+1 (return to Standard)

---

## State 4: Gate Lock

A governance overlay state. All panels remain at their normal widths, but a prominent Governance Bar appears at the top of the Orchestrate panel. This state enforces that the user addresses a pending gate before continuing other work.

### Layout

| Panel | Width | Content |
|-------|-------|---------|
| Signal | ~280px | Normal. Gate-related signal cards are highlighted / pinned to top. |
| Orchestrate | flex-1 | Normal content with Governance Bar pinned to top. Content scrolls beneath it. |
| Control | ~300px | Normal. Approvals tab auto-selected. Approve/Reject/Clarify/Hold buttons prominent. |

### Trigger

- A gate becomes active on the current channel AND the logged-in user is a designated reviewer
- Automatic: the state activates without user action when conditions are met

### Governance Bar

A sticky horizontal bar at the top of the Orchestrate panel, appearing below the standard header bar.

```
+-----------------------------------------------------------------------+
| [Gate Icon] GATE REVIEW REQUIRED    [SLA: 2h 14m]   [Approve] [Reject] [Clarify] [Hold] |
+-----------------------------------------------------------------------+
```

| Property | Value |
|----------|-------|
| Height | 56px |
| Background | `--airlock-surface` with subtle red/amber top border (2px) based on urgency |
| Position | Sticky, top of Orchestrate (below header bar) |
| Z-index | Above scrolling content |
| Gate icon | Lucide Shield, 20px, gate color |
| Label | "GATE REVIEW REQUIRED", Fira Sans 13px semibold uppercase, `--airlock-text` |
| SLA countdown | Fira Code, 15px bold. Color follows SLA rules (green/amber/red). |

### Action Buttons in Governance Bar

| Button | Style | Action |
|--------|-------|--------|
| Approve | Green background (#22C55E), white text | Approves the gate, advances lifecycle |
| Reject | Red background (#EF4444), white text | Rejects, triggers rejection workflow |
| Clarify | Amber background (#F59E0B), dark text | Requests additional information from submitter |
| Hold | `--airlock-card` background, `--airlock-text` | Places the gate in hold state, pauses SLA |

All buttons: Fira Sans, 13px semibold, 8px 16px padding, 4px border radius.

### Governance Bar Urgency

| SLA Status | Top Border Color | Background Tint |
|------------|-----------------|------------------|
| > 50% time remaining | Green (#22C55E) | None |
| 25-50% time remaining | Amber (#F59E0B) | Subtle amber tint (2% opacity) |
| < 25% or overdue | Red (#EF4444) | Subtle red tint (3% opacity) |

### Exit

- User takes a gate action (Approve, Reject, Clarify, Hold)
- Gate is resolved by another reviewer
- Keyboard: Cmd+1 returns to Standard (but Governance Bar reappears if gate is still active)

---

## Quick Escape

All non-Standard states support fast return to Standard:

| Method | Behavior |
|--------|----------|
| Escape key | Returns to Standard from any state |
| Click outside focused content | Returns to Standard (Artifact Focus and Action Focus) |
| Cmd+1 | Force return to Standard |

## Keyboard Overrides

Power users can force a view state with keyboard shortcuts, overriding the automatic context detection.

| Shortcut | State |
|----------|-------|
| Cmd+1 | Standard |
| Cmd+2 | Artifact Focus |
| Cmd+3 | Action Focus |
| Cmd+4 | Gate Lock (only if a gate is active; no-op otherwise) |

## Transitions

| From | To | Duration | Easing |
|------|-----|----------|--------|
| Standard | Artifact Focus | 300ms | ease-in-out |
| Standard | Action Focus | 300ms | ease-in-out |
| Standard | Gate Lock | 200ms | ease (governance bar slides down) |
| Any | Standard | 250ms | ease-out |

Panel width transitions animate smoothly. Content within panels crossfades during state changes (150ms opacity transition).
