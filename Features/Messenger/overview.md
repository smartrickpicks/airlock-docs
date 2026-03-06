# Messenger — Design Spec

> **Status:** SPECCED — Real-time team communication available as a dock tool, not a module.

> **Key Insight:** Messenger is a cross-cutting communication layer, not a top-level module. It lives in the right-side tool dock and opens as a slide-over drawer. Vault threads bridge messenger conversations with the vault's Signal panel, so chat context is always co-located with work context.

> **Tech:** WebSocket + Redis Pub/Sub (shared infrastructure with [RealTime](../RealTime/overview.md)). Novu for external notification delivery (future).

---

## Summary

Messenger provides real-time team communication within Airlock. It is a dock tool (not a module) that overlays as a 340px slide-over drawer from the right side. Conversations can be scoped to the current module context or viewed globally. Every vault automatically gets a chat thread, and messages flow bidirectionally between the Messenger drawer and the vault's Signal panel.

---

## Position in the Application

| Property | Value |
|----------|-------|
| Access point | Chat icon in the right-side tool dock |
| Drawer type | Slide-over (340px, absolute positioned, overlays content) |
| Module bar entry | **None** — Messenger is NOT a module |
| Availability | Every screen in the application |
| Unread indicator | Red dot with numeric count on the dock icon |

### Right-Side Tool Dock Context

The tool dock contains six icons. Messenger (Chat) is one of two tools that use slide-over drawer behavior:

| Icon | Tool | Panel Behavior | Width |
|------|------|---------------|-------|
| Tasks | Task dock panel | Push (shrinks content) | 320px |
| CRM | CRM dock panel | Push (shrinks content) | 320px |
| Calendar | Calendar dock panel | Push (shrinks content) | 320px |
| Docs | Documents dock panel | Push (shrinks content) | 320px |
| **Chat** | **Messenger** | **Slide-over (overlays content)** | **340px** |
| Otto | AI Assistant | Slide-over (overlays content) | 340px |

Chat and Otto overlay the main content rather than shrinking it, preserving the user's working layout while providing quick access to communication and AI assistance.

---

## Conversation Types

Messenger supports four conversation types, each serving a distinct communication pattern:

| Type | Creation | Scope | Icon | Example |
|------|----------|-------|------|---------|
| **Vault Thread** | Auto-created when a vault is created | Messages tied to a specific vault | Gate dot (colored by current chamber) | "Henderson MSA" thread |
| **Direct Message** | User-initiated | Private 1-on-1 between two users | User avatar + online indicator | DM with Ana Chen |
| **Team Conversation** | User-created, joinable by workspace members | Named group chat (like Slack channels) | Group icon | "#design-reviews" |
| **Module Conversation** | Auto-created per module | Module-wide discussion not tied to a specific vault | Module icon | "#contracts-general" |

### Vault Threads

- Auto-created when any vault is created across any module
- Thread name matches the vault name
- Messages appear BOTH in the Messenger conversation AND as `message.sent` events in the vault's Signal panel (bidirectional sync)
- Thread shows the vault's current chamber as a colored gate dot
- Thread persists for the lifetime of the vault

### Direct Messages

- Initiated by any user by selecting another workspace member
- Private to the two participants
- Cannot be joined by others (create a Team Conversation for group chat)
- Always visible regardless of scope toggle setting

### Team Conversations

- Created by any user with a chosen name (prefixed with `#` in display)
- Any workspace member can join or be invited
- Persist until explicitly archived by the creator or a workspace admin
- Support pinned messages (future)

### Module Conversations

- One auto-created per module: `#contracts-general`, `#crm-general`, `#tasks-general`, `#calendar-general`, `#documents-general`
- For module-wide discussions that are not tied to a specific vault
- All workspace members with access to the module are auto-joined
- Cannot be deleted or renamed

---

## Scope Toggle Behavior

Every dock panel has a scope toggle. For Messenger, this controls which conversations are visible in the list.

### Toggle UI

A pill bar at the top of the drawer, below the search bar:

```
+--------------------------------------+
|  [ Contracts  |  All ]               |
+--------------------------------------+
```

The left pill shows the name of the currently active module and updates dynamically as the user navigates between modules.

### Module Scope (Default)

When module scope is active, the conversation list shows:

- Vault threads for vaults belonging to the active module
- The active module's module conversation (e.g., `#contracts-general`)
- All DMs (DMs are always shown regardless of scope)
- Team conversations tagged to the active module (future: conversation tagging)

### Global Scope

When global scope is active, the conversation list shows:

- ALL vault threads across all modules
- ALL module conversations
- ALL DMs
- ALL team conversations

### Scope Memory

The toggle state persists per session. Switching modules in the left module bar updates the module scope label but does not reset the toggle position. If the user chose "All," it stays on "All" until they switch back.

---

## Conversation List UI

The default view when the Messenger drawer opens is the conversation list.

### Layout

```
+--------------------------------------+
|  [<search icon>  Search...        ]  |
|  [ Contracts  |  All ]               |
+--------------------------------------+
|  [gate dot] Henderson MSA            |
|  Ana: Field extraction looks go...   |
|  2m ago                          (3) |
|--------------------------------------|
|  [avatar] Ana Chen           [green] |
|  Hey, can you check the SLA...       |
|  15m ago                             |
|--------------------------------------|
|  [group] #design-reviews             |
|  Marco: Pushed the updated wi...     |
|  1h ago                          (1) |
|--------------------------------------|
|  [module] #contracts-general         |
|  System: 5 new vaults created t...   |
|  3h ago                              |
+--------------------------------------+
```

### Sorting

Conversations are sorted by last message timestamp, most recent first. Pinned conversations (future) would appear above the sorted list.

### Conversation List Item Anatomy

| Element | Description |
|---------|-------------|
| Avatar/Icon | Vault gate dot, user avatar, group icon, or module icon depending on type |
| Conversation name | Vault name, user display name, `#team-name`, or `#module-general` |
| Last message preview | Author name + truncated message content (single line, ellipsis) |
| Timestamp | Relative time ("2m ago", "1h ago", "Yesterday") |
| Unread badge | Numeric count in a pill, only shown when unread count > 0 |
| Online indicator | Green dot on DM avatars when the other user is online |
| Gate dot color | For vault threads, colored by the vault's current chamber |

### Gate Dot Color Mapping

| Chamber | Dot Color |
|---------|-----------|
| Discover | Blue |
| Build | Amber |
| Review | Purple |
| Ship | Green |

### Search

The search bar at the top filters conversations by name. Typing filters the list in real time (client-side filter on loaded conversations). Full-text message search is a future feature.

---

## Chat Interface

Clicking a conversation in the list opens the chat view within the same 340px drawer.

### Layout

```
+--------------------------------------+
|  [<] Henderson MSA        [gate dot] |
+--------------------------------------+
|                                      |
|  [system] Thread created             |
|  Mar 1, 2026                         |
|                                      |
|  [avatar] Ana Chen          10:30 AM |
|  Extraction completed. 42 fields     |
|  found, 5 need review.              |
|                                      |
|  [avatar] Marco Li          10:45 AM |
|  Looking at the flagged fields now.  |
|                                      |
|  [avatar] Ana Chen          10:47 AM |
|  Great. The SLA timer started at     |
|  10:30 — we have until 2:30 PM.     |
|                                      |
|  Ana is typing...                    |
+--------------------------------------+
|  [+] [  Type a message...   ] [send] |
+--------------------------------------+
```

### Header

- Back button (`<`) returns to the conversation list
- Conversation name (truncated if needed)
- Context indicator: gate dot for vault threads, online dot for DMs, member count for team conversations

### Message List

- Reverse-chronological scroll: newest messages at the bottom
- Auto-scrolls to bottom on new message (unless user has scrolled up to read history)
- "New messages" pill appears when the user is scrolled up and new messages arrive
- Infinite scroll upward to load older messages (paginated, 50 messages per page)
- Date separators between messages from different days

### Message Types

