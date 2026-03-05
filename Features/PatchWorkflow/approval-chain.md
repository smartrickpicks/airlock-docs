# Approval Chain

## Summary

The approval chain is a vertical timeline displayed in the Control panel that visualizes each gate a patch must pass through before it can be applied. Each step in the chain shows its current state, the assigned reviewer, and available actions. The chain enforces self-approval prevention and evidence review requirements.

## Location

The approval chain renders inside the **Control panel** (~300px right zone of the triptych). It is visible in both Standard view (when viewing a submitted patch) and Action Focus view (during authoring, showing the upcoming steps).

## Visual Design

The timeline is a vertical track with connected dots, one per approval step. Steps are ordered top-to-bottom in chronological sequence.

```
  [green dot]   Submitted         jane.doe     2025-06-01 14:22
       |
  [cyan dot]    Verifier Review   john.smith   SLA: 02:34:18
       |
  [gray dot]    Admin Review      (unassigned)
       |
  [gray dot]    Applied           (system)
```

## Step States

| State | Visual | Description |
|-------|--------|-------------|
| **Completed** | Green filled dot | Step has been completed successfully |
| **Active** | Cyan pulsing dot | Step is currently awaiting action |
| **Pending** | Gray outlined dot | Step has not yet been reached |
| **Rejected** | Red filled dot | Step resulted in a rejection |
| **Returned** | Amber filled dot | Step was sent back for clarification |

### Step Detail Display

Each step shows the following information:

| Element | Completed | Active | Pending | Rejected | Returned |
|---------|-----------|--------|---------|----------|----------|
| Dot color | Green | Cyan (pulsing) | Gray | Red | Amber |
| Step label | Gate name | Gate name | Gate name | Gate name | Gate name |
| Actor | Name + timestamp | Assigned reviewer | "(unassigned)" or role | Name + reason | Name + "Clarification requested" |
| SLA timer | -- | Countdown display | -- | -- | -- |
| Actions | -- | Action buttons | -- | -- | Author response form |

## Self-Approval Block

When the currently logged-in user is the patch author:

- The **"Approve"** button is **hidden** (not disabled, not grayed out -- completely removed from the DOM)
- A subtle tooltip-trigger icon appears in its place; hovering shows: "You cannot approve your own patch"
- This applies to both the Verifier_Approved and Admin_Approved transitions
- The Reject, Request Clarification, and Hold buttons remain visible (authors can see feedback options but not self-approve)

## Evidence Requirement

Before a reviewer can approve a patch, they must have reviewed all attached evidence:

- The **"Approve"** button is **disabled** until the reviewer has viewed all evidence items
- Evidence viewing is tracked via scroll-to-bottom detection on the evidence panel
- Each evidence item (file attachment, before/after comparison) has an "unseen" indicator until scrolled into view
- Once all items have been scrolled past, the Approve button enables
- This requirement does not apply to Reject, Request Clarification, or Hold actions

## Actions Per Step

When a step is in the **Active** state, the assigned reviewer sees the following action buttons:

| Action | Button Style | Requirements | Result |
|--------|-------------|--------------|--------|
| **Approve** | Primary (green) | All evidence viewed; actor is not the author | Advances to next state |
| **Reject** | Destructive (red) | Reason required (text input) | Transitions to Rejected |
| **Request Clarification** | Secondary (amber) | Message required (text input) | Transitions to Needs_Clarification; returns to author |
| **Hold** | Secondary (gray) | Reason required (text input); admin only | Transitions to Admin_Hold; SLA timer pauses |

### Action Confirmation

- **Approve:** Single click (no confirmation modal, since evidence review is the gate)
- **Reject:** Inline text input expands below button; must enter reason, then confirm
- **Request Clarification:** Inline text input expands below button; must enter message, then confirm
- **Hold:** Inline text input expands below button; must enter reason, then confirm; admin role required

## Timeline Interaction

- Clicking a completed step expands it to show full details (actor, timestamp, any notes)
- Clicking an active step scrolls to the action buttons
- Completed steps are collapsible to save vertical space
- The timeline auto-scrolls to keep the active step visible

## Edge Cases

| Scenario | Behavior |
|----------|----------|
| Patch has no SLA configured | Active step shows "No deadline" instead of timer |
| Reviewer is reassigned | Active step updates to new reviewer; SLA timer resets |
| Patch is in Admin_Hold | Active step shows "On Hold" badge; SLA timer paused |
| Multiple patches on same contract | Each patch has its own independent approval chain |
| Patch returns from Otto | A new "Otto Return Review" step appears at the bottom of the chain |

## Related Specs

- [Patch Workflow Overview](./overview.md) -- States, transitions, authorization rules
- [SLA Timer](./sla-timer.md) -- Timer component displayed on active steps
- [Action Focus](./action-focus.md) -- Layout when authoring a new patch
