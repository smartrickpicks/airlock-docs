# Event Creation & Meeting Scheduling -- Design Spec

> **Status:** SPECCED
> **Source:** Calendar overview brainstorm section (Sources 4-5, Smart Scheduling, Calendar Sharing), CRM Communications Google Meet context, user requirements for manual event creation (deferred from v1)

## Summary

The Calendar module currently operates as a computed view -- it renders dates extracted from vaults and task due dates but stores no events of its own. This spec promotes manual event creation from "deferred to v2" to a fully specified feature. It defines the data model, creation UX, meeting scheduling with Google Meet integration, two-way Google Calendar sync, and Otto-powered smart scheduling. All new capabilities sit behind Feature Control Plane flags and integrate cleanly with the existing Schedule-X rendering pipeline.

---

## Event Type Taxonomy

The Calendar module renders five distinct event types. The first three are computed (read-only projections from other modules). The last two are new stored records introduced by this spec.

### Computed Events (Existing, Read-Only)

#### 1. Vault Date Events
- **Source:** Extracted date fields from vaults (effective_date, termination_date, renewal_date, expiry_date, custom date fields)
- **Color:** Module color of the source vault (green for effective, red for termination/expiry, amber for renewal)
- **Interaction:** Click navigates to the source vault's triptych in its module
- **Editing:** Cannot be edited from the Calendar -- user must edit the vault field in its source module, and the calendar event updates automatically
- **Ownership:** System-generated; no creator attribution

#### 2. Task Due Date Events
- **Source:** Universal task table (`tasks.due_at`)
- **Color:** Module color of the source task's parent module
- **Interaction:** Click navigates to the Tasks module Focus Mode for that task
- **Editing:** Changing the event time updates the task's `due_at` (with confirmation)
- **Ownership:** Attributed to task assignee

#### 3. Alert Events
- **Source:** Auto-generated advance warnings for critical dates (termination -90/-30/-7 days, renewal -60/-14/-3 days, SLA deadlines)
- **Color:** Severity-based (info = module color, warning = amber, urgent = red with pulsating indicator)
- **Interaction:** Dismissable but logged to the audit trail; click navigates to source vault
- **Ownership:** System-generated

### New Event Types (This Spec)

#### 4. Manual Events
- **Source:** User-created via event creation modal or inline quick-create
- **Color:** User-selected from preset palette, or contextual module color if linked to a vault
- **Interaction:** Full CRUD by creator and participants; click opens detail popover
- **Editing:** Creator and workspace admins can edit/delete; participants can view
- **Ownership:** Attributed to creating workspace member
- **Storage:** `calendar_events` table (new)

#### 5. Meeting Events
- **Source:** User-created via meeting scheduling flow; optionally imported from Google Calendar
- **Color:** Cyan (consistent with AI/meeting color defined in Calendar brainstorm)
- **Interaction:** Click opens detail popover with Google Meet link, participant list, and prep brief
- **Pre-meeting state:** Shows participant RSVP status, agenda, and Meet join button
- **Post-meeting state:** Shows duration, summary link (if Meeting Intelligence is enabled), and action items
- **Editing:** Organizer and workspace admins can edit/delete; participants can RSVP
- **Storage:** `calendar_events` table with `event_type = 'meeting'` and populated `meeting_id`, `google_meet_link`

---

## Data Model

### calendar_events Table (NEW)

All manual and meeting events are stored in this table. Computed events (vault dates, task dues, alerts) are NOT stored here -- they continue to be assembled at render time from their source tables.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| `id` | `uuid` | NOT NULL | Primary key, default `gen_random_uuid()` |
| `workspace_id` | `uuid` | NOT NULL | FK to `workspaces`. RLS enforcement column for tenant isolation |
| `title` | `text` | NOT NULL | Event title (max 500 chars) |
| `description` | `text` | NULL | Event description or agenda. Supports markdown. Encrypted at rest |
| `event_type` | `enum('manual','meeting')` | NOT NULL | Discriminator for event behavior |
| `all_day` | `boolean` | NOT NULL | Default `false`. When true, `start_at` and `end_at` represent date-only boundaries |
| `start_at` | `timestamptz` | NOT NULL | Event start time. Stored in UTC, displayed in user's timezone |
| `end_at` | `timestamptz` | NOT NULL | Event end time. Default: `start_at + interval '1 hour'` |
| `recurrence_rule` | `text` | NULL | iCal RRULE format string (e.g., `FREQ=WEEKLY;BYDAY=MO,WE,FR`). Null for non-recurring events |
| `recurrence_end` | `timestamptz` | NULL | Recurrence series end date. Null means recur indefinitely |
| `recurrence_parent_id` | `uuid` | NULL | FK to `calendar_events.id`. When editing a single occurrence, the exception references the parent recurring event |
| `location` | `text` | NULL | Physical location string or `"Google Meet"` |
| `google_event_id` | `text` | NULL | Google Calendar event ID for synced events. Unique within workspace. Indexed for sync lookups |
| `google_meet_link` | `text` | NULL | Auto-generated Google Meet URL. Populated when meeting type + Google Meet toggle enabled |
| `meeting_id` | `uuid` | NULL | FK to `meetings` table (Meeting Intelligence module). Links to full meeting lifecycle record |
| `vault_id` | `uuid` | NULL | FK to `vaults`. Optional contextual link -- "this event is about this vault" |
| `module_id` | `text` | NULL | Module context for color coding (e.g., `'contracts'`, `'crm'`). Defaults to `'calendar'` |
| `color` | `text` | NULL | Hex color override (e.g., `#F59E0B`). When null, falls back to `module_id` color mapping |
| `created_by` | `uuid` | NOT NULL | FK to `workspace_members`. Event creator |
| `reminder` | `text` | NOT NULL | Default `'15min'`. One of: `'none'`, `'5min'`, `'15min'`, `'30min'`, `'1hr'`, `'1day'` |
| `status` | `enum('confirmed','tentative','cancelled')` | NOT NULL | Default `'confirmed'`. Cancelled events are soft-deleted (hidden from calendar but retained for audit) |
| `created_at` | `timestamptz` | NOT NULL | Default `now()` |
| `updated_at` | `timestamptz` | NOT NULL | Default `now()`, auto-updated on modification |

