# Triptych

## Summary

The Triptych is the three-panel workspace that occupies all remaining space to the right of the Channel Sidebar. It follows a **Signal | Orchestrate | Control** pattern. All content loads inline within these panels -- users never navigate away from the triptych.

## Panel Overview

| Panel | Position | Default Width | Role | Interactivity |
|-------|----------|---------------|------|---------------|
| Signal | Left | ~280px | Read-only intelligence feed | Passive (read, dismiss, pin) |
| Orchestrate | Center | flex-1 (remaining) | Primary workspace | Full interactivity (edit, inspect, create) |
| Control | Right | ~300px | Action and governance controls | Interactive (approve, configure, trigger) |

## Layout

```
+------------------+---------------------------+-------------------+
|     Signal       |       Orchestrate         |      Control      |
|     ~280px       |        flex-1             |      ~300px       |
|                  |                           |                   |
|  Read-only       |  Main workspace           |  Interactive      |
|  intelligence    |  Record Inspector         |  controls         |
|  feed            |  Document Viewer          |  Lifecycle state  |
|                  |  Editors                  |  SLA timer        |
|                  |                           |  Approvals        |
|                  |                           |  Audit trail      |
+------------------+---------------------------+-------------------+
```

Total width: viewport width minus Module Bar (72px) minus Channel Sidebar (240px).

---

## Signal Panel (Left)

### Purpose

Signal is the read-only intelligence feed. It surfaces notifications, events, AI-generated suggestions, gate status, and health scores relevant to the currently active channel.

### Content Types

| Content | Description | Visual |
|---------|-------------|--------|
| Gate Status | Current gate state with color indicator | Color-coded card with gate dot and label |
| Health Score | Numeric health percentage with trend arrow | Large number + sparkline or trend indicator |
| Notifications | Time-ordered events (status changes, assignments, SLA warnings) | Card stack, newest on top |
| AI Suggestions | AI-generated recommendations, risk flags, next-action prompts | Card with cyan left border and subtle glow |
| Activity Feed | Recent actions taken on this record (edits, approvals, comments) | Timeline-style list with timestamps |

### Signal Card Anatomy

```
[Color Border] [Icon] Card Title        [Timestamp]
               Card body text or metric
               [Action link if applicable]
```

