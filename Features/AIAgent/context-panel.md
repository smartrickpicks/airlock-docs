# Context Panel

The Context Panel is the AI Agent's dedicated conversational interface, rendered as a tab in the **Control panel** (right side of the triptych).

---

## Placement

| Attribute | Value |
|---|---|
| Panel | Control panel (right column of the triptych) |
| Tab siblings | Lifecycle, Approvals, Audit Trail |
| Activation | Click tab directly, or auto-opened by `@AI` mentions in Signal feed |

---

## Layout

The panel uses a standard chat layout within the Control panel tab.

```
+-------------------------------+
|  Tab bar: [Lifecycle] [Approvals] [Audit Trail] [AI Context]  |
+-------------------------------+
|                               |
|  Scrollable message area      |
|                               |
|  [AI] Response with           |
|       enrichment indicator    |
|                               |
|  [You] Follow-up question     |
|                               |
|  [AI] Response with           |
|       enrichment indicator    |
|                               |
+-------------------------------+
|  Chat input                   |
|  [Send]                       |
+-------------------------------+
```

### Message Area

- Scrollable, reverse-chronological (newest at bottom).
- Messages are visually distinguished by sender (analyst vs. AI).
- AI messages include an **"enrichment used" indicator** -- a compact badge or expandable row showing which of the 9 enrichment sources were available for that response.

### Chat Input

- Text input at the bottom of the panel.
- Send button (or Enter key).
- No file upload -- the agent reads from the contract's existing corpus.

---

## Conversation Model

| Attribute | Detail |
|---|---|
| Turn style | Multi-turn conversation |
| Session persistence | Sessions stored in `otto_sessions`, messages in `otto_messages` |
| Session scope | One session per contract per analyst (resumes on revisit) |
| Context window | Full session history sent with each agent call (plus enrichment) |

---

## Dual Presence: Signal Feed Integration

The AI Agent has simultaneous presence in both the Context Panel and the Signal feed.

| Behavior | Detail |
|---|---|
| `@AI` in Signal feed | Triggers agent call and auto-opens Context Panel tab |
| AI response rendering | Appears as a **cyan event** in Signal feed AND as a message in Context Panel |
| Analyst reply in Context Panel | Does NOT echo to Signal feed (private working conversation) |
| Analyst `@AI` in Signal feed | Visible to all viewers of the contract channel |

This dual model gives analysts two interaction modes:
- **Signal feed `@AI`**: Collaborative -- visible to the team, appears in the shared timeline.
- **Context Panel chat**: Focused -- private working conversation for deeper analysis.

---

## Enrichment Indicator

Each AI message displays which enrichment sources contributed to the response.

| State | Display |
|---|---|
| Source available and used | Filled icon or green chip |
| Source available but not relevant | Muted icon or gray chip |
| Source unavailable (error/timeout) | Absent from indicator |

The indicator is collapsible -- compact by default (e.g., "6/9 sources"), expandable to show the full list.

---

## Provider Configuration

| Setting | Detail |
|---|---|
| Env variable | `OTTO_AGENT_WEBHOOK_URL` |
| Supported backends | Claude, GPT, Grok, local models |
| Swap mechanism | Change the URL; no code or UI changes required |

The Context Panel is backend-agnostic. It sends the conversation history and enrichment payload to whatever endpoint is configured. The response format is standardized regardless of provider.

---

## Graceful Degradation

When the agent webhook is unavailable:

| Scenario | Behavior |
|---|---|
| Webhook timeout | Returns stub response containing available enrichment context |
| Webhook 5xx | Returns stub response containing available enrichment context |
| Network error | Returns stub response containing available enrichment context |
| Enrichment source failure | Agent call proceeds; failed sources omitted from payload |

The stub response surfaces whatever enrichment data was successfully gathered, so the analyst still receives useful context even when the AI backend is down. The stub is clearly labeled as a fallback (not presented as an AI-generated response).