**Indexes:**
- `idx_calendar_events_workspace_range` on `(workspace_id, start_at, end_at)` -- primary query path for calendar rendering
- `idx_calendar_events_google_id` on `(workspace_id, google_event_id)` WHERE `google_event_id IS NOT NULL` -- sync lookups
- `idx_calendar_events_vault` on `(vault_id)` WHERE `vault_id IS NOT NULL` -- vault-linked event lookups
- `idx_calendar_events_created_by` on `(created_by)` -- "my events" filter

**RLS Policy:**
```sql
CREATE POLICY calendar_events_workspace_isolation ON calendar_events
  USING (workspace_id = current_setting('app.workspace_id')::uuid);
```

### calendar_event_participants Table (NEW)

Tracks participants for both manual and meeting events. Supports internal workspace members and external contacts (from CRM or ad-hoc email).

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| `id` | `uuid` | NOT NULL | Primary key, default `gen_random_uuid()` |
| `event_id` | `uuid` | NOT NULL | FK to `calendar_events.id`. ON DELETE CASCADE |
| `member_id` | `uuid` | NULL | FK to `workspace_members`. Set for internal participants |
| `contact_id` | `uuid` | NULL | FK to `vault_contacts` (CRM contacts). Set for external participants linked to CRM |
| `email` | `text` | NOT NULL | Participant email address. Canonical identifier for Google Calendar sync |
| `display_name` | `text` | NOT NULL | Rendered name in participant chips |
| `rsvp_status` | `enum('accepted','declined','tentative','pending')` | NOT NULL | Default `'pending'`. Updated via RSVP endpoint or Google Calendar sync |
| `is_organizer` | `boolean` | NOT NULL | Default `false`. Exactly one participant per event must be organizer |
| `notified_at` | `timestamptz` | NULL | When the invitation notification was sent (via Novu) |

**Constraints:**
- `CHECK (member_id IS NOT NULL OR email IS NOT NULL)` -- must have at least one identifier
- `UNIQUE (event_id, email)` -- no duplicate participants per event

**RLS Policy:** Inherits workspace isolation through the `calendar_events` join.

### calendar_sync_state Table (NEW)

Tracks per-user Google Calendar sync state for incremental sync.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| `id` | `uuid` | NOT NULL | Primary key |
| `workspace_id` | `uuid` | NOT NULL | FK to `workspaces` |
| `member_id` | `uuid` | NOT NULL | FK to `workspace_members` |
| `google_calendar_id` | `text` | NOT NULL | Google Calendar ID (usually `'primary'`) |
| `sync_token` | `text` | NULL | Google Calendar incremental sync token |
| `last_synced_at` | `timestamptz` | NULL | Last successful sync timestamp |
| `webhook_channel_id` | `text` | NULL | Google Pub/Sub channel ID for push notifications |
| `webhook_expiration` | `timestamptz` | NULL | When the webhook channel expires (must renew) |
| `sync_status` | `enum('active','paused','error')` | NOT NULL | Default `'active'` |
| `error_message` | `text` | NULL | Last sync error, if any |

**Constraints:**
- `UNIQUE (workspace_id, member_id, google_calendar_id)` -- one sync state per user per calendar

---

## Event Creation UX

### Entry Points

Users can create events from five locations in the Airlock shell:

| Entry Point | Action | Opens |
|-------------|--------|-------|
| **Toolbar Create button** | Click "New Event" or "Schedule Meeting" from the Create dropdown | Full event creation modal |
| **Cmd+K palette** | Type `/new-event` or `/schedule-meeting` | Full event creation modal (pre-set to event or meeting type) |
| **Calendar click (Month view)** | Click a date cell | Inline quick-create popover, date pre-filled |
| **Calendar click (Week/Day view)** | Click an empty time slot | Inline quick-create popover, date + time pre-filled |
| **Calendar drag (Week/Day view)** | Click and drag across time slots | Inline quick-create popover, date + start/end pre-filled from drag range |

### Event Creation Modal

Centered modal, 480px width, dark surface background (`--airlock-surface`). Two modes toggled by a segmented control at the top: **Event** | **Meeting**.

#### Event Mode (Default)

```
+----------------------------------------------+
|  New Event                            [x]    |
|----------------------------------------------|
|  [Event] [Meeting]     <- segmented control  |
|                                              |
|  Title *                                     |
|  [________________________________]          |
|                                              |
|  [ ] All day                                 |
|                                              |
|  Start *            End *                    |
|  [Mar 5, 2026]      [Mar 5, 2026]           |
|  [10:00 AM   ]      [11:00 AM   ]           |
|                                              |
|  Location                                    |
|  [________________________________]          |
|                                              |
|  Description                                 |
|  [________________________________]          |
|  [________________________________]          |
|  [________________________________]          |
|                                              |
|  Color          Reminder                     |
|  [o o o o o]    [15 min before    v]         |
|                                              |
|  Recurrence                                  |
|  [Does not repeat          v]                |
|                                              |
|  Link to vault (optional)                    |
|  [Search vaults...            ]              |
|                                              |
|  [Cancel]              [Create Event]        |
+----------------------------------------------+
```

