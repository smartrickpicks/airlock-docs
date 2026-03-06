# Meeting Intelligence -- Design Spec

> **Status:** SPECCED
> **Source:** CRM Communications brainstorm (Google Meet SDK section), Google Meet REST API / Add-ons SDK / Media API research, open-source meeting bot research (Vexa, Meetily), conversation context threading research
> **Added:** 2026-03-05

## Summary

Meeting Intelligence is a cross-cutting capability -- not a standalone module -- that weaves video conferencing, AI-powered transcription, action item extraction, and conversation memory into Airlock's existing architecture. Meetings launch from any vault, Calendar event, Messenger conversation, or Cmd+K command. Post-meeting, Otto processes the transcript to generate summaries, extract action items into the universal task system, and build a contact-level conversation memory that surfaces as prep briefs before future meetings. The core insight: when Billy meets Dave about a marketing initiative and later schedules a meeting with Mike on the same topic, all prior context is automatically available -- no manual searching, no lost institutional memory.

---

## Position in the Architecture

Meeting Intelligence is **not a module**. It does not appear in the Module Bar. It is a cross-cutting capability that integrates with multiple existing systems:

| Integration Point | Role |
|---|---|
| **CRM Module** | Primary launch point -- start a meeting from a vault detail page. Contact enrichment from participants. |
| **Calendar Module** | Scheduled meetings appear as calendar events (Source 4 in Calendar spec). Prep briefs surface on event click. |
| **Messenger** | Launch a meeting from the video icon in a conversation header. Meeting summary posts to the vault thread. |
| **Tasks Module** | AI-extracted action items create tasks in the universal task table. |
| **Otto AI Agent** | Processes transcripts for summaries, action items, topic extraction, and sentiment analysis. |
| **Search (MeiliSearch)** | Meeting summaries and transcripts are indexed for cross-module search. |
| **Signal Panel** | Meeting lifecycle events appear in the vault's Signal feed. |
| **Feature Control Plane** | Seven flags control progressive enablement of each sub-capability. |
| **Workflow Engine** | Meeting events can trigger automations (e.g., "meeting completed" triggers a follow-up workflow). |

### Where Meeting Intelligence Lives in the Shell

```
Module Bar  |  Sub-Panel  |  Triptych (Signal | Orchestrate | Control)
            |             |
            |             |  Signal: meeting.scheduled, meeting.started,
            |             |          meeting.ended, meeting.summary events
            |             |
            |             |  Orchestrate: prep brief panel (pre-meeting),
            |             |               transcript viewer (post-meeting)
            |             |
            |             |  Control: "Start Meeting" button in vault header,
            |             |           meeting history tab, participant list
```

Meetings attach to vaults. A meeting without a vault is valid (e.g., a general team standup) but loses vault-scoped intelligence features like prep briefs and topic threading. The system encourages vault-linked meetings by defaulting the vault association when launching from a vault context.

---

## Video Conferencing Layer

### Google Meet Integration

#### Why Google Meet

Google Meet provides the richest API surface for programmatic meeting management. The Google Workspace ecosystem (Calendar, Drive, Docs) means meetings, recordings, transcripts, and AI-generated notes all flow through APIs Airlock can consume without building custom infrastructure.

#### API Surface (Research Findings)

| API / Platform | Status | What It Does | Airlock Use |
|---|---|---|---|
| **Jitsi Meet (self-hosted)** | Stable (Apache 2.0) | Embeddable WebRTC video conferencing with iframe API, Docker deployment | **Primary**: Embedded video calls inside Airlock -- users never leave the app |
| **Jitsi iframe API** | Stable | Programmatic control of embedded Jitsi calls (mute, record, events) | Full meeting lifecycle control from Airlock's React frontend |
| **Google Calendar API** | GA | Create events with `conferenceData` to auto-generate Meet links | Meeting scheduling with Google Meet links for external participants |
| **Google Meet REST API** | GA | Get transcripts/recordings post-meeting, manage participants | Post-meeting data retrieval when Meet is used for external calls |
| **Google Meet Add-ons SDK** | GA | Embed YOUR application inside the Meet interface (side panel / main stage) | Optional: Airlock sidebar inside Meet showing vault context (for external calls) |
| **Google Drive API** | GA | Access Gemini meeting notes saved as Google Docs | Retrieve AI-generated notes post-meeting (when Meet + Gemini is used) |
| **Google Docs API** | GA | Parse Gemini notes documents for structured content | Extract attendees, topics, action items from Gemini notes |
| **Google Meet Media API** | Developer Preview | Raw WebRTC audio/video streams | NOT recommended -- strict enrollment, Jitsi covers this natively |

#### Embedded Video: Jitsi Meet (Primary -- In-App Calls)

Airlock embeds video calls **directly inside the application** so users never leave. Google Meet cannot be iframed (`X-Frame-Options: DENY`), so Airlock uses **Jitsi Meet** (Apache 2.0) as the primary embedded video engine.

**Why Jitsi Meet:**
- Open source (Apache 2.0), self-hosted, Docker-ready
- Full iframe API with programmatic JavaScript control
- WebRTC-based: HD video, screen sharing, chat, recording, E2EE
- Battle-tested at scale (used by 8x8, Matrix/Element, hundreds of enterprises)
- No account required for participants
- Runs as a Docker Compose sidecar alongside Airlock's existing infrastructure

**Jitsi Docker Compose sidecar:**

```yaml
# Added to Airlock's docker-compose.yml
jitsi-web:
  image: jitsi/web:stable
  ports:
    - "8443:443"
  environment:
    - ENABLE_AUTH=1
    - AUTH_TYPE=jwt
    - JWT_APP_ID=airlock
    - JWT_APP_SECRET=${JITSI_JWT_SECRET}
    - ENABLE_RECORDING=1
    - ENABLE_TRANSCRIPTIONS=1
  volumes:
    - jitsi-web-config:/config
    - jitsi-transcripts:/usr/share/jitsi-meet/transcripts

jitsi-prosody:
  image: jitsi/prosody:stable
  environment:
    - AUTH_TYPE=jwt
    - JWT_APP_ID=airlock
    - JWT_APP_SECRET=${JITSI_JWT_SECRET}
  volumes:
    - jitsi-prosody-config:/config

jitsi-jicofo:
  image: jitsi/jicofo:stable
  volumes:
    - jitsi-jicofo-config:/config

jitsi-jvb:
  image: jitsi/jvb:stable
  ports:
    - "10000:10000/udp"
  volumes:
    - jitsi-jvb-config:/config
```

**Embedding in Airlock (iframe API):**

```javascript
// Embed Jitsi call inside Airlock's Orchestrate panel
const api = new JitsiMeetExternalAPI("jitsi.airlock.internal", {
  roomName: `vault-${vault.id}-${meetingId}`,
  parentNode: document.getElementById("orchestrate-video-container"),
  width: "100%",
  height: "100%",
  jwt: userJwtToken,  // Airlock generates JWT for authenticated access
  configOverwrite: {
    startWithAudioMuted: true,
    startWithVideoMuted: false,
    prejoinPageEnabled: false,  // Skip pre-join, user already authenticated
    toolbarButtons: [
      "camera", "chat", "desktop", "microphone",
      "participants-pane", "raisehand", "tileview",
      "toggle-camera", "fullscreen"
    ],
    // Airlock branding
    DEFAULT_LOGO_URL: "/assets/airlock-logo.svg",
    SHOW_JITSI_WATERMARK: false,
    SHOW_WATERMARK_FOR_GUESTS: false,
  },
  interfaceConfigOverwrite: {
    SHOW_CHROME_EXTENSION_BANNER: false,
    MOBILE_APP_PROMO: false,
  }
});

// Listen for Jitsi events to sync with Airlock
api.addEventListener("videoConferenceJoined", (data) => {
  updateMeetingState(meetingId, "active", { participant: data.id });
});

api.addEventListener("videoConferenceLeft", () => {
  checkIfLastParticipant(meetingId);
});

api.addEventListener("recordingStatusChanged", (status) => {
  if (status.on) startTranscription(meetingId);
});
```