| Property | Value |
|----------|-------|
| Background | `--airlock-card` (#151923) |
| Border radius | 6px |
| Left border | 3px, color varies by type |
| Padding | 12px |
| Margin | 0 0 8px 0 |
| Title font | Fira Sans, 13px semibold, `--airlock-text` |
| Body font | Fira Sans, 12px, `--airlock-muted` |
| Timestamp | Fira Code, 11px, `--airlock-muted`, right-aligned |

### Signal Card Left Border Colors

| Card Type | Border Color |
|-----------|-------------|
| Gate alert / critical | #EF4444 (red) |
| SLA warning | #F59E0B (amber) |
| AI suggestion | #00D1FF (cyan) |
| Status update | #22C55E (green) |
| Neutral info | `--airlock-border` (#1E2330) |

### Interactions

- Scroll: Independent vertical scroll within the panel
- Dismiss: Swipe right or click X to dismiss a notification card
- Pin: Click pin icon to keep a card at the top of the feed
- No editing: Signal cards are strictly read-only

---

## Orchestrate Panel (Center)

### Purpose

Orchestrate is the primary workspace. It hosts the main content for the active channel: Record Inspector, Document Viewer, editors, forms, and any other content the user directly works with.

### Content Modes

| Mode | Trigger | Description |
|------|---------|-------------|
| Record Inspector | Default for contract/CRM channels | Sectioned field viewer with inline editing |
| Document Viewer | Opening a document or attachment | PDF/document renderer with annotation tools |
| Editor | Creating a new patch or editing content | Full editor with structured input fields |
| Triage Grid | Opening Triage Dashboard channel | Filtered data grid of actionable items |
| Generator | Opening Generator channel | Contract generation wizard/form |

### Layout Rules

- Orchestrate always fills the remaining horizontal space (flex-1)
- Content scrolls vertically within the panel
- Sticky headers (e.g., record title bar, governance bar) pin to the top of the Orchestrate panel
- In Artifact Focus view state, Orchestrate expands to ~90% of triptych width

### Header Bar

A sticky bar at the top of the Orchestrate panel showing context for the active content.

| Property | Value |
|----------|-------|
| Height | 48px |
| Background | `--airlock-surface` (#0F1219) |
| Border | 1px bottom, `--airlock-border` |
| Content | [Back breadcrumb] Record/Document title [Action buttons] |
| Font | Fira Sans, 15px semibold, `--airlock-text` |

---

## Control Panel (Right)

### Purpose

Control is the interactive governance and action panel. It provides lifecycle management, SLA monitoring, approval workflows, audit trails, and the AI Agent interface.

### Tabs

The Control panel uses a tabbed interface at the top.

| Tab | Icon | Content |
|-----|------|---------|
| Lifecycle | Lucide GitBranch | State machine visualization, current state, transition buttons |
| SLA | Lucide Clock | SLA countdown timer, deadline list, escalation status |
| Approvals | Lucide CheckCircle | Approval chain visualization, approve/reject/clarify/hold buttons |
| Audit | Lucide ScrollText | Chronological audit trail of all actions and state changes |
| AI Agent | Lucide Bot | AI conversation interface, automated action suggestions, agent logs |

### Tab Bar

| Property | Value |
|----------|-------|
| Height | 40px |
| Background | `--airlock-surface` (#0F1219) |
| Tab font | Fira Sans, 12px, `--airlock-muted` (inactive), `--airlock-text` (active) |
| Active indicator | 2px bottom border in `--airlock-cyan` |
| Icon size | 16px, left of tab label |

### Lifecycle Tab

- State diagram: Visual node graph showing lifecycle states and transitions
- Current state: Highlighted node with pulsing border
- Transition buttons: Available transitions shown as action buttons below the diagram
- State metadata: Entry date, duration in current state, assigned owner

### SLA Tab

- Countdown timer: Large Fira Code display (HH:MM:SS or DD:HH:MM) with color coding
  - Green: > 50% time remaining
  - Amber: 25-50% time remaining
  - Red: < 25% time remaining or overdue
- Deadline list: Upcoming SLA milestones in chronological order
- Escalation status: Current escalation level and next escalation trigger

### Approvals Tab

- Approval chain: Vertical stepper showing each approver in sequence
- Current approver: Highlighted with pulsing indicator
- Action buttons: Approve, Reject, Clarify, Hold (only visible when user is the current reviewer)
- History: Previous approval decisions with timestamps and comments

### Audit Tab

- Chronological log: Newest entries at top
- Entry format: [Timestamp] [Actor] [Action] [Details]
- Timestamp font: Fira Code, 11px
- Filterable by action type, actor, date range

### AI Agent Tab

- Chat interface: Message bubbles for AI agent communication
- AI messages: Cyan-tinted background, subtle glow effect
- Suggested actions: Clickable action chips below AI messages
- Agent logs: Collapsible section showing agent reasoning and actions taken

---

## Panel Resize

Users can resize panels by dragging the borders between them.

| Property | Value |
|----------|-------|
| Drag handle | 4px wide hit area on panel borders, cursor: col-resize |
| Visual indicator | 1px line in `--airlock-border`, brightens to `--airlock-cyan` on hover/drag |
| Minimum widths | Signal: 200px, Orchestrate: 400px, Control: 200px |
| Collapsed width | ~80px (icon-only mode, see view-states.md) |
| Transition | 200ms ease when auto-resizing (view state changes); instant when user-dragging |
| Persistence | User-set widths persist per channel type (not per individual channel) |

---

## Inline Principle

Everything happens within the triptych. Specific behaviors enforced:

- Clicking a document opens it in the Orchestrate panel (Document Viewer mode), not in a new page
- Clicking an approval button triggers the flow within the Control panel
- AI Agent conversations happen in the Control panel's AI Agent tab
- Errors and confirmations appear as inline toasts or modals overlaying the Orchestrate panel
- Deep links load the full shell with the correct module, channel, and triptych state