**Field specifications:**

| Field | Type | Required | Behavior |
|-------|------|----------|----------|
| Title | Text input | Yes | Autofocus on modal open. Max 500 chars |
| All day | Toggle | No | When enabled, hides time pickers; sets `start_at` to 00:00 and `end_at` to 23:59 of selected date(s) |
| Start date | Date picker | Yes | Default: today (from toolbar) or clicked date (from calendar) |
| Start time | Time picker | Yes (unless all-day) | Default: next round hour, or clicked time slot |
| End date | Date picker | Yes | Default: same as start date |
| End time | Time picker | Yes (unless all-day) | Default: start time + 1 hour. Auto-adjusts if start time changes |
| Location | Text input | No | Free text. No auto-complete in v1 |
| Description | Textarea | No | Supports markdown. 3 visible rows, expandable. Encrypted at rest |
| Color | Preset picker | No | 8 preset colors: module colors (blue, green, amber, red, purple, cyan, teal) + gray. Default: amber (Calendar module color) |
| Reminder | Dropdown | No | Options: None, 5 min, 15 min, 30 min, 1 hour, 1 day. Default: 15 min |
| Recurrence | Dropdown | No | Options: Does not repeat, Daily, Weekly on [day], Monthly on [date], Every weekday, Custom. Default: Does not repeat. Feature flag: `ENABLE_EVENT_RECURRENCE` |
| Vault link | Searchable dropdown | No | Searches vaults across all modules. Shows vault name + module icon. Links event contextually to a vault |

**Custom Recurrence Sub-Modal (when "Custom" selected):**

```
+----------------------------------------------+
|  Custom Recurrence                           |
|----------------------------------------------|
|  Repeat every [1] [week(s)   v]             |
|                                              |
|  Repeat on                                   |
|  [S] [M] [T] [W] [T] [F] [S]              |
|   o   x   o   x   o   x   o                |
|                                              |
|  Ends                                        |
|  (o) Never                                   |
|  ( ) On date  [___________]                  |
|  ( ) After [__] occurrences                  |
|                                              |
|  [Cancel]              [Done]                |
+----------------------------------------------+
```

Generates a standard iCal RRULE string stored in `recurrence_rule`.

#### Meeting Mode

When user switches to "Meeting" tab or enters via "Schedule Meeting", the form extends with meeting-specific fields:

```
+----------------------------------------------+
|  Schedule Meeting                     [x]    |
|----------------------------------------------|
|  [Event] [Meeting]     <- Meeting selected   |
|                                              |
|  Title *                                     |
|  [________________________________]          |
|                                              |
|  Start *            End *                    |
|  [Mar 5, 2026]      [Mar 5, 2026]           |
|  [10:00 AM   ]      [10:30 AM   ]           |
|                                              |
|  [x] Add Google Meet                         |
|      Meet link will be generated on save     |
|                                              |
|  Participants                                |
|  [Search members or contacts...      ]       |
|  [Ana Builder x] [Dev Patel x] [+ Add]      |
|                                              |
|  [Find best time]    <- Otto smart schedule  |
|                                              |
|  Agenda                                      |
|  [________________________________]          |
|  [________________________________]          |
|  [________________________________]          |
|  [________________________________]          |
|                                              |
|  [ ] Generate prep brief                     |
|      Otto will compile meeting context       |
|                                              |
|  Link to vault (optional)                    |
|  [Search vaults...            ]              |
|                                              |
|  Reminder          Color                     |
|  [15 min before v] [o o o o o]              |
|                                              |
|  [Cancel]           [Schedule Meeting]       |
+----------------------------------------------+
```

**Meeting-specific fields:**

| Field | Type | Required | Behavior |
|-------|------|----------|----------|
| Google Meet toggle | Checkbox | No | Default: on. When enabled, generates Meet link via Google Calendar API `conferenceDataVersion=1` on save. Feature flag: `ENABLE_MEETING_SCHEDULING` |
| Participants | Search + chip input | No | Searches workspace members (by name/email) and CRM contacts (by name/company/email). Each selected participant appears as a removable chip. "Add external" option at bottom of dropdown allows entering an email address directly |
| Find best time | Button | No | Opens smart scheduling panel (see Smart Scheduling section). Only visible when `ENABLE_SMART_SCHEDULING` is enabled and at least one participant is added |
| Agenda | Rich textarea | No | Replaces the Description field. Supports markdown with bullet lists, headers. 4 visible rows. Encrypted at rest |
| Generate prep brief | Checkbox | No | When enabled, Otto compiles a pre-meeting context packet from participant interaction history, linked vault details, and recent activity. Visible only when a vault is linked or participants have CRM contact records |

### Inline Quick-Create (Calendar Grid)

For rapid event creation directly from the calendar, without opening the full modal.

**Trigger:** Click an empty time slot (Week/Day view) or date cell (Month view).

```
+-----------------------------------+
|  [Title...                    ]   |
|  Mar 5, 10:00 AM - 11:00 AM      |
|                                   |
|  [More options]  [Save] [Cancel]  |
+-----------------------------------+
```

**Behavior:**
- Popover appears anchored to the clicked cell/slot
- Title input is autofocused
- Date and time are pre-filled from the clicked position
- In Week/Day view, user can click-and-drag to define duration before the popover appears
- Press Enter to save with defaults (manual event, no participants, 15min reminder)
- "More options" opens the full event creation modal, pre-filled with entered title and times
- Press Escape or click outside to cancel
- Created event appears immediately on the calendar (optimistic UI, reverts on API error)

