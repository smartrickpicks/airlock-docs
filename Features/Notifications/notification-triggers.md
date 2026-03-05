# Notification Triggers

## Summary

This document defines every event that generates a notification in Airlock, its priority level, badge behavior, and where it surfaces. This is the canonical reference for the notification system.

## Trigger Table

| Trigger | Priority | Badge Scope | Surfaces In |
|---------|----------|-------------|-------------|
| SLA overdue | URGENT (red pulse) | Channel + Module | Signal panel, Triage Dashboard, Module bar badge |
| Patch rejected | URGENT (red) | Channel + Module | Signal panel, Triage Dashboard |
| Contract failed 3x | URGENT (red) | Module | Triage Dashboard, Admin module |
| Patch needs review | ACTION (amber) | Channel + Module | Signal panel, Triage Dashboard |
| Triage item assigned | ACTION (amber) | Channel + Module | Signal panel, Triage Dashboard |
| Entity needs disambiguation | ACTION (amber) | Channel | Signal panel |
| SLA approaching (<1hr) | ACTION (amber) | Channel | Signal panel, Control panel SLA timer |
| Extraction completed | INFO (blue) | Channel | Signal panel |
| Lifecycle state changed | INFO (blue) | Channel | Signal panel |
| Health score changed >10% | INFO (blue) | Channel | Signal panel |
| Batch processing completed | INFO (blue) | Module | Triage Dashboard |
| AI suggestion available | AI (cyan glow) | Channel | Signal panel, AI Agent tab (Control panel) |

## Priority Definitions

### URGENT (Red)

- **Visual:** Red text/icon with pulse animation
- **Badge:** Contributes to both channel sidebar blue dot AND module bar red dot + count
- **Behavior:** Persists until explicitly resolved or acknowledged
- **Future:** Will trigger browser push notifications and email

### ACTION (Amber)

- **Visual:** Amber text/icon, no animation
- **Badge:** Contributes to both channel sidebar blue dot AND module bar red dot + count
- **Behavior:** Persists until the required action is taken (review completed, item triaged, etc.)
- **Future:** Will appear in daily email digest if unresolved

### INFO (Blue)

- **Visual:** Blue text/icon, no animation
- **Badge:** Contributes to channel sidebar blue dot ONLY (does not increment module badge)
- **Behavior:** Automatically marked as read when the user opens the channel and scrolls past
- **Future:** No push or email; in-app only

### AI (Cyan Glow)

- **Visual:** Cyan text/icon with subtle glow effect
- **Badge:** Contributes to channel sidebar blue dot ONLY (does not increment module badge)
- **Behavior:** Appears in Signal panel as a feed event AND in the AI Agent tab of the Control panel
- **Future:** No push or email; in-app only

## Trigger Details

### SLA Overdue

- **Source:** SLA timer component in the approval chain
- **Fires when:** Timer crosses `00:00:00` into negative time
- **Payload:** Patch ID, gate name, assigned reviewer, overdue duration
- **Recipient:** All workspace admins + assigned reviewer
- **Resolution:** Reviewer takes action, admin extends SLA, or admin reassigns

### Patch Rejected

- **Source:** Approval chain rejection action
- **Fires when:** A reviewer clicks Reject on any approval gate
- **Payload:** Patch ID, rejecting reviewer, rejection reason, gate name
- **Recipient:** Patch author + all previous approvers in the chain
- **Resolution:** Author creates a new patch or the event ages out

### Contract Failed 3x

- **Source:** Contract lifecycle engine
- **Fires when:** A contract has failed processing three or more times
- **Payload:** Contract ID, failure count, last failure reason, last failure timestamp
- **Recipient:** All workspace admins
- **Resolution:** Admin investigates and resolves the underlying issue

### Patch Needs Review

- **Source:** Patch workflow state transition (Draft to Submitted, or Verifier_Responded)
- **Fires when:** A patch enters a state that requires reviewer action
- **Payload:** Patch ID, author name, target field(s), summary of proposed change
- **Recipient:** Assigned reviewer for the active gate
- **Resolution:** Reviewer approves, rejects, or requests clarification

### Triage Item Assigned

- **Source:** Triage assignment engine (auto or manual)
- **Fires when:** A triage item is assigned to a specific user
- **Payload:** Triage item ID, contract reference, assignment reason, assigner name
- **Recipient:** Assigned user
- **Resolution:** User opens and processes the triage item

