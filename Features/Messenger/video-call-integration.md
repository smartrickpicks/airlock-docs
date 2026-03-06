# Video Call Integration -- Design Spec

> **Status:** SPECCED
> **Source:** Messenger overview, Meeting Intelligence spec, Google Meet SDK research, CRM comms brainstorm

## Summary

Adds video calling capability to Messenger. Users can start or schedule video calls directly from any conversation. **Calls are embedded inside Airlock** using self-hosted Jitsi Meet (Apache 2.0) -- users never leave the app. For meetings with external participants, Google Meet links are generated as a fallback. Call state is tracked in real-time via Redis and broadcast over WebSocket. Post-call meeting intelligence -- transcription, summary, action items -- flows back into the conversation thread as special system message types. For Vault Threads, call artifacts are automatically linked to the vault and surface in the Signal panel.

---

## Video Call in the Messenger Context

### Why Messenger, Not Just Calendar?

Messenger is the natural entry point for ad-hoc calls. The typical workflow is conversational: a user is chatting with a colleague about a vault, realizes the discussion needs voice/video, and clicks a video icon -- no context switch required.

| Reason | Detail |
|--------|--------|
| **Natural UX** | Chatting with someone leads to "let's hop on a call" -- one click from the conversation header |
| **Contextual linking** | The call is automatically linked to the conversation (and its vault, if a Vault Thread) |
| **Post-call artifacts** | Summary and action items appear inline in the conversation thread |
| **No context switch** | Call renders as a floating overlay or expands into the Orchestrate panel -- users never leave Airlock |

Calendar remains the right tool for scheduled, multi-participant meetings. Messenger handles the spontaneous "let's talk now" pattern.

### Relationship to Meeting Intelligence

Messenger and Meeting Intelligence have a clear division of responsibility:

- **Messenger** is the trigger and the display surface -- it initiates calls and renders post-call artifacts
- **Meeting Intelligence** is the processing engine -- it handles transcription, summarization, and action item extraction
- Data flows in one direction during a call (Messenger -> Meeting Intelligence) and reverses after (Meeting Intelligence -> Messenger)

```
User clicks video icon
        |
        v
Messenger creates Jitsi room via Airlock backend
        |
        v
"Call started" system message posted to conversation
        |
        v
Jitsi call renders as floating overlay (480x360) inside Airlock
   |                                              |
   +-- User can resize, go full-screen,           |
   |   or expand into Orchestrate panel            |
   |                                              |
   +-- Other participants click "Join"             |
        in the conversation -> same embedded call  |
        |
        v
Call ends -> Meeting Intelligence pipeline triggered
        |
        v
Summary + action items flow back to Messenger as system messages
        |
        v
(If Vault Thread) Artifacts also appear in vault Signal feed
```

---

## Call Initiation UX

### Video Call Button

| Property | Value |
|----------|-------|
| Location | Conversation header, right side, grouped with action icons |
| Icon | Video camera (Lucide: `Video`) |
| Size | 20px icon, 32px touch target |
| Tooltip | "Start or schedule a video call" |
| Visible in | Direct Messages, Team Conversations, Vault Threads |
| Hidden in | Module Conversations (too many potential participants; use Calendar) |

### Click Flow

1. User clicks the video icon in the conversation header.
2. A dropdown appears with two options:
   - **Start Instant Call** -- Launch an embedded Jitsi call immediately inside Airlock
   - **Schedule Meeting** -- Opens a meeting scheduler modal (reuses the Calendar event creation form)
3. For **Start Instant Call**:
   1. Client sends `POST /api/conversations/{id}/call` to the backend
   2. Backend creates a Jitsi room (`vault-{vaultId}-{meetingId}`) and generates a JWT token for authenticated access
   3. Backend inserts a `call_started` system message into the conversation
   4. Backend publishes `call.started` WebSocket event to all conversation members
   5. Backend stores call state in Redis (`call:{meeting_id}`)
   6. Client embeds the Jitsi call as a **floating overlay** (480×360, resizable, draggable) using the Jitsi iframe API
   7. Other conversation members see the system message with a "Join Call" button that opens the same embedded Jitsi room
   8. Users can expand the overlay to full Orchestrate panel width or go full-screen (Cmd+Shift+F)
4. For **Schedule Meeting**:
   1. Opens the meeting scheduler modal pre-populated with conversation participants
   2. Platform toggle: **Jitsi (embedded)** or **Google Meet** (for external participants)
   3. Follows the standard Calendar event creation flow
   4. A `call_scheduled` system message is posted to the conversation with date/time
   5. At the scheduled time, participants receive a notification with a "Join Call" button that opens the embedded call

### Instant Call Dropdown