| Type | Rendering |
|------|-----------|
| **Text** | Author avatar, author name, timestamp, message content. Supports basic markdown (bold, italic, code, links). |
| **File attachment** | Preview card with file icon, file name, file size. Click to open in document viewer or download. |
| **System message** | Italic text, muted color, no avatar. Used for thread creation, member joins/leaves, vault lifecycle events. |

### Message Anatomy

| Element | Description |
|---------|-------------|
| Author avatar | 24px avatar image, circular |
| Author name | Display name, bold, same line as timestamp |
| Timestamp | HH:MM format for today, "Yesterday HH:MM" for yesterday, "MMM D" for older |
| Content | Message body, wraps within the drawer width. Max-width prevents single long words from breaking layout. |

### Composer

| Element | Description |
|---------|-------------|
| Attach button (`+`) | Opens file picker. Selected file shows as preview chip above the input. |
| Text input | Auto-expanding textarea (1-5 lines). Placeholder: "Type a message..." |
| Emoji picker button | Future: opens emoji picker popover |
| Send button | Enabled only when input is non-empty. Sends on click. |
| Keyboard shortcuts | `Enter` to send, `Shift+Enter` for new line |

---

## Real-Time Delivery

Messenger shares the WebSocket infrastructure defined in [RealTime/overview.md](../RealTime/overview.md).

### Subscriptions

On WebSocket connect, the client subscribes to conversation topics for all conversations the user is a member of:

```json
{ "type": "subscribe", "topic": "conversation:{conversation_id}" }
```

New messages are pushed as events:

```json
{
  "type": "event",
  "topic": "conversation:conv_abc123",
  "event": {
    "id": "msg_xyz789",
    "event_type": "message.sent",
    "payload": {
      "author_id": "user_456",
      "author_name": "Ana Chen",
      "content": "Extraction completed. 42 fields found.",
      "message_type": "text"
    },
    "created_at": "2026-03-04T10:30:00Z"
  }
}
```

### Typing Indicators

- Client sends a `typing` frame when the user starts typing:
  ```json
  { "type": "typing", "conversation_id": "conv_abc123" }
  ```
- Server broadcasts to other conversation members
- Indicator shows for 3 seconds after the last keystroke
- Client re-sends the `typing` frame on each keystroke (throttled to once per 2 seconds)
- Display: "Ana is typing..." or "Ana and Marco are typing..." (max 2 names, then "3 people are typing...")

### Read Receipts

- Per-conversation `last_read_at` timestamp stored in `conversation_members`
- Updated when user opens a conversation (sets `last_read_at = now()`)
- Also updated on scroll-to-bottom when the conversation is already open
- Unread count = messages with `created_at > last_read_at` for that conversation
- No per-message read receipts in v1 (only per-conversation)

### Online Presence

- Users broadcast presence via WebSocket heartbeat (every 30 seconds)
- Server maintains an in-memory presence map: `user_id -> last_heartbeat`
- A user is "online" if their last heartbeat is within 60 seconds
- Presence changes broadcast to DM partners and conversation members
- Presence is best-effort, not guaranteed (network delays may cause brief inaccuracy)

---

## Vault Thread Integration

This is the most important architectural feature of Messenger: vault threads are bidirectionally synced with the vault's Signal panel.

### Bidirectional Message Flow

```
Messenger Drawer                     Vault Signal Panel
+------------------+                 +------------------+
| Henderson MSA    |                 | Signal Feed      |
|                  |    sync         |                  |
| [msg] Ana: ...   | <============> | [evt] message    |
| [msg] Marco: ... |                | [evt] message    |
| [msg] System:... |                | [evt] extraction |
+------------------+                 +------------------+
```

### When a User Sends a Message in the Messenger Thread

