# Toggle System -- Design Spec

## Purpose

Replace the current environment-variable-based feature flags with a database-backed toggle system that supports instant activation, per-workspace targeting, gradual rollout, and full audit logging.

---

## Migration: Env Vars to DB-Backed Flags

### Current Flow (OrcestrateOS)

1. Developer edits `.env` file on server.
2. Server restarts to load new values.
3. Flag is globally on or off -- no targeting, no rollout percentage.
4. No record of who changed the flag or when.

### Target Flow (Airlock)

1. Admin opens Feature Flags channel in Admin UI.
2. Admin flips a toggle, sets rollout %, or targets specific workspaces.
3. HTTP PATCH request updates the flag in the database.
4. Flag takes effect immediately -- no restart required.
5. Audit event is emitted with actor, timestamp, previous state, and new state.

---

## Flag Schema

Each feature flag record contains:

| Field | Type | Description |
|---|---|---|
| `flag_key` | string | Unique identifier (e.g., `extractor.tier1.enabled`) |
| `display_name` | string | Human-readable name shown in Admin UI |
| `description` | string | What the flag controls |
| `enabled` | boolean | Global default state |
| `rollout_pct` | integer (0-100) | Percentage of evaluations that return true |
| `workspace_overrides` | map | Per-workspace enable/disable overrides |
| `circuit_breaker` | object | Error threshold and cooldown configuration |
| `created_at` | timestamp | When the flag was created |
| `updated_at` | timestamp | Last modification time |
| `updated_by` | string | User ID of last modifier |

---

## HTTP Toggle Endpoint

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/admin/flags` | List all flags with current state |
| `GET` | `/api/admin/flags/:key` | Get single flag detail |
| `PATCH` | `/api/admin/flags/:key` | Update flag state (toggle, rollout %, overrides) |

### PATCH Request Body

Supports partial updates. Only included fields are modified.

| Field | Optional | Description |
|---|---|---|
| `enabled` | yes | Set global on/off |
| `rollout_pct` | yes | Set rollout percentage (0-100) |
| `workspace_overrides` | yes | Add or remove per-workspace overrides |
| `circuit_breaker` | yes | Update error threshold or cooldown |

### Response

Returns the full updated flag record plus the emitted audit event ID.

---

## Per-Workspace Targeting

Workspace overrides take precedence over the global flag state.

### Evaluation Order

1. Check `workspace_overrides` for the current workspace ID.
2. If an override exists, use it (ignore global state and rollout %).
3. If no override exists, evaluate `rollout_pct` against a deterministic hash of workspace ID + flag key.
4. If `rollout_pct` is 100, return `enabled`.
5. If `rollout_pct` is 0, return `false`.
6. Otherwise, hash and compare.

### Override Structure

| Field | Type | Description |
|---|---|---|
| `workspace_id` | string | Target workspace |
| `enabled` | boolean | Override state for this workspace |
| `reason` | string | Why this override exists |
| `set_by` | string | User who created the override |
| `set_at` | timestamp | When the override was created |

---

## Rollout Percentage

Gradual rollout uses deterministic hashing so the same workspace always gets the same result for a given flag at a given percentage.

| Rollout % | Behavior |
|---|---|
| 0 | Flag is off for all workspaces (unless workspace override says otherwise) |
| 1-99 | Deterministic hash of `workspace_id + flag_key` mod 100; if < rollout_pct, flag is on |
| 100 | Flag is on for all workspaces (unless workspace override says otherwise) |

---

## Flag Evaluation Audit Log

Every flag evaluation is optionally loggable for debugging. In production, only state-change events are logged by default.

| Event Type | Logged Fields |
|---|---|
| `flag.toggled` | flag_key, previous_enabled, new_enabled, actor, timestamp |
| `flag.rollout_changed` | flag_key, previous_pct, new_pct, actor, timestamp |
| `flag.override_set` | flag_key, workspace_id, enabled, reason, actor, timestamp |
| `flag.override_removed` | flag_key, workspace_id, actor, timestamp |
| `flag.circuit_breaker_tripped` | flag_key, error_count, threshold, cooldown_until, timestamp |

---

## Features Requiring Toggles

| Feature | Default Flag Key | Default State |
|---|---|---|
| Extractor: Party Names | `extractor.party_names.enabled` | on |
| Extractor: Dates | `extractor.dates.enabled` | on |
| Extractor: Financial Terms | `extractor.financial_terms.enabled` | on |
| Extractor: Territory | `extractor.territory.enabled` | on |
| Extractor: Rights | `extractor.rights.enabled` | on |
| Extractor: Obligations | `extractor.obligations.enabled` | on |
| Extractor: Metadata | `extractor.metadata.enabled` | on |
| OCR Pipeline | `ocr.enabled` | on |
| Preflight | `preflight.enabled` | on |
| Identity Resolver | `identity_resolver.enabled` | on |
| AI Flags | `ai_flags.enabled` | off |
| Contract Generator | `contract_generator.enabled` | on |
| Health Scoring | `health_scoring.enabled` | on |
| Batch Processor | `batch_processor.enabled` | on |
| Export | `export.enabled` | on |
| Lifecycle Transitions | `lifecycle.transitions.enabled` | on |
| Otto Chat (FAB) | `otto.chat.enabled` | off |

---

## Circuit Breaker

When a feature's error rate exceeds a configured threshold, the circuit breaker automatically disables the flag and emits an alert.

| Parameter | Default | Description |
|---|---|---|
| `error_threshold` | 10 | Number of errors in the evaluation window |
| `evaluation_window` | 60s | Time window for counting errors |
| `cooldown_period` | 300s | How long the flag stays disabled after tripping |
| `auto_recover` | true | Whether the flag re-enables after cooldown expires |
