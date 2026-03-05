# SLA Timer

## Summary

The SLA timer is a countdown component displayed on the active step of the approval chain. It tracks the time remaining for a reviewer to take action on a patch gate. The timer provides escalating visual urgency as the deadline approaches, and triggers notifications when overdue.

## When Active

An SLA timer appears when all of the following are true:

- A patch gate is assigned to a specific reviewer
- The gate has a configured SLA deadline
- The gate is in an active (non-held) state

If no SLA is configured for a gate, the active step shows "No deadline" in place of the timer.

## Display

### Countdown Format

- **Font:** Monospace
- **Format:** `HH:MM:SS` remaining
- **Updates:** Every second (real-time decrement)

### Color Coding

| Time Remaining | Color | Additional Effect |
|----------------|-------|-------------------|
| More than 4 hours | Green | None |
| Less than 1 hour | Amber | None |
| Less than 15 minutes | Red | None |
| Less than 5 minutes | Blinking Red | Pulsing animation (0.5s interval) |
| Overdue (past deadline) | Red | Negative time displayed (e.g., `-00:12:34`); notification sent to admin |

### Progress Ring

- A circular progress ring wraps around the timer digits
- The ring depletes clockwise as time passes (full circle at start, empty at deadline)
- Ring color matches the countdown color coding above
- Ring stroke width: 3px
- Ring diameter: sized to enclose the countdown text comfortably

### Reviewer Context

Displayed beside the timer:

- **Reviewer name** (full display name)
- **Role badge** (e.g., "Verifier", "Admin") as a small pill/tag

```
  +-------------------+
  |   [progress ring] |   john.smith  [Verifier]
  |    02:34:18       |
  +-------------------+
```

## Admin Actions

Two action buttons appear below the timer, visible only to users with the Admin role:

| Button | Label | Behavior |
|--------|-------|----------|
| **Extend** | "Extend" | Opens an inline input to add time (hours/minutes); extends the SLA deadline |
| **Reassign** | "Reassign" | Opens a reviewer picker to change the assigned reviewer |

### Extend

- Admin clicks "Extend" and a small inline form appears
- Input: hours and minutes to add (e.g., +2h 30m)
- Confirm adds the specified time to the current deadline
- The progress ring recalculates to reflect the new total duration
- A history entry is logged: "SLA extended by [amount] by [admin] at [timestamp]"

### Reassign

- Admin clicks "Reassign" and a reviewer picker dropdown appears
- Lists eligible reviewers for the current gate type (verifiers for verifier gates, admins for admin gates)
- Selecting a new reviewer updates the assignment
- The SLA timer resets to the full SLA duration for the new reviewer
- A history entry is logged: "Reassigned from [old] to [new] by [admin] at [timestamp]"

## Timer Pause (Hold State)

When a gate enters the **Admin_Hold** state:

- The countdown freezes at its current value
- The progress ring stops depleting
- A "PAUSED" label overlays the timer in a muted style
- The timer color resets to a neutral gray
- Time does not accumulate while on hold

When the hold is released (transition to Admin_Approved or Rejected):

- If the gate is still active, the countdown resumes from where it was paused
- The progress ring resumes depleting

## Timer Reset

The SLA timer resets to the full configured duration when:

- The assigned reviewer changes (via Reassign)
- The patch transitions back from Needs_Clarification to an active review state

The timer does **not** reset when:

- An admin extends the deadline (time is added, not reset)
- The gate goes on hold and then resumes (paused time is preserved)

## Overdue Behavior

When the timer reaches `00:00:00` and continues past the deadline:

- The display switches to negative time: `-HH:MM:SS` in red
- An **URGENT** notification is sent to all workspace admins
- The notification appears in:
  - The channel's Signal panel
  - The Triage Dashboard "Recent Events" tab
  - The Module bar badge (red dot increment)
- The overdue state persists until the reviewer takes action or an admin intervenes

## SLA Configuration

| Setting | Scope | Default |
|---------|-------|---------|
| SLA enabled | Per workspace | Disabled (no SLA) |
| Default SLA duration | Per workspace | Manual setting per gate |
| Per-gate override | Per gate type (Verifier, Admin) | Inherits workspace default |
| Custom deadline | Per individual patch gate | Overrides all defaults |

SLA configuration is managed through the Admin module's workspace settings.

## Related Specs

- [Approval Chain](./approval-chain.md) -- Where the timer is displayed
- [Patch Workflow Overview](./overview.md) -- States and transitions that affect the timer
- [Notification Triggers](/Features/Notifications/notification-triggers.md) -- SLA overdue and approaching triggers
