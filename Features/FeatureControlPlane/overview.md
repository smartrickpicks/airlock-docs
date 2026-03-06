# Feature Control Plane -- Design Spec

## Purpose

The Feature Control Plane ensures that **every ported feature is individually controllable, observable, and debuggable** from the Admin UI. No feature ships without an entry in the control plane.

---

## Core Principle

Every feature in Airlock must expose four capabilities through the control plane:

| Capability | Description |
|---|---|
| **Toggle On/Off** | Instantly enable or disable the feature without a restart |
| **Manual Calibration** | Adjust thresholds, weights, and confidence scores through Admin UI |
| **Expanded Audit Log** | Every evaluation, state change, and calibration update is recorded |
| **Failure Flagging** | Automatic detection and surfacing of feature failures with circuit-breaker behavior |

---

## Current State (OrcestrateOS)

| Aspect | Current Reality |
|---|---|
| Flag count | 24 environment variable flags |
| Toggle mechanism | Binary on/off via `.env` file |
| Activation | Requires server restart to take effect |
| Storage | No database backing -- lives in process memory after env load |
| Audit trail | None -- no record of who changed what or when |
| Targeting | Global only -- same flags for all workspaces |
| Rollout control | None -- 0% or 100%, nothing in between |

---

## Airlock Vision

| Aspect | Target State |
|---|---|
| Flag storage | Database-backed with versioned history |
| Toggle mechanism | HTTP PATCH endpoint -- instant flip, no restart |
| Targeting | Per-workspace flag overrides |
| Rollout control | 0-100% percentage for gradual deployment |
| Audit trail | Every flag change recorded with actor, timestamp, before/after |
| Circuit breaker | Automatic disable when error rate exceeds threshold |
| Observability | Real-time flag evaluation metrics in System Health channel |

---

## Managed Features

The following features require control plane entries:

| Category | Features | Flags | Calibration Params |
|---|---|---|---|
| **Extractors** | BooleanExtractor (5 fields), SplitExtractor (~30 fields), PicklistExtractor (~20 fields), DateExtractor (~10 fields), PatternExtractor (~15 fields), TextExtractor (~20 fields, spaCy NER), Dispatcher (all 152 fields) | 7 individual + 1 dispatcher | 3 (tier-1/tier-2 confidence, proximity window) |
| **OCR** | OCR pipeline toggle and engine selection | 1 | -- |
| **Preflight** | Preflight gate evaluation (3,895 lines), Phase 1 RED hard stops, Phase 2 YELLOW quality checks | 1 | 7 (mojibake ratio, control chars, page chars min, image ratio, counterparty conf, entity score, doc mode majority) |
| **Identity Resolver** | Composite context scorer: `(name * 0.85) + address + account_context - service_penalty` | 1 | 7 (4 score weights + 3 match thresholds) |
| **Health Scoring** | Section weights (5), gate penalties (3), band thresholds (2), calibration model selection | 1 | 10 (5 weights, 3 penalties, 2 band floors) |
| **Contract Generator** | Template loading, clause matching (188 clauses, 24 types), generation pipeline | 1 | -- |
| **Batch Processor** | Async pipeline with semaphore(5), 15-state lifecycle | 1 | 3 (concurrency, retries, PDF timeout) |
| **Auto System Pass** | GREEN gate + no triage items = auto-advance | 1 | -- |
| **Triage Creation** | QA rules + preflight → triage items | 1 | -- |
| **Patch Workflow** | 12 states, 20 transitions, self-approval block | 1 | -- |
| **AI Flags** | AI-powered anomaly flagging | 1 | -- |
| **Otto AI Agent** | PydanticAI + LiteLLM runtime | 1 | See AI Configuration |
| **Export** | PDF, CSV, JSON, DOCX export pipelines | 4 | -- |
| **Lifecycle Transitions** | State machine transition enforcement (15 states) | 1 | -- |
| **Canonical Copy** | Normalized text output | 1 | -- |
| **External Integrations** | Google Drive, Salesforce Sync | 2 | -- |
| **Audit Logging** | Full action audit trail | 1 | -- |

**Totals:** 27 feature flags, 32 calibration parameters, 14 health-monitored engines

---

## AI Configuration

The Feature Control Plane includes a dedicated AI Configuration section for managing the Otto AI Agent runtime:

### Model Providers (LiteLLM Gateway)

| Alias | Model | Purpose | Status |
|---|---|---|---|
| `otto-default` | `anthropic/claude-sonnet-4-20250514` | Primary model for all agent interactions | Connected |
| `otto-fast` | `anthropic/claude-haiku-4-20250414` | Quick responses, tool-only calls | Connected |
| `otto-local` | `ollama/llama3.1` | Local dev / air-gapped environments | Offline |
| `gpt-fallback` | `openai/gpt-4o` | Fallback when Anthropic is unavailable | Connected |

**Fallback chain:** `otto-default → otto-fast → gpt-fallback`
**Routing strategy:** `simple-shuffle` with 2 retries, 30s timeout

### Agent Roles

Configurable AI agent personalities with different system prompts, skills, and model selections. Each role defines:

- **System prompt** — personality and instructions
- **Skills** — which of the 8 tools the agent can use (`create_patch`, `resolve_entity`, `classify_contract`, `search_corpus`, `explain_field`, `get_gate_blockers`, `get_health_breakdown`, `get_patch_queue`)
- **Model** — which LiteLLM model alias to use
- **Temperature** — response creativity (0.0–1.0)
- **Max tokens** — response length limit

### Runtime Settings

| Setting | Default | Description |
|---|---|---|
| Streaming | Enabled | Token-by-token SSE delivery |
| Tool Use | Enabled | Agent can invoke typed tools |
| Max Tool Calls | 5 | Per-turn tool invocation limit |
| Enrichment Timeout | 2s | Per-source enrichment timeout |
| Session Persistence | Enabled | Multi-turn conversation storage |
| Cost Tracking | Enabled | Per-message token + cost logging |

---

## Sub-Specs

| Document | Scope |
|---|---|
| [Toggle System](./toggle-system.md) | DB-backed flags, HTTP toggle, per-workspace targeting, rollout % |
| [Calibration](./calibration.md) | Thresholds, weights, confidence scores as admin-editable parameters |
| [Failure Detection](./failure-detection.md) | Health states, error pipeline, circuit breaker logic, Admin UI, graceful degradation |

---

## Demo

See [Feature Control Plane Demo](../Admin/feature-control-plane-demo.html) — comprehensive interactive prototype with all 14 health-monitored engines, 27 feature flags, 32 calibration parameters, AI configuration panel, failure feed, and audit log.

---

## Related Specs

- [Admin / Settings](../Admin/overview.md) — Parent admin overlay spec
- [AI Agent Runtime](../AIAgent/ai-runtime.md) — PydanticAI agent definition, VaultContext, tools
- [LiteLLM Gateway](../AIAgent/litellm-gateway.md) — Model proxy configuration
- [Contract Generator](../ContractGenerator/overview.md) — 188 clauses, 24 contract types