### Drag-to-Create (Week/Day View)

**Trigger:** Click and drag vertically across time slots.

**Behavior:**
1. On mousedown, a translucent event block appears at the cursor position
2. As user drags down, the block extends to cover the drag range (snaps to 15-minute increments)
3. On mouseup, the inline quick-create popover appears with start and end times pre-filled from the drag range
4. Minimum duration: 15 minutes
5. Maximum drag: within a single day (no cross-day drag)

---

## Event Detail Popover

When a user clicks an existing manual or meeting event on the calendar, a detail popover appears (anchored to the event element, max-width 360px).

### Manual Event Popover

```
+-----------------------------------+
|  Team Planning Session            |
|  -------------------------------- |
|  Mar 5, 10:00 AM - 11:00 AM      |
|  Location: Conference Room B      |
|                                   |
|  Review Q1 deliverables and       |
|  assign Q2 priorities.            |
|                                   |
|  Vault: henderson-msa             |
|  -------------------------------- |
|  [Edit]  [Delete]  [Open vault]   |
+-----------------------------------+
```

### Meeting Event Popover

```
+-----------------------------------+
|  Henderson MSA Review             |
|  -------------------------------- |
|  Mar 5, 10:00 AM - 10:30 AM      |
|  Google Meet                      |
|                                   |
|  Participants:                    |
|  Ana Builder (accepted)           |
|  Dev Patel (pending)              |
|  Jack Nova (declined)             |
|                                   |
|  Agenda:                          |
|  - Review extraction results      |
|  - Discuss renewal strategy       |
|                                   |
|  Vault: henderson-msa             |
|  -------------------------------- |
|  [Join Meet] [Prep Brief]         |
|  [Edit]  [Delete]  [Open vault]   |
+-----------------------------------+
```

**Post-meeting state** (after meeting end time has passed):

```
+-----------------------------------+
|  Henderson MSA Review             |
|  -------------------------------- |
|  Mar 5, 10:00 AM - 10:32 AM      |
|  Completed (32 min)               |
|                                   |
|  Participants: 3 attended         |
|                                   |
|  Action Items: 4                  |
|  -------------------------------- |
|  [View Summary] [View Actions]    |
|  [Open vault]                     |
+-----------------------------------+
```

### Computed Event Popover (Vault Dates / Task Dues)

Computed events are read-only from the Calendar's perspective:

```
+-----------------------------------+
|  henderson-msa -- Renewal         |
|  -------------------------------- |
|  Mar 15, 2026 (all day)           |
|  Source: Contracts > henderson-msa|
|                                   |
|  Extracted from contract on       |
|  Feb 28, 2026                     |
|  -------------------------------- |
|  [Edit in source]                 |
+-----------------------------------+
```

"Edit in source" navigates to the vault's triptych in its source module, where the user can modify the extracted date field.

---

## Event Editing

### Edit Flow

1. User clicks event on calendar --> detail popover appears
2. User clicks [Edit] --> full event creation modal opens, pre-filled with all current event data
3. User modifies fields and clicks [Save Changes]
4. API call: `PUT /api/calendar/events/{id}`
5. Calendar re-renders with updated event (optimistic UI)
6. If Google Calendar sync is enabled, change is pushed to Google Calendar

### Recurring Event Edit

When editing a recurring event, a dialog appears:

```
+-----------------------------------+
|  Edit Recurring Event             |
|  -------------------------------- |
|  ( ) This event only              |
|  ( ) This and following events    |
|  ( ) All events in the series     |
|  -------------------------------- |
|  [Cancel]              [Continue] |
+-----------------------------------+
```

- **This event only:** Creates an exception record (`recurrence_parent_id` set to the series parent) with modified fields
- **This and following events:** Splits the series: truncates original at edit point, creates new series from edit point forward
- **All events in the series:** Modifies the parent record; all computed occurrences reflect the change

### Delete Flow

1. User clicks [Delete] in the detail popover
2. Confirmation dialog: "Delete this event? This cannot be undone."
3. For recurring events, same three-option dialog as edit
4. API call: `DELETE /api/calendar/events/{id}` (sets `status = 'cancelled'`, soft delete)
5. Event disappears from calendar immediately
6. If Google Calendar sync is enabled, deletion is pushed to Google Calendar

### Permission Model

| Action | Creator | Participant | Workspace Admin |
|--------|---------|-------------|-----------------|
| View event | Yes | Yes | Yes |
| Edit event | Yes | No | Yes |
| Delete event | Yes | No | Yes |
| RSVP | Yes | Yes | No (unless participant) |
| View participant list | Yes | Yes | Yes |

---

## Google Calendar Sync

### Architecture Overview

Two-way sync between Airlock calendar events and a user's Google Calendar. Each workspace member independently connects their Google Calendar -- sync is per-user, not workspace-wide.

**Feature flag:** `ENABLE_GOOGLE_CALENDAR_SYNC` (default: `false`)

### OAuth Flow

1. User navigates to Calendar Settings (accessible from Calendar sub-panel gear icon)
2. Clicks "Connect Google Calendar"
3. OAuth consent screen requests scopes:
   - `https://www.googleapis.com/auth/calendar` (read/write events)
   - `https://www.googleapis.com/auth/calendar.events` (event CRUD)
4. On successful auth, Airlock stores the OAuth refresh token (encrypted at rest, per security principles)
5. Sync state record created in `calendar_sync_state`

