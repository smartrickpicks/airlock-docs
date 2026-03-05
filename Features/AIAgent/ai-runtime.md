# AI Runtime -- PydanticAI Agent Specification

> **Tech:** [PydanticAI](https://ai.pydantic.dev/) -- typed agent framework with dependency injection, tool definitions, and streaming.

---

## VaultContext (Dependency Injection)

Every Otto agent call receives a `VaultContext` -- a typed dataclass injected via PydanticAI's dependency system. This replaces the untyped enrichment dict from OrcestrateOS.

### Schema

```python
@dataclass
class VaultContext:
    # Identity
    vault_id: str
    workspace_id: str
    user_id: str
    user_role: str                          # 'builder' | 'gatekeeper' | 'owner'

    # Enrichment results (populated async, any can be None if source failed)
    gate_state: GateState | None            # Source 1
    field_summary: FieldSummary | None      # Source 2
    contract_health: ContractHealth | None   # Source 3
    domain_rules: DomainRules | None        # Source 4
    preflight_sections: list[PreflightSection] | None  # Source 5
    corpus_lines: list[str] | None          # Source 6
    extraction_meta: ExtractionMeta | None  # Source 7
    patch_summary: PatchSummary | None      # Source 8
    deal_fields: DealFields | None          # Source 9
```

### Enrichment Sub-Types

```python
@dataclass
class GateState:
    gate_color: str          # 'green' | 'yellow' | 'red'
    health_score: float      # 0.0 - 1.0
    processing_status: str   # 'pending' | 'in_progress' | 'complete' | 'failed'

@dataclass
class FieldSummary:
    pass_count: int
    fail_count: int
    review_count: int
    skip_count: int

@dataclass
class ContractHealth:
    score: float             # 0.0 - 1.0
    gate_color: str
    total_checks: int

@dataclass
class DomainRules:
    categories: list[str]    # Available classification categories

@dataclass
class PreflightSection:
    name: str                # e.g., 'accounts', 'catalog', 'pricing', 'terms'
    status: str              # 'pending' | 'running' | 'passed' | 'failed' | 'skipped'

@dataclass
class ExtractionMeta:
    page_count: int
    encoding_type: str
    mojibake_count: int

@dataclass
class PatchSummary:
    open: int
    in_review: int
    resolved: int
    dismissed: int

@dataclass
class DealFields:
    territory: str | None
    term_length: str | None
    contract_type: str | None
    deal_type: str | None
    counterparty_type: str | None
    legal_entity: str | None
```

---

## Enrichment Pipeline

All 9 enrichment sources fire in parallel before the agent processes the user's message.

### Execution Model

```python
async def build_vault_context(vault_id: str, workspace_id: str, user_id: str, user_role: str) -> VaultContext:
    """
    Assemble VaultContext by running all 9 enrichment sources in parallel.
    Each source has an independent timeout. Failed sources return None.
    """
    results = await asyncio.gather(
        with_timeout(enrich_gate_state(vault_id), timeout=ENRICHMENT_TIMEOUT),
        with_timeout(enrich_field_summary(vault_id), timeout=ENRICHMENT_TIMEOUT),
        with_timeout(enrich_contract_health(vault_id), timeout=ENRICHMENT_TIMEOUT),
        with_timeout(enrich_domain_rules(workspace_id), timeout=ENRICHMENT_TIMEOUT),
        with_timeout(enrich_preflight_sections(vault_id), timeout=ENRICHMENT_TIMEOUT),
        with_timeout(enrich_corpus(vault_id), timeout=ENRICHMENT_TIMEOUT),
        with_timeout(enrich_extraction_meta(vault_id), timeout=ENRICHMENT_TIMEOUT),
        with_timeout(enrich_patch_summary(vault_id), timeout=ENRICHMENT_TIMEOUT),
        with_timeout(enrich_deal_fields(vault_id), timeout=ENRICHMENT_TIMEOUT),
        return_exceptions=True,
    )

    return VaultContext(
        vault_id=vault_id,
        workspace_id=workspace_id,
        user_id=user_id,
        user_role=user_role,
        gate_state=results[0] if not isinstance(results[0], Exception) else None,
        field_summary=results[1] if not isinstance(results[1], Exception) else None,
        # ... same pattern for all 9
    )
```

### Behavioral Properties

| Property | Value | Configurable |
|---|---|---|
| Execution model | `asyncio.gather()` (all 9 in parallel) | No |
| Per-source timeout | 2 seconds | Yes -- `otto.enrichment_timeout` calibration key |
| Failure handling | Failed sources set to `None`, agent proceeds | No |
| Caching | Results cached per vault per session | No |
| Cache invalidation | On vault state change (gate transition, patch applied, health recalc) | No |
| Retry | No retry on failure (fire-and-forget) | No |

### Enrichment Source → OrcestrateOS Function Mapping

Each enrichment source ports directly from an existing OrcestrateOS function:

| Source | OrcestrateOS Function | Airlock Function | Changes |
|---|---|---|---|
| Gate State | `_enrich_gate_state()` | `enrich_gate_state()` | Returns typed `GateState` instead of dict |
| Field Summary | `_enrich_field_summary()` | `enrich_field_summary()` | Returns typed `FieldSummary` |
| Contract Health | `_enrich_contract_health()` | `enrich_contract_health()` | Returns typed `ContractHealth` |
| Domain Rules | `_enrich_domain_rules()` | `enrich_domain_rules()` | Returns typed `DomainRules` |
| Preflight Sections | `_enrich_preflight_sections()` | `enrich_preflight_sections()` | Returns typed `list[PreflightSection]` |
| Corpus Search | `_enrich_corpus()` | `enrich_corpus()` | Returns `list[str]` (up to 10 lines) |
| Extraction Meta | `_enrich_extraction_meta()` | `enrich_extraction_meta()` | Returns typed `ExtractionMeta` |
| Patch Summary | `_enrich_patch_summary()` | `enrich_patch_summary()` | Returns typed `PatchSummary` |
| Deal Fields | `_enrich_deal_fields()` | `enrich_deal_fields()` | Returns typed `DealFields` |

---

## Agent Definition

```python
from pydantic_ai import Agent
from pydantic_ai_litellm import LiteLLMModel

otto_agent = Agent(
    model=LiteLLMModel(model_name="otto-default"),
    deps_type=VaultContext,
    result_type=str,
    system_prompt=OTTO_SYSTEM_PROMPT,
)
```

### System Prompt

The system prompt is dynamic -- it injects VaultContext data into a template. The prompt structure:

```
[Role definition]
You are Otto, an AI assistant for the Airlock contract management platform.
You help analysts understand, review, and correct contract data.

[Behavioral constraints]
- Never take autonomous actions. Always propose changes for human approval.
- When suggesting corrections, use the create_patch tool to draft a patch.
- When asked about entity matches, use the resolve_entity tool to show candidates.
- Always cite enrichment sources when making claims about the contract.

[Vault context (injected from VaultContext)]
Current vault: {vault_id}
Gate status: {gate_state.gate_color} | Health: {contract_health.score}
Fields: {field_summary.pass_count} pass, {field_summary.fail_count} fail, {field_summary.review_count} review
Open triage items: {patch_summary.open}

[Role-specific additions (from workspace config)]
{custom_role_prompt}
```

### Dynamic System Prompt Sections

| Section | Source | Always included |
|---|---|---|
| Role definition | Hardcoded | Yes |
| Behavioral constraints | Hardcoded | Yes |
| Vault context summary | VaultContext fields | Yes (with `None` handling) |
| Role-specific prompt | `workspace_agent_config.system_prompt` | Only if configured |
| Skill restrictions | `workspace_agent_config.enabled_skills` | Only if configured |

---

## Tool Definitions

PydanticAI tools allow the agent to call back into Airlock during a conversation. All tools are **read-only or draft-only** -- no tool can apply changes without human approval.

### Tool Catalog

| # | Tool | Purpose | Returns | Side Effects |
|---|---|---|---|---|
| 1 | `create_patch` | Draft a data correction for a specific field | Patch ID + diff preview | Creates Draft-state patch (requires human approval to apply) |
| 2 | `resolve_entity` | Show entity match candidates with confidence | Top 5 candidates + scores | Marks a suggestion (not a confirmed resolution) |
| 3 | `classify_contract` | Suggest contract type and subtype | Classification + confidence | Informational only |
| 4 | `search_corpus` | Full-text search in contract document text | Matching lines + context | Read-only |
| 5 | `explain_field` | Get extraction evidence for a specific field | Anchor, evidence window, confidence score | Read-only |
| 6 | `get_gate_blockers` | List what's blocking gate advancement | Blocker list with reasons + remediation hints | Read-only |
| 7 | `get_health_breakdown` | Show per-section health scores | Section scores + contributing factors | Read-only |
| 8 | `get_patch_queue` | List open triage items for this vault | Triage items grouped by status + severity | Read-only |

### Tool Implementation Pattern

Each tool follows the same PydanticAI pattern:

```python
@otto_agent.tool
async def create_patch(ctx: RunContext[VaultContext], field_code: str, proposed_value: str, reason: str) -> str:
    """Draft a data correction for a field. Creates a Draft-state patch requiring human approval."""
    vault_id = ctx.deps.vault_id
    user_id = ctx.deps.user_id

    # Validate field exists in schema
    if field_code not in MASTER_SCHEMA:
        return f"Unknown field code: {field_code}"

    # Create draft patch via internal API
    patch = await create_draft_patch(
        vault_id=vault_id,
        field_code=field_code,
        proposed_value=proposed_value,
        reason=reason,
        author_id=user_id,
        author_type="ai_assisted",  # Tracks that AI helped draft this
    )

    return f"Draft patch created (ID: {patch.id}). Proposed: {field_code} = '{proposed_value}'. Requires human approval to apply."
```

### Tool Safety Rules

| Rule | Enforcement |
|---|---|
| No autonomous changes | `create_patch` creates Draft state only -- never Applied |
| No self-approval | Patches created via tool require a different user to approve |
| RBAC filtering | Tools available to the agent are filtered by `user_role` |
| Audit trail | Every tool call logged in `otto_messages` with `role='tool'` |
| Rate limiting | Max 5 tool calls per message (prevents runaway loops) |

### RBAC-Based Tool Availability

| Tool | Builder | Gatekeeper | Owner |
|---|---|---|---|
| `create_patch` | Yes | Yes | Yes |
| `resolve_entity` | Yes | Yes | Yes |
| `classify_contract` | Yes | Yes | Yes |
| `search_corpus` | Yes | Yes | Yes |
| `explain_field` | Yes | Yes | Yes |
| `get_gate_blockers` | Yes | Yes | Yes |
| `get_health_breakdown` | Yes | Yes | Yes |
| `get_patch_queue` | Yes | Yes | Yes |

All tools are available to all roles in the initial implementation. Future: workspace admins can restrict tools per role.

---

## Feature Control Plane Integration

Otto is a toggleable feature with its own calibration parameters.

### Feature Flag

| Flag | Key | Default | Description |
|---|---|---|---|
| Otto AI Agent | `otto.enabled` | `true` | Master on/off for the AI agent |
| Otto Tool Use | `otto.tools_enabled` | `true` | Allow agent to invoke tools |
| Otto Streaming | `otto.streaming_enabled` | `true` | Enable SSE streaming (false = buffered response) |

### Calibration Parameters

| Parameter | Key | Default | Range | Unit |
|---|---|---|---|---|
| Model selection | `otto.model` | `otto-default` | enum | model alias |
| Max response tokens | `otto.max_tokens` | 2048 | 256 - 8192 | tokens |
| Temperature | `otto.temperature` | 0.3 | 0.0 - 1.0 | score |
| Request timeout | `otto.timeout` | 30 | 5 - 120 | seconds |
| Enrichment timeout | `otto.enrichment_timeout` | 2 | 1 - 10 | seconds |
| Max tool calls per message | `otto.max_tool_calls` | 5 | 1 - 20 | count |

### Circuit Breaker

| Setting | Value |
|---|---|
| Error threshold | 5 errors in 60 seconds |
| Cooldown period | 300 seconds |
| Health check | LiteLLM `/health` endpoint |
| Recovery | Automatic after cooldown; manual override in Admin |

---

## File Layout

```
services/airlock-api/server/otto/
    __init__.py
    agent.py          # PydanticAI agent definition + system prompt
    deps.py           # VaultContext dataclass + enrichment sub-types
    tools.py          # 8 tool definitions
    enrichment.py     # 9 enrichment source functions (ported from OrcestrateOS)
    feature_gate.py   # Feature flag + circuit breaker checks
```

---

## Related Specs

- [litellm-gateway.md](./litellm-gateway.md) -- Model routing and Docker config
- [streaming-protocol.md](./streaming-protocol.md) -- SSE endpoint and frontend hook
- [session-persistence.md](./session-persistence.md) -- DB schema for sessions and messages
- [enrichment-sources.md](./enrichment-sources.md) -- Detailed source catalog
