# Streaming Protocol -- SSE Endpoint & Frontend Integration

> **Tech:** FastAPI SSE + [Vercel AI SDK](https://sdk.vercel.ai/) wire format + PydanticAI streaming.

---

## Purpose

Otto delivers responses as a real-time token stream, not a buffered synchronous response. This gives the analyst immediate feedback as Otto "thinks" and allows tool calls to render inline as they happen.

The streaming pipeline:

```
PydanticAI agent.run_stream()
    |
    v
FastAPI StreamingResponse (SSE)
    |
    v
Vercel AI SDK useChat hook (frontend)
    |
    v
Context Panel chat UI (renders tokens incrementally)
```

---

## Backend: FastAPI SSE Endpoint

### Route

```
POST /api/v3/vaults/{vault_id}/otto/chat
```

### Request

```json
{
  "message": "What's wrong with this contract?",
  "session_id": "ots_abc123"          // optional, omit to start new session
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `message` | string | Yes | User's message text |
| `session_id` | string | No | Existing session ID to continue; omit to create new session |

### Response

Content-Type: `text/event-stream`

The response is a Server-Sent Events stream following the Vercel AI SDK wire format.

### Endpoint Implementation Pattern

```python
from fastapi import APIRouter
from fastapi.responses import StreamingResponse
from pydantic_ai import Agent

router = APIRouter()

@router.post("/api/v3/vaults/{vault_id}/otto/chat")
async def otto_chat(vault_id: str, body: OttoChatRequest, user = Depends(get_current_user)):
    # 1. Check feature flag
    if not is_feature_enabled("otto.enabled", user.workspace_id):
        raise HTTPException(503, "Otto AI Agent is disabled")

    # 2. Check circuit breaker
    if is_circuit_open("otto", user.workspace_id):
        return stub_response(vault_id, user)

    # 3. Load or create session
    session = await get_or_create_session(vault_id, user.id, body.session_id)

    # 4. Build VaultContext (async enrichment)
    context = await build_vault_context(vault_id, user.workspace_id, user.id, user.role)

    # 5. Load message history
    history = await load_session_messages(session.id)

    # 6. Stream response
    return StreamingResponse(
        stream_otto_response(context, history, body.message, session),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",
            "Connection": "keep-alive",
            "X-Accel-Buffering": "no",    # Disable nginx buffering
        },
    )
```

### Streaming Generator

```python
async def stream_otto_response(context, history, message, session):
    """
    Yields SSE events in Vercel AI SDK wire format.
    """
    try:
        async with otto_agent.run_stream(
            message,
            deps=context,
            message_history=history,
        ) as stream:
            # Stream text tokens
            async for chunk in stream.stream_text(delta=True):
                yield format_sse_text(chunk)

            # After streaming completes, get result
            result = stream.result()

            # Save messages to DB
            await save_messages(session.id, message, result)

            # Emit finish event
            yield format_sse_finish("stop")

    except Exception as e:
        # Log error, increment circuit breaker counter
        await record_otto_error(e)
        yield format_sse_error(str(e))

    finally:
        yield format_sse_done()
```

---

## SSE Wire Format (Vercel AI SDK Compatible)

The SSE stream uses the Vercel AI SDK data stream protocol. Each event is a single line prefixed with a type code.

### Event Types

| Prefix | Name | Payload | When |
|---|---|---|---|
| `0:` | Text delta | `"token text"` | During generation -- each token/chunk |
| `2:` | Tool call | `[{toolCallId, toolName, args}]` | Agent invokes a tool |
| `8:` | Tool result | `[{toolCallId, result}]` | Tool execution completed |
| `9:` | Tool call streaming | `{toolCallId, argsTextDelta}` | Tool arguments streaming (partial) |
| `e:` | Finish | `{finishReason, usage}` | Generation complete |
| `d:` | Done | `[DONE]` | Stream ended |

### Example Stream

```
0:"I can see "
0:"several issues "
0:"with this contract.\n\n"
0:"Let me check "
0:"the gate blockers.\n\n"
2:[{"toolCallId":"tc_1","toolName":"get_gate_blockers","args":{"vault_id":"vault_henderson"}}]
8:[{"toolCallId":"tc_1","result":"2 blockers found: low entity confidence (38%), missing payment threshold"}]
0:"Based on the gate analysis, "
0:"there are **2 blockers**:\n\n"
0:"1. **Low entity confidence** (38%) -- "
0:"the counterparty match needs confirmation\n"
0:"2. **Missing payment threshold** -- "
0:"no `payment_threshold` value was extracted\n\n"
0:"Would you like me to "
0:"help resolve either of these?"
e:{"finishReason":"stop","usage":{"promptTokens":1847,"completionTokens":203}}
d:[DONE]
```

### SSE Formatting Functions

```python
import json

def format_sse_text(delta: str) -> str:
    """Format a text chunk as SSE event."""
    return f"0:{json.dumps(delta)}\n"

def format_sse_tool_call(tool_call_id: str, tool_name: str, args: dict) -> str:
    """Format a tool invocation as SSE event."""
    payload = [{"toolCallId": tool_call_id, "toolName": tool_name, "args": args}]
    return f"2:{json.dumps(payload)}\n"

def format_sse_tool_result(tool_call_id: str, result: str) -> str:
    """Format a tool result as SSE event."""
    payload = [{"toolCallId": tool_call_id, "result": result}]
    return f"8:{json.dumps(payload)}\n"

