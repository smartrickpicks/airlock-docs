# Session Persistence -- Database Schema & Lifecycle

> **Replaces:** `kiwi_chat_sessions` and `kiwi_chat_messages` tables from OrcestrateOS.

---

## Purpose

Otto maintains persistent, multi-turn conversations per vault per user. Sessions resume when an analyst revisits a vault's Context Panel. Messages are stored with full metadata for audit, cost tracking, and enrichment visibility.

---

## Database Schema

### Migration: `030_otto_chat.sql`

```sql
-- Otto AI Agent: Session & Message Persistence
-- Replaces kiwi_chat_sessions and kiwi_chat_messages

CREATE TABLE otto_sessions (
    id TEXT PRIMARY KEY,                          -- prefix: ots_ (e.g., ots_abc123def456)
    vault_id UUID NOT NULL,                       -- references channels(id) via linked_record
    user_id UUID NOT NULL REFERENCES users(id),
    workspace_id UUID NOT NULL REFERENCES workspaces(id),
    model_used TEXT,                               -- litellm model alias (e.g., 'otto-default')
    enrichment_snapshot JSONB DEFAULT '{}',        -- cached enrichment at session start
    tool_calls_count INT DEFAULT 0,                -- total tool invocations in this session
    total_tokens INT DEFAULT 0,                    -- cumulative token usage
    total_cost NUMERIC(10, 6) DEFAULT 0,           -- cumulative cost in USD
    message_count INT DEFAULT 0,                   -- total messages (user + assistant + tool)
    created_at TIMESTAMPTZ DEFAULT now(),
    last_message_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_otto_sessions_vault ON otto_sessions(vault_id);
CREATE INDEX idx_otto_sessions_user ON otto_sessions(user_id);
CREATE INDEX idx_otto_sessions_workspace ON otto_sessions(workspace_id);
CREATE INDEX idx_otto_sessions_last_msg ON otto_sessions(last_message_at DESC);

CREATE TABLE otto_messages (
    id TEXT PRIMARY KEY,                          -- prefix: otm_ (e.g., otm_xyz789ghi012)
    session_id TEXT NOT NULL REFERENCES otto_sessions(id) ON DELETE CASCADE,
    role TEXT NOT NULL,                            -- 'user' | 'assistant' | 'tool'
    content TEXT NOT NULL,                         -- message text or tool result
    tool_calls JSONB,                              -- tool invocations (if role='assistant' and tools were called)
    tool_results JSONB,                            -- tool execution results (if role='tool')
    tool_name TEXT,                                -- specific tool name (if role='tool')
    model TEXT,                                    -- model used for this response (if role='assistant')
    tokens_used INT,                               -- tokens consumed by this message
    cost NUMERIC(10, 6),                           -- cost of this message in USD
    enrichment_sources_used TEXT[],                -- which of the 9 sources were active for this response
    finish_reason TEXT,                            -- 'stop' | 'tool_calls' | 'error' | 'length'
    created_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_otto_messages_session ON otto_messages(session_id, created_at);
CREATE INDEX idx_otto_messages_role ON otto_messages(session_id, role);
```

---

## ID Prefixes

| Table | Prefix | Example | Purpose |
|---|---|---|---|
| `otto_sessions` | `ots_` | `ots_a1b2c3d4e5f6` | Distinguishable in logs and URLs |
| `otto_messages` | `otm_` | `otm_x7y8z9g0h1i2` | Distinguishable from session IDs |

Prefix convention matches OrcestrateOS pattern (`kcs_` / `kcm_` for Kiwi).

---

## Session Lifecycle

### Creation

A new session is created when:

1. An analyst opens the Context Panel for a vault for the first time, **OR**
2. An analyst sends an `@AI` mention in the Signal feed for a vault they haven't chatted with before

```python
async def get_or_create_session(vault_id: str, user_id: str, session_id: str | None) -> OttoSession:
    if session_id:
        session = await load_session(session_id)
        if session and session.user_id == user_id:
            return session

    # Check for existing session for this vault + user
    existing = await find_session(vault_id=vault_id, user_id=user_id)
    if existing:
        return existing

    # Create new session
    return await create_session(
        vault_id=vault_id,
        user_id=user_id,
        workspace_id=workspace_id,
    )
```

### Resumption

Sessions resume automatically when the analyst revisits the Context Panel:

1. Frontend calls `GET /api/v3/vaults/{vault_id}/otto/session` to check for existing session
2. If a session exists, load its message history
3. If enrichment has changed since `enrichment_snapshot`, refresh and update the snapshot
4. Display full conversation history in the Context Panel

### Message Storage

After every streaming response completes:

```python
async def save_messages(session_id: str, user_message: str, agent_result):
    # Save user message
    await insert_message(
        session_id=session_id,
        role="user",
        content=user_message,
    )

    # Save assistant response
    await insert_message(
        session_id=session_id,
        role="assistant",
        content=agent_result.text,
        tool_calls=agent_result.tool_calls,
        model=agent_result.model,
        tokens_used=agent_result.usage.total_tokens,
        cost=agent_result.usage.cost,
        enrichment_sources_used=agent_result.enrichment_sources,
        finish_reason=agent_result.finish_reason,
    )

    # Save tool messages (if any)
    for tool_call in agent_result.tool_calls or []:
        await insert_message(
            session_id=session_id,
            role="tool",
            content=tool_call.result,
            tool_name=tool_call.tool_name,
            tool_results={"args": tool_call.args, "result": tool_call.result},
        )

    # Update session counters
    await update_session_counters(
        session_id=session_id,
        tokens=agent_result.usage.total_tokens,
        cost=agent_result.usage.cost,
        tool_calls=len(agent_result.tool_calls or []),
        messages=2 + len(agent_result.tool_calls or []),  # user + assistant + tools
    )
```