```
+----------------------------------+
|  ▶ Start Instant Call            |
|    Call opens inside Airlock     |
|----------------------------------|
|  📅 Schedule Meeting             |
|    Pick a date and time          |
|----------------------------------|
|  🔗 Google Meet Link             |
|    For external participants     |
+----------------------------------+
```

The "Google Meet Link" option is only shown when `ENABLE_GOOGLE_MEET_INTEGRATION` is enabled. It generates a Meet link and opens it in a new tab (for when external participants need a standard Google Meet experience).

The dropdown dismisses on click outside or Escape key.

---

## System Messages for Calls

Call-related events are rendered as system messages -- visually distinct from regular user messages. They use muted colors, no author avatar, and centered/inline layout within the message stream.

### System Message Types

| Event | System Message Text | Visual Treatment |
|-------|-------------------|------------------|
| Call started | "[User] started a video call" + Join Call button | Icon: video camera. Blue accent. Prominent Join button. Clicking Join opens the embedded Jitsi call overlay. |
| Call scheduled | "[User] scheduled a call for [date/time]" + Add to Calendar link | Icon: calendar. Muted blue. |
| Participant joined | "[User] joined the call" | Subtle gray text, small font. No button. |
| Participant left | "[User] left the call" | Subtle gray text, small font. No button. |
| Call ended | "Call ended -- [duration]" + View Summary link (when available) | Icon: video-off. Muted. Duration in bold. |
| Summary ready | "Meeting summary available" + expandable summary card | Icon: clipboard. Purple accent. Expandable. |
| Action items created | "[N] action items created from this call" + item list | Icon: check-circle. Green accent. Expandable. |

### System Message Rendering in the 340px Drawer

```
+--------------------------------------+
|                                      |
|     [video icon] Ana started a       |
|     video call                       |
|     [ Join Call -> ]                  |
|                                      |
|     Dave joined the call             |
|                                      |
|  [avatar] Marco Li          11:02 AM |
|  Joining now, give me a sec.         |
|                                      |
|     Marco joined the call            |
|                                      |
|     Call ended -- 23 min             |
|                                      |
|     +------------------------------+ |
|     | Meeting Summary -- Mar 5     | |
|     | 23 min -- 3 participants     | |
|     |                              | |
|     | - Agreed on Q2 budget        | |
|     | - Dave to draft proposal     | |
|     | - Follow-up next Wednesday   | |
|     |                              | |
|     | Action Items (3)             | |
|     |  [ ] Draft proposal -> Dave  | |
|     |  [ ] Review quotes -> Ana    | |
|     |  [ ] Schedule follow-up      | |
|     |                              | |
|     | [Transcript] [Edit Summary]  | |
|     +------------------------------+ |
|                                      |
+--------------------------------------+
```

---

## Call State in Conversation Header

When a call is active for a conversation, the header updates to show a persistent call indicator.

### Active Call Header States

**No active call (default):**
```
+--------------------------------------+
|  [<] Henderson MSA   [video] [...]   |
+--------------------------------------+
```

**Active call:**
```
+--------------------------------------+
|  [<] Henderson MSA       [video][..] |
|  [green dot] Call in progress        |
|  3 participants -- 12:34   [Join ->] |
+--------------------------------------+
```

### Header Behavior

| State | Video Icon | Banner | Action |
|-------|-----------|--------|--------|
| No call | Default icon, muted | Hidden | Click opens dropdown (Start/Schedule) |
| Call active, user not in call | Pulsing green dot on icon | Visible: participant count + duration | Click "Join" opens Meet in new tab |
| Call active, user in call | Solid green dot on icon | Visible: "You're in this call" + duration | Click opens Meet tab (brings to front) |
| Call ended, summary processing | Default icon | Brief flash: "Processing summary..." (3s) | None |

### Duration Counter

- Starts counting from `call.started` event timestamp
- Updates every second via client-side timer (not WebSocket -- too chatty)
- Format: `MM:SS` for calls under 1 hour, `H:MM:SS` for longer calls
- Timer stops when `call.ended` event is received

---

## Conversation Types and Call Behavior

### Direct Messages

| Property | Behavior |
|----------|----------|
| Call type | 1:1 video call |
| Participants | The two DM participants only |
| Auto-invite | Both participants see the Join button |
| Meeting record | Auto-created, linked to both user profiles |
| Post-call summary | Appears in the DM thread |
| CRM integration | If either participant is linked to a CRM contact, the call is logged on the contact record |

### Vault Threads

Vault Threads have the richest call integration because calls are contextually tied to the vault's lifecycle.