### Sync Direction and Scope

| Direction | What Syncs | Behavior |
|-----------|-----------|----------|
| Airlock --> Google Calendar | Manual events created by this user | Full CRUD: create, update, delete propagated |
| Airlock --> Google Calendar | Meeting events organized by this user | Full CRUD + conferenceData for Google Meet links |
| Google Calendar --> Airlock | External events from user's Google Calendar | Imported as read-only events with `event_type = 'external'` display type. Not stored in `calendar_events` table -- held in a cache layer and merged at render time |
| NOT synced | Vault date events | Internal only -- no business data leaves the workspace |
| NOT synced | Task due date events | Internal only -- use task management tools for external task sync |
| NOT synced | Alert events | Internal only -- generated from vault date thresholds |

### Sync Mechanism

#### Initial Sync
1. On first connection, full sync via Google Calendar API `events.list` with `syncToken` request
2. All existing Airlock manual/meeting events for this user are pushed to Google Calendar
3. All Google Calendar events within the configured window (past 30 days to future 365 days) are imported
4. `sync_token` stored in `calendar_sync_state` for incremental updates

#### Ongoing Sync (Push)
1. Register a Google Calendar webhook via `events.watch`
2. Google sends push notification to `POST /api/calendar/webhooks/google` when any event changes
3. Airlock fetches changed events using the stored `sync_token`
4. Merge changes into local state
5. Webhook channels expire (typically 7 days); a background job renews them before expiration

#### Ongoing Sync (Fallback Polling)
- If webhook registration fails or webhook delivery stops, fall back to polling every 5 minutes
- Polling uses incremental sync via `syncToken` to minimize API usage
- Background job managed by BullMQ (`calendar-sync-poll` queue)

#### Conflict Resolution
- **Last-write-wins** with user notification
- If the same event is modified in both Airlock and Google Calendar between sync cycles:
  1. The more recent `updated` timestamp wins
  2. A notification is sent to the event creator: "Event '{title}' was updated in {winning source}. Your changes in {losing source} were overwritten."
  3. Conflict events are logged in the audit trail for transparency

### Sync Status UI

In the Calendar sub-panel footer:

```
+------------------------+
| Google Calendar: Synced |
| Last sync: 2 min ago   |
+------------------------+
```

States:
- **Synced** (green dot): Last sync successful, webhook active
- **Syncing** (amber spinner): Sync in progress
- **Error** (red dot): Last sync failed, shows error message on hover
- **Paused** (gray dot): User paused sync
- **Not connected** (no indicator): Feature flag off or user hasn't connected

---

## Smart Scheduling (Otto-Powered)

### Overview

Otto can suggest optimal meeting times by analyzing participant availability, vault deadlines, user preferences, and meeting load. This feature requires Google Calendar access for availability data.

**Feature flag:** `ENABLE_SMART_SCHEDULING` (default: `false`)
**Dependency:** Requires `ENABLE_GOOGLE_CALENDAR_SYNC` to be enabled, and at least the organizing user must have connected Google Calendar.

### User Flow

1. User opens Meeting creation form and adds participants
2. "Find best time" button appears below the participants section (only when at least 1 participant is added and smart scheduling is enabled)
3. User clicks "Find best time"
4. A scheduling panel slides in below the participants, replacing the time pickers temporarily:

```
+----------------------------------------------+
|  Suggested Times                       [x]   |
|----------------------------------------------|
|  Otto analyzed availability for 3             |
|  participants across 5 business days.         |
|                                              |
|  1. Tue Mar 10, 2:00 PM - 2:30 PM           |
|     All 3 available | No conflicts    [Pick] |
|                                              |
|  2. Wed Mar 11, 10:00 AM - 10:30 AM         |
|     All 3 available | 1 back-to-back  [Pick] |
|                                              |
|  3. Thu Mar 12, 3:00 PM - 3:30 PM           |
|     2 of 3 available | Dev tentative  [Pick] |
|                                              |
|  4. Fri Mar 13, 11:00 AM - 11:30 AM         |
|     2 of 3 available | Ana busy       [Pick] |
|                                              |
|  [ Show more options ]                        |
+----------------------------------------------+
```

5. User clicks [Pick] on a suggestion
6. Start/end time fields in the meeting form are populated with the selected slot
7. Scheduling panel closes, user can continue filling out the meeting form

### Ranking Algorithm

Otto ranks suggested time slots using a weighted scoring system:

| Factor | Weight | Description |
|--------|--------|-------------|
| Participant availability | 40% | More available participants = higher score. All available = maximum |
| No back-to-back meetings | 20% | Penalize slots where participants have meetings immediately before or after |
| Preferred hours | 15% | Respect user-configured preferred meeting times (default: 9AM-12PM, 2PM-5PM) |
| Vault deadline avoidance | 15% | Don't schedule during active SLA countdowns or critical vault deadlines |
| Time zone fairness | 10% | For distributed teams, favor slots during reasonable hours for all time zones |

### Availability Data Sources

| Source | How Accessed | Accuracy |
|--------|-------------|----------|
| Google Calendar (connected users) | Free/busy API: `freebusy.query` | High -- shows busy blocks without event details (privacy-preserving) |
| Google Calendar (unconnected users) | Not available | Unknown -- shown as gray dot in availability display |
| Airlock calendar events | Direct database query | High -- manual + meeting events for that user |
| Vault deadlines | Vault extraction dates within 48 hours | Medium -- inferred from date proximity |
| User preferences | `workspace_member_settings.preferred_meeting_hours` | High -- user-configured |

### Availability Indicators