def format_sse_finish(reason: str, prompt_tokens: int = 0, completion_tokens: int = 0) -> str:
    """Format the finish event."""
    payload = {"finishReason": reason, "usage": {"promptTokens": prompt_tokens, "completionTokens": completion_tokens}}
    return f"e:{json.dumps(payload)}\n"

def format_sse_done() -> str:
    """Format the stream termination event."""
    return "d:[DONE]\n"

def format_sse_error(message: str) -> str:
    """Format an error event."""
    payload = {"finishReason": "error", "error": message}
    return f"e:{json.dumps(payload)}\n"
```

---

## Frontend: Vercel AI SDK `useChat` Hook

### `useOttoChat` Hook

```typescript
import { useChat } from '@ai-sdk/react';
import { useModuleStore } from '@/stores/modules';

interface UseOttoChatOptions {
  vaultId: string;
}

export function useOttoChat({ vaultId }: UseOttoChatOptions) {
  const sessionId = useOttoSessionStore((s) => s.getSessionId(vaultId));

  const chat = useChat({
    api: `/api/v3/vaults/${vaultId}/otto/chat`,
    body: {
      session_id: sessionId,
    },
    onToolCall: async ({ toolCall }) => {
      // Tool calls are handled server-side by PydanticAI
      // Frontend receives tool call + result events for display
      // No client-side tool execution needed
    },
    onFinish: (message) => {
      // Update session store with latest message
      // Trigger Signal panel event refresh (if dual presence enabled)
    },
    onError: (error) => {
      // Show error state in Context Panel
      // If circuit breaker tripped, show "Otto unavailable" banner
    },
  });

  return {
    messages: chat.messages,
    input: chat.input,
    handleInputChange: chat.handleInputChange,
    handleSubmit: chat.handleSubmit,
    isLoading: chat.isLoading,
    error: chat.error,
    // Additional Otto-specific state
    isOttoAvailable: !chat.error,
  };
}
```

### Context Panel Integration

The `useOttoChat` hook plugs directly into the Control Panel's Context tab:

```typescript
function OttoContextPanel({ vaultId }: { vaultId: string }) {
  const { messages, input, handleInputChange, handleSubmit, isLoading } = useOttoChat({ vaultId });

  return (
    <div className="otto-context-panel">
      {/* Message list */}
      <div className="otto-messages">
        {messages.map((msg) => (
          <OttoMessage key={msg.id} message={msg} />
        ))}
        {isLoading && <OttoTypingIndicator />}
      </div>

      {/* Composer */}
      <form onSubmit={handleSubmit}>
        <textarea value={input} onChange={handleInputChange} />
        <button type="submit" disabled={isLoading}>Send</button>
      </form>
    </div>
  );
}
```

### Tool Call Rendering

When the stream includes tool calls (prefix `2:`) and results (prefix `8:`), the frontend renders them inline:

| Tool | UI Rendering |
|---|---|
| `create_patch` | Inline patch card with diff preview + "Review Patch" button |
| `resolve_entity` | Entity candidate list with confidence bars |
| `classify_contract` | Classification badge with confidence score |
| `search_corpus` | Expandable code block with matching lines |
| `explain_field` | Field evidence card with anchor + context window |
| `get_gate_blockers` | Blocker list with severity icons |
| `get_health_breakdown` | Section health table with score bars |
| `get_patch_queue` | Triage item list grouped by status |

---

## Graceful Degradation

### Level 1: Partial Enrichment

If some enrichment sources fail, the agent proceeds with available context. The enrichment indicator in the Context Panel shows which sources contributed (e.g., "7/9 sources").

### Level 2: Model Fallback

If the primary model (Claude) is unavailable, LiteLLM routes to the fallback (GPT). The response includes a metadata badge showing which model was used. The analyst sees a subtle indicator: "Powered by GPT-4o (fallback)".

### Level 3: Full Stub Response

If all models are unavailable (circuit breaker tripped or all providers down):

```json
{
  "type": "stub",
  "message": "Otto is temporarily unavailable. Here's the vault context I gathered:",
  "enrichment": {
    "gate_state": { "gate_color": "yellow", "health_score": 0.68 },
    "field_summary": { "pass": 37, "fail": 12, "review": 28, "skip": 5 },
    "patch_summary": { "open": 3, "in_review": 1 }
  }
}
```

The Context Panel renders this as a structured card (not a chat message) with:
- "Otto unavailable" banner
- Enrichment data formatted as readable cards
- "Retry" button that checks if the circuit breaker has recovered

### Level 4: Feature Disabled

If `otto.enabled` flag is `false`, the Context Panel tab shows:
- "Otto is disabled by your workspace administrator"
- No chat input visible
- Link to Admin settings (if user has admin role)

---

## Error Handling

| Error Type | Backend Behavior | Frontend Behavior |
|---|---|---|
| Enrichment timeout | Source returns `None`, agent proceeds | Enrichment indicator shows gap |
| LiteLLM timeout | Returns partial response if any tokens streamed | Shows partial message + error banner |
| LiteLLM 5xx | Circuit breaker counter increments | Retry prompt or stub fallback |
| PydanticAI tool error | Tool returns error string, agent incorporates | Error shown inline in chat |
| Network error | Stream terminates | "Connection lost" banner + auto-retry |
| Rate limit (429) | LiteLLM queues or retries | "Otto is busy, please wait" message |

---

## Related Specs

- [ai-runtime.md](./ai-runtime.md) -- PydanticAI agent that generates the stream
- [litellm-gateway.md](./litellm-gateway.md) -- LiteLLM proxy that routes the model call
- [session-persistence.md](./session-persistence.md) -- Where messages are stored after streaming
- [context-panel.md](./context-panel.md) -- UI that consumes the stream