| Property | Behavior |
|----------|----------|
| Call type | Group call for vault collaborators |
| Participants | All vault members invited (can decline) |
| Auto-invite | All members of the vault's conversation receive the Join button |
| Meeting record | Linked to the vault -- appears in the vault's Signal feed as a `call.completed` event |
| Post-call summary | Appears in both the Messenger thread AND the vault Signal panel |
| Action items | Auto-linked to the vault; appear in the vault's task list |
| Chamber context | The call record stores the vault's current Chamber at call time (for audit) |

**Vault Signal Panel Integration:**

When a call is started from a Vault Thread, the Signal panel shows:

```
Signal Feed
+--------------------------------------+
| [call icon] Video call started       |
| Ana Chen -- 11:00 AM                 |
| 3 participants -- 23 min             |
|                                      |
| [call icon] Meeting summary          |
| 11:25 AM                             |
| - Agreed on Q2 budget allocation     |
| - 3 action items created             |
| [View Full Summary ->]               |
+--------------------------------------+
```

These Signal events follow the same bidirectional sync pattern defined in the Messenger overview: call system messages in Messenger are mirrored as events in the vault Signal feed.

### Team Conversations

| Property | Behavior |
|----------|----------|
| Call type | Group call for team members |
| Participants | All team conversation members invited |
| Meeting record | Linked to the team conversation |
| Post-call summary | Appears in the team conversation thread |
| Action items | Created as standalone tasks, assignable to any team member |

### Module Conversations

| Property | Behavior |
|----------|----------|
| Call type | **Not supported** |
| Reason | Module Conversations can have many members (all module users). Large group calls should be scheduled via Calendar with explicit invitee selection. |
| UI treatment | Video icon is hidden in Module Conversation headers |

---

## Meet Add-on: Airlock Sidebar in Google Meet

### Feature Flag

`ENABLE_MEET_ADDON` (default: `false`)

This is an optional enhancement that embeds Airlock context inside the Google Meet window. It requires Google Workspace admin configuration and is only available to organizations using Google Workspace.

### What It Does

The Google Meet Add-ons SDK allows third-party applications to render a sidebar panel inside the Meet window. The Airlock Meet Add-on shows contextual information during a call so participants do not need to switch back to Airlock.

### Sidebar Content (by Conversation Type)

**Vault Thread calls:**
- Vault name, current Chamber, and Gate status
- Key vault fields (from the Control panel summary)
- Recent Signal feed events (last 5)
- Live note-taking textarea (syncs to conversation as messages)

**DM calls:**
- Participant profile cards (from CRM if linked)
- Recent conversation history (last 10 messages)
- Live note-taking textarea

**Team Conversation calls:**
- Team name and member list
- Meeting agenda (if set via scheduled meeting)
- Live note-taking textarea

### Implementation Details

| Property | Value |
|----------|-------|
| App type | Separate lightweight React app |
| Route | Served at `/meet-addon` on the Airlock domain |
| Auth | Same OAuth tokens as main Airlock app (passed via postMessage handshake) |
| Communication | REST API calls to Airlock backend (not WebSocket -- simpler for the add-on context) |
| Registration | Registered as a Google Workspace Marketplace app by the workspace admin |
| Availability | Only for Google Workspace customers with admin-approved marketplace apps |

### Add-on Heartbeat

When the Meet Add-on is active, it sends a heartbeat every 15 seconds to the Airlock backend:

```json
{
  "meeting_id": "uuid",
  "user_id": "uuid",
  "timestamp": "2026-03-05T11:15:00Z"
}
```

This heartbeat provides more accurate participant tracking than relying solely on "Join Call" clicks from Airlock (see "How Airlock Knows Call State" below).

---

## Real-Time Call State

### State Machine

```
idle -> ringing -> active -> ended -> processing -> complete
```

| State | Description | Duration |
|-------|-------------|----------|
| `idle` | No call exists for this conversation | Indefinite |
| `ringing` | Call created, Meet link generated, waiting for participants | 60 seconds timeout, then auto-transition to `ended` if no one joins |
| `active` | At least one participant has joined | Until last participant leaves or explicit end |
| `ended` | All participants have left or call was explicitly ended | Brief (transitions to `processing` if Meeting Intelligence enabled) |
| `processing` | Meeting Intelligence pipeline running (transcription, summary) | Typically 1-5 minutes |
| `complete` | Summary and action items delivered to conversation | Terminal state |

### State Storage in Redis

Each active call is stored as a Redis hash with a TTL:

**Key:** `call:{meeting_id}`

| Field | Type | Description |
|-------|------|-------------|
| `status` | string | Current state from the state machine |
| `conversation_id` | UUID | The conversation this call belongs to |
| `vault_id` | UUID | Nullable. Set if this is a Vault Thread call. |
| `initiator_id` | UUID | User who started the call |
| `google_meet_link` | string | The generated Meet URL |
| `google_meet_code` | string | The Meet room code (e.g., `abc-defg-hij`) |
| `google_calendar_event_id` | string | The Calendar event ID used to generate the link |
| `started_at` | ISO timestamp | When the call was initiated |
| `ended_at` | ISO timestamp | When the call ended (set on transition to `ended`) |
| `participants` | JSON array | List of `{user_id, joined_at, left_at}` objects |

