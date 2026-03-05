# AI Agent (Otto)

> **Status:** SPECCED — PydanticAI + LiteLLM runtime.

> **Decision:** The AI Agent has dual presence -- feed events in the Signal panel AND a dedicated tab in the Control panel.

## Design Principles

| Principle | Detail |
|---|---|
| Human in the loop | AI proposes, humans approve. No autonomous actions. |
| Provider-agnostic | Pluggable model backend via LiteLLM proxy -- Claude, GPT, Grok, Ollama. Swap model routing without code changes. |
| Configurable roles | Organizations define agent roles with custom skills and prompts. No fixed role taxonomy. |
| Tool-enabled | Agent can call back into Airlock (create patches, resolve entities, search corpus) via typed PydanticAI tools. |
| Streaming-first | Token-by-token SSE delivery via Vercel AI SDK wire format. No synchronous request/response. |

## Tech Stack

| Library | Version | License | Purpose |
|---|---|---|---|
| [PydanticAI](https://ai.pydantic.dev/) | latest | MIT | Agent framework -- typed tools, dependency injection, streaming |
| [LiteLLM](https://github.com/BerryDev/litellm) | latest | MIT | LLM gateway proxy -- 100+ providers behind OpenAI-compatible API |
| [pydantic-ai-litellm](https://pypi.org/project/pydantic-ai-litellm/) | latest | MIT | PydanticAI provider that routes through LiteLLM |
| [Vercel AI SDK](https://sdk.vercel.ai/) (`@ai-sdk/react`) | latest | Apache-2.0 | Frontend `useChat` hook for SSE streaming |

## OrcestrateOS Lineage

| Backend artifact | Detail |
|---|---|
| Origin | Evolution of Otto Chat (formerly Kiwi Chat in OrcestrateOS) |
| Original route | `server/routes/kiwi_chat.py` (392 lines) -- **replaced** by PydanticAI agent |
| Original sessions table | `kiwi_chat_sessions` -- **replaced** by `otto_sessions` |
| Original messages table | `kiwi_chat_messages` -- **replaced** by `otto_messages` |
| What carries forward | 9 enrichment source functions (ported to async), dual presence model, session-per-vault pattern |
| What changes | Webhook call replaced by in-process PydanticAI agent with typed tools and streaming |

---

## Dual Presence Model

The AI Agent surfaces in two places simultaneously within the vault triptych.

### Signal Panel (left/center)

- Agent responses appear as **cyan-colored events** in the vault's Signal feed.
- Visually distinct from human events, system events, and preflight events.
- `@AI` mentions in the Signal feed trigger the agent and auto-open the Context Panel tab.

### Control Panel (right)

- Dedicated **Context Panel** tab alongside Lifecycle, Approvals, and Audit Trail.
- Full multi-turn chat interface with session persistence.
- See [context-panel.md](./context-panel.md) for detailed spec.

---

## Agent Architecture

```
Analyst input (message or @AI mention)
    |
    v
FastAPI SSE endpoint (/api/v3/vaults/{vault_id}/otto/chat)
    |
    v
VaultContext assembly (9 enrichment sources, async parallel, 2s timeout each)
    |
    v
PydanticAI Agent (typed tools, system prompt, VaultContext dependency)
    |
    v
LiteLLM Proxy (model routing: Claude -> GPT -> Ollama fallback)
    |
    v
Streaming SSE response (Vercel AI SDK wire format)
    |
    v
Signal feed event (cyan) + Context Panel message (multi-turn)
```

### Key Difference from OrcestrateOS (Kiwi)

| Aspect | Kiwi (OrcestrateOS) | PydanticAI + LiteLLM (Otto) |
|---|---|---|
| Execution | External webhook call | In-process agent execution |
| Tool use | None -- agent receives pre-composed text only | 8 typed tools -- agent can call back into Airlock |
| Streaming | Synchronous request/response | Token-by-token SSE |
| Model routing | Single webhook URL | LiteLLM proxy with fallback chains |
| Context injection | Untyped dict composed into text blob | Typed `VaultContext` dataclass via dependency injection |
| Local dev | Required external service | Runs locally with Ollama via LiteLLM |
| Error handling | Stub response with enrichment text | Circuit breaker + partial response + stub fallback |

### Enrichment

Before every agent call, the system pre-injects context from up to 9 enrichment sources. Each source is independently toggleable via the Feature Control Plane. Enrichment is non-blocking -- if a source times out or errors, the call proceeds without it.

**New behavior:** Enrichment runs as `asyncio.gather()` with per-source timeouts (default 2s, configurable via calibration). Results are cached per vault per session -- follow-up messages reuse cached enrichment unless the vault state changed.

See [enrichment-sources.md](./enrichment-sources.md) for the full source catalog.

### Provider Configuration

| Setting | Detail |
|---|---|
| Model gateway | LiteLLM proxy at `http://litellm:4000` (Docker sidecar) |
| Default model | `otto-default` alias (routes to Claude Sonnet) |
| Fast model | `otto-fast` alias (routes to Claude Haiku) |
| Fallback chain | `otto-default` -> `otto-fast` (automatic on failure) |
| Local dev | `otto-local` alias (routes to Ollama) |
| Swap mechanism | Edit `litellm-config.yaml` and restart LiteLLM container |

See [litellm-gateway.md](./litellm-gateway.md) for full configuration spec.

### Graceful Degradation

Three levels of degradation:

| Level | Trigger | Behavior |
|---|---|---|
| **Partial enrichment** | 1+ enrichment sources timeout | Agent proceeds with available context, "enrichment used" indicator shows gaps |
| **Model fallback** | Primary model unavailable | LiteLLM routes to fallback model automatically |
| **Full stub** | All models unavailable | System returns enrichment context as structured stub, clearly labeled "AI unavailable" |

When errors exceed threshold (5 errors in 60 seconds), the circuit breaker trips and Otto shows "unavailable" status in the Context Panel for a configurable cooldown period (default 300 seconds).

---

## Configurable Agent Framework

Organizations are not locked into predefined agent behaviors.

| Concept | Detail |
|---|---|
| Agent roles | Org-defined, not system-defined |
| Skills | Composable capabilities attached to roles |
| Prompts | Custom system prompts per role |
| Assignment | Roles can be scoped to teams, modules, or individual analysts |

This allows orgs to create specialized agents (e.g., "Compliance Reviewer," "Data Quality Auditor," "Onboarding Assistant") without code changes.

---

## Related Specs

| File | Content |
|---|---|
| [ai-runtime.md](./ai-runtime.md) | PydanticAI agent definition, VaultContext, 8 tools, enrichment pipeline |
| [litellm-gateway.md](./litellm-gateway.md) | LiteLLM proxy config, model routing, Docker, monitoring |
| [streaming-protocol.md](./streaming-protocol.md) | SSE endpoint, Vercel AI SDK wire format, `useOttoChat` hook |
| [session-persistence.md](./session-persistence.md) | `otto_sessions` + `otto_messages` DB schema |
| [enrichment-sources.md](./enrichment-sources.md) | 9 enrichment sources catalog |
| [context-panel.md](./context-panel.md) | Control panel UI/UX spec |
| [feed-events.md](./feed-events.md) | Signal panel event rendering |