### Entity Needs Disambiguation

- **Source:** Entity extraction pipeline
- **Fires when:** An extracted entity matches multiple existing records and cannot be auto-resolved
- **Payload:** Entity name, match candidates (with confidence scores), contract context
- **Recipient:** Contract channel owner / assigned analyst
- **Resolution:** User selects the correct entity match in the Signal panel inline resolver

### SLA Approaching (<1hr)

- **Source:** SLA timer component
- **Fires when:** Timer drops below 1 hour remaining
- **Payload:** Patch ID, gate name, assigned reviewer, time remaining
- **Recipient:** Assigned reviewer
- **Resolution:** Reviewer takes action before the deadline

### Extraction Completed

- **Source:** Data extraction pipeline
- **Fires when:** An extraction job finishes (success or partial success)
- **Payload:** Contract ID, extraction type, fields extracted count, confidence summary
- **Recipient:** Contract channel subscribers
- **Resolution:** Informational; no action required

### Lifecycle State Changed

- **Source:** Contract lifecycle engine
- **Fires when:** A contract transitions from one lifecycle state to another
- **Payload:** Contract ID, previous state, new state, trigger (manual or automatic)
- **Recipient:** Contract channel subscribers
- **Resolution:** Informational; no action required

### Health Score Changed >10%

- **Source:** Contract health scoring engine
- **Fires when:** A contract's health score changes by more than 10 percentage points in either direction
- **Payload:** Contract ID, previous score, new score, contributing factors
- **Recipient:** Contract channel subscribers
- **Resolution:** Informational; may prompt investigation if score dropped

### Batch Processing Completed

- **Source:** Batch processing pipeline
- **Fires when:** A batch job finishes across multiple contracts
- **Payload:** Batch ID, total contracts processed, success count, failure count, duration
- **Recipient:** Batch initiator + workspace admins
- **Resolution:** Informational; failures may generate separate Contract Failed 3x events

### AI Suggestion Available

- **Source:** AI agent framework
- **Fires when:** The AI generates a suggestion for a contract (patch proposal, classification, anomaly detection)
- **Payload:** Contract ID, suggestion type, confidence score, summary
- **Recipient:** Contract channel subscribers
- **Resolution:** User reviews the suggestion in the AI Agent tab and accepts, modifies, or dismisses

## Read/Unread Mechanics

| Concept | Implementation |
|---------|---------------|
| Per-user tracking | Each channel stores a `last_read` timestamp per user |
| Unread definition | Events with `timestamp > last_read` for that user |
| Mark as read | Opening a channel updates `last_read` to current time |
| Channel badge | Blue dot appears if any events are unread in that channel |
| Module badge | Red dot + count = sum of unread ACTION + URGENT events across all channels |
| INFO/AI events | Contribute to channel blue dot only, not to module count |

## Future Notification Channels (Design Now, Build Later)

### Per-User Notification Preferences

Users will be able to customize their notification experience:

- Override priority level per trigger type (e.g., downgrade "Health score changed" from INFO to muted)
- Set quiet hours (suppress non-URGENT notifications during specified time ranges)
- Choose notification grouping (individual vs. batched digest)

### Channel Muting

- Users can mute specific contract channels
- Muted channels do not contribute to the module badge
- Muted channels do not show the blue dot in the sidebar
- Events are still recorded; they become visible if the user unmutes

### Browser Push Notifications

- URGENT events only (SLA overdue, patch rejected, contract failed 3x)
- Requires Service Worker registration and user permission
- Clicking the push notification navigates directly to the relevant channel

### Email Notifications

| Email Trigger | Recipients | Timing |
|---------------|-----------|--------|
| SLA overdue | All workspace admins | Immediate |
| Approval chain completed (patch Applied) | Patch author + all reviewers | Immediate |
| Daily unresolved urgent digest | All workspace admins | Daily at configured time |

## Related Specs

- [Notifications Overview](./overview.md) -- Architecture, badge system, read/unread tracking
- [Patch Workflow / SLA Timer](/Features/PatchWorkflow/sla-timer.md) -- SLA overdue and approaching triggers
- [Patch Workflow / Approval Chain](/Features/PatchWorkflow/approval-chain.md) -- Patch rejection and review triggers
- [Triage Dashboard](/Features/Triage/) -- Module-level aggregation of notifications