**TTL:** 24 hours after transition to `complete`. This allows late-arriving summary data to be correlated.

**Redis Pub/Sub channel:** `call_state:{conversation_id}` -- broadcasts state changes to WebSocket subscribers.

### WebSocket Events

| Event | Payload | Subscribers |
|-------|---------|------------|
| `call.started` | `{meeting_id, conversation_id, vault_id, initiator_id, google_meet_link, started_at}` | All conversation members |
| `call.joined` | `{meeting_id, user_id, display_name}` | All conversation members |
| `call.left` | `{meeting_id, user_id, display_name}` | All conversation members |
| `call.ended` | `{meeting_id, duration_seconds, participant_count}` | All conversation members |
| `call.processing` | `{meeting_id, pipeline_stage}` | All conversation members |
| `call.summary_ready` | `{meeting_id, summary_message_id}` | All conversation members |
| `call.action_items_ready` | `{meeting_id, action_item_ids[], task_ids[]}` | All conversation members |

These events are published on the existing `conversation:{conversation_id}` WebSocket topic, alongside regular message events. Clients distinguish them by event_type prefix (`call.*` vs `message.*`).

### How Airlock Knows Call State

Google Meet does not expose real-time participant events to external applications. Airlock uses a layered approach to track call state:

| Layer | Mechanism | Accuracy | Available When |
|-------|-----------|----------|---------------|
| 1. Click tracking | Track who clicked "Join Call" in Airlock UI | Good for join events, cannot detect leave | Always |
| 2. Meet Add-on heartbeat | 15-second heartbeat from the add-on sidebar | High accuracy for join AND leave detection | Only when `ENABLE_MEET_ADDON` is true and user has the add-on installed |
| 3. Post-call reconciliation | Google Calendar API `events.get` returns actual attendees and duration after the event | Definitive, but delayed | After call ends |

**Fallback for call end detection (without add-on):**
- If no "Join Call" clicks have occurred for 60 seconds after call creation, transition to `ended` (nobody joined)
- If the last known participant clicked "Join" more than 2 hours ago and no heartbeat is received, prompt the initiator: "Is the call still active?"
- When any participant returns to the Airlock tab, client sends a check: "Am I still in a call?" which triggers post-call reconciliation

---

## Post-Call Flow

### When a Call Ends

The post-call pipeline is a sequence of events that delivers meeting artifacts back to the conversation:

```
Call ends
  |
  v
1. Redis call state -> "ended"
  |
  v
2. System message posted: "Call ended -- {duration}"
  |
  v
3. Meeting Intelligence pipeline triggered (async)
  |   a. Retrieve Gemini auto-generated notes from Google Drive (if available)
  |   b. Retrieve Vexa bot transcript (if Vexa was running)
  |   c. Combine sources and send to Otto for summarization
  |   d. Otto extracts action items with assignees
  |
  v
4. Redis call state -> "processing"
  |
  v
5. WebSocket: call.processing event
  |
  v
6. Otto completes summarization
  |
  v
7. call_summary system message inserted into conversation
  |
  v
8. call_action_items system message inserted into conversation
  |
  v
9. Tasks created in Task system (linked to vault if applicable)
  |
  v
10. Redis call state -> "complete"
  |
  v
11. WebSocket: call.summary_ready + call.action_items_ready events
```

### Summary Card Rendering

The `call_summary` message renders as an expandable card within the conversation thread:

```
+--------------------------------------+
| Meeting Summary -- Mar 5, 2026       |
| Duration: 23 min -- 3 participants   |
|                                      |
| - Agreed on Q2 marketing budget      |
|   allocation of $50K                 |
| - Dave to draft the vendor proposal  |
|   by end of week                     |
| - Follow-up meeting scheduled for    |
|   next Wednesday                     |
|                                      |
| Action Items (3)                     |
|   [ ] Draft Q2 proposal             |
|       -> Dave -- Due Fri Mar 7       |
|   [ ] Review vendor quotes           |
|       -> Ana -- Due Tue Mar 11       |
|   [ ] Schedule follow-up meeting     |
|       -> Billy -- Due Wed Mar 12     |
|                                      |
| [View Full Transcript] [Edit Summary]|
+--------------------------------------+
```

**Card behavior:**
- Collapsed by default: shows only "Meeting summary available -- 23 min, 3 participants" with an expand chevron
- Expanded: shows full summary bullets and action items
- "View Full Transcript" opens the transcript in the Document Suite viewer (Orchestrate panel)
- "Edit Summary" allows any conversation member to correct the AI-generated summary (edits are tracked)
- Action item checkboxes are interactive -- checking one marks the linked Task as complete