When participants are added to a meeting form and a time is selected, availability dots appear next to each participant chip:

| Indicator | Meaning |
|-----------|---------|
| Green dot | Available at the selected time |
| Red dot | Busy -- conflicting event at the selected time |
| Yellow dot | Tentative -- has a tentative event at the selected time |
| Gray dot | Unknown -- no calendar access (Google Calendar not connected) |

---

## API Endpoints

All endpoints are scoped to the authenticated user's workspace via RLS. Authentication is via workspace session token in the `Authorization` header.

### Event CRUD

| Method | Path | Description | Request Body | Response |
|--------|------|-------------|-------------|----------|
| `POST` | `/api/calendar/events` | Create a new event | `CreateEventRequest` | `201` + `CalendarEvent` |
| `GET` | `/api/calendar/events` | List events in date range | Query: `start`, `end`, `type[]`, `module_id` | `200` + `CalendarEvent[]` |
| `GET` | `/api/calendar/events/{id}` | Get single event detail | -- | `200` + `CalendarEvent` (with participants) |
| `PUT` | `/api/calendar/events/{id}` | Update event | `UpdateEventRequest` | `200` + `CalendarEvent` |
| `DELETE` | `/api/calendar/events/{id}` | Soft-delete event (set `status = 'cancelled'`) | -- | `204` |

### Participants and RSVP

| Method | Path | Description | Request Body | Response |
|--------|------|-------------|-------------|----------|
| `POST` | `/api/calendar/events/{id}/participants` | Add participant | `{ email, display_name, member_id?, contact_id? }` | `201` + `Participant` |
| `DELETE` | `/api/calendar/events/{id}/participants/{pid}` | Remove participant | -- | `204` |
| `POST` | `/api/calendar/events/{id}/rsvp` | RSVP to event (authenticated user) | `{ status: 'accepted'\|'declined'\|'tentative' }` | `200` |

### Scheduling Intelligence

| Method | Path | Description | Request Body | Response |
|--------|------|-------------|-------------|----------|
| `GET` | `/api/calendar/availability` | Check free/busy for participants | Query: `emails[]`, `start`, `end` | `200` + `AvailabilityResponse` |
| `POST` | `/api/calendar/suggest-times` | Otto smart scheduling | `{ participant_emails[], duration_minutes, date_range_start, date_range_end, preferences? }` | `200` + `SuggestedSlot[]` |

### Google Calendar Sync

| Method | Path | Description | Request Body | Response |
|--------|------|-------------|-------------|----------|
| `POST` | `/api/calendar/sync/google/connect` | Initiate Google OAuth flow | -- | `302` redirect to Google consent |
| `GET` | `/api/calendar/sync/google/callback` | OAuth callback handler | Query: `code`, `state` | `302` redirect to Calendar settings |
| `POST` | `/api/calendar/sync/google/trigger` | Force a manual sync | -- | `202` (sync queued) |
| `GET` | `/api/calendar/sync/status` | Sync health check | -- | `200` + `SyncStatus` |
| `POST` | `/api/calendar/sync/google/disconnect` | Revoke access and stop sync | -- | `200` |
| `POST` | `/api/calendar/webhooks/google` | Google push notification receiver | Google webhook payload | `200` |

### Request/Response Schemas

```typescript
interface CreateEventRequest {
  title: string;                      // required, max 500 chars
  description?: string;               // markdown, encrypted at rest
  event_type: 'manual' | 'meeting';   // required
  all_day: boolean;                   // default false
  start_at: string;                   // ISO 8601 timestamptz
  end_at: string;                     // ISO 8601 timestamptz
  recurrence_rule?: string;           // iCal RRULE
  recurrence_end?: string;            // ISO 8601 timestamptz
  location?: string;
  enable_google_meet?: boolean;       // for meetings, generates Meet link
  vault_id?: string;                  // uuid, optional vault link
  module_id?: string;                 // for color coding
  color?: string;                     // hex color override
  reminder?: 'none' | '5min' | '15min' | '30min' | '1hr' | '1day';
  participants?: Array<{
    email: string;
    display_name: string;
    member_id?: string;
    contact_id?: string;
  }>;
  generate_prep_brief?: boolean;      // Otto pre-meeting context
}

interface CalendarEvent {
  id: string;
  workspace_id: string;
  title: string;
  description?: string;
  event_type: 'manual' | 'meeting';
  all_day: boolean;
  start_at: string;
  end_at: string;
  recurrence_rule?: string;
  recurrence_end?: string;
  location?: string;
  google_event_id?: string;
  google_meet_link?: string;
  meeting_id?: string;
  vault_id?: string;
  vault_name?: string;                // denormalized for display
  module_id?: string;
  color?: string;
  created_by: string;
  creator_name: string;               // denormalized for display
  reminder: string;
  status: 'confirmed' | 'tentative' | 'cancelled';
  participants?: Participant[];
  created_at: string;
  updated_at: string;
}

interface Participant {
  id: string;
  event_id: string;
  member_id?: string;
  contact_id?: string;
  email: string;
  display_name: string;
  rsvp_status: 'accepted' | 'declined' | 'tentative' | 'pending';
  is_organizer: boolean;
}

interface AvailabilityResponse {
  participants: Array<{
    email: string;
    display_name: string;
    busy_slots: Array<{ start: string; end: string }>;
    status: 'known' | 'unknown';       // unknown if no calendar access
  }>;
}

interface SuggestedSlot {
  start_at: string;
  end_at: string;
  score: number;                        // 0-100 ranking score
  available_count: number;              // participants available
  total_count: number;                  // total participants
  conflicts: Array<{
    email: string;
    display_name: string;
    reason: 'busy' | 'tentative';
  }>;
  notes: string[];                      // e.g., "Avoids back-to-back for Ana"
}

interface SyncStatus {
  connected: boolean;
  google_calendar_id?: string;
  last_synced_at?: string;
  sync_status: 'active' | 'paused' | 'error' | 'not_connected';
  error_message?: string;
  webhook_active: boolean;
  webhook_expiration?: string;
}
```