### No Expiry

Sessions persist indefinitely. They can be:

- **Cleared manually** by the analyst ("Clear conversation" button in Context Panel)
- **Cleared by admin** via Admin panel (for compliance or cost reasons)
- **Soft deleted** (marked as cleared, messages retained for audit)

---

## Session-Level Enrichment Caching

### On Session Create

The initial enrichment snapshot is stored in `enrichment_snapshot` JSONB:

```json
{
  "captured_at": "2024-03-04T10:00:00Z",
  "gate_state": { "gate_color": "yellow", "health_score": 0.68 },
  "field_summary": { "pass": 37, "fail": 12, "review": 28, "skip": 5 },
  "extraction_meta": { "page_count": 12, "encoding_type": "utf-8", "mojibake_count": 0 }
}
```

### On Follow-Up Messages

For subsequent messages in the same session:

1. Check if vault state has changed since `enrichment_snapshot.captured_at`
   - Gate transition occurred?
   - Patch applied?
   - Health recalculated?
2. If changed: re-run enrichment, update snapshot, include fresh context
3. If unchanged: reuse cached enrichment (avoids redundant DB queries)

---

## API Endpoints

### Session Management

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/v3/vaults/{vault_id}/otto/session` | Get existing session for current user (returns session + message history) |
| `POST` | `/api/v3/vaults/{vault_id}/otto/chat` | Send message (SSE streaming response) |
| `DELETE` | `/api/v3/vaults/{vault_id}/otto/session` | Clear session (soft delete, preserves audit trail) |

### Session Response

```json
{
  "session": {
    "id": "ots_abc123",
    "vault_id": "vault_henderson",
    "model_used": "otto-default",
    "message_count": 8,
    "total_tokens": 4210,
    "created_at": "2024-03-04T10:00:00Z",
    "last_message_at": "2024-03-04T10:15:00Z"
  },
  "messages": [
    {
      "id": "otm_001",
      "role": "user",
      "content": "What's wrong with this contract?",
      "created_at": "2024-03-04T10:00:15Z"
    },
    {
      "id": "otm_002",
      "role": "assistant",
      "content": "I can see 2 blockers...",
      "model": "claude-sonnet-4-20250514",
      "tokens_used": 203,
      "enrichment_sources_used": ["gate_state", "field_summary", "contract_health", "patch_summary"],
      "created_at": "2024-03-04T10:00:18Z"
    }
  ]
}
```

---

## Cost Tracking

Every assistant message records token usage and cost:

| Field | Source | Purpose |
|---|---|---|
| `tokens_used` | LiteLLM usage response | Per-message token count |
| `cost` | LiteLLM cost tracking | Per-message cost in USD |
| `model` | LiteLLM model routing | Which model was used |

Session-level aggregates (`total_tokens`, `total_cost`) are updated after each message for quick reporting.

### Admin Visibility

The Admin module can query:

```sql
-- Cost per workspace (last 30 days)
SELECT workspace_id, SUM(total_cost) as cost_30d, SUM(total_tokens) as tokens_30d
FROM otto_sessions
WHERE created_at > now() - interval '30 days'
GROUP BY workspace_id;

-- Cost per user (last 30 days)
SELECT user_id, SUM(total_cost) as cost_30d, COUNT(*) as sessions
FROM otto_sessions
WHERE created_at > now() - interval '30 days'
GROUP BY user_id
ORDER BY cost_30d DESC;
```

---

## Audit Trail Integration

Otto messages integrate with the existing audit system:

| Event | Trigger | Logged To |
|---|---|---|
| `otto.session_created` | New session started | `audit_events` |
| `otto.tool_called` | Agent invoked a tool | `audit_events` + `otto_messages` (role='tool') |
| `otto.patch_drafted` | `create_patch` tool used | `audit_events` + creates `patches` row |
| `otto.session_cleared` | User clears conversation | `audit_events` |
| `otto.error` | Agent call failed | `audit_events` + error logged |

---

## Migration from Kiwi Tables

| Kiwi Table | Otto Table | Migration Strategy |
|---|---|---|
| `kiwi_chat_sessions` | `otto_sessions` | No data migration -- old sessions are read-only archives |
| `kiwi_chat_messages` | `otto_messages` | No data migration -- old messages are read-only archives |

The Kiwi tables are not dropped. They remain in the database as historical records. New conversations use `otto_*` tables exclusively.

---

## Related Specs

- [ai-runtime.md](./ai-runtime.md) -- Agent that produces the messages
- [streaming-protocol.md](./streaming-protocol.md) -- How messages are streamed to the frontend
- [context-panel.md](./context-panel.md) -- UI that displays the conversation