### Where Post-Call Artifacts Appear

| Surface | What Shows | Condition |
|---------|-----------|-----------|
| Messenger conversation thread | Full summary card + action items list | Always (for conversations with video call support) |
| Vault Signal feed | `call.completed` event with summary snippet + link to full card | Only for Vault Thread calls |
| Calendar event detail | Summary attached to the Google Calendar event description | Always (via Calendar API update) |
| CRM contact timeline | Call log entry with summary snippet | Only if participants are linked to CRM contacts |
| Task system | Individual task records for each action item | Only if `ENABLE_AUTO_ACTION_ITEMS` is true |

---

## Call History

### Per-Conversation Call History

Accessible via the "..." overflow menu in the conversation header, selecting "Call History."

**Layout:**
```
+--------------------------------------+
|  [<] Call History                     |
|  Henderson MSA                       |
+--------------------------------------+
|                                      |
|  Mar 5, 2026 -- 11:00 AM            |
|  23 min -- Ana, Dave, Marco          |
|  "Agreed on Q2 marketing budget..." |
|  [View Summary ->]                   |
|                                      |
|  Feb 28, 2026 -- 3:15 PM            |
|  45 min -- Ana, Dave                 |
|  "Reviewed vendor proposals and..."  |
|  [View Summary ->]                   |
|                                      |
|  Feb 20, 2026 -- 10:00 AM           |
|  12 min -- Ana, Marco               |
|  No summary available                |
|  [View Details ->]                   |
|                                      |
+--------------------------------------+
```

**Fields per entry:**
- Date and start time
- Duration
- Participant names (up to 3, then "+N more")
- Summary snippet (first bullet point, truncated to one line)
- Link to full meeting intelligence record

**Pagination:** 20 entries per page, infinite scroll upward for older calls.

### Global Call Log

Accessible via:
- `Cmd+K` command palette -> `/call-history`
- Calendar module -> filter by "Meetings" or "Video Calls"

The global call log renders in the Orchestrate panel (center of the Triptych) with richer filtering:

| Filter | Options |
|--------|---------|
| Date range | Preset ranges (Today, This Week, This Month) or custom date picker |
| Participants | Multi-select user picker |
| Vault | Vault selector (only for Vault Thread calls) |
| Module | Module selector |
| Has action items | Boolean toggle |
| Has transcript | Boolean toggle |

**Table columns:** Date, Duration, Conversation, Participants, Summary Snippet, Action Items Count, Status (complete/processing/no-summary).

---

## Data Model Extensions

These extensions build on the existing Messenger data model defined in the overview.

### messages table -- new message_type enum values

Add to the existing `message_type` ENUM:

| New Value | Description |
|-----------|-------------|
| `call_started` | System message when a call is initiated |
| `call_scheduled` | System message when a call is scheduled for a future time |
| `call_joined` | System message when a participant joins (subtle rendering) |
| `call_left` | System message when a participant leaves (subtle rendering) |
| `call_ended` | System message when the call ends, includes duration |
| `call_summary` | Expandable summary card with bullets and action items |
| `call_action_items` | Action items list (may be combined with `call_summary` in rendering) |

### messages.metadata (JSONB) -- structured data per call message type

The existing `messages` table gains a `metadata` JSONB column (nullable) to carry structured data for call-related system messages.

**For `call_started`:**
```json
{
  "meeting_id": "uuid",
  "google_meet_link": "https://meet.google.com/abc-defg-hij",
  "google_calendar_event_id": "string",
  "initiator_id": "uuid"
}
```

**For `call_scheduled`:**
```json
{
  "meeting_id": "uuid",
  "google_meet_link": "https://meet.google.com/abc-defg-hij",
  "google_calendar_event_id": "string",
  "scheduled_at": "2026-03-07T14:00:00Z",
  "initiator_id": "uuid"
}
```

**For `call_joined` and `call_left`:**
```json
{
  "meeting_id": "uuid",
  "user_id": "uuid",
  "display_name": "Dave Kim"
}
```

**For `call_ended`:**
```json
{
  "meeting_id": "uuid",
  "duration_seconds": 1380,
  "participant_count": 3,
  "participants": [
    {"user_id": "uuid", "display_name": "Ana Chen"},
    {"user_id": "uuid", "display_name": "Dave Kim"},
    {"user_id": "uuid", "display_name": "Marco Li"}
  ]
}
```