**Where the embedded call renders:**

| Context | Render Location | Size |
|---|---|---|
| From vault detail | Orchestrate panel replaces current view | Full triptych width |
| From Messenger | Floating overlay (PiP-style) | 480×360 resizable |
| From Calendar | Orchestrate panel | Full triptych width |
| Full-screen mode | Full browser viewport | Cmd+Shift+F toggle |

#### Google Meet (Secondary -- External Participants)

For meetings with **external participants** who use Google Workspace (clients, vendors, partners), Airlock can generate Google Meet links as an alternative:

1. Airlock creates a Google Calendar event with `conferenceData` via Calendar API
2. External participants receive a standard Google Meet invite
3. Internal Airlock users join via Jitsi (embedded) by default, or optionally via the Meet link
4. Post-meeting: Airlock retrieves Gemini notes via Drive API regardless of which platform was used

This dual approach means: **internal calls stay embedded, external calls use Meet when needed.**

#### OAuth Scopes Required

| Scope | Purpose | When Requested |
|---|---|---|
| `https://www.googleapis.com/auth/calendar` | Create calendar events with conferenceData | On first meeting schedule |
| `https://www.googleapis.com/auth/meetings.space.readonly` | Read meeting metadata (participants, state) | On first meeting schedule |
| `https://www.googleapis.com/auth/drive.readonly` | Access Gemini meeting notes in organizer's Drive | On first post-meeting processing |
| `https://www.googleapis.com/auth/documents.readonly` | Parse Gemini notes Google Docs | On first post-meeting processing |

Scopes are requested incrementally -- only when the user first triggers the capability that requires them. The "Connect Google Workspace" flow in Admin Settings handles OAuth consent. This extends the existing Google Drive connector defined in the CRM communications spec.

#### Meeting Link Generation

Airlock generates Meet links by creating Google Calendar events with embedded conference data:

```python
# Google Calendar API: create event with auto-generated Meet link
event = {
    "summary": f"Airlock: {vault.name}",
    "description": f"Meeting linked to vault: {vault.name}\nWorkspace: {workspace.name}",
    "start": {"dateTime": start_iso, "timeZone": timezone},
    "end": {"dateTime": end_iso, "timeZone": timezone},
    "attendees": [{"email": p.email} for p in participants],
    "conferenceData": {
        "createRequest": {
            "requestId": str(meeting_id),
            "conferenceSolutionKey": {"type": "hangoutsMeet"}
        }
    }
}

created_event = calendar_service.events().insert(
    calendarId="primary",
    body=event,
    conferenceDataVersion=1
).execute()

meet_link = created_event["conferenceData"]["entryPoints"][0]["uri"]
google_event_id = created_event["id"]
```

#### Meet Add-ons SDK (Optional Enhancement)

The Meet Add-ons SDK allows Airlock to embed a side panel **inside** the Google Meet interface. This is the inverse of embedding Meet inside Airlock -- instead, Airlock travels with the user into the meeting.

**Side Panel Content:**
- Vault name and current chamber
- Key vault metadata (counterparty, contract type, health score)
- Prep brief summary (attendee context, recent activity)
- Quick-action buttons: create task, flag for review, add note

**Implementation:** A separate lightweight React app deployed at a public URL, registered as a Meet Add-on via Google Workspace Marketplace. The add-on communicates with Airlock's backend via authenticated API calls.

**Feature flag:** `ENABLE_MEET_ADDON_SIDEBAR` (default: false). This is Phase 3+ and requires Workspace Marketplace registration.

### Meeting Launch Points

Users can start or schedule a meeting from five locations in Airlock:

| Launch Point | Context | Behavior |
|---|---|---|
| **Vault detail page** | Any vault in any module (CRM, Contracts, etc.) | "Start Meeting" button in Control panel header. Pre-fills vault association, attendees from vault contacts. |
| **Calendar module** | Day/week view or "New Event" action | Click "New Meeting" or schedule an event with Meet link. Prompts for vault association. |
| **Messenger** | Video icon in conversation header | Starts a meeting linked to the conversation's vault (for vault threads) or unlinked (for DMs/team chats). |
| **Cmd+K palette** | Any screen | `/schedule-meeting` or `/start-meeting` command. Prompts for vault, participants, time. |
| **Toolbar quick actions** | Right-click a vault in the sub-panel | "Schedule Meeting" in the context menu. |

#### "Start Meeting" Button UX

The button appears in the Control panel header when viewing a vault:

```
+------------------------------------------+
| Control Panel                            |
|   Henderson MSA                          |
|   [Start Meeting]  [Schedule Meeting]    |
|                                          |
|   Tabs: Lifecycle | Approvals | Audit    |
|              | Meetings | Otto AI        |
+------------------------------------------+
```

- **Start Meeting** (green): Creates an instant Jitsi room, embeds video call directly in the Orchestrate panel
- **Schedule Meeting** (outline): Opens a scheduling form with date/time picker, attendee selector, agenda field, platform toggle (Jitsi embedded / Google Meet link)

### Meeting State Machine

Meetings progress through a deterministic state machine:

```
scheduled ──> joining ──> active ──> ended ──> processing ──> ready
    |                                  |           |
    +── cancelled                      |           +── failed
                                       |
                                       +── abandoned (no post-processing)
```

| State | Description | Trigger |
|---|---|---|
| `scheduled` | Meeting created with future start time | POST /api/meetings (with scheduled_start) |
| `joining` | At least one participant has joined | Meet REST API: participant joined event |
| `active` | Meeting is in progress | First participant joins (or explicit start) |
| `ended` | Meeting has concluded | Meet REST API: meeting ended event, or manual end |
| `processing` | Otto is processing the transcript | Automatic after `ended` |
| `ready` | All intelligence artifacts are available | Otto processing complete |
| `cancelled` | Meeting was cancelled before starting | User action or calendar event deletion |
| `abandoned` | Meeting ended but post-processing skipped | Feature flags disabled, or no transcript available |
| `failed` | Post-processing encountered an error | Otto error, API failure |

**State storage:** Meeting state is persisted in the `meetings` table (`status` column). During active meetings, the current state is also cached in Redis for low-latency WebSocket broadcasts:

```
# Redis key for active meeting state
HSET meeting:{meeting_id} status "active" started_at 1709640000 participant_count 3
EXPIRE meeting:{meeting_id} 86400  # TTL: 24 hours
```

**WebSocket events:** Meeting state changes broadcast to all vault subscribers:

```json
{
  "type": "event",
  "topic": "vault:{vault_id}",
  "event": {
    "event_type": "meeting.status_changed",
    "payload": {
      "meeting_id": "mtg_abc123",
      "old_status": "active",
      "new_status": "ended",
      "ended_at": "2026-03-05T15:30:00Z",
      "duration_seconds": 1800
    }
  }
}
```

---

## Meeting Intelligence Agent

### Architecture Decision: Dual-Source Transcript Strategy

Airlock uses a dual-source strategy for obtaining meeting transcripts, prioritizing zero-infrastructure Google-native tools with a self-hosted fallback for organizations that need it.

#### Primary Source: Google Gemini Meeting Notes

Google Workspace Business Standard+ plans include Gemini-powered meeting notes. After a meeting ends, Gemini automatically generates a Google Doc in the organizer's Drive containing:

- Attendee list
- Topic summaries organized by discussion theme
- Action items in a "Next Steps" section (with assignee names)
- Key decisions

**How Airlock retrieves Gemini notes:**

