# Action Focus -- Patch Authoring

## Summary

When a user initiates a new patch, the triptych automatically switches to the **Action Focus** view state. This is a dedicated full-editor experience that takes over the Orchestrate and Control panels, narrowing the Signal panel to an icon-only strip. Action Focus provides a structured authoring environment on the left and a live preview on the right.

**DECISION:** Patch authoring uses Action Focus (full editor), not inline editing from field cards.

## Trigger

- User clicks the **"New Patch"** button (available in the Control panel or via keyboard shortcut)
- The triptych auto-switches to Action Focus layout
- No manual toggle needed; the system detects the authoring context

## Layout

```
+--------+-----------+-----------------------------------------------------+
| Module | Channel   |                  Triptych (Action Focus)             |
|  Bar   | Sidebar   |  Signal  |    Editor     |    Preview    | Control  |
| 72px   | 240px     |  ~80px   |   flex-1      |   flex-1      | ~300px   |
|        |           | (icons)  |               |               |          |
+--------+-----------+----------+---------------+---------------+----------+
```

| Zone | Width | Behavior in Action Focus |
|------|-------|--------------------------|
| Signal panel | Narrows to ~80px | Icon-only mode; event icons still visible, no text |
| Orchestrate (left half) | flex-1 | Patch editor form |
| Orchestrate (right half) | flex-1 | Live preview / diff |
| Control panel | ~300px | Patch workflow state, submit, history |

## Editor Content (Orchestrate Left)

The editor presents a structured form for composing the patch evidence pack:

### 1. Field Selector

- Dropdown or searchable picker to select which contract field(s) to patch
- Shows the current value of the selected field for reference
- Supports multi-field patches (each field gets its own when/then pair)

### 2. Intent Input

- Free text input describing the change and why it is needed
- Serves as a quick summary before the detailed clauses below
- Placeholder: "What are you changing and why?"

### 3. When Clause Builder

- Visual builder that produces a JSON condition
- Drag-and-drop or form-based interface for constructing conditional logic
- Supports field references, comparison operators, logical AND/OR grouping
- Raw JSON toggle for power users
- Output stored as `when_clause` in the evidence pack

### 4. Then Clause Builder

- Visual builder that produces a JSON action
- Defines the specific data mutation: which field, what new value, what operation
- Supports set, append, remove, increment operations
- Raw JSON toggle for power users
- Output stored as `then_clause` in the evidence pack

### 5. Because Clause

- Free text reasoning field
- Supports multi-line input
- Explains the business rationale for the change
- Output stored as `because_clause` in the evidence pack

### 6. Evidence Attachment

- File upload area (drag-and-drop or browse)
- Displays before/after values for the targeted field
- Accepted formats: PDF, PNG, JPG, CSV, XLSX, TXT
- File size limit: configurable per workspace

## Preview Content (Orchestrate Right)

The preview panel provides real-time feedback on the proposed patch:

### Live Diff Preview

- Side-by-side comparison of the original value vs. the proposed change
- Uses standard diff coloring: red for removed, green for added
- Updates as the author modifies the editor fields

### Preflight Simulation

- Shows what would change if this patch were applied to the current contract state
- Lists all affected fields, downstream calculations, and related records
- Warnings for any conflicts with other pending patches on the same field

### AI Confidence Indicator

- Displayed only if the AI agent is reviewing the patch in real time
- Shows a confidence score (percentage) for the proposed change
- Color-coded: green (high confidence), amber (medium), red (low)
- Expandable to show the AI's reasoning summary

## Control Panel (Right)

When in Action Focus, the Control panel shows:

| Section | Content |
|---------|---------|
| Patch state badge | Current lifecycle state (Draft while authoring) |
| Submit button | Primary action to transition Draft to Submitted |
| Save Draft button | Persist work without submitting |
| Approval chain | Vertical timeline showing upcoming approval steps (all Pending while in Draft) |
| History | Transition log for this patch (empty until first submission) |

## Keyboard Shortcuts

| Shortcut | Action |
|----------|--------|
| `Cmd+3` | Force switch to Action Focus view state (from any state) |
| `Cmd+P` | Open patch authoring (equivalent to clicking "New Patch") |
| `Escape` | Return to Standard view state (prompts to save draft if unsaved changes) |
| `Cmd+Enter` | Submit patch (same as clicking Submit button) |
| `Cmd+S` | Save draft |

## Transition Back to Standard

- Pressing Escape or completing submission returns the triptych to Standard view
- Signal panel re-expands to its normal ~280px width
- Orchestrate returns to single-pane content
- If the user has unsaved changes, a confirmation dialog appears before exiting

## Related Specs

- [Patch Workflow Overview](./overview.md) -- States, transitions, evidence pack
- [Approval Chain](./approval-chain.md) -- Vertical timeline in Control panel
- [SLA Timer](./sla-timer.md) -- Timer displayed during active review gates
- [Shell / View States](/Features/Shell/view-states.md) -- Triptych layout modes