**For `call_summary`:**
```json
{
  "meeting_id": "uuid",
  "summary_bullets": [
    "Agreed on Q2 marketing budget allocation of $50K",
    "Dave to draft the vendor proposal by end of week",
    "Follow-up meeting scheduled for next Wednesday"
  ],
  "action_item_count": 3,
  "transcript_available": true,
  "transcript_document_id": "uuid",
  "processed_by": "otto",
  "processing_duration_seconds": 45
}
```

**For `call_action_items`:**
```json
{
  "meeting_id": "uuid",
  "action_items": [
    {
      "task_id": "uuid",
      "description": "Draft Q2 proposal",
      "assignee_id": "uuid",
      "assignee_name": "Dave Kim",
      "due_date": "2026-03-07"
    },
    {
      "task_id": "uuid",
      "description": "Review vendor quotes",
      "assignee_id": "uuid",
      "assignee_name": "Ana Chen",
      "due_date": "2026-03-11"
    }
  ]
}
```

### New Table: meetings

A persistent record of every call, separate from the ephemeral Redis state.

| Column | Type | Description |
|--------|------|-------------|
| `id` | UUID | Primary key |
| `workspace_id` | UUID | FK to workspaces. Required for RLS tenant isolation. |
| `conversation_id` | UUID | FK to conversations |
| `vault_id` | UUID | Nullable. FK to vaults. Set for Vault Thread calls. |
| `google_calendar_event_id` | VARCHAR(255) | Google Calendar event ID used to generate the Meet link |
| `google_meet_code` | VARCHAR(50) | The Meet room code (encrypted at rest) |
| `initiator_id` | UUID | FK to users. Who started the call. |
| `status` | ENUM | `scheduled`, `active`, `ended`, `processing`, `complete` |
| `started_at` | TIMESTAMPTZ | When the call was initiated |
| `ended_at` | TIMESTAMPTZ | Nullable. When the call ended. |
| `duration_seconds` | INTEGER | Nullable. Computed from ended_at - started_at. |
| `participant_count` | INTEGER | Nullable. Final participant count after reconciliation. |
| `summary_message_id` | UUID | Nullable. FK to messages. The call_summary system message. |
| `transcript_document_id` | UUID | Nullable. FK to documents. The stored transcript. |
| `chamber_at_call_time` | VARCHAR(50) | Nullable. The vault's Chamber when the call was made (Discover, Build, Review, Ship). |
| `created_at` | TIMESTAMPTZ | Record creation timestamp |

**Indexes:**
- `(workspace_id)` -- RLS enforcement
- `(conversation_id, started_at DESC)` -- per-conversation call history
- `(vault_id, started_at DESC)` -- per-vault call history
- `(initiator_id, started_at DESC)` -- "my calls" view
- `(status)` -- find active/processing calls

**RLS Policy:** `workspace_id = current_setting('app.workspace_id')` -- same pattern as all other tables.

### New Table: meeting_participants

| Column | Type | Description |
|--------|------|-------------|
| `meeting_id` | UUID | FK to meetings |
| `user_id` | UUID | FK to users |
| `joined_at` | TIMESTAMPTZ | When the participant joined |
| `left_at` | TIMESTAMPTZ | Nullable. When the participant left. |
| `joined_via` | ENUM | `airlock_click`, `meet_addon`, `direct_link`, `unknown` |
| `confirmed` | BOOLEAN | Default false. Set to true after post-call reconciliation with Google Calendar API. |

**Primary key:** `(meeting_id, user_id, joined_at)` -- composite to support re-joins.

### Entity Relationship Extension

```
conversations 1---* meetings
meetings 1---* meeting_participants *---1 users
meetings *---1 vaults (nullable, vault_thread calls only)
meetings 1---0..1 messages (summary_message_id)
meetings 1---0..1 documents (transcript_document_id)
```

---

## API Endpoints

### Call Management

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/conversations/{id}/call` | Start an instant call. Creates Calendar event, generates Meet link, posts system message. Returns `{meeting_id, google_meet_link}`. |
| `POST` | `/api/conversations/{id}/call/schedule` | Schedule a future call. Body: `{scheduled_at, title?, agenda?}`. Creates Calendar event with conferenceData. Returns `{meeting_id, google_meet_link, google_calendar_event_id}`. |
| `POST` | `/api/conversations/{id}/call/join` | Record that the current user clicked "Join Call." Updates Redis state and posts `call_joined` system message. |
| `POST` | `/api/conversations/{id}/call/leave` | Record that the current user left the call. Updates Redis state and posts `call_left` system message. |
| `POST` | `/api/conversations/{id}/call/end` | Explicitly end the call. Transitions state to `ended`, triggers post-call pipeline. Only callable by the initiator or a workspace admin. |
| `GET` | `/api/conversations/{id}/calls` | Call history for a conversation. Paginated: `?before={timestamp}&limit=20`. Returns list of meeting records with summary snippets. |
| `GET` | `/api/conversations/{id}/call/active` | Get the currently active call for a conversation, if any. Returns `null` or `{meeting_id, google_meet_link, status, participants, started_at}`. |

### Global Call History

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/meetings` | Global call log. Supports filters: `?vault_id=`, `?module_id=`, `?participant_id=`, `?from=`, `?to=`, `?has_action_items=true`. Paginated. |
| `GET` | `/api/meetings/{meeting_id}` | Full meeting detail: participants, summary, action items, transcript link. |

