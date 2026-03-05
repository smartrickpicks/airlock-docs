# Notifications

> **Status:** SPECCED — Two-level architecture powered by Novu for delivery infrastructure.

> **Tech Stack:** [Novu](https://github.com/novuhq/novu) self-hosted (MIT). See [Tech Stack](../TechStack/overview.md).

## Summary

Airlock uses a two-level notification architecture to surface events at both the individual vault level and across the entire workspace. Notifications are in-app only for the POC phase; external channels (push, email) are designed now but built later.

**DECISION:** Each module's triage board (backed by the [Universal Task System](../TaskSystem/overview.md)) IS the module-level aggregator. Home triage widgets provide the cross-module overview. In-app only for POC.

## Two-Level Architecture

### Vault-Level Notifications

- **Where:** Signal panel (left zone of the triptych) within a vault
- **Scope:** Events for THIS vault only
- **Examples:** Patch submitted, extraction completed, SLA approaching, AI suggestion available
- **Visual:** Events appear as timestamped entries in the Signal feed

### Module-Level Notifications

- **Where:** Module triage board (backed by universal task table) + module bar badge
- **Scope:** Events across ALL vaults in the module
- **Examples:** SLA overdue, patch rejected, vault failed 3x, batch processing completed
- **Visual:** Tasks in the module's triage view; red badge on module icon

### Workspace-Level Notifications

- **Where:** Home workspace triage widgets
- **Scope:** Events across ALL modules
- **Examples:** Cross-module summary of urgent/action items
- **Visual:** Per-module triage widget tables on the Home page

## Badge System

### Module Bar Badge

- **Location:** On the Contracts module icon in the far-left Module Bar
- **Visual:** Red dot with numeric count
- **Count:** Sum of open triage items across all contracts (action-required and urgent events)
- **Updates:** Real-time via WebSocket; increments on new events, decrements when items are resolved or read

### Channel Sidebar Badge

- **Location:** Next to each contract channel name in the Channel Sidebar
- **Visual:** Blue dot (no count)
- **Condition:** Appears when the channel has unread events (events after the user's `last_read` timestamp for that channel)
- **Clears:** When the user opens the channel (which updates `last_read`)

### Badge Priority

| Badge Type | Color | Trigger |
|------------|-------|---------|
| Module bar | Red dot + count | Unread action/urgent events across all contracts |
| Channel sidebar | Blue dot | Any unread events in that specific channel |

## Read/Unread Tracking

### Per-User, Per-Channel Timestamps

- Each channel maintains a `last_read` timestamp for every user
- Events with a timestamp after `last_read` are considered **unread**
- Opening a channel automatically updates `last_read` to the current time
- The blue dot on a channel clears when `last_read` catches up to the latest event

### Module Badge Calculation

- Module badge count = sum of unread events with **ACTION** or **URGENT** priority across all channels
- INFO-level events contribute to the channel blue dot but NOT to the module badge count
- AI-level events contribute to the channel blue dot but NOT to the module badge count

## Event Flow

```
  Event occurs (e.g., SLA overdue)
       |
       v
  +-- Channel-level --+          +-- Module-level --+
  |  Signal panel      |          |  Triage Dashboard |
  |  (this contract)   |          |  "Recent Events"  |
  |  Blue dot on       |          |  Red dot + count   |
  |  channel sidebar   |          |  on Module bar     |
  +--------------------+          +--------------------+
```

Not all events propagate to both levels. See [notification-triggers.md](./notification-triggers.md) for the full mapping.

## Notification Priority Levels

| Priority | Color | Badge Contribution | Typical Events |
|----------|-------|--------------------|----------------|
| **URGENT** | Red (pulse animation) | Channel + Module | SLA overdue, patch rejected, contract failed 3x |
| **ACTION** | Amber | Channel + Module | Patch needs review, triage item assigned, SLA approaching |
| **INFO** | Blue | Channel only | Extraction completed, lifecycle state changed, health score changed |
| **AI** | Cyan (glow effect) | Channel only | AI suggestion available |

## Novu Integration Architecture

### POC (Phase 1): In-App Only

Novu self-hosted runs as a Docker service alongside Airlock API.

```
Vault Event (e.g., SLA overdue)
    |
    +--[BullMQ]--> Create task in universal task table
    |               → appears in module triage, Home widgets
    |
    +--[Novu SDK]--> Trigger in-app notification
                      → badge on module bar
                      → blue dot on vault in sub-panel
                      → Novu React notification center (if opened)
```

**Novu workflow templates:**

| Template | Trigger | Channel | Recipients |
|----------|---------|---------|------------|
| `sla-warning` | SLA < threshold | in-app | Assigned reviewer |
| `patch-submitted` | Patch state → Submitted | in-app | Assigned gatekeeper |
| `patch-approved` | Patch state → Applied | in-app | Patch author |
| `vault-failed` | Batch processing failure 3x | in-app | Workspace admins |
| `entity-needs-resolution` | Ambiguous entity match | in-app | Assigned builder |
| `health-dropped` | Health score drops >10% | in-app | Vault members |

### V2: Email Channel

Add email delivery to existing Novu templates:

| Template | Email Behavior |
|----------|---------------|
| `sla-warning` (URGENT) | Immediate email to assigned reviewer |
| `vault-failed` | Immediate email to workspace admins |
| `daily-digest` | NEW template: batched summary of unresolved urgent items, sent daily |

### V3: Push Channel

Browser push notifications for URGENT events via Novu's push channel + Service Worker.

## Future (Design Now, Build Later)

The following capabilities are designed into the data model and UI spec but are not implemented in the POC:

| Feature | Description | Novu Feature |
|---------|-------------|--------------|
| **Per-user notification preferences** | Users choose which event types they want to see | Novu subscriber preferences |
| **Vault muting** | Users can mute a specific vault to suppress its badge | Novu topic unsubscribe |
| **Push notifications (browser)** | Browser push for URGENT events only | Novu push channel + Service Worker |
| **Email notifications** | Email digest for SLA overdue and approval chain completion | Novu email channel + digest workflow |
| **Notification batching** | Group similar notifications (e.g., "5 contracts completed extraction") | Novu digest step in workflows |

## Related Specs

- [Notification Triggers](./notification-triggers.md) -- Full table of event triggers, priorities, and badge behavior
- [Universal Task System](../TaskSystem/overview.md) -- Tasks as the durable notification record
- [Tech Stack](../TechStack/overview.md) -- Novu library details
- [Shell / Module Bar](../Shell/module-bar.md) -- Where the module badge renders
- [Patch Workflow / SLA Timer](../PatchWorkflow/sla-timer.md) -- Source of SLA-related notifications