---

## Schedule-X Integration

### New Event Source Adapter

The existing Calendar module renders computed events via Schedule-X. This spec adds a new event source adapter that fetches stored events from `calendar_events` alongside computed events.

```typescript
// Existing: computed event sources
const vaultDateEvents = useVaultDateEvents(dateRange);
const taskDueEvents = useTaskDueEvents(dateRange);
const alertEvents = useAlertEvents(dateRange);

// New: stored event source
const manualEvents = useCalendarEvents(dateRange, {
  types: ['manual', 'meeting'],
  enabled: featureFlags.ENABLE_MANUAL_EVENTS,
});

// New: external Google Calendar events (read-only cache)
const externalEvents = useExternalCalendarEvents(dateRange, {
  enabled: featureFlags.ENABLE_GOOGLE_CALENDAR_SYNC && syncStatus.connected,
});

// Merge all sources for Schedule-X
const allEvents = useMemo(() => [
  ...vaultDateEvents,
  ...taskDueEvents,
  ...alertEvents,
  ...manualEvents,
  ...externalEvents,
], [vaultDateEvents, taskDueEvents, alertEvents, manualEvents, externalEvents]);
```

### Event Rendering

Schedule-X renders events with these customizations:

| Event Type | Visual Treatment |
|------------|-----------------|
| Vault date | Module color bar, vault icon, italic title |
| Task due | Module color bar, task icon, checkbox indicator |
| Alert | Red/amber bar, warning icon, pulsating border for urgent |
| Manual | User-selected color bar, calendar icon |
| Meeting | Cyan bar, video icon, participant count badge |
| External (Google) | Dotted border, Google Calendar icon, muted color |

### Click Handler Routing

```typescript
const handleEventClick = (event: ScheduleXEvent) => {
  switch (event.meta.source) {
    case 'vault_date':
      navigateToVault(event.meta.vault_id, event.meta.module_id);
      break;
    case 'task_due':
      navigateToTask(event.meta.task_id);
      break;
    case 'alert':
      navigateToVault(event.meta.vault_id, event.meta.module_id);
      break;
    case 'manual':
    case 'meeting':
      openEventPopover(event.meta.event_id);
      break;
    case 'external':
      openExternalEventPopover(event); // read-only, shows title + time only
      break;
  }
};
```

### Real-Time Updates

When another workspace member creates or modifies a calendar event, the change is broadcast via WebSocket:

- **Channel:** `workspace:{workspace_id}:calendar`
- **Events:** `event.created`, `event.updated`, `event.deleted`, `event.rsvp`
- **Payload:** Full `CalendarEvent` object (for create/update) or `{ id }` (for delete)
- **Client behavior:** Optimistically update the Schedule-X event list; no full re-fetch required

---

## Notification Integration (Novu)

### Event Reminders

When a user creates an event with a reminder set, a delayed notification job is scheduled:

| Reminder Setting | Novu Trigger | Channel |
|-----------------|--------------|---------|
| 5 min before | `calendar.reminder.5min` | In-app + browser push |
| 15 min before | `calendar.reminder.15min` | In-app + browser push |
| 30 min before | `calendar.reminder.30min` | In-app + browser push |
| 1 hour before | `calendar.reminder.1hr` | In-app + browser push + email |
| 1 day before | `calendar.reminder.1day` | In-app + email |

Reminder jobs are managed by BullMQ with delayed execution (`calendar-reminders` queue). If an event is rescheduled, the existing reminder job is cancelled and a new one is scheduled.

### Meeting Invitations

When a meeting is created with participants, each participant receives:
- **In-app notification:** "You've been invited to '{title}' on {date} at {time}"
- **Email notification:** Standard meeting invitation email with RSVP buttons (accept/tentative/decline)
- **RSVP actions:** Clicking RSVP in the email hits `/api/calendar/events/{id}/rsvp` with a signed token

### RSVP Updates

When a participant RSVPs, the organizer receives:
- **In-app notification:** "{name} {accepted/declined/tentatively accepted} your meeting '{title}'"

---

## Sub-Panel Updates

The Calendar sub-panel (defined in the overview) gains new elements when manual events are enabled:

```
+------------------------+
| CALENDAR               |
| > Mini Dashboard       |
|   3 Today              |
|   8 This Week          |
|   2 Overdue            |
|   1 Meeting today      |  <-- NEW: meeting count
|------------------------+
| > VIEWS                |
|   o Month              |
|   o Week               |
|   o Day                |  <-- NEW: day view
|   o Agenda             |
|                        |
| > FILTERS              |
|   o All                |
|   o Contracts          |
|   o Tasks              |
|   o CRM                |
|   o Meetings           |  <-- NEW: meeting filter
|   o My Events          |  <-- NEW: user's manual events
|------------------------+
| > UPCOMING             |
|   Mar 5: Henderson MSA |
|     review meeting     |
|   Mar 5: henderson-msa |
|     renewal deadline   |
|   Mar 8: sony-dist     |
|     termination        |
|------------------------+
| Google Calendar: Synced |  <-- NEW: sync status footer
| Last sync: 2 min ago   |
+------------------------+
```