### Meet Add-on

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/meet-addon/heartbeat` | Heartbeat from the Meet Add-on. Body: `{meeting_id, user_id}`. Updates participant presence in Redis. |
| `GET` | `/api/meet-addon/context/{meeting_id}` | Get vault/conversation context for display in the Meet Add-on sidebar. |
| `POST` | `/api/meet-addon/note` | Save a note from the Meet Add-on. Body: `{meeting_id, content}`. Posts as a regular message in the linked conversation. |

### Authentication

All endpoints require the standard Airlock JWT auth token. The Meet Add-on endpoints additionally accept a short-lived add-on session token issued during the OAuth handshake.

---

## Google Calendar API Integration

### Creating a Meet Link

Airlock generates Google Meet links by creating Google Calendar events with conference data. This is the only supported method -- direct Meet link generation is not available via public API.

```python
# Pseudocode for Meet link generation
event = {
    "summary": f"Airlock Call: {conversation_name}",
    "start": {"dateTime": now_iso, "timeZone": user_timezone},
    "end": {"dateTime": one_hour_later_iso, "timeZone": user_timezone},
    "attendees": [{"email": email} for email in participant_emails],
    "conferenceData": {
        "createRequest": {
            "requestId": str(uuid4()),
            "conferenceSolutionKey": {"type": "hangoutsMeet"}
        }
    }
}
result = calendar_service.events().insert(
    calendarId="primary",
    body=event,
    conferenceDataVersion=1
).execute()

meet_link = result["conferenceData"]["entryPoints"][0]["uri"]
meet_code = result["conferenceData"]["conferenceId"]
```

### Required OAuth Scopes

| Scope | Purpose |
|-------|---------|
| `https://www.googleapis.com/auth/calendar.events` | Create and read calendar events with conference data |
| `https://www.googleapis.com/auth/drive.readonly` | Read Gemini auto-generated meeting notes from Drive (post-call) |

These scopes are requested during the Google Workspace OAuth connection flow, which is configured per workspace in Admin settings.

### Embedding Constraint

Google Meet uses `X-Frame-Options: SAMEORIGIN`, which prevents embedding via iframe. All Meet interactions happen in a separate browser tab. The Airlock UI provides the link and tracks state, but the video experience itself is fully within Google Meet.

---

## Feature Control Plane Flags

Integration with the [Feature Control Plane](../FeatureControlPlane/overview.md) for gradual rollout and dependency management.

| Flag | Default | Dependencies | Description |
|------|---------|-------------|-------------|
| `ENABLE_VIDEO_CALLS` | `false` | `ENABLE_GOOGLE_WORKSPACE_INTEGRATION` | Master toggle for all video call features in Messenger |
| `ENABLE_MEET_ADDON` | `false` | `ENABLE_VIDEO_CALLS` | Airlock sidebar inside Google Meet |
| `ENABLE_CALL_SUMMARIES_IN_CHAT` | `true` | `ENABLE_VIDEO_CALLS`, `ENABLE_AI_MEETING_SUMMARY` | Post-call summary cards in conversation thread |
| `ENABLE_AUTO_ACTION_ITEMS` | `true` | `ENABLE_CALL_SUMMARIES_IN_CHAT`, `ENABLE_ACTION_ITEM_EXTRACTION` | Automatic task creation from call action items |
| `ENABLE_CALL_HISTORY` | `true` | `ENABLE_VIDEO_CALLS` | Per-conversation and global call history views |

### Degraded States

| Scenario | Behavior |
|----------|----------|
| `ENABLE_VIDEO_CALLS` = false | Video icon hidden in all conversation headers. No call-related system messages. |
| Google Workspace not connected | Video icon visible but disabled with tooltip: "Connect Google Workspace to enable video calls" |
| `ENABLE_CALL_SUMMARIES_IN_CHAT` = false | Calls work normally, but no summary card or action items are posted post-call. `call_ended` message still appears with duration. |
| `ENABLE_AUTO_ACTION_ITEMS` = false | Summary card appears but action items are not auto-created as Tasks. Users can manually create tasks from the summary. |
| Meeting Intelligence pipeline fails | `call_ended` message appears normally. After pipeline timeout (10 min), a system message posts: "Meeting summary could not be generated. [Retry]" |

---

## Security Considerations