1. Meeting ends -- Airlock receives the `ended` state via Meet REST API
2. After a configurable delay (default: 60 seconds, allowing Gemini processing time), poll the organizer's Google Drive for new Docs in the "Meeting Notes" folder
3. Match the Doc to the meeting via title pattern (`"Meeting notes - {meeting title}"`) or creation timestamp
4. Parse the Doc via Google Docs API:
   - Extract attendees from the participants section
   - Extract bullet-point summary from the body
   - Extract action items from "Next Steps" section
   - Extract topics from section headings
5. Store the `google_notes_doc_id` on the meeting record

**Advantages:**
- Zero infrastructure cost -- Google handles transcription and summarization
- Works automatically for Business Standard+ customers
- High-quality transcription via Google's speech models
- Notes are also available in the user's Google Docs for sharing outside Airlock

**Limitations:**
- Requires Google Workspace Business Standard or higher
- Notes appear as Google Docs, not raw transcript -- some detail lost
- Gemini notes may take 1-5 minutes to appear after meeting ends
- No real-time transcript access during the meeting

#### Secondary Source: Vexa Meeting Bot (Self-Hosted)

[Vexa](https://github.com/vexa-ai/vexa) (Apache-2.0, 1.7k GitHub stars) is the best open-source meeting bot available. It joins meetings as a bot participant and provides real-time transcription via OpenAI Whisper.

**Capabilities:**
- Joins Google Meet, Microsoft Teams, and Zoom meetings as a participant
- Real-time Whisper transcription (configurable model size)
- MCP server for tool integration
- Speaker diarization (identifies who said what)
- Supports multiple simultaneous meetings

**Deployment:**

```yaml
# Docker Compose addition for Vexa bot
vexa-bot:
  image: vexa/meeting-bot:latest
  environment:
    WHISPER_MODEL: base  # or small, medium, large-v3
    WEBHOOK_URL: http://api:8000/webhooks/vexa
    MAX_CONCURRENT_MEETINGS: 5
  volumes:
    - vexa_data:/data
  deploy:
    resources:
      limits:
        memory: 2G  # Whisper model memory requirement
```

**When to use Vexa:**
- Organization does not have Google Workspace Business Standard+
- Real-time transcription is needed (not just post-meeting)
- Meetings happen on Microsoft Teams or Zoom (not just Google Meet)
- Organization requires on-premises transcript processing (no Google Docs dependency)

**Feature flag:** `ENABLE_MEETING_BOT` (default: false)

**Privacy:** The Vexa bot joins as a visible participant. All attendees see "Airlock Assistant" (or configurable name) in the participant list. The bot's presence serves as the transparency notice that the meeting is being recorded/transcribed. Organizations must configure this name to comply with their recording disclosure policies.

#### Meetily Reference (Local-Only Alternative)

[Meetily](https://github.com/meetily) (MIT) offers 100% local meeting intelligence with a Rust-based audio pipeline and Ollama for summarization. While not selected as the primary fallback, its architecture informs the local-dev story:

- In local development, developers can use Meetily's approach (system audio capture + local Whisper) for testing the meeting intelligence pipeline without any cloud dependencies
- Production deployments should use Gemini notes or Vexa

### Transcription Pipeline

```
Meeting ends
    |
    v
BullMQ job: "meeting.post_process" { meeting_id }
    |
    +-- Step 1: Check for Gemini notes (if ENABLE_MEETING_TRANSCRIPTION)
    |     |
    |     +-- Poll Drive API for notes doc (retry 3x with 60s delay)
    |     +-- If found: parse with Docs API -> raw_transcript, summary, action_items
    |     +-- Store google_notes_doc_id on meeting record
    |
    +-- Step 2: Check for Vexa transcript (if ENABLE_MEETING_BOT)
    |     |
    |     +-- Query Vexa API for transcript by meeting_id
    |     +-- If found: structured transcript with speaker labels + timestamps
    |
    +-- Step 3: Merge / deduplicate (if both sources available)
    |     |
    |     +-- Vexa transcript is richer (timestamps, speaker labels)
    |     +-- Gemini notes provide better summaries
    |     +-- Merge: use Vexa for raw_transcript, Gemini for initial summary
    |     +-- transcript_source: "merged" (or "gemini" / "vexa" if single source)
    |
    +-- Step 4: Otto AI processing (PydanticAI + LiteLLM)
    |     |
    |     +-- Input: raw transcript + meeting metadata + vault context
    |     +-- PII redaction BEFORE sending to LLM (non-negotiable)
    |     +-- Tools available to Otto:
    |     |     - summarize_meeting -> {bullets: [], one_liner: ""}
    |     |     - extract_action_items -> [{description, assignee_name, suggested_due}]
    |     |     - extract_topics -> [{name, confidence}]
    |     |     - analyze_sentiment -> {overall, per_participant}
    |     |     - match_contacts -> [{participant_email, contact_id, confidence}]
    |     |
    |     +-- Output: MeetingIntelligence result object
    |
    +-- Step 5: Persist results
    |     |
    |     +-- INSERT into meeting_intelligence table
    |     +-- UPDATE meeting status -> "ready"
    |     +-- Create tasks from action items (if ENABLE_ACTION_ITEM_EXTRACTION)
    |     +-- Update conversation_threads (if ENABLE_CONVERSATION_THREADING)
    |     +-- Index in MeiliSearch (meetings index)
    |
    +-- Step 6: Notify
          |
          +-- WebSocket event: meeting.intelligence_ready
          +-- Signal feed events: meeting.summary, meeting.action_items
          +-- Novu notification to meeting participants
```

### PII Redaction for LLM Processing

Per Airlock security principles (non-negotiable rule 9: "PII never reaches LLM providers unredacted"), the transcript undergoes PII redaction before being sent to Otto:

```python
# PII redaction pipeline (runs before LLM call)
redacted_transcript = pii_redactor.redact(
    text=raw_transcript,
    entities_to_redact=[
        "PHONE_NUMBER", "EMAIL_ADDRESS", "SSN",
        "CREDIT_CARD", "BANK_ACCOUNT", "PASSPORT_NUMBER"
    ],
    replacement_strategy="placeholder"  # "[PHONE_1]", "[EMAIL_1]", etc.
)

# Rehydration after LLM processing (maps placeholders back to real values)
final_summary = pii_redactor.rehydrate(otto_response.summary, redaction_map)
```

Names and company names are NOT redacted (they are required for action item assignment and contact matching). Only regulated PII categories are scrubbed.

### Action Item Extraction

Otto parses the transcript for commitments, decisions, and follow-ups using a structured extraction prompt:

**Otto System Prompt (action item extraction):**
```
You are analyzing a meeting transcript. Extract all action items -- explicit commitments
where a person agreed to do something. For each action item, provide:

1. description: What needs to be done (imperative form, e.g., "Send revised schedule")
2. assignee_name: Who committed to doing it (exact name from transcript)
3. suggested_due: When it should be done (ISO date if mentioned, null if not)
4. urgency: high/medium/low based on language used
5. context: The relevant quote from the transcript (max 100 words)

Only extract items where someone clearly committed. Do not infer action items
from general discussion. If the assignee is ambiguous, set assignee_name to null.
```

**Action item to task mapping:**

| Action Item Field | Task Table Field | Notes |
|---|---|---|
| `description` | `title` | Direct mapping |
| `assignee_name` | `assignee_id` | Resolved via workspace member name matching |
| `suggested_due` | `due_at` | Parsed from natural language ("by Friday" -> date) |
| `urgency` | `priority` | high -> P1, medium -> P2, low -> P3 |
| `context` | `description` | Quote from transcript stored as task description |
| -- | `source_type` | Set to `meeting_ai` |
| -- | `source_id` | Set to `meeting_id` |
| -- | `vault_id` | Inherited from meeting's vault association |
| -- | `module` | Inherited from vault's module |
| -- | `status` | Set to `open` (not auto-completed) |

**Assignee resolution:** Otto returns a `assignee_name` string. The system matches it against workspace members using a fuzzy name match:

1. Exact match on `display_name` (case-insensitive)
2. Partial match (first name only, if unique in workspace)
3. Email match (if Otto extracted an email reference)
4. If no match found, task is created with `assignee_id = null` and a `needs_assignment` flag

**AI-generated task labeling:** All tasks created by Meeting Intelligence carry a `source_type = 'meeting_ai'` tag and display a subtle "AI-generated" badge in the task UI. This allows users to review, edit, or dismiss AI suggestions before treating them as committed work.

---

## Conversation Memory & Context Threading

### The Core Pattern

The user's vision: conversations are not isolated events. They form threads of context that persist across meetings, participants, and time. Meeting Intelligence builds this memory layer.

Every meeting generates:
1. **Contact-level history** -- what was discussed with each participant
2. **Topic-level threads** -- cross-meeting linkages by subject matter
3. **Vault-level timeline** -- chronological meeting history per vault

These three dimensions create a mesh of context that powers prep briefs.

### Contact-Level Aggregation

Each contact (vault_contact or workspace_member) accumulates a conversation history compiled from multiple sources:

| Source | Data Captured | Storage |
|---|---|---|
| **Meeting summaries** | Bullet points, topics, sentiment, action items from meetings where they participated | `meeting_intelligence.summary` joined via `meeting_participants` |
| **Messenger DMs/threads** | Message content from vault threads and DMs | `messages` table (existing Messenger spec) |
| **CRM communications** | Email, iMessage, SMS via Smart Line | `communications` table (existing CRM comms spec) |
| **Vault Signal events** | Events they triggered (patches, approvals, extractions) | `channel_events` table |

**Contact conversation history endpoint:**

```
GET /api/contacts/{contact_id}/conversation-history
    ?workspace_id=uuid
    &since=2026-01-01          (optional date filter)
    &source=meetings,messenger  (optional source filter)
    &limit=20                   (pagination)
```

Response:

```json
{
  "contact": {
    "id": "ct_123",
    "name": "Jack Burke",
    "email": "jburke@nova.com",
    "company": "Nova Entertainment"
  },
  "history": [
    {
      "type": "meeting",
      "date": "2026-03-01T14:00:00Z",
      "summary": "Discussed marketing initiative timeline. Jack committed to Q2 rollout plan.",
      "topics": ["marketing initiative", "Q2 planning"],
      "sentiment": "positive",
      "action_items": [{"description": "Send Q2 rollout plan", "status": "open"}],
      "meeting_id": "mtg_abc",
      "vault_name": "Nova Entertainment"
    },
    {
      "type": "communication",
      "date": "2026-02-28T10:30:00Z",
      "channel": "imessage",
      "summary": "Follow-up on contract renewal terms",
      "vault_name": "Nova Entertainment"
    },
    {
      "type": "messenger",
      "date": "2026-02-25T09:00:00Z",
      "content": "Shared the updated proposal document",
      "vault_name": "Nova Entertainment"
    }
  ],
  "stats": {
    "total_meetings": 5,
    "total_communications": 23,
    "last_interaction": "2026-03-01T14:00:00Z",
    "avg_sentiment": "positive",
    "open_action_items": 2
  }
}
```

### Topic Threading

Topics are the connective tissue between meetings. When Billy discusses "marketing initiative" with Dave, and later meets Mike about the same topic, the system links these conversations.

**Topic extraction:** Otto identifies topics from meeting transcripts during post-processing. Each topic is a short phrase (2-5 words) with a confidence score.

**Topic registry:**

```
Topics table:
  "marketing initiative"  -- mentioned in mtg_001 (Billy+Dave), mtg_007 (Billy+Mike)
  "Q2 rollout"           -- mentioned in mtg_001 (Billy+Dave), mtg_003 (Dave+Sarah)
  "contract renewal"     -- mentioned in mtg_002 (Billy+Jack), mtg_005 (Ana+Jack)
```

**Topic matching algorithm (v1 -- deterministic):**

1. Otto extracts topics from new meeting transcript
2. For each extracted topic, search existing `conversation_threads` by text similarity
3. If similarity > 0.85 (configurable), link the new meeting to the existing thread
4. If no match, create a new thread
5. Similarity computed via PostgreSQL `pg_trgm` trigram matching (v1) or pgvector embedding distance (future)

```sql
-- Find matching topics using trigram similarity
SELECT id, topic, similarity(topic, :new_topic) AS sim
FROM conversation_threads
WHERE workspace_id = :workspace_id
  AND similarity(topic, :new_topic) > 0.85
ORDER BY sim DESC
LIMIT 1;
```

**Topic status lifecycle:**

| Status | Meaning |
|---|---|
| `active` | Topic has been discussed in the last 30 days |
| `stale` | No mentions in 30+ days |
| `resolved` | Manually marked as resolved by a user |

### Prep Briefs (Pre-Meeting Intelligence)

The prep brief is the culmination of conversation memory. When a meeting is scheduled, Airlock auto-generates a contextual brief that surfaces everything relevant.

**Prep brief generation trigger:**
- Meeting enters `scheduled` state with at least one participant
- BullMQ job: `meeting.generate_prep_brief` fires immediately and again 15 minutes before `scheduled_start`

**Prep brief structure:**

```json
{
  "meeting_id": "mtg_xyz",
  "generated_at": "2026-03-05T09:45:00Z",
  "sections": {
    "attendee_profiles": [
      {
        "contact_id": "ct_123",
        "name": "Jack Burke",
        "company": "Nova Entertainment",
        "role": "VP Business Development",
        "last_interaction": "2026-03-01T14:00:00Z",
        "interaction_count": 12,
        "sentiment_trend": "positive",  // trending over last 5 interactions
        "open_action_items": [
          {"description": "Send Q2 rollout plan", "due": "2026-03-08", "status": "overdue"}
        ]
      }
    ],
    "related_meetings": [
      {
        "meeting_id": "mtg_abc",
        "date": "2026-03-01T14:00:00Z",
        "title": "Marketing Initiative Review",
        "participants": ["Billy Chen", "Dave Ruiz"],
        "summary": "Agreed on Q2 timeline. Dave raised budget concerns.",
        "unresolved_items": ["Budget approval pending CFO sign-off"]
      }
    ],
    "active_vaults": [
      {
        "vault_id": "v_456",
        "name": "Nova Entertainment",
        "module": "CRM",
        "chamber": "Build",
        "health": 78,
        "recent_activity": "Patch submitted 2 days ago, awaiting review"
      }
    ],
    "open_tasks": [
      {
        "task_id": "t_789",
        "title": "Review Nova proposal",
        "assignee": "Billy Chen",
        "due": "2026-03-07",
        "status": "in_progress"
      }
    ],
    "recent_communications": [
      {
        "channel": "imessage",
        "date": "2026-03-04T16:20:00Z",
        "from": "Jack Burke",
        "preview": "Thanks for the update. Let's discuss tomorrow."
      }
    ],
    "topic_threads": [
      {
        "topic": "marketing initiative",
        "first_mentioned": "2026-02-15T10:00:00Z",
        "meeting_count": 3,
        "key_decisions": ["Q2 timeline agreed", "Budget pending CFO"],
        "related_vault_names": ["Nova Entertainment", "Nova Marketing Campaign"]
      }
    ]
  }
}
```

**Where prep briefs appear:**

| Location | Trigger | Display |
|---|---|---|
| **Calendar event detail** | User clicks a scheduled meeting event | Full prep brief in the Orchestrate panel |
| **Meeting dock panel** | User opens the meeting from Control panel "Meetings" tab | Collapsible prep brief above meeting details |
| **Notification** | 15 minutes before scheduled_start | Push notification: "Prep brief ready for your meeting with Jack Burke" with link |
| **Signal feed** | When prep brief is generated | `meeting.prep_brief_ready` event with one-line summary |

**Prep brief caching:** Briefs are cached in `meeting_intelligence.prep_brief` (JSONB). They are regenerated:
- When the meeting attendee list changes
- 15 minutes before the meeting (to capture last-minute context)
- On explicit user request ("Refresh prep brief" button)

---

## Data Model

### meetings

Extends and replaces the `meetings` table from the CRM communications brainstorm with additional fields for the full intelligence pipeline.

```sql
CREATE TABLE meetings (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id        UUID NOT NULL REFERENCES workspaces(id),
    vault_id            UUID REFERENCES vaults(id),          -- nullable: meetings can be unlinked
    title               TEXT NOT NULL,
    description         TEXT,                                 -- agenda / notes
    scheduled_start     TIMESTAMPTZ,                          -- when meeting is supposed to start
    scheduled_end       TIMESTAMPTZ,                          -- when meeting is supposed to end
    actual_start        TIMESTAMPTZ,                          -- when meeting actually started
    actual_end          TIMESTAMPTZ,                          -- when meeting actually ended
    duration_seconds    INT,                                  -- computed: actual_end - actual_start
    google_meet_link    TEXT,                                 -- meet.google.com/xxx-yyy-zzz
    google_event_id     TEXT,                                 -- Google Calendar event ID
    google_meet_id      TEXT,                                 -- Google Meet space resource name
    google_notes_doc_id TEXT,                                 -- Gemini notes Google Doc ID (nullable)
    status              TEXT NOT NULL DEFAULT 'scheduled'
                        CHECK (status IN (
                            'scheduled', 'joining', 'active', 'ended',
                            'processing', 'ready', 'cancelled', 'abandoned', 'failed'
                        )),
    organizer_id        UUID NOT NULL REFERENCES workspace_members(id),
    created_by          UUID NOT NULL REFERENCES workspace_members(id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Indexes
CREATE INDEX idx_meetings_workspace ON meetings(workspace_id, scheduled_start DESC);
CREATE INDEX idx_meetings_vault ON meetings(vault_id, scheduled_start DESC);
CREATE INDEX idx_meetings_status ON meetings(workspace_id, status);
CREATE INDEX idx_meetings_organizer ON meetings(organizer_id);
CREATE INDEX idx_meetings_google_event ON meetings(google_event_id) WHERE google_event_id IS NOT NULL;

-- RLS policy (workspace isolation)
ALTER TABLE meetings ENABLE ROW LEVEL SECURITY;
CREATE POLICY meetings_workspace_isolation ON meetings
    USING (workspace_id = current_setting('app.current_workspace_id')::uuid);
```

### meeting_participants

```sql
CREATE TABLE meeting_participants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id      UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    member_id       UUID REFERENCES workspace_members(id),   -- nullable for external participants
    contact_id      UUID REFERENCES vault_contacts(id),      -- nullable for internal participants
    email           TEXT NOT NULL,
    display_name    TEXT NOT NULL,
    rsvp_status     TEXT NOT NULL DEFAULT 'pending'
                    CHECK (rsvp_status IN ('accepted', 'declined', 'tentative', 'pending')),
    attended        BOOLEAN DEFAULT false,                    -- actually joined the call
    joined_at       TIMESTAMPTZ,                              -- when they joined
    left_at         TIMESTAMPTZ,                              -- when they left
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Indexes
CREATE INDEX idx_participants_meeting ON meeting_participants(meeting_id);
CREATE INDEX idx_participants_member ON meeting_participants(member_id) WHERE member_id IS NOT NULL;
CREATE INDEX idx_participants_contact ON meeting_participants(contact_id) WHERE contact_id IS NOT NULL;
CREATE UNIQUE INDEX idx_participants_unique ON meeting_participants(meeting_id, email);
```

### meeting_intelligence

Stores the output of Otto's post-meeting processing. One row per meeting (1:1 with meetings table).

```sql
CREATE TABLE meeting_intelligence (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    meeting_id          UUID NOT NULL UNIQUE REFERENCES meetings(id) ON DELETE CASCADE,
    transcript_source   TEXT NOT NULL CHECK (transcript_source IN ('gemini', 'vexa', 'merged', 'manual')),
    raw_transcript      TEXT,                                 -- full transcript (ENCRYPTED at rest via DEK)
    summary             JSONB NOT NULL DEFAULT '{}',
    -- summary schema: {"bullets": ["..."], "one_liner": "...", "key_decisions": ["..."]}
    action_items        JSONB NOT NULL DEFAULT '[]',
    -- action_items schema: [{"description": "...", "assignee_name": "...",
    --   "assignee_id": uuid|null, "suggested_due": iso_date|null,
    --   "task_id": uuid|null, "urgency": "high|medium|low", "context": "..."}]
    topics              JSONB NOT NULL DEFAULT '[]',
    -- topics schema: [{"name": "...", "confidence": 0.0-1.0,
    --   "related_vault_ids": [uuid], "thread_id": uuid|null}]
    sentiment           JSONB NOT NULL DEFAULT '{}',
    -- sentiment schema: {"overall": "positive|neutral|negative",
    --   "score": 0.0-1.0, "per_participant": {"email": {"sentiment": "...", "score": 0.0}}}
    prep_brief          JSONB,                                -- cached prep brief (generated before meeting)
    processed_at        TIMESTAMPTZ,                          -- when Otto finished processing
    processing_model    TEXT,                                  -- LLM model used (e.g., "claude-sonnet")
    processing_duration_ms INT,                               -- how long processing took
    created_at          TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Index for querying by meeting
CREATE INDEX idx_intelligence_meeting ON meeting_intelligence(meeting_id);
```

### conversation_threads

Cross-meeting topic threads that link conversations by subject matter.

```sql
CREATE TABLE conversation_threads (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id    UUID NOT NULL REFERENCES workspaces(id),
    topic           TEXT NOT NULL,
    topic_normalized TEXT NOT NULL,                            -- lowercased, trimmed, for matching
    -- topic_embedding vector(1536),                          -- pgvector (future: semantic search)
    first_mentioned TIMESTAMPTZ NOT NULL,
    last_mentioned  TIMESTAMPTZ NOT NULL,
    mention_count   INT NOT NULL DEFAULT 1,
    status          TEXT NOT NULL DEFAULT 'active'
                    CHECK (status IN ('active', 'resolved', 'stale')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Junction table: meetings <-> threads (many-to-many)
CREATE TABLE meeting_thread_links (
    meeting_id  UUID NOT NULL REFERENCES meetings(id) ON DELETE CASCADE,
    thread_id   UUID NOT NULL REFERENCES conversation_threads(id) ON DELETE CASCADE,
    confidence  REAL NOT NULL DEFAULT 1.0,                    -- how confident the topic match is
    PRIMARY KEY (meeting_id, thread_id)
);

-- Junction table: threads <-> vaults (many-to-many)
CREATE TABLE thread_vault_links (
    thread_id   UUID NOT NULL REFERENCES conversation_threads(id) ON DELETE CASCADE,
    vault_id    UUID NOT NULL REFERENCES vaults(id) ON DELETE CASCADE,
    PRIMARY KEY (thread_id, vault_id)
);

-- Indexes
CREATE INDEX idx_threads_workspace ON conversation_threads(workspace_id, last_mentioned DESC);
CREATE INDEX idx_threads_status ON conversation_threads(workspace_id, status);
CREATE INDEX idx_threads_topic_trgm ON conversation_threads USING gin(topic_normalized gin_trgm_ops);
```

### Entity Relationships

```
meetings 1---* meeting_participants *---1 workspace_members (nullable)
meetings 1---* meeting_participants *---1 vault_contacts (nullable)
meetings 1---1 meeting_intelligence
meetings *---1 vaults (nullable)
meetings *---* conversation_threads (via meeting_thread_links)
conversation_threads *---* vaults (via thread_vault_links)
```

---

## API Endpoints

### Meeting Lifecycle

| Method | Endpoint | Description | Auth |
|---|---|---|---|
| `POST` | `/api/meetings` | Schedule or start an instant meeting | `create_meeting` permission |
| `GET` | `/api/meetings/{id}` | Get meeting detail with intelligence | Meeting participant or vault member |
| `PATCH` | `/api/meetings/{id}` | Update meeting (title, time, participants) | Organizer or admin |
| `DELETE` | `/api/meetings/{id}` | Cancel a scheduled meeting | Organizer or admin |
| `POST` | `/api/meetings/{id}/join` | Mark current user as joining | Meeting participant |
| `POST` | `/api/meetings/{id}/end` | Manually mark meeting as ended | Organizer |

### Post-Meeting Intelligence

| Method | Endpoint | Description | Auth |
|---|---|---|---|
| `POST` | `/api/meetings/{id}/process` | Trigger or re-trigger post-processing | Organizer or admin |
| `GET` | `/api/meetings/{id}/transcript` | Get raw transcript | Meeting participant |
| `GET` | `/api/meetings/{id}/summary` | Get AI summary | Meeting participant or vault member |
| `GET` | `/api/meetings/{id}/action-items` | Get extracted action items | Meeting participant or vault member |
| `POST` | `/api/meetings/{id}/action-items/{item_index}/create-task` | Create a task from a specific action item | Meeting participant |
| `POST` | `/api/meetings/{id}/action-items/create-all` | Create tasks from all action items | Meeting participant |

### Prep Briefs

| Method | Endpoint | Description | Auth |
|---|---|---|---|
| `GET` | `/api/meetings/{id}/prep-brief` | Get or generate prep brief | Meeting participant |
| `POST` | `/api/meetings/{id}/prep-brief/refresh` | Force regeneration of prep brief | Meeting participant |

### Conversation Memory

| Method | Endpoint | Description | Auth |
|---|---|---|---|
| `GET` | `/api/contacts/{id}/conversation-history` | Get contact's conversation history | `view_contacts` permission |
| `GET` | `/api/topics` | Search conversation threads | `view_vaults` permission |
| `GET` | `/api/topics/{id}` | Get thread detail with linked meetings | `view_vaults` permission |
| `PATCH` | `/api/topics/{id}` | Update thread status (resolve, reactivate) | `manage_vaults` permission |
| `GET` | `/api/vaults/{id}/meetings` | Get all meetings linked to a vault | Vault member |

### Request/Response Examples

**POST /api/meetings** (schedule a meeting):

```json
// Request
{
  "vault_id": "v_456",
  "title": "Nova Entertainment - Q2 Planning",
  "scheduled_start": "2026-03-10T14:00:00Z",
  "scheduled_end": "2026-03-10T14:30:00Z",
  "participants": [
    {"email": "jburke@nova.com", "display_name": "Jack Burke"},
    {"email": "billy@airlock.app", "display_name": "Billy Chen"}
  ],
  "description": "Review Q2 marketing rollout plan and budget allocation"
}

// Response (201 Created)
{
  "id": "mtg_xyz789",
  "vault_id": "v_456",
  "title": "Nova Entertainment - Q2 Planning",
  "scheduled_start": "2026-03-10T14:00:00Z",
  "scheduled_end": "2026-03-10T14:30:00Z",
  "google_meet_link": "https://meet.google.com/abc-defg-hij",
  "google_event_id": "evt_google_123",
  "status": "scheduled",
  "participants": [
    {"email": "jburke@nova.com", "display_name": "Jack Burke", "rsvp_status": "pending"},
    {"email": "billy@airlock.app", "display_name": "Billy Chen", "rsvp_status": "accepted"}
  ],
  "prep_brief_status": "generating"
}
```

**GET /api/meetings/{id}** (meeting with intelligence):

```json
{
  "id": "mtg_abc123",
  "vault_id": "v_456",
  "title": "Nova Entertainment - Marketing Review",
  "status": "ready",
  "scheduled_start": "2026-03-01T14:00:00Z",
  "actual_start": "2026-03-01T14:02:00Z",
  "actual_end": "2026-03-01T14:35:00Z",
  "duration_seconds": 1980,
  "google_meet_link": "https://meet.google.com/abc-defg-hij",
  "participants": [
    {"display_name": "Billy Chen", "email": "billy@airlock.app", "attended": true},
    {"display_name": "Dave Ruiz", "email": "dave@airlock.app", "attended": true}
  ],
  "intelligence": {
    "transcript_source": "gemini",
    "summary": {
      "one_liner": "Agreed on Q2 marketing timeline with $50K budget allocation",
      "bullets": [
        "Q2 rollout plan finalized: launch April 15",
        "Budget: $50K allocated from marketing reserve",
        "Dave to coordinate with design team on creative assets",
        "Follow-up meeting scheduled for March 15"
      ],
      "key_decisions": [
        "April 15 launch date confirmed",
        "Budget approved at $50K"
      ]
    },
    "action_items": [
      {
        "description": "Send Q2 rollout plan document",
        "assignee_name": "Dave Ruiz",
        "assignee_id": "mem_456",
        "suggested_due": "2026-03-08",
        "task_id": "t_created_789",
        "urgency": "high"
      }
    ],
    "topics": [
      {"name": "marketing initiative", "confidence": 0.95, "thread_id": "thr_001"},
      {"name": "Q2 planning", "confidence": 0.88, "thread_id": "thr_002"}
    ],
    "sentiment": {
      "overall": "positive",
      "score": 0.82
    },
    "processed_at": "2026-03-01T14:40:00Z",
    "processing_model": "claude-sonnet"
  }
}
```

---

## Signal Feed Events

Meeting lifecycle events appear in the vault's Signal panel alongside other vault events:

| Event Type | Trigger | Signal Panel Display |
|---|---|---|
| `meeting.scheduled` | Meeting created and linked to vault | "[User] scheduled a meeting: [Title] on [Date]" with cyan dot |
| `meeting.started` | Meeting enters `active` state | "Meeting started: [Title] -- [N] participants" with green dot |
| `meeting.ended` | Meeting ends | "Meeting ended: [Title] -- [Duration]" with gray dot |
| `meeting.summary` | Otto generates summary | AI summary bullets displayed inline (cyan event styling, same as Otto events) |
| `meeting.action_items` | Action items extracted | "[N] action items extracted from [Title]" with task links |
| `meeting.prep_brief_ready` | Prep brief generated | "Prep brief ready for [Title]" with link to view |

Events use the same WebSocket infrastructure as all vault events (topic: `vault:{vault_id}`).

---

## MeiliSearch Integration

Meeting intelligence is indexed in MeiliSearch for cross-module search:

**New MeiliSearch index: `meetings`**

| Field | Source | Searchable | Filterable |
|---|---|---|---|
| `id` | `meetings.id` | No | Yes |
| `title` | `meetings.title` | Yes (weight: highest) | No |
| `summary_text` | `meeting_intelligence.summary.one_liner` | Yes (weight: high) | No |
| `summary_bullets` | `meeting_intelligence.summary.bullets` joined | Yes (weight: medium) | No |
| `topics` | `meeting_intelligence.topics[].name` joined | Yes (weight: high) | Yes |
| `participants` | `meeting_participants[].display_name` joined | Yes (weight: medium) | Yes |
| `vault_name` | `vaults.name` | Yes (weight: medium) | Yes |
| `module` | Vault's module | No | Yes |
| `date` | `meetings.scheduled_start` | No | Yes |
| `status` | `meetings.status` | No | Yes |
| `workspace_id` | `meetings.workspace_id` | No | Yes |

**Badge:** Results from the meetings index display a "Meetings" badge in search results.

**Sync pipeline:** Same as existing MeiliSearch sync -- `pg_notify` on meetings/meeting_intelligence INSERT/UPDATE triggers a BullMQ job that pushes the denormalized document to MeiliSearch.

---

## Triptych Integration

### Control Panel: Meetings Tab

A new "Meetings" tab is added to the Control panel (alongside Lifecycle, Approvals, Audit Trail, and Otto AI):

```
+------------------------------------------+
| Control Panel                            |
|   Henderson MSA                          |
|   [Start Meeting]  [Schedule Meeting]    |
|------------------------------------------|
|   Lifecycle | Approvals | Meetings | ... |
|------------------------------------------|
|                                          |
|   UPCOMING                               |
|   Mar 10, 2:00 PM — Q2 Planning         |
|     Jack Burke, Billy Chen               |
|     [View Prep Brief] [Join]             |
|                                          |
|   PAST MEETINGS                          |
|   Mar 1, 2:00 PM — Marketing Review     |
|     Billy Chen, Dave Ruiz                |
|     30 min — 2 action items              |
|     [View Summary] [View Transcript]     |
|                                          |
|   Feb 25, 11:00 AM — Contract Review    |
|     Ana Chen, Jack Burke                 |
|     45 min — 4 action items              |
|     [View Summary] [View Transcript]     |
|                                          |
+------------------------------------------+
```

### Orchestrate Panel: Prep Brief & Transcript Viewer

When a user clicks "View Prep Brief" or "View Transcript," the content loads in the Orchestrate panel (center panel of the triptych):

**Prep Brief View:**
- Attendee profiles with interaction history
- Related past meetings with summaries
- Active vaults and open tasks
- Topic threads with cross-meeting context

**Transcript View:**
- Full transcript with speaker labels
- Timestamp navigation (click a time to jump)
- Action items highlighted inline
- Topics tagged as colored pills
- Search within transcript

---

## Feature Control Plane Flags

| Flag | Default | Description | Impact Description |
|---|---|---|---|
| `ENABLE_GOOGLE_MEET_INTEGRATION` | `false` | Google Meet link generation and Calendar API integration | When on: "Start Meeting" buttons appear on vault pages. When off: no meeting UI visible. |
| `ENABLE_MEETING_BOT` | `false` | Vexa meeting bot for real-time transcription | When on: Vexa bot joins meetings as a participant. When off: only Gemini notes available. Requires Vexa Docker container running. |
| `ENABLE_MEETING_TRANSCRIPTION` | `false` | Post-meeting transcript retrieval and storage | When on: transcripts are fetched from Gemini/Vexa after meetings end. When off: meetings are logged but no transcript processing occurs. |
| `ENABLE_AI_MEETING_SUMMARY` | `false` | Otto AI generates meeting summaries | When on: summaries appear in Signal feed and Meetings tab. When off: raw transcript available but no AI processing. Requires ENABLE_MEETING_TRANSCRIPTION. |
| `ENABLE_ACTION_ITEM_EXTRACTION` | `false` | Otto extracts action items and creates tasks | When on: action items auto-populate in task system. When off: no automatic task creation from meetings. Requires ENABLE_AI_MEETING_SUMMARY. |
| `ENABLE_PREP_BRIEFS` | `false` | Pre-meeting context brief generation | When on: prep briefs auto-generate for scheduled meetings. When off: no pre-meeting context surfaced. |
| `ENABLE_CONVERSATION_THREADING` | `false` | Cross-meeting topic tracking and contact memory | When on: topics are extracted and threaded across meetings. When off: each meeting is isolated (no memory). |

**Dependency chain:**

```
ENABLE_GOOGLE_MEET_INTEGRATION
    |
    +-> ENABLE_MEETING_TRANSCRIPTION
            |
            +-> ENABLE_AI_MEETING_SUMMARY
            |       |
            |       +-> ENABLE_ACTION_ITEM_EXTRACTION
            |
            +-> ENABLE_CONVERSATION_THREADING
                    |
                    +-> ENABLE_PREP_BRIEFS

ENABLE_MEETING_BOT (independent -- can be enabled alongside or instead of Gemini)
```

### Calibration Parameters

| Key | Default | Range | Description |
|---|---|---|---|
| `meetings.gemini_poll_delay_seconds` | 60 | 10-300 | How long to wait after meeting ends before polling for Gemini notes |
| `meetings.gemini_poll_retries` | 3 | 1-10 | Number of retry attempts to find Gemini notes |
| `meetings.topic_similarity_threshold` | 0.85 | 0.5-1.0 | Minimum similarity for linking a new topic to an existing thread |
| `meetings.stale_thread_days` | 30 | 7-90 | Days without mention before a topic thread becomes "stale" |
| `meetings.prep_brief_advance_minutes` | 15 | 5-60 | Minutes before meeting to regenerate prep brief and send notification |
| `meetings.max_transcript_length` | 100000 | 10000-500000 | Maximum characters stored in raw_transcript |
| `meetings.action_item_confidence_floor` | 0.7 | 0.3-1.0 | Minimum Otto confidence to auto-create a task from an action item |

### Health Monitoring

| Metric | Alert Threshold | Description |
|---|---|---|
| Google OAuth token health | Token refresh fails 3x | Google Workspace connection lost |
| Gemini notes retrieval rate | < 70% of meetings with notes | Notes not generating (plan issue?) |
| Vexa bot connection | Disconnected > 60s | Bot cannot join meetings |
| Post-processing success rate | < 90% in 24h window | Otto processing pipeline failing |
| Prep brief generation time | > 30 seconds | Brief generation too slow |
| Action item extraction accuracy | > 30% human-override rate | Otto extracting unreliable items |

---

## Security Considerations

### Encryption

- **Transcripts are PII-rich.** The `raw_transcript` column in `meeting_intelligence` is encrypted at rest using the DEK/KEK envelope encryption pattern defined in [Security/data-cipher-model.md](../Security/data-cipher-model.md).
- **Prep briefs contain aggregated PII.** The `prep_brief` JSONB column is also encrypted at rest.
- **In transit:** All API responses containing transcript or intelligence data are served over TLS only.

### PII Redaction for LLM

Per security principle 9 ("PII never reaches LLM providers unredacted"):
- Raw transcripts undergo PII redaction before being sent to Otto (LiteLLM proxy)
- Redaction categories: phone numbers, email addresses, SSN, credit card numbers, bank accounts, passport numbers
- Names and company names are NOT redacted (required for functionality)
- Redaction is reversible via a local redaction map (never sent to the LLM)
- The redaction map is stored encrypted alongside the transcript

### Meeting Recordings

**Airlock does NOT store meeting recordings.** Google handles recording storage in Google Drive. Airlock stores only the Google Drive URL reference (`recording_url` is NOT stored in Airlock's database -- recordings are accessed via Google Drive API on demand).

Rationale: recordings are large binary files that create storage cost, encryption complexity, and compliance risk. Google's infrastructure handles retention, access control, and geographic compliance for recordings. Airlock focuses on the text intelligence layer.

### Vexa Bot Transcript Storage

When Vexa is enabled, transcripts are processed in-memory on the Vexa container and transmitted to the Airlock API over the internal Docker network (encrypted in transit). The Vexa container does not persist transcripts to disk after processing. The only persistent copy lives in Airlock's `meeting_intelligence` table (encrypted at rest).

### Access Control

| Resource | Who Can Access |
|---|---|
| Meeting record | Meeting participants + vault members + workspace admins |
| Raw transcript | Meeting participants only (not vault members who did not attend) |
| AI summary | Meeting participants + vault members |
| Action items | Meeting participants + vault members + task assignees |
| Prep brief | Meeting participants |
| Conversation history | Users with `view_contacts` permission |

**Audit logging:** All access to transcript data is logged to the append-only audit trail:

```json
{
  "event_type": "meeting.transcript_accessed",
  "actor_id": "mem_123",
  "meeting_id": "mtg_abc",
  "accessed_at": "2026-03-05T10:30:00Z",
  "ip_address": "10.0.0.1"
}
```

### Consent and Compliance

| Requirement | Implementation |
|---|---|
| **Recording consent** | Google Meet enforces recording disclosure (visual indicator + announcement). Airlock does not bypass this. |
| **Transcription notice** | When Vexa bot joins, its participant name ("Airlock Assistant") serves as notice. Organizations configure the name via Admin Settings. |
| **GDPR right to erasure** | Deleting a contact triggers crypto-shredding of their participation data across all meeting_participants records. Transcripts containing their name are flagged for re-processing (redact their contributions). |
| **Data retention** | Meeting intelligence follows workspace-level retention policy. Expired records are crypto-shredded. |
| **Cross-border data** | Transcripts encrypted at rest in Airlock's database. Google handles geographic compliance for recordings in Drive. |

---

## Implementation Phases

### Phase 1: Google Meet Link Generation + Calendar Integration

**Goal:** Users can schedule and join meetings from Airlock. No intelligence yet.

**Deliverables:**
- Google OAuth flow for Calendar API scope
- `POST /api/meetings` endpoint creates Google Calendar event with Meet link
- "Start Meeting" and "Schedule Meeting" buttons in Control panel
- Meeting state machine (scheduled -> active -> ended)
- Calendar module renders scheduled meetings as Source 4 events (cyan)
- Meetings tab in Control panel shows upcoming and past meetings
- Feature flag: `ENABLE_GOOGLE_MEET_INTEGRATION`

**Database:** `meetings` + `meeting_participants` tables

**Effort estimate:** 2-3 weeks

### Phase 2: Gemini Notes Ingestion + Otto Summarization

**Goal:** After meetings end, Airlock retrieves Gemini notes and Otto generates summaries and action items.

**Deliverables:**
- Google OAuth flow for Drive + Docs API scopes
- BullMQ job: `meeting.post_process` pipeline
- Gemini notes retrieval (Drive API polling + Docs API parsing)
- Otto meeting summarization (PydanticAI tools: `summarize_meeting`, `extract_action_items`)
- PII redaction pipeline for transcripts
- Signal feed events: `meeting.summary`, `meeting.action_items`
- Transcript viewer in Orchestrate panel
- Action item -> task creation flow
- MeiliSearch meetings index
- Feature flags: `ENABLE_MEETING_TRANSCRIPTION`, `ENABLE_AI_MEETING_SUMMARY`, `ENABLE_ACTION_ITEM_EXTRACTION`

**Database:** `meeting_intelligence` table

**Effort estimate:** 3-4 weeks

### Phase 3: Vexa Bot Integration for Real-Time Transcription

**Goal:** Self-hosted meeting bot for organizations without Business Standard+ or needing Teams/Zoom support.

**Deliverables:**
- Vexa Docker container added to docker-compose
- Webhook endpoint: `/webhooks/vexa` for real-time transcript delivery
- Vexa bot auto-joins meetings when `ENABLE_MEETING_BOT` is on
- Transcript merge logic (Gemini + Vexa deduplication)
- Vexa health monitoring in Feature Control Plane
- Feature flag: `ENABLE_MEETING_BOT`

**Effort estimate:** 2 weeks

### Phase 4: Conversation Threading + Prep Briefs

**Goal:** Cross-meeting context memory and pre-meeting intelligence.

**Deliverables:**
- Topic extraction from meeting transcripts (Otto tool: `extract_topics`)
- `conversation_threads` table + junction tables
- Topic matching algorithm (pg_trgm similarity)
- Contact-level conversation history endpoint
- Prep brief generation pipeline (BullMQ job: `meeting.generate_prep_brief`)
- Prep brief UI in Calendar event detail and Control panel
- 15-minute pre-meeting notification with prep brief link
- Topic search via MeiliSearch
- Feature flags: `ENABLE_CONVERSATION_THREADING`, `ENABLE_PREP_BRIEFS`

**Database:** `conversation_threads` + `meeting_thread_links` + `thread_vault_links` tables

**Effort estimate:** 3-4 weeks

### Phase 5 (Future): Meet Add-ons SDK + Semantic Search

**Goal:** Airlock sidebar inside Google Meet + vector-based topic matching.

**Deliverables:**
- Meet Add-ons SDK integration (side panel with vault context)
- pgvector embedding column on `conversation_threads`
- Semantic topic matching (replaces trigram similarity)
- Real-time topic suggestions during meeting (via Vexa + live Otto processing)

**Not scheduled for MVP.**

---

## Workflow Engine Integration

Meeting events can trigger Workflow Engine automations:

| Event | Workflow Use Case |
|---|---|
| `meeting.scheduled` | Send prep brief to all participants via email |
| `meeting.ended` | Trigger post-processing pipeline (already handled by BullMQ, but workflows can add custom steps) |
| `meeting.summary` | If sentiment is negative, create an escalation task and notify manager |
| `meeting.action_items` | If action items exceed 5, create a follow-up meeting suggestion |
| `meeting.prep_brief_ready` | Push brief to Slack/Teams channel (future integration) |

These events are published to the cross-module event bus (BullMQ + Trigger.dev) and can be used as workflow triggers in the visual builder.

---

## Cmd+K Commands

New commands available in the command palette:

| Command | Action |
|---|---|
| `/schedule-meeting` | Open meeting scheduling form (prompts for vault, participants, time) |
| `/start-meeting` | Instant meeting creation linked to current vault (if in vault context) |
| `/meeting-history` | Show recent meetings across all vaults |
| `/meeting-history {vault}` | Show meetings for a specific vault |
| `/prep-brief {meeting}` | Open prep brief for a scheduled meeting |

---

## Performance Considerations

| Concern | Mitigation |
|---|---|
| **Gemini notes polling latency** | Configurable delay + retries. BullMQ exponential backoff. Does not block user. |
| **Otto processing time** | Async BullMQ job. User sees "Processing..." state. Target: < 60 seconds for a 30-minute meeting transcript. |
| **Prep brief generation** | Cached in `meeting_intelligence.prep_brief`. Regenerated on-demand or 15 min before meeting. Target: < 10 seconds. |
| **Contact conversation history** | Paginated API. Denormalized summary data avoids expensive JOINs at read time. |
| **Topic matching at scale** | pg_trgm index on `conversation_threads.topic_normalized`. Sub-50ms for workspaces with < 10K topics. |
| **MeiliSearch sync** | Same pipeline as existing search indexing. < 2 second sync latency. |
| **Transcript storage size** | `max_transcript_length` calibration parameter (default 100K chars). Older transcripts can be archived. |

---

## Related Specs

- **[CRM Communications](../CRM/crm-comms-brainstorm.md)** -- Google Meet SDK section (original brainstorm). Meeting Intelligence supersedes and expands that section.
- **[Calendar](../Calendar/overview.md)** -- Meeting events as Source 4. Prep briefs surface in Calendar event detail.
- **[Messenger](../Messenger/overview.md)** -- Video call launch from conversation header. Meeting summaries post to vault threads.
- **[AI Agent (Otto)](../AIAgent/overview.md)** -- Otto processes transcripts. New PydanticAI tools: `summarize_meeting`, `extract_action_items`, `extract_topics`, `analyze_sentiment`, `match_contacts`.
- **[Search](../Search/overview.md)** -- MeiliSearch `meetings` index for cross-module search.
- **[Security](../Security/overview.md)** -- Cipher model for transcript encryption. PII redaction before LLM. Audit logging requirements.
- **[Feature Control Plane](../FeatureControlPlane/overview.md)** -- 7 feature flags + 7 calibration parameters + 6 health metrics.
- **[Universal Task System](../TaskSystem/overview.md)** -- Action items create tasks via the master task table.
- **[Workflow Engine](../WorkflowEngine/overview.md)** -- Meeting events as workflow triggers.
- **[Shell](../Shell/overview.md)** -- Triptych integration (Control panel Meetings tab, Orchestrate panel transcript viewer).
- **[Roles & Permissions](../Roles/overview.md)** -- Access control for transcripts and intelligence data.