1. Message is inserted into the `messages` table (conversation_id = vault's conversation)
2. A `message.sent` event is inserted into the `channel_events` table (channel_id = vault_id)
3. The message appears instantly in the Messenger drawer via WebSocket (`conversation:{id}` topic)
4. The event appears instantly in the Signal panel via WebSocket (`vault:{vault_id}` topic)

### When a User Sends a Message From the Signal Panel

1. The same dual-write happens: `messages` row + `channel_events` row
2. Both surfaces update in real time

### What Appears Where

| Source | Messenger Thread | Signal Panel |
|--------|-----------------|--------------|
| Messages sent via Messenger | Yes | Yes (as `message.sent` event) |
| Messages sent via Signal panel | Yes | Yes |
| System events (extraction, patch, gate advance) | No | Yes |
| File attachments via Messenger | Yes | Yes (as `file.shared` event) |

System events from the vault lifecycle (extractions, patches, gate advances) appear ONLY in the Signal panel, not in the Messenger thread. The Messenger thread contains only human-authored messages and file shares. This keeps the chat readable without being cluttered by automated system events.

---

## Notifications

### Unread Badge on Dock Icon

- The Chat dock icon shows a red dot with numeric count
- Count = total unread messages across all non-muted conversations
- Badge updates in real time via WebSocket
- Badge clears when all conversations have been read

### Notification Priority

| Event | Priority | Badge | Push/Email (future) |
|-------|----------|-------|---------------------|
| DM received | High | Yes | Yes |
| @mention in any conversation | High | Yes | Yes |
| Message in vault thread (unmuted) | Normal | Yes | No |
| Message in team conversation (unmuted) | Normal | Yes | No |
| Message in module conversation (unmuted) | Low | Yes | No |
| Message in muted conversation | None | No | No |

### Mute Controls

- Per-conversation mute toggle available from the conversation list (long-press or context menu)
- Muting suppresses:
  - Badge count contribution
  - Push/email notifications (future)
- Muting does NOT suppress:
  - Message delivery (messages still appear in the conversation)
  - @mention highlighting within the conversation

### Novu Integration (Future)

- DMs and @mentions trigger Novu workflows for push and email delivery
- Email delivery: batched digest (e.g., "You have 3 unread DMs") after 5 minutes of inactivity
- Push delivery: immediate for DMs, batched for @mentions
- Users configure notification preferences per conversation type in Personal Settings

---

## Data Model

### conversations

| Column | Type | Description |
|--------|------|-------------|
| `id` | UUID | Primary key |
| `type` | ENUM | `vault_thread`, `dm`, `team`, `module` |
| `name` | VARCHAR(255) | Display name. NULL for DMs (derived from participants). |
| `module_id` | VARCHAR(50) | Nullable. Set for vault threads and module conversations. |
| `vault_id` | UUID | Nullable. Set only for vault threads. FK to vaults table. |
| `created_by` | UUID | User who created the conversation. System for auto-created. |
| `created_at` | TIMESTAMPTZ | Creation timestamp |
| `archived_at` | TIMESTAMPTZ | Nullable. Set when conversation is archived. |

**Indexes:** `(type)`, `(module_id)`, `(vault_id)` unique for vault threads.

### conversation_members

| Column | Type | Description |
|--------|------|-------------|
| `conversation_id` | UUID | FK to conversations |
| `user_id` | UUID | FK to users |
| `last_read_at` | TIMESTAMPTZ | Last time user read this conversation |
| `muted` | BOOLEAN | Default false. Suppresses badge and push. |
| `joined_at` | TIMESTAMPTZ | When the user joined the conversation |

**Primary key:** `(conversation_id, user_id)`

### messages

| Column | Type | Description |
|--------|------|-------------|
| `id` | UUID | Primary key |
| `conversation_id` | UUID | FK to conversations |
| `author_id` | UUID | FK to users. NULL for system messages. |
| `content` | TEXT | Message body (plain text with basic markdown) |
| `message_type` | ENUM | `text`, `file`, `system` |
| `file_url` | VARCHAR(500) | Nullable. S3/storage URL for file attachments. |
| `file_name` | VARCHAR(255) | Nullable. Original file name for display. |
| `file_size` | INTEGER | Nullable. File size in bytes. |
| `parent_message_id` | UUID | Nullable. FK to messages. For threaded replies (future). |
| `created_at` | TIMESTAMPTZ | Message timestamp. Append-only: no edits or deletes in v1. |

**Indexes:** `(conversation_id, created_at)` for paginated message loading.

### Entity Relationship

```
conversations 1---* conversation_members *---1 users
conversations 1---* messages *---1 users (author)
conversations *---1 vaults (nullable, vault_thread only)
messages *---1 messages (nullable, parent for threading)
```

### Important Constraints

- Messages are **append-only** in v1. No edit or delete operations.
- A vault can have exactly one vault_thread conversation (unique constraint on `vault_id`).
- DM conversations are unique per user pair (enforced at application level).
- Module conversations are unique per module (enforced at application level).

---

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/conversations` | List conversations for the current user. Supports `?scope=module&module_id=contracts` or `?scope=global`. |
| `GET` | `/api/conversations/{id}` | Get conversation details + member list |
| `POST` | `/api/conversations` | Create a team conversation or initiate a DM |
| `GET` | `/api/conversations/{id}/messages` | Paginated message list. `?before={timestamp}&limit=50` |
| `POST` | `/api/conversations/{id}/messages` | Send a message. Body: `{ content, message_type, file_url? }` |
| `PUT` | `/api/conversations/{id}/read` | Mark conversation as read (updates `last_read_at`) |
| `PUT` | `/api/conversations/{id}/mute` | Toggle mute for the current user |
| `POST` | `/api/conversations/{id}/members` | Add a member to a team conversation |
| `DELETE` | `/api/conversations/{id}/members/{user_id}` | Remove a member (or leave) |

All endpoints require authentication. Conversation access is scoped to members only (except module conversations, which are accessible to all module members).

---

## Performance Considerations

| Concern | Mitigation |
|---------|-----------|
| **Message volume** | Paginated loading (50 per page). Only load messages for the open conversation. |
| **WebSocket subscriptions** | Subscribe to all user conversations on connect. Unsubscribe is not needed per-conversation (server filters by membership). |
| **Typing indicator spam** | Client-side throttle: send typing frame at most once per 2 seconds. Server-side: drop duplicate typing events within 1 second window. |
| **Presence overhead** | Presence map is in-memory only (Redis hash). No database writes for heartbeats. |
| **Large file uploads** | Files upload directly to S3 (presigned URL). Message contains only the reference URL. |
| **Conversation list staleness** | Conversation list metadata (last message, unread count) updated via WebSocket events, not polling. |

---

## Future Enhancements

Features designed into the data model but not built in v1:

| Feature | Description | Data Model Support |
|---------|-------------|--------------------|
| **Threaded replies** | Reply to a specific message within a conversation | `parent_message_id` column in messages |
| **Message editing** | Edit sent messages within a time window | Add `edited_at` column, remove append-only constraint |
| **Message deletion** | Soft-delete messages | Add `deleted_at` column |
| **Reactions** | Emoji reactions on messages | New `message_reactions` table |
| **Pinned messages** | Pin important messages to conversation top | New `pinned_messages` table |
| **Conversation tagging** | Tag team conversations to modules for scope filtering | New `conversation_tags` table |
| **Full-text search** | Search message content across all conversations | PostgreSQL full-text index on `messages.content` |
| **Rich embeds** | Link previews, code blocks with syntax highlighting | Extend message rendering, no schema change |

---

## Related Specs

- **[RealTime](../RealTime/overview.md)** — WebSocket + Redis Pub/Sub architecture shared by Messenger
- **[Notifications](../Notifications/overview.md)** — Badge system, Novu integration, notification priority levels
- **[Shell](../Shell/overview.md)** — Master layout including the right-side tool dock where Messenger lives
- **[Shell / Triptych](../Shell/triptych.md)** — Signal panel where vault thread messages also appear
- **[EventBus](../EventBus/overview.md)** — Server-side event routing that creates vault lifecycle events
- **[Video Call Integration](./video-call-integration.md)** — Google Meet calls from Messenger, call state, post-call summaries
- **[Meeting Intelligence](../MeetingIntelligence/overview.md)** — Transcription engine, action item extraction, conversation memory