---

## Feature Control Plane Flags

All new capabilities are gated behind Feature Control Plane flags. Flags follow the progressive disclosure pattern -- each flag enables a larger surface area.

| Flag | Default | Dependencies | Description |
|------|---------|-------------|-------------|
| `ENABLE_MANUAL_EVENTS` | `false` | None | Enables manual event creation (Event mode). Gate for all other calendar creation features |
| `ENABLE_MEETING_SCHEDULING` | `false` | `ENABLE_MANUAL_EVENTS` | Enables Meeting mode in event creation form. Requires Google Meet integration |
| `ENABLE_GOOGLE_CALENDAR_SYNC` | `false` | `ENABLE_MANUAL_EVENTS` | Enables two-way Google Calendar sync per user |
| `ENABLE_SMART_SCHEDULING` | `false` | `ENABLE_MEETING_SCHEDULING`, `ENABLE_GOOGLE_CALENDAR_SYNC` | Enables Otto-powered "Find best time" suggestions |
| `ENABLE_EVENT_RECURRENCE` | `false` | `ENABLE_MANUAL_EVENTS` | Enables recurring event creation (daily, weekly, monthly, custom) |
| `ENABLE_EVENT_REMINDERS` | `true` | `ENABLE_MANUAL_EVENTS` | Enables reminder notifications for manual/meeting events |

**Dependency chain:** Manual Events --> Meeting Scheduling --> Smart Scheduling (requires sync)

When a flag is disabled, its UI elements are hidden entirely (not grayed out). No feature-flagged data is created or stored when the flag is off.

---

## Security Considerations

### Data Encryption

- Event `description` and `agenda` fields are encrypted at rest using the workspace DEK (Data Encryption Key) via the envelope encryption model defined in the Security spec
- Google OAuth refresh tokens are encrypted at rest with a separate application-level key, never stored in plaintext
- Event titles are encrypted at rest (may contain sensitive vault/deal names)

### Google Calendar Integration

- OAuth consent is per-user, not workspace-wide. Each user independently authorizes Google Calendar access
- Airlock never reads the content/details of external Google Calendar events -- only free/busy status is used for availability queries
- External events displayed in Airlock show title and time only (not description, attendees, or attachments from Google Calendar)
- Google OAuth tokens can be revoked at any time by the user via Calendar Settings or by revoking Airlock's access in Google Account settings
- Webhook endpoints validate the `X-Goog-Channel-ID` and `X-Goog-Resource-ID` headers against stored `calendar_sync_state` records

### Access Control

- Only the event creator and workspace admins can edit or delete events
- Participants can view event details and RSVP but cannot modify event content
- Vault date events and task due events remain read-only from the Calendar -- edits must go through their source module
- Meeting Google Meet links are only visible to confirmed participants and the organizer (not declined participants)
- External participant emails are validated (RFC 5322 format) before invitation

### Audit Trail

- Event creation, modification, deletion, and RSVP actions are logged to the append-only audit log
- Google Calendar sync operations (connect, disconnect, sync conflicts) are logged
- Failed sync attempts and OAuth token refresh failures are logged with error context

### PII Handling

- Participant email addresses are considered PII and are encrypted at rest
- When Otto generates prep briefs, PII is redacted before reaching the LLM provider (per Security Principle 9: "PII never reaches LLM providers unredacted")
- External participant contact information is not exposed in calendar exports or iCal feeds

---

## Migration Path

### From v1 (Computed-Only Calendar) to v2 (Manual Events)

1. **Database migration:** Create `calendar_events`, `calendar_event_participants`, and `calendar_sync_state` tables with RLS policies
2. **Feature flag rollout:** Enable `ENABLE_MANUAL_EVENTS` for internal testing workspaces
3. **Schedule-X adapter:** Add `useCalendarEvents` hook alongside existing computed event hooks
4. **UI rollout:** Event creation modal + inline quick-create + detail popover
5. **Google sync:** Enable `ENABLE_GOOGLE_CALENDAR_SYNC` after OAuth infrastructure is tested
6. **Meeting scheduling:** Enable `ENABLE_MEETING_SCHEDULING` after Google Meet API integration is validated
7. **Smart scheduling:** Enable `ENABLE_SMART_SCHEDULING` last, after availability data proves reliable

### Rollback Plan

All features are behind flags. Disabling a flag:
- Hides all related UI elements immediately
- Stops creating new events of that type
- Existing stored events are retained but not displayed
- Google Calendar sync is paused (not disconnected) -- webhook is deregistered, polling stops
- No data is deleted on flag disable

---

## Related Specs

- [Calendar Overview](./overview.md) -- Parent spec: computed events architecture, Schedule-X config, sub-panel layout, alert thresholds
- [CRM Communications](../CRM/crm-comms-brainstorm.md) -- Google Meet SDK integration context, meeting lifecycle
- [Search / Cmd+K](../Search/overview.md) -- `/new-event` and `/schedule-meeting` command palette actions
- [Feature Control Plane](../FeatureControlPlane/overview.md) -- Flag management, progressive disclosure patterns
- [Notifications](../Notifications/overview.md) -- Novu integration for reminders and meeting invitations
- [Security Overview](../Security/overview.md) -- Encryption at rest, DEK/KEK envelope model, PII redaction, audit trail
- [Workflow Engine](../WorkflowEngine/overview.md) -- Workflow deadline events (Source 5 from brainstorm)
- [Roles](../Roles/overview.md) -- Permission model for event CRUD operations
- [Shell](../Shell/overview.md) -- Toolbar Create button, triptych layout context
