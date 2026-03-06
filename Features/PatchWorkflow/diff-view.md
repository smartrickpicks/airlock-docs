# Diff View

## Summary

The diff view provides a visual comparison between the original contract data and proposed changes from a patch. It appears in two primary contexts: the **Orchestrate panel's preview half** during Action Focus (where authors see their changes in real time) and the **Signal panel** during Gate Lock (where reviewers inspect what changed before approving or rejecting).

The diff view supports three types of comparisons -- field-level diffs for structured data, text diffs for free-text content, and table diffs for financial data -- and three layout modes that adapt to the available screen space and review context.

**DECISION:** Diffs are computed client-side using the `jsdiff` library and rendered via TipTap's decoration API. The diff engine does not modify the document -- it overlays visual annotations on a read-only view.

## Diff Types

### Field-Level Diff

For structured field changes (territory, royalty rates, dates, enums). Each changed field is rendered as a discrete comparison card.

**Card Structure:**

```
+-------------------------------------------------------------------+
| territory                                              [Modified] |
|-------------------------------------------------------------------|
| Original: North America                                           |
| Proposed: Worldwide                                               |
| Delta:    Value changed from "North America" to "Worldwide"       |
+-------------------------------------------------------------------+
```

**Color Coding:**

| Change Type | Visual Treatment | CSS Variable |
|-------------|-----------------|--------------|
| New value (field had no prior value) | Green left border (3px solid) | `--airlock-success` (#22C55E) |
| Removed value (field cleared to null) | Red strikethrough on original value | `--airlock-danger` (#EF4444) |
| Modified value | Amber left border (3px solid) | `--airlock-warning` (#F59E0B) |
| Unchanged (shown for context) | No border, muted text | `--airlock-muted` (#64748B) |

**Delta Summary:**

Each field card includes a human-readable delta summary generated from the then clause:

| Operation | Delta Text |
|-----------|-----------|
| `set` | `Value changed from "{before}" to "{after}"` |
| `increment` | `Value increased by {delta} (from {before} to {after})` |
| `append` | `"{value}" added to list` |
| `remove` | `"{value}" removed from list` |
| `clear` | `Value cleared (was "{before}")` |

---

### Text Diff

For free-text fields such as clauses, descriptions, notes, and legal prose. Uses character-level or word-level diffing depending on the field length.

**Diff Algorithm Selection:**

| Field Length | Algorithm | Rationale |
|-------------|-----------|-----------|
| < 500 characters | Character-level diff | Short fields benefit from precise highlighting |
| >= 500 characters | Word-level diff | Long fields are more readable with word-level grouping |

**Color Coding:**

| Change Type | Visual Treatment | CSS |
|-------------|-----------------|-----|
| Addition | Cyan background | `background: rgba(6, 182, 212, 0.2)` |
| Deletion | Red strikethrough with faint red background | `text-decoration: line-through; background: rgba(239, 68, 68, 0.15)` |
| Modification | Amber background | `background: rgba(245, 158, 11, 0.15)` |

**TipTap Integration:**

- Uses the **Diff Block** custom node from the Document Suite
- Additions and deletions are rendered as TipTap decorations (not actual document edits)
- **Version Marker** nodes track which version each section belongs to
- The diff view is always read-only -- no editing from the diff panel

---

### Table Diff

For financial data presented in tabular form (royalty rate schedules, territory splits, payment terms, advance schedules).

**Row-Level Comparison:**

```
+-------------------------------------------------------------------+
| Royalty Rate Schedule                                              |
|-------------------------------------------------------------------|
| Territory        | Rate (Original) | Rate (Proposed) | Delta     |
|------------------|----------------|-----------------|-----------|
| North America    | 15.0%          | 17.0%           | +2.0%     |  <- amber cell
| Europe           | 12.0%          | 12.0%           | --        |
| Asia Pacific     | --             | 10.0%           | NEW       |  <- green row
| South America    | 8.0%           | --              | REMOVED   |  <- red row
+-------------------------------------------------------------------+
```

**Visual Treatment:**

| Change Type | Row Treatment | Cell Treatment |
|-------------|--------------|----------------|
| Added row | Green left border (3px solid `#22C55E`) | All cells have faint green background |
| Removed row | Red left border (3px solid `#EF4444`) | All cells have faint red background, text strikethrough |
| Changed cell | No row-level change | Amber background on the specific changed cell |
| Unchanged row | No visual treatment | Standard cell rendering |

---

## Layout Modes

The diff view supports three layout modes. The default mode depends on the context, and users can toggle between modes via a segmented control.

### Side-by-Side (Default for Action Focus)

Original content on the left, revised content on the right. Both columns scroll in sync (scroll-locked).

```
+---------------------------------+---------------------------------+
|           ORIGINAL              |            REVISED              |
|---------------------------------|---------------------------------|
| territory: North America        | territory: Worldwide            |
| royalty_rate: 0.15              | royalty_rate: 0.17              |
| distribution: [Physical]        | distribution: [Physical,Digital]|
|                                 |                                 |
| "The licensor grants rights     | "The licensor grants rights     |
|  within the territory of        |  within the territory of        |
|  North America..."              |  Worldwide..."                  |
+---------------------------------+---------------------------------+
```

| Property | Value |
|----------|-------|
| Column split | 50/50 |
| Scroll behavior | Scroll-locked (both columns scroll together) |
| Minimum column width | 300px |
| Overflow behavior | If viewport < 600px, automatically switches to Inline mode |
| Divider | 1px `--airlock-border`, non-draggable |

### Inline (Default for Signal Panel)

Additions and deletions interleaved within a single column. Compact display suitable for narrow panels and event cards.

```
+-------------------------------------------------------------------+
| territory: North America -> Worldwide                             |
| royalty_rate: 0.15 -> 0.17 (+0.02)                               |
| distribution: +Digital                                            |
|                                                                   |
| "The licensor grants rights within the territory of               |
|  [-North America-]{+Worldwide+}..."                               |
+-------------------------------------------------------------------+
```

| Property | Value |
|----------|-------|
| Width | Single column, fills available width |
| Additions | Cyan background inline |
| Deletions | Red strikethrough inline |
| Use case | Signal panel event cards, narrow viewports, compact summaries |

### Unified (Default for Approval Chain Evidence Review)

Combined view with line-by-line markers in the left gutter, similar to a Git unified diff.

```
+-------------------------------------------------------------------+
|   | territory: North America                                      |
| - | territory: North America                                      |
| + | territory: Worldwide                                          |
|   | royalty_rate: 0.15                                            |
| - | royalty_rate: 0.15                                            |
| + | royalty_rate: 0.17                                            |
|   |                                                               |
|   | "The licensor grants rights within the territory of           |
| - |  North America for the duration of..."                        |
| + |  Worldwide for the duration of..."                            |
+-------------------------------------------------------------------+
```

| Property | Value |
|----------|-------|
| Gutter width | 24px |
| Gutter markers | `-` (red) for removed lines, `+` (green) for added lines, blank for context |
| Context lines | 3 lines of unchanged content shown above and below each change block |
| Use case | Approval chain evidence review, full-page diff reports |

### Mode Toggle

A segmented control in the diff header bar allows switching between modes:

```
[ Side-by-Side | Inline | Unified ]
```

| Property | Value |
|----------|-------|
| Component | Segmented control (3 segments) |
| Default selection | Context-dependent (see defaults per layout mode above) |
| Persistence | User's last selection is stored per-session in local state |
| Keyboard shortcut | `Cmd+Shift+D` cycles through modes |

---

## Diff Header Bar

A fixed header bar appears at the top of every diff view, providing context and controls.

```
+-------------------------------------------------------------------+
| PATCH-2847  Expand territory to Worldwide                         |
| @jane.doe                          [Submitted]                    |
| Changes: 2 fields modified, 1 added, 0 removed                   |
| [ Side-by-Side | Inline | Unified ]   [ Accept All | Reject All ]|
+-------------------------------------------------------------------+
```

### Header Bar Contents

| Element | Description | Position |
|---------|-------------|----------|
| Patch ID | Monospace badge, e.g., `PATCH-2847` | Top-left |
| Patch title | Free text title from the patch | Top-left, after ID |
| Author | Avatar (24px circle) + display name | Below title, left |
| State badge | Colored pill showing current patch state | Below title, right |
| Change summary | `"Changes: X fields modified, Y added, Z removed"` | Below author |
| Layout mode toggle | Segmented control | Bottom-right |
| Accept All / Reject All | Action buttons (Gate Lock only) | Bottom-right, before toggle |

### State Badge Colors

| State | Badge Color | Text Color |
|-------|------------|------------|
| Draft | `--airlock-muted` (#64748B) | `--airlock-text` |
| Submitted | `--airlock-accent` (#3B82F6) | White |
| Needs_Clarification | `--airlock-warning` (#F59E0B) | Dark text |
| Verifier_Approved | `--airlock-success` (#22C55E) | Dark text |
| Admin_Hold | `--airlock-warning` (#F59E0B) | Dark text |
| Admin_Approved | `--airlock-success` (#22C55E) | White |
| Applied | `--airlock-success` (#22C55E) | White |
| Rejected | `--airlock-danger` (#EF4444) | White |
| Cancelled | `--airlock-muted` (#64748B) | `--airlock-text` |

### Accept All / Reject All Buttons

These buttons appear only when the diff view is rendered in the **Gate Lock** state (State 4) and the current user is a designated reviewer (and not the patch author).

| Button | Style | Behavior |
|--------|-------|----------|
| Accept All | Green background (`#22C55E`), white text | Marks all individual changes as accepted |
| Reject All | Red background (`#EF4444`), white text | Marks all individual changes as rejected |

Both buttons trigger a confirmation dialog before executing. The dialog shows the count of changes being accepted or rejected.

---

## Per-Change Actions

During review (Gate Lock state only), reviewers can act on individual changes within the diff. This enables partial approvals -- accepting some changes while rejecting or questioning others.

### Available Actions

| Action | Icon | Description | Availability |
|--------|------|-------------|-------------|
| Approve change | Green checkmark (CheckCircle) | Marks this specific change as accepted | Gate Lock, reviewer only |
| Reject change | Red X (XCircle) | Marks this specific change as rejected | Gate Lock, reviewer only |
| Comment | Message bubble (MessageSquare) | Opens an inline comment thread on this change | Gate Lock and Standard (read-only in Standard) |
| Request clarification | Amber question mark (HelpCircle) | Flags this specific change for author response | Gate Lock, reviewer only |

### Per-Change Action Bar

Each change card (field diff, text diff block, or table row group) renders a floating action bar on hover:

```
+-------------------------------------------------------------------+
| territory                                              [Modified] |
|-------------------------------------------------------------------+
| Original: North America                                           |
| Proposed: Worldwide                                               |
|                                    [check] [x] [comment] [?]     |
+-------------------------------------------------------------------+
```

| Property | Value |
|----------|-------|
| Visibility | Appears on hover (desktop) or on tap (mobile) |
| Position | Bottom-right of the change card |
| Icon size | 20px |
| Spacing | 8px between icons |
| Tooltip | Each icon shows a tooltip on hover (e.g., "Approve this change") |

### Inline Comment Thread

When a reviewer clicks the comment icon, an inline thread expands below the change card:

```
+-------------------------------------------------------------------+
| territory                                              [Modified] |
|-------------------------------------------------------------------|
| Original: North America                                           |
| Proposed: Worldwide                                               |
|                                                                   |
|   +---------------------------------------------------------------+
|   | @admin.user (2h ago):                                         |
|   | "Is there a signed amendment supporting this expansion?"      |
|   +---------------------------------------------------------------+
|   | @jane.doe (1h ago):                                           |
|   | "Yes, see attached amendment-v2.pdf in the evidence pack."    |
|   +---------------------------------------------------------------+
|   | [Type a reply...]                               [Send]        |
|   +---------------------------------------------------------------+
+-------------------------------------------------------------------+
```

| Property | Value |
|----------|-------|
| Thread storage | Stored as JSONB array on the patch record, keyed by field name |
| Notifications | Comment creates a notification for the patch author and all previous commenters |
| Timestamps | Relative format (e.g., "2h ago"), full timestamp on hover |
| Max thread depth | Flat (no nested replies) -- all comments are at the same level |

### Per-Change State Aggregation

The patch's overall approval state considers individual change states:

| Scenario | Result |
|----------|--------|
| All changes approved | Patch can proceed to next approval step |
| Any change rejected | Patch can be transitioned to Rejected or Needs_Clarification |
| Mixed (some approved, some pending) | Approve button remains disabled until all changes are resolved |
| All changes have comments but no approval/rejection | Patch remains in current state, reviewer must explicitly act |

---

## Diff Rendering with TipTap

The diff view leverages the Document Suite's TipTap infrastructure for rendering text diffs in free-text fields and clause content.

### Custom Nodes Used

| Node | Role in Diff View |
|------|-------------------|
| **Diff Block** | Container node wrapping a before/after comparison. Renders with appropriate background coloring. |
| **Version Marker** | Inline node marking which version (v1, v2, clarification response, etc.) a text segment belongs to. |

### Diff Computation Pipeline

1. **Source values**: Retrieve the before value and after value from the evidence pack snapshot
2. **Tokenization**: Split text into tokens (characters for short fields, words for long fields)
3. **Diff algorithm**: Run `jsdiff.diffWords()` or `jsdiff.diffChars()` to produce a change list
4. **Decoration mapping**: Convert the jsdiff output into TipTap Decoration objects
5. **Rendering**: Apply decorations to a read-only TipTap editor instance

### Decoration Types

| Decoration | CSS Class | Visual |
|-----------|-----------|--------|
| `diff-addition` | `.diff-add` | Cyan background, no strikethrough |
| `diff-deletion` | `.diff-del` | Red background, strikethrough |
| `diff-modification` | `.diff-mod` | Amber background |
| `diff-context` | `.diff-ctx` | No background, standard text |

### Performance Considerations

| Concern | Mitigation |
|---------|-----------|
| Large text fields (> 10,000 chars) | Diff computed in a Web Worker to avoid blocking the main thread |
| Many fields in a single patch | Virtual scrolling -- only visible diff cards are rendered |
| Rapid edits in Action Focus | Debounced recomputation (300ms) to avoid excessive rerenders |
| Memory | Diff decorations are lightweight (position + class name only) |

---

## Version Timeline Scrubber

The version timeline scrubber provides a horizontal navigation bar for browsing the patch's version history. It appears at the top of the diff view, below the diff header bar.

### Layout

```
+-------------------------------------------------------------------+
| DIFF HEADER BAR                                                   |
+-------------------------------------------------------------------+
| [Draft v1] --- [Draft v2] --- [Submitted] --- [Clarification] --- [Resubmitted] |
|     o-----------o--------------o----------------o------------------o              |
|                                                 ^                                |
|                                            (selected)                            |
+-------------------------------------------------------------------+
| DIFF CONTENT                                                      |
+-------------------------------------------------------------------+
```

### Scrubber Nodes

Each node on the timeline represents a distinct version of the patch evidence:

| Event | Node Label | Description |
|-------|-----------|-------------|
| Initial draft save | `Draft v1` | First saved version of the evidence pack |
| Subsequent draft edits | `Draft v2`, `v3`, etc. | Each save during Draft state creates a new version |
| Submission | `Submitted` | Evidence frozen at submission time |
| Clarification request | `Clarification` | Marks the point where reviewer requested more info |
| Author response | `Response R1` | Author's clarification response (round 1, 2, etc.) |
| Resubmission | `Resubmitted` | Evidence re-frozen after clarification cycle |
| Application | `Applied` | Final state with actual values |

### Interaction

| Action | Behavior |
|--------|----------|
| Click a single node | Shows the diff between that version and the immediately previous version |
| Click-drag to select a range | Shows the diff between the start node and the end node (arbitrary version comparison) |
| Shift+click two nodes | Selects those two nodes as the comparison endpoints |
| Hover a node | Tooltip shows: version label, timestamp, actor name |
| Keyboard left/right arrows | Moves the selected node forward or backward (when scrubber is focused) |

### Visual Design

| Property | Value |
|----------|-------|
| Scrubber height | 48px |
| Background | `--airlock-surface` (#0F1219) |
| Track line | 2px solid `--airlock-border` (#1E2330) |
| Node circle | 12px diameter, `--airlock-muted` fill by default |
| Selected node | 12px diameter, `--airlock-accent` (#3B82F6) fill, 2px white ring |
| Range highlight | Track segment between selected nodes changes to `--airlock-accent` at 40% opacity |
| Node label | Fira Sans, 11px, `--airlock-muted`, positioned below the node |
| Overflow | Horizontally scrollable if more than 8 nodes, with fade-out gradients at edges |

### Version Data Model

Each version is stored as an entry in the patch's history JSONB array:

```json
{
  "version": 3,
  "label": "Draft v3",
  "timestamp": "2025-12-01T11:45:00Z",
  "actor_id": "user_123",
  "snapshot": {
    "when_clause": { "..." },
    "then_clause": { "..." },
    "because_clause": "...",
    "attachments": ["att_001", "att_002"]
  }
}
```

The diff engine computes the difference between any two version snapshots on demand. Snapshots are immutable once created.

---

## Contextual Behavior by View State

The diff view adapts its behavior depending on which triptych view state it appears in:

| View State | Location | Layout Default | Per-Change Actions | Accept/Reject All | Scrubber |
|------------|----------|---------------|-------------------|-------------------|----------|
| Action Focus (State 3) | Orchestrate panel, right half | Side-by-Side | No (author cannot self-review) | No | No (only Draft versions visible) |
| Gate Lock (State 4) | Signal panel + Orchestrate governance overlay | Unified | Yes | Yes | Yes |
| Standard (State 1) | Orchestrate panel (when viewing a patch) | Side-by-Side | Comment only (read-only) | No | Yes |
| Artifact Focus (State 2) | Orchestrate panel (expanded) | Side-by-Side | Comment only (read-only) | No | Yes |

---

## Related Specs

- [Patch Workflow Overview](./overview.md) -- States, transitions, optimistic locking
- [Evidence Pack](./evidence-pack.md) -- Structured evidence payload (when/then/because clauses, attachments, snapshots)
- [Action Focus](./action-focus.md) -- Triptych layout for patch authoring (editor + preview)
- [Approval Chain](./approval-chain.md) -- Vertical timeline UI for approval steps
- [SLA Timer](./sla-timer.md) -- Countdown timer for reviewer deadlines
- [Document Suite](/Features/DocumentSuite/overview.md) -- TipTap editor, Diff Block and Version Marker custom nodes, PDF.js viewer
- [Shell / View States](/Features/Shell/view-states.md) -- Triptych view states (Standard, Artifact Focus, Action Focus, Gate Lock)
