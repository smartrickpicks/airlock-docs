# Calendar Module

> **Status:** SPECCED — Calendar is a date-centric view across all modules, powered by extracted dates and task due dates.

> **Key Insight:** The Calendar module doesn't store its own events. It renders dates from two sources: (1) dates extracted from vaults (contract effective dates, termination dates, renewal deadlines) and (2) task due dates from the universal task table. It's a temporal lens over existing data.

> **Tech Stack:** [Schedule-X](https://github.com/schedule-x/schedule-x) MIT core. See [Tech Stack](../TechStack/overview.md).

---

## Module Identity

| Property | Value |
|----------|-------|
| Module bar icon | Calendar (Lucide) |
| Module name | Calendar |
| Module color | Amber accent |
| Primary data | Dates extracted from vaults + task due dates |
| Chamber lifecycle | Not applicable — Calendar doesn't follow chambers |

---

## Data Sources

Calendar events are **computed views**, not stored records. They're assembled at render time from:

### Source 1: Vault Dates (from extraction)

| Extracted Field | Calendar Event Type | Color |
|----------------|--------------------|----|
| `effective_date` | Contract start | Green |
| `termination_date` | Contract end | Red |
| `renewal_date` | Renewal deadline | Amber |
| `expiry_date` | Expiration | Red |
| Custom date fields | Module-specific | Module color |

Query: vault extraction metadata where field `data_type = 'date'` and value is not null.

### Source 2: Task Due Dates

```sql
SELECT * FROM tasks
WHERE workspace_id = :ws
  AND due_at IS NOT NULL
  AND due_at BETWEEN :calendar_start AND :calendar_end
```

Each task renders as a calendar event color-coded by source module.

### Source 3: Manual Calendar Events (Future)

Users will be able to create manual calendar events (meetings, reminders). For POC, this is deferred — Schedule-X supports it, but the data model isn't needed until v2.

---

## Sub-Panel

```
+------------------------+
| CALENDAR               |
| > Mini Dashboard       |
|   3 Today              |
|   8 This Week          |
|   2 Overdue            |
|------------------------+
| > VIEWS                |
|   o Month              |   <-- Default landing
|   o Week               |
|   o Agenda             |   <-- List view, sorted by date
|                        |
| > FILTERS              |
|   o All Modules        |
|   o Contracts Only     |
|   o Tasks Only         |
|   o CRM Only           |
|------------------------+
| > UPCOMING             |
|   Mar 5: henderson-msa |   <-- Next 5 events
|     renewal deadline   |
|   Mar 8: sony-dist     |
|     termination        |
|   Mar 12: onboarding   |
|     task due           |
+------------------------+
```

Calendar sub-panel doesn't use chambers (Discover/Build/Review/Ship). Instead it has VIEWS (display modes) and FILTERS (data source filtering). The UPCOMING section is a quick-glance list of the next 5 events.

---

## Main Content: Calendar Views

### Month View (Default)

Standard month grid rendered by Schedule-X.

- Each day cell shows event dots (color-coded by module/type)
- Click a day → zooms to day view or agenda list for that day
- Click an event → navigates to source vault in its module
- Events show: vault name (truncated) + event type icon
- Today highlighted with accent border
- SLA overdue events pulsate with red indicator

### Week View

7-column grid with time slots (like Google Calendar week view).

- Better for seeing time-of-day patterns (if events have times)
- Task due dates render as all-day events at the top
- Vault dates render as all-day events unless they have a specific time

### Agenda View

Flat list sorted by date, grouped by day.

```
+----------------------------------------------------------+
| TODAY — March 3, 2026                                     |
|----------------------------------------------------------|
| [Red dot] henderson-msa — Renewal Deadline                |
|           Contracts > henderson-msa                       |
|           Due in 4h 30m                                   |
|----------------------------------------------------------|
| [Blue dot] Enrich Acme Inc account — Task Due             |
|           CRM > Acme Inc                                  |
|           Due in 6h                                       |
|----------------------------------------------------------|
|                                                           |
| TOMORROW — March 4, 2026                                  |
|----------------------------------------------------------|
| [Green dot] sony-dist-2024 — Effective Date               |
|           Contracts > sony-dist-2024                      |
|----------------------------------------------------------|
| [Amber dot] Review Q1 reports — Task Due                  |
|           Tasks                                           |
+----------------------------------------------------------+
```

Each row is clickable → navigates to source vault/task.

---

## Calendar Event Anatomy

| Field | Source | Display |
|-------|--------|---------|
| Title | Vault name + field label (e.g., "henderson-msa — Renewal") | Event title |
| Date | Extracted date value or task `due_at` | Position on calendar |
| Color | Source module color or severity color | Event dot/bar color |
| Module badge | Source module icon | Small icon on event |
| Vault link | `vault_id` | Click navigates to vault |
| Alert threshold | 90 days before termination, 30 days before renewal (configurable) | Early warning events |

### Auto-Generated Alert Events

The system creates **advance warning events** for critical dates:

| Date Type | Alert At | Severity |
|-----------|----------|----------|
| Termination date | -90 days, -30 days, -7 days | info, warning, urgent |
| Renewal deadline | -60 days, -14 days, -3 days | info, warning, urgent |
| SLA deadline | -4 hours, -1 hour, -15 min | info, warning, urgent |
| Task due date | -1 day, -2 hours | info, warning |

Alert events create tasks in the universal task table (type: `sla_warning`) and trigger Novu notifications.

---

## Cross-Module Integration

### Calendar ← Contracts

When extraction runs on a contract vault, any date fields are automatically available to the Calendar module:
- `effective_date` → green event on start date
- `termination_date` → red event on end date + advance warnings
- `renewal_date` → amber event with reminders

### Calendar ← Tasks

All tasks with `due_at` set appear on the calendar, regardless of source module.

### Calendar → Navigation

Clicking any calendar event navigates to the source:
- Vault date → switches to source module, opens vault triptych
- Task due date → switches to Tasks module, opens Focus Mode for that task

---

## Schedule-X Configuration

```typescript
// Calendar component setup
import { ScheduleXCalendar } from '@schedule-x/react'
import { createCalendar, viewMonth, viewWeek, viewDay } from '@schedule-x/calendar'

const calendar = createCalendar({
  views: [viewMonth, viewWeek, viewDay],
  defaultView: viewMonth.name,
  theme: 'airlock-dark',  // custom theme
  events: calendarEvents, // computed from vaults + tasks
  callbacks: {
    onEventClick: (event) => navigateToSource(event),
  },
})
```

Custom theme tokens map from Airlock palette:
- Background: `--airlock-bg`
- Surface: `--airlock-surface`
- Text: `--airlock-text`
- Today highlight: `--airlock-cyan`
- Event colors: per-module accent colors

---

---

## Improvement Brainstorm (2026-03-04)

> The Calendar module needs more depth beyond just rendering dates. These ideas expand it into a proper scheduling and planning tool.

### Source 4: Meeting Events (Google Meet Integration)

Once Google Meet SDK is integrated (see [CRM Comms Brainstorm](../CRM/crm-comms-brainstorm.md)), meetings become a calendar data source:

| Source | Calendar Event | Color |
|--------|---------------|-------|
| Scheduled Google Meet | Meeting block with participants | Cyan (AI/meeting) |
| Completed meeting | Past event with duration + summary link | Gray |
| Follow-up from meeting AI | Auto-scheduled follow-up | Amber |

### Source 5: Workflow Deadlines

Events from the workflow engine (lead qualification deadlines, onboarding milestones, approval SLAs):

| Source | Calendar Event | Color |
|--------|---------------|-------|
| SLA deadline | Approval due (countdown) | Red when <1hr |
| Onboarding milestone | Setup task due | Module color |
| Qualification deadline | Lead response SLA | Amber |

### Enhanced Day View

The current spec has Month, Week, and Agenda views. Add a **Day View** with time-block detail:

```
+----------------------------------------------------------+
| MONDAY, MARCH 4, 2026                                     |
|----------------------------------------------------------|
| 09:00  [Meeting] Henderson MSA Review — Google Meet       |
|        Participants: Ana, Dev, Jack (Nova)                 |
|        Duration: 30min                                     |
|----------------------------------------------------------|
| 10:00  (open)                                              |
|----------------------------------------------------------|
| 11:00  [SLA] Sony Distribution — Verifier review due      |
|        Assigned: Dev Patel — 2h 15m remaining              |
|----------------------------------------------------------|
| 14:00  [Task] Enrich Acme account — Due today             |
|        CRM > Acme Inc                                      |
|----------------------------------------------------------|
| ALL DAY                                                    |
|   [Contract] henderson-msa — Renewal deadline              |
|   [Contract] warner-amendment — Effective date             |
+----------------------------------------------------------+
```

### Smart Scheduling

Otto AI can suggest optimal meeting times based on:
- Participant availability (if Google Calendar is connected)
- SLA deadlines (don't schedule a meeting that conflicts with a deadline)
- Time zones (if workspace members are distributed)
- Meeting load (prevent back-to-back days)

### Calendar Sharing

Export calendar events as:
- iCal feed (`.ics`) — subscribe from any calendar app
- Google Calendar sync (two-way via Google Calendar API)
- Email digest — daily/weekly summary of upcoming events

### Capacity Planning View

A specialized calendar view for pipeline forecasting:

```
+----------------------------------------------------------+
| CAPACITY PLANNER — March 2026                              |
|----------------------------------------------------------|
|         Week 1     Week 2     Week 3     Week 4           |
| Batch    ████░░     ██████     ████░░     ██░░░░          |
| Review   ██░░░░     ████░░     ██████     ████░░          |
| Export   ░░░░░░     ██░░░░     ████░░     ██████          |
|                                                            |
| Projected load: 45 contracts/week                          |
| Current capacity: 60 contracts/week                        |
| Headroom: 25%                                              |
+----------------------------------------------------------+
```

Shows pipeline throughput projected over time — how many contracts are in each stage, when batches are expected to complete, where bottlenecks will emerge.

### Feature Control Plane

| Flag | Default | Description |
|------|---------|-------------|
| ENABLE_CALENDAR_MEETINGS | false | Show Google Meet events on calendar |
| ENABLE_CALENDAR_SHARING | false | iCal export + Google Calendar sync |
| ENABLE_CAPACITY_PLANNER | false | Pipeline capacity planning view |

---

## Related Specs

- **[Tech Stack](../TechStack/overview.md)** — Schedule-X library details
- **[Universal Task System](../TaskSystem/overview.md)** — Task due dates as calendar events
- **[Vault Hierarchy](../VaultHierarchy/overview.md)** — Vault dates as calendar events
- **[Notifications](../Notifications/overview.md)** — Advance warning alerts via Novu
- **[CRM Communications](../CRM/crm-comms-brainstorm.md)** — Google Meet integration
- **[Batch Processing](../BatchProcessing/overview.md)** — Pipeline capacity data
