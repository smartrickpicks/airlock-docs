# LiteLLM Gateway -- Model Routing Specification

> **Tech:** [LiteLLM](https://github.com/BerryDev/litellm) -- LLM gateway proxy supporting 100+ providers behind a unified OpenAI-compatible API.

---

## Purpose

LiteLLM runs as a Docker sidecar alongside the Airlock API. All LLM calls from PydanticAI route through the LiteLLM proxy, which handles:

1. **Model routing** -- Route to Claude, GPT, or Ollama via a single model alias
2. **Automatic fallback** -- If Claude is down, transparently fall back to GPT
3. **Cost tracking** -- Log token usage and cost per request
4. **Rate limiting** -- Enforce per-model rate limits
5. **Key management** -- API keys stored in LiteLLM, not in application code

---

## Architecture

```
PydanticAI Agent
    |
    v
pydantic-ai-litellm provider
    |
    v
LiteLLM Proxy (http://litellm:4000)
    |
    +---> Claude API (anthropic/claude-sonnet-4-20250514)     [primary]
    +---> OpenAI API (openai/gpt-4o)                    [fallback]
    +---> Ollama (ollama/llama3.1)                      [local dev]
```

---

## Model Configuration

### `litellm-config.yaml`

```yaml
model_list:
  # Primary model -- Claude Sonnet (best reasoning + tool use)
  - model_name: otto-default
    litellm_params:
      model: anthropic/claude-sonnet-4-20250514
      api_key: os.environ/ANTHROPIC_API_KEY
      max_tokens: 4096

  # Fallback model -- GPT-4o (when Claude is unavailable)
  - model_name: otto-default
    litellm_params:
      model: openai/gpt-4o
      api_key: os.environ/OPENAI_API_KEY
      max_tokens: 4096

  # Fast model -- Claude Haiku (quick responses, lower cost)
  - model_name: otto-fast
    litellm_params:
      model: anthropic/claude-haiku-4-20250414
      api_key: os.environ/ANTHROPIC_API_KEY
      max_tokens: 2048

  # Local dev model -- Ollama (no API keys needed, air-gapped)
  - model_name: otto-local
    litellm_params:
      model: ollama/llama3.1
      api_base: http://ollama:11434

router_settings:
  routing_strategy: simple-shuffle    # Tries models in order, auto-failover
  num_retries: 2                      # Retry on transient failures
  timeout: 30                         # Per-request timeout (seconds)
  fallbacks:
    - otto-default: [otto-fast]       # If default fails, try fast model
```

### Model Alias Strategy

| Alias | Primary | Fallback | Use Case |
|---|---|---|---|
| `otto-default` | Claude Sonnet | GPT-4o | Standard conversations, tool use, analysis |
| `otto-fast` | Claude Haiku | -- | Quick responses, high-volume, cost-sensitive |
| `otto-local` | Ollama Llama 3.1 | -- | Local development, air-gapped environments |

When multiple providers are listed under the same `model_name`, LiteLLM tries them in order and automatically fails over.

---

## Environment Variables

| Variable | Description | Required | Default |
|---|---|---|---|
| `ANTHROPIC_API_KEY` | Claude API key for primary model | Yes (prod) | -- |
| `OPENAI_API_KEY` | GPT fallback API key | No | -- |
| `LITELLM_MASTER_KEY` | Admin key for LiteLLM management API | Yes | -- |
| `LITELLM_BASE_URL` | LiteLLM proxy URL (internal Docker network) | No | `http://litellm:4000` |
| `OTTO_MODEL` | Default model alias used by PydanticAI | No | `otto-default` |
| `OTTO_FAST_MODEL` | Fast model alias for quick tasks | No | `otto-fast` |

---

## Docker Compose

```yaml
# Added to airlock/docker-compose.yml
services:
  litellm:
    image: ghcr.io/berriai/litellm:main-latest
    ports:
      - "4000:4000"          # Exposed for local dev dashboard
    volumes:
      - ./config/litellm-config.yaml:/app/config.yaml
    command: ["--config", "/app/config.yaml"]
    environment:
      LITELLM_MASTER_KEY: ${LITELLM_MASTER_KEY}
      ANTHROPIC_API_KEY: ${ANTHROPIC_API_KEY}
      OPENAI_API_KEY: ${OPENAI_API_KEY:-}
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:4000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped

  # Ollama for local dev (optional, not started in prod)
  ollama:
    image: ollama/ollama:latest
    ports:
      - "11434:11434"
    volumes:
      - ollama-data:/root/.ollama
    profiles:
      - local                # Only starts with: docker compose --profile local up

volumes:
  ollama-data:
```

### Local Development

For local dev without API keys:

```bash
# Start all services including Ollama
docker compose --profile local up

# Pull a model into Ollama
docker compose exec ollama ollama pull llama3.1

# Set OTTO_MODEL=otto-local in .env to route all calls to Ollama
```

---

## Monitoring & Observability

### LiteLLM Dashboard

LiteLLM includes a web dashboard at `http://localhost:4000/ui` (local dev only) showing:

- Request logs: model used, tokens consumed, latency, cost
- Error rates: per-model failure counts
- Usage analytics: requests/min, avg latency, cost over time

### Health Endpoint

| Endpoint | Returns | Used By |
|---|---|---|
| `GET /health` | `{"status": "healthy"}` or error | Feature Control Plane health strip |
| `GET /model/info` | List of available models + status | Admin UI model selector |

### Feature Control Plane Integration

Otto's health card in the Admin Feature Control Plane dashboard pulls from:

| Metric | Source | Display |
|---|---|---|
| Status | LiteLLM `/health` + circuit breaker state | Green/Amber/Red dot |
| Requests/min | LiteLLM request logs | Sparkline chart |
| Avg latency | LiteLLM request logs | `Xms` label |
| Model | Active model alias | Model badge |
| Error rate | LiteLLM error counts | Percentage |
| Cost (24h) | LiteLLM cost tracking | `$X.XX` label |

---

## Calibration Parameters

These parameters are editable in the Admin Calibration panel and control LiteLLM behavior:

| Parameter | Key | Default | Range | Unit | Effect |
|---|---|---|---|---|---|
| Model selection | `otto.model` | `otto-default` | enum | model alias | Which LiteLLM model alias to use |
| Max response tokens | `otto.max_tokens` | 2048 | 256 - 8192 | tokens | Passed to LiteLLM as `max_tokens` |
| Temperature | `otto.temperature` | 0.3 | 0.0 - 1.0 | score | Controls response creativity |
| Request timeout | `otto.timeout` | 30 | 5 - 120 | seconds | LiteLLM per-request timeout |

### Model Selection Behavior

When an admin changes `otto.model` in the calibration panel:

1. The change takes effect immediately (no restart needed)
2. Active sessions continue with the previous model until the next message
3. New sessions use the updated model
4. An audit event is emitted: `calibration.changed` with `otto.model`

---

## Security Considerations

| Concern | Mitigation |
|---|---|
| API key exposure | Keys stored as env vars, never in config files or DB |
| Internal-only access | LiteLLM port 4000 not exposed in production (only via Docker internal network) |
| Prompt injection | PydanticAI handles input sanitization; LiteLLM adds no additional risk |
| Cost runaway | LiteLLM supports per-model budget limits (configurable) |
| Data privacy | No request/response data leaves the Airlock network unless routed to external API |

---

## Related Specs

- [ai-runtime.md](./ai-runtime.md) -- PydanticAI agent that connects to LiteLLM
- [streaming-protocol.md](./streaming-protocol.md) -- SSE endpoint that streams LiteLLM responses
- [Feature Control Plane](../FeatureControlPlane/overview.md) -- Health strip + calibration UI