### Encryption and Storage

| Data | At Rest | In Transit | Notes |
|------|---------|-----------|-------|
| Google Meet link | Encrypted (column-level) | TLS 1.3 | Meet codes are sensitive -- anyone with the link can join |
| Meeting transcripts | Encrypted (DEK/KEK envelope) | TLS 1.3 | Same encryption as all Document Suite content |
| Call metadata in Redis | Encrypted (workspace-scoped key) | TLS (Redis 7 native TLS) | TTL ensures ephemeral data does not persist indefinitely |
| Summary bullets | Encrypted (column-level, JSONB) | TLS 1.3 | Stored as message metadata |
| Google OAuth tokens | Encrypted (KMS-backed KEK) | TLS 1.3 | Stored in workspace credentials vault, never in browser |

### Access Control

- Only conversation members can see call system messages (enforced by existing conversation membership checks)
- Only conversation members can call the `/call`, `/call/join`, `/call/leave`, `/call/end` endpoints
- The `call/end` endpoint is restricted to the call initiator or workspace admins
- Meeting records inherit workspace RLS -- no cross-workspace call data leakage
- The Meet Add-on authenticates via the same user OAuth session -- no separate credentials

### Audit Trail

All call events are logged to the append-only audit log:

| Event | Logged Fields |
|-------|--------------|
| `call.initiated` | initiator_id, conversation_id, vault_id, meeting_id, timestamp |
| `call.joined` | user_id, meeting_id, joined_via, timestamp |
| `call.ended` | meeting_id, duration_seconds, participant_count, timestamp |
| `call.summary_generated` | meeting_id, processed_by, processing_duration_seconds, timestamp |
| `call.action_items_created` | meeting_id, task_ids[], assignee_ids[], timestamp |

### PII Considerations

- Meeting transcripts may contain PII spoken during the call. Transcripts are encrypted at rest and subject to the same PII redaction rules as all vault data.
- PII in transcripts is NOT sent to external LLM providers unredacted. The Meeting Intelligence pipeline applies the standard PII redaction layer before Otto processes the transcript (per Security principle #9).
- Google Meet recordings are controlled by Google Meet settings, not by Airlock. Airlock does not access, store, or manage video recordings.

---

## Performance Considerations

| Concern | Mitigation |
|---------|-----------|
| Meet link generation latency | Google Calendar API call takes 500ms-2s. Show optimistic "Starting call..." state in UI while waiting. |
| Redis call state memory | Each active call is approximately 2KB in Redis. TTL of 24 hours ensures cleanup. |
| WebSocket event volume during calls | Join/leave events are low frequency. Typing indicators are suppressed during active calls to reduce noise. |
| Post-call pipeline duration | Summary generation is async (1-5 min). Users see "Processing..." indicator and are notified when ready. |
| Call history query performance | Indexed by `(conversation_id, started_at DESC)`. Paginated with cursor-based pagination. |
| Large workspace call volume | Global call log uses the same pagination pattern. Filters narrow the query scope. |

---

## Implementation Notes

### Google Meet Link Cannot Be Iframed

Google Meet sets `X-Frame-Options: SAMEORIGIN` on all responses, which prevents embedding in an iframe. This is a hard constraint, not a configuration issue. All implementations that show video "inline" in a chat app are using their own WebRTC infrastructure (e.g., Slack Huddles, Discord Voice). Airlock's approach of opening Meet in a new tab is the correct pattern for Google Meet integration.

### Calendar Event as Meet Link Vehicle

Google does not offer a standalone "create Meet link" API. The only public API method is to create a Google Calendar event with `conferenceDataVersion=1` and extract the Meet link from the response. The Calendar event serves as the container for the Meet room. This has a side benefit: the call automatically appears in participants' Google Calendars.

### Reconciliation Is Critical

Because Airlock cannot observe Meet room state in real-time (without the add-on), post-call reconciliation via the Google Calendar API is the authoritative source for:
- Who actually attended (vs. who clicked "Join" in Airlock)
- Actual call duration
- Whether the call actually happened (someone might close the tab immediately)

Reconciliation runs as a background job 5 minutes after the last known participant activity, and again 1 hour later to catch late Calendar API updates.

---

## Related Specs

- [Messenger Overview](./overview.md) -- Parent spec: conversation types, data model, real-time delivery, vault thread integration
- [RealTime](../RealTime/overview.md) -- WebSocket + Redis Pub/Sub infrastructure shared by Messenger and call state
- [FeatureControlPlane](../FeatureControlPlane/overview.md) -- Feature flags for gradual rollout
- [Security](../Security/overview.md) -- Encryption at rest, audit logging, PII redaction principles
- [CRM Communications](../CRM/crm-comms-brainstorm.md) -- Google Meet integration in the CRM context
