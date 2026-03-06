# Failure Detection -- Design Spec

## Purpose

Failure detection is the automated monitoring and response system that detects when features malfunction, surfaces failures in the Admin UI before they are found in code or logs, and provides circuit-breaker behavior to prevent cascading failures. Where the [Toggle System](./toggle-system.md) handles on/off and rollout, and [Calibration](./calibration.md) handles threshold tuning, this spec handles **what happens when features break**.

Every health-monitored engine (14 total per [overview.md](./overview.md)) reports health status continuously. The failure detection system collects error signals, evaluates health state, trips circuit breakers when thresholds are exceeded, and routes alerts through the notification and task systems so admins can respond before users notice.

---

## Health States

Each managed feature reports one of five health states. These states drive the System Health strip in the Admin dashboard, the Failure Feed, alert routing, and circuit breaker logic.

| State | Color | Dot Style | Meaning | Trigger |
|---|---|---|---|---|
| **Healthy** | Green (`#22c55e`) | Solid | Feature operating normally | Error rate < `degraded_threshold` (default 1%) |
| **Degraded** | Amber (`#f59e0b`) | Solid | Feature operating with elevated errors | Error rate >= `degraded_threshold` and < `erroring_threshold` |
| **Erroring** | Red (`#ef4444`) | Pulsing | Feature failing frequently | Error rate >= `erroring_threshold` (default 10%) OR circuit breaker tripped |
| **Disabled** | Gray (`#6b7280`) | Solid | Feature intentionally turned off via toggle | Feature flag `enabled = false` (manual or rollout 0%) |
| **Unknown** | Dim gray (`#374151`) | Hollow ring | No health data available | Feature not yet evaluated in current window |

### State Priority

When multiple conditions apply, the highest-severity state wins:

```
Erroring > Degraded > Healthy > Unknown > Disabled
```

A disabled feature does not report health data. If a feature is both disabled AND has residual errors from before disable, state = Disabled (the toggle takes precedence).

---

## Error Classification

Not all exceptions are equal. The failure detection system classifies errors into four types to enable accurate health evaluation and meaningful Admin UI filtering.

| Error Type | Badge Color | Counts Toward Health? | Description |
|---|---|---|---|
| `feature_error` | Red | Yes | The feature itself broke -- bug in logic, unhandled edge case, null reference |
| `input_error` | Amber | No | Bad input data -- malformed PDF, missing required fields, invalid format |
| `timeout_error` | Orange | Yes | Downstream dependency is slow -- OCR service, LLM API, database query timeout |
| `config_error` | Purple | Yes | Misconfiguration -- invalid calibration value, missing API key, broken template |

### Why Input Errors Don't Count

Input errors reflect data quality, not feature health. A 50% input error rate on a batch of corrupted PDFs does not mean the extraction engine is broken. Input errors are logged and visible in the Failure Feed but excluded from health state calculation and circuit breaker evaluation.

---

## Error Collection Pipeline

### Error Flow

```
Feature Engine Call
    |
    +-- try/catch wrapper (every engine call)
    |       |
    |       +-- Success: increment success counter
    |       |
    |       +-- Failure: classify error type
    |               |
    |               +-- Build error event
    |               |
    |               +-- Write to Redis sorted set (score = timestamp)
    |               |
    |               +-- Increment per-feature error counter
    |               |
    |               +-- Evaluate circuit breaker
    |
    +-- Health evaluator (runs every evaluation_window)
            |
            +-- Calculate error rate per feature
            |
            +-- Update health state
            |
            +-- Emit state-change audit event (if state changed)
            |
            +-- Route alerts (if threshold crossed)
```

### Error Event Structure

Every error captured by the health collector contains:

| Field | Type | Description |
|---|---|---|
| `event_id` | uuid | Unique error event identifier |
| `feature_key` | string | Feature flag key (e.g., `preflight.enabled`) |
| `error_type` | enum | `feature_error`, `input_error`, `timeout_error`, `config_error` |
| `error_message` | string | Human-readable error description |
| `error_class` | string | Exception class name (e.g., `ExtractionError`, `TimeoutError`) |
| `stack_trace` | string | Full stack trace (truncated to 5,000 chars) |
| `input_context` | object | Sanitized input data -- vault_id, document_id, field name. NO PII, NO file contents |
| `timestamp` | timestamp | When the error occurred (UTC) |
| `workspace_id` | string | Workspace where the error occurred |
| `vault_id` | string | Vault that triggered the error (nullable for system-level errors) |
| `duration_ms` | integer | How long the call ran before failing |
| `suggested_fix` | string | Optional auto-generated suggestion (e.g., "Check OCR service connectivity") |

### Redis Storage

Errors are stored in Redis sorted sets for fast windowed aggregation:

| Key Pattern | Score | Value | TTL |
|---|---|---|---|
| `health:errors:{feature_key}` | Unix timestamp | Serialized error event | 1 hour |
| `health:counts:{feature_key}:{window}` | -- | Error count integer | 5 minutes |
| `health:state:{feature_key}` | -- | Current health state enum | No TTL (persistent) |
| `health:breaker:{feature_key}` | -- | Circuit breaker state object | No TTL (persistent) |

### Aggregation

Error counts are aggregated per feature per evaluation window:

- **Evaluation window** defaults to 60 seconds (configurable via `circuit_breaker_window`).
- At each window boundary, the evaluator counts errors of types that count toward health (`feature_error`, `timeout_error`, `config_error`).
- Error rate = `(health_counted_errors / total_evaluations) * 100` within the window.
- If `total_evaluations` is zero in the window, state = Unknown.

---

## Circuit Breaker Logic

The circuit breaker is a state machine that automatically disables a feature when it fails too frequently, then cautiously tests recovery after a cooldown period.

### State Machine

```
                 error_count >= threshold
    +--------+  (in evaluation window)   +---------+
    |        | ------------------------> |         |
    | CLOSED |                           |  OPEN   |
    |        | <---------+               |         |
    +--------+           |               +---------+
        ^                |                    |
        |           N successes               | cooldown expires
        |           in half-open              |
        |                |                    v
        |           +-----------+             |
        |           |           | <-----------+
        +---------- | HALF-OPEN |
         (success)  |           | -----------+
                    +-----------+            |
                         |                   |
                         | any error         |
                         +----> OPEN (re-trip)
```

### State Transitions

| From | To | Trigger | Action |
|---|---|---|---|
| **Closed** | **Open** | Error count in window >= `circuit_breaker_count` (default 10 errors in 60s) | Feature returns fallback response. Emit `flag.circuit_breaker_tripped` audit event. Create URGENT task. Notify workspace admins via Novu. |
| **Open** | **Half-Open** | Cooldown period expires (`circuit_breaker_cooldown`, default 300s) | Allow next `recovery_success_count` evaluations through to test recovery. |
| **Half-Open** | **Closed** | Next N evaluations succeed (`recovery_success_count`, default 3 consecutive successes) | Resume normal operation. Emit `flag.circuit_breaker_recovered` audit event. Resolve URGENT task. |
| **Half-Open** | **Open** | Any single error during half-open testing | Re-trip the breaker. Restart cooldown timer. Emit `flag.circuit_breaker_retripped` audit event. |

### Breaker State Record

Stored in Redis at `health:breaker:{feature_key}`:

| Field | Type | Description |
|---|---|---|
| `state` | enum | `closed`, `open`, `half_open` |
| `tripped_at` | timestamp | When the breaker last tripped (null if closed) |
| `cooldown_until` | timestamp | When the cooldown expires (null if closed) |
| `error_count_in_window` | integer | Current error count in the active window |
| `half_open_successes` | integer | Consecutive successes during half-open (resets on error) |
| `trip_count` | integer | Total number of times this breaker has tripped (lifetime) |
| `last_error` | object | Most recent error event that contributed to the trip |

### Manual Override

Admins can force a circuit breaker state from the Admin UI:

| Action | Effect |
|---|---|
| **Force Close** | Immediately close the breaker and resume normal operation. Emits `flag.circuit_breaker_force_closed` audit event. |
| **Force Open** | Immediately trip the breaker regardless of error rate. Emits `flag.circuit_breaker_force_opened` audit event. |
| **Reset Counter** | Clear the error count without changing state. Useful after a known transient issue. |

---

## Fallback Strategies

When a circuit breaker is Open, the feature cannot execute normally. Each feature defines a fallback strategy that determines what happens instead.

| Strategy | Behavior | Best For |
|---|---|---|
| `return_last_good` | Return the cached result from the last successful execution for this vault | Scoring, analytics -- stale data is better than no data |
| `return_default` | Return a hardcoded safe default value defined in the feature config | Gates, status fields -- neutral state that doesn't block or auto-pass |
| `return_error` | Return an explicit error to the caller with a user-facing message | User-initiated actions -- user needs to know it failed |
| `skip_silently` | Act as if the feature is disabled; skip execution, no error surfaced | Background processing -- items stay queued, nothing breaks |

### Per-Feature Fallback Configuration

| Feature | Flag Key | Fallback Strategy | User-Visible Effect |
|---|---|---|---|
| **Extraction** | `extractor.*.enabled` | `return_default` | Fields show "Not extracted" with gray status badge. Manual entry remains available. |
| **Preflight** | `preflight.enabled` | `return_default` | Gate shows "Pending" -- does not block advancement, does not auto-pass. Manual review prompt shown. |
| **Health Scoring** | `health_scoring.enabled` | `return_last_good` | Last known health score displayed with "stale" badge and timestamp of last successful evaluation. |
| **Identity Resolver** | `identity_resolver.enabled` | `return_default` | Entity shows "Unresolved" with manual resolution prompt. Counterparty field editable. |
| **Otto AI Agent** | `otto.chat.enabled` | `return_error` | Context panel shows "AI unavailable -- service recovering" message. Chat input disabled with tooltip. |
| **Contract Generator** | `contract_generator.enabled` | `return_error` | Error banner: "Contract generation temporarily unavailable" with retry button. |
| **Batch Processor** | `batch_processor.enabled` | `skip_silently` | Queued items remain in queue. No new items processed. Admin alerted via System Health strip. |
| **Export (PDF)** | `export.pdf.enabled` | `return_error` | Export dialog shows failure message with retry button. Other export formats remain available. |
| **Export (CSV)** | `export.csv.enabled` | `return_error` | Export dialog shows failure message with retry button. |
| **Export (JSON)** | `export.json.enabled` | `return_error` | Export dialog shows failure message with retry button. |
| **Export (DOCX)** | `export.docx.enabled` | `return_error` | Export dialog shows failure message with retry button. |
| **OCR Pipeline** | `ocr.enabled` | `return_error` | Upload flow shows "OCR unavailable" error. Document stored but not processed. |
| **Lifecycle Transitions** | `lifecycle.transitions.enabled` | `return_default` | State transitions blocked. Current state preserved. Admin notified. |
| **AI Flags** | `ai_flags.enabled` | `skip_silently` | No AI-generated flags appear. Manual flagging unaffected. |

---

## Admin UI: System Health Strip

The System Health strip is the primary monitoring surface in the Workspace Admin Dashboard. It provides at-a-glance status for all 14 health-monitored engines.

### Layout

```
+------------------------------------------------------------------+
| SYSTEM HEALTH                                          [Refresh]  |
|------------------------------------------------------------------|
| [!] Preflight  | [!] Identity  |  Extraction  |  Health Score    |
|  ** ERRORING   |  * DEGRADED   |   HEALTHY    |   HEALTHY        |
|  23 errors/hr  |  7 errors/hr  |   0 err/hr   |   0 err/hr       |
|  ~~~~ 245rpm   |  ~~~~ 120rpm  |   ~~~~ 890rpm|   ~~~~ 340rpm    |
|  42ms avg      |  180ms avg    |   12ms avg   |   8ms avg        |
|  2m ago        |  5m ago       |   --         |   --             |
|------------------------------------------------------------------|
|  Otto AI       |  Generator    |  Batch       |  Export (4)      |
|   HEALTHY      |   HEALTHY     |   HEALTHY    |   HEALTHY        |
|   0 err/hr     |   0 err/hr    |   0 err/hr   |   0 err/hr       |
|   ~~~~ 45rpm   |   ~~~~ 12rpm  |   ~~~~ 8rpm  |   ~~~~ 30rpm     |
|   320ms avg    |   95ms avg    |   2100ms avg |   55ms avg       |
|   --           |   --          |   --         |   --             |
+------------------------------------------------------------------+
```

### Card Anatomy

Each feature card displays:

| Element | Position | Content |
|---|---|---|
| **Health dot** | Top-left | Colored dot matching health state. Pulsing animation for Erroring. |
| **Feature name** | Top, bold | Engine display name |
| **Health label** | Below name, colored text | State name in caps (HEALTHY, DEGRADED, ERRORING, DISABLED, UNKNOWN) |
| **Error count** | Below label | `{count} errors/hr` -- errors in the last 60 minutes |
| **Sparkline** | Middle | Requests-per-minute over last 30 minutes. Miniature line chart, 60px wide. |
| **Avg latency** | Below sparkline | `{ms}ms avg` -- p50 latency over last 5 minutes |
| **Last error** | Bottom | Relative timestamp of most recent error (e.g., "2m ago"), or `--` if none |

### Card Sorting

Cards are sorted by severity, then by error count:

1. Erroring features (sorted by error count descending) -- always float to the left
2. Degraded features (sorted by error count descending)
3. Healthy features (alphabetical)
4. Unknown features (alphabetical)
5. Disabled features (alphabetical)

### Card Expansion (Click)

Clicking a card expands it into a detail panel below the strip:

| Section | Content |
|---|---|
| **Recent Errors** | Table of last 10 errors: timestamp, error type badge, message (truncated to 120 chars), vault link, duration |
| **Error Rate Chart** | 24-hour error rate chart with degraded/erroring threshold lines overlaid |
| **Circuit Breaker** | Current state (Closed/Open/Half-Open), trip count, last trip time, cooldown remaining (countdown) |
| **Manual Controls** | Buttons: Force Close, Force Open, Reset Counter. Each requires confirmation dialog. |
| **Fallback Config** | Current fallback strategy with description. Not editable here -- links to feature config. |
| **Quick Links** | Jump to: Feature Flag toggle, Calibration params, Audit Log (filtered to this feature) |

### Filtering

| Filter | Options |
|---|---|
| **Health state** | Checkboxes: Healthy, Degraded, Erroring, Disabled, Unknown |
| **Category** | Dropdown: All, Engines, AI, Integrations, Export |
| **Search** | Text search on feature name |

---

## Admin UI: Failure Feed

The Failure Feed is a real-time, reverse-chronological stream of error events across all features. It provides the detail that the System Health strip summarizes.

### Layout

```
+------------------------------------------------------------------+
| FAILURE FEED                    [Auto-refresh: 10s]  [Filters v]  |
|------------------------------------------------------------------|
| [RED] Preflight          feature_error              12:34:05 PM   |
|       "Phase 1 RED gate evaluation failed:                        |
|        NoneType has no attribute 'text'"                          |
|       Vault: Henderson MSA                                        |
|------------------------------------------------------------------|
| [AMB] Identity Resolver  timeout_error              12:33:42 PM   |
|       "Fuzzy match query timed out after 5000ms"                  |
|       Vault: Acme Corp Division                                   |
|------------------------------------------------------------------|
| [RED] Preflight          feature_error              12:33:18 PM   |
|       "Mojibake ratio calculation failed:                         |
|        division by zero"                                          |
|       Vault: Global Media License                                 |
|------------------------------------------------------------------|
| [PUR] Otto AI Agent      config_error               12:31:55 PM   |
|       "LiteLLM gateway returned 401: invalid API key              |
|        for model alias 'otto-default'"                            |
|       Vault: --                                                   |
+------------------------------------------------------------------+
```

### Entry Anatomy

Each entry in the Failure Feed displays:

| Element | Content |
|---|---|
| **Health badge** | Colored dot matching the feature's current health state |
| **Feature name** | Engine display name, colored by health state |
| **Error type badge** | Pill badge: `feature_error` (red), `input_error` (amber), `timeout_error` (orange), `config_error` (purple) |
| **Timestamp** | Time of the error event, formatted in workspace timezone |
| **Error message** | First 120 characters of the error message. Monospace font. |
| **Vault context** | Vault name (linked to vault) or `--` for system-level errors |

### Entry Expansion (Click)

Clicking an entry expands to show:

| Section | Content |
|---|---|
| **Full Error Message** | Complete error message, untruncated |
| **Stack Trace** | Collapsible, monospace, syntax-highlighted. First 5,000 characters. |
| **Input Context** | Sanitized input data: vault ID, document ID, field name, workspace. No PII. |
| **Duration** | How long the call ran before failing |
| **Suggested Fix** | If available, a plain-English suggestion (e.g., "Restart the OCR service" or "Check LiteLLM API key configuration") |
| **Related Errors** | Link to filter the feed to show only errors from this feature |

### Filters

| Filter | Options |
|---|---|
| **Feature** | Dropdown with all managed features |
| **Error type** | Checkboxes: feature_error, input_error, timeout_error, config_error |
| **Severity** | Health state at time of error: Degraded, Erroring |
| **Time range** | Presets: Last 15 min, Last hour, Last 6 hours, Last 24 hours. Custom range picker. |
| **Vault** | Search by vault name |

### Auto-Refresh

- Feed auto-refreshes every 10 seconds while the admin is actively viewing (tab is focused).
- A "paused" indicator appears when scrolled away from the top, preventing jarring content shifts.
- Scroll to top resumes auto-refresh.
- New entries since last refresh are highlighted with a subtle pulse for 3 seconds.

---

## Alert Thresholds

All failure detection thresholds are configurable per feature via the calibration system. These parameters are stored in the calibration table alongside other calibration records (see [Calibration](./calibration.md)).

### Parameters

| Parameter | Key Pattern | Default | Range | Unit | Description |
|---|---|---|---|---|---|
| Degraded threshold | `failure.{feature}.degraded_threshold` | 1 | 0.1 - 50.0 | % | Error rate that triggers Degraded state |
| Erroring threshold | `failure.{feature}.erroring_threshold` | 10 | 1.0 - 100.0 | % | Error rate that triggers Erroring state |
| Circuit breaker count | `failure.{feature}.circuit_breaker_count` | 10 | 1 - 100 | count | Errors in window to trip breaker |
| Circuit breaker window | `failure.{feature}.circuit_breaker_window` | 60 | 10 - 600 | seconds | Time window for counting errors |
| Circuit breaker cooldown | `failure.{feature}.circuit_breaker_cooldown` | 300 | 30 - 3600 | seconds | Cooldown before half-open test |
| Recovery success count | `failure.{feature}.recovery_success_count` | 3 | 1 - 20 | count | Consecutive successes to close breaker |

### Impact Descriptions

| Parameter | What It Controls | Raise It | Lower It |
|---|---|---|---|
| **Degraded threshold** | Error rate percentage at which the health dot turns amber and an ACTION task is created. | More tolerant -- feature can have higher error rates before being flagged. Risk: problems go unnoticed longer. | More sensitive -- feature flagged sooner. Good for critical engines. Risk: noise from transient spikes. |
| **Erroring threshold** | Error rate percentage at which the health dot turns red and an URGENT task is created. | More tolerant -- only severe failure rates trigger red status. Risk: significant failures may not surface fast enough. | More aggressive -- lower error rates trigger red. Good for zero-tolerance features. Risk: frequent false alarms. |
| **Circuit breaker count** | Absolute number of errors (not rate) needed to trip the breaker within the window. | Breaker trips less often -- tolerates bursts of errors. Good for features with naturally variable error rates. | Breaker trips sooner -- protective but may over-react to brief spikes. |
| **Circuit breaker window** | Time window over which errors are counted. | Wider window -- errors spread over more time are still counted. Catches slow-burn failures. | Narrower window -- only dense bursts of errors trip the breaker. Ignores spread-out failures. |
| **Circuit breaker cooldown** | Time the breaker stays open before attempting half-open recovery. | Longer wait before retrying -- gives more time for downstream recovery but extends outage. | Faster retry -- shorter outage but risks re-tripping if the issue is not resolved. |
| **Recovery success count** | How many consecutive successes required during half-open before closing the breaker. | More proof of recovery required -- safer but slower return to service. | Faster recovery -- closes breaker sooner but less confidence the issue is truly resolved. |

---

## Cross-Module Alerts

When features fail, the failure detection system routes alerts through two channels: the universal task system (durable, trackable) and the notification system (immediate, attention-getting).

### Alert Routing Table

| Condition | Task Created | Task Priority | Novu Notification | Recipients |
|---|---|---|---|---|
| Feature enters **Degraded** state | Yes -- Admin module triage | ACTION (amber) | No | -- |
| Feature enters **Erroring** state | Yes -- Admin module triage | URGENT (red) | No | -- |
| Circuit breaker **tripped** | Yes -- Admin module triage (if not already created by Erroring) | URGENT (red) | Yes -- `feature-breaker-tripped` template | All workspace admins |
| Feature erroring for **> 1 hour** | Escalation task (linked to original) | URGENT (red) | Yes -- `feature-escalation` template | All workspace admins + Architect role |
| Circuit breaker **recovered** | Resolve original URGENT task | -- | Yes -- `feature-recovered` template | Original task assignees |

### Task Structure

Tasks created by failure detection follow the universal task system schema (see [Task System](../TaskSystem/overview.md)):

| Field | Value |
|---|---|
| `module` | `admin` |
| `type` | `system_alert` |
| `title` | `{Feature Name}: {State}` (e.g., "Preflight: Erroring") |
| `description` | Error summary, error count, circuit breaker state, link to System Health |
| `priority` | ACTION or URGENT (per table above) |
| `auto_resolve` | true -- resolved when feature returns to Healthy |
| `created_by` | `system:failure_detection` |

### Novu Notification Templates

| Template ID | Trigger | Channel (POC) | Content |
|---|---|---|---|
| `feature-breaker-tripped` | Circuit breaker state = Open | In-app | "{Feature} circuit breaker tripped. {error_count} errors in {window}s. Fallback: {strategy}." |
| `feature-escalation` | Erroring state duration > 1 hour | In-app | "{Feature} has been erroring for {duration}. {error_count} total errors. Manual intervention may be required." |
| `feature-recovered` | Circuit breaker state = Closed (from Open/Half-Open) | In-app | "{Feature} has recovered. Circuit breaker closed after {recovery_successes} successful evaluations." |

---

## Graceful Degradation Patterns

Each feature's fallback behavior is designed to maintain user workflow continuity while making the degraded state visible. The principle: **degrade visually, never silently corrupt data**.

### Degradation Visibility

| Indicator | Where | When |
|---|---|---|
| **Stale badge** | Next to the affected value | `return_last_good` strategy is serving cached data |
| **Gray status** | Field cards in Record Inspector | `return_default` strategy returned placeholder values |
| **Error banner** | Top of the relevant view | `return_error` strategy needs user attention |
| **Queue badge** | Batch processing panel | `skip_silently` strategy has paused processing |

### Feature-Specific Degradation Details

#### Extraction (return_default)

- All extraction fields for new documents show "Not extracted" in gray italic text.
- Field cards display with a hollow status ring instead of a filled confidence dot.
- Manual entry fields remain fully functional -- users can type values directly.
- Previously extracted documents retain their existing values (no retroactive clearing).

#### Preflight (return_default)

- Gate status shows "Pending" with a gray clock icon.
- The gate does NOT auto-pass (would be dangerous -- bad documents could advance).
- The gate does NOT hard-block (would halt all work).
- A banner above the gate reads: "Preflight evaluation is temporarily unavailable. Manual review is recommended before advancing."

#### Health Scoring (return_last_good)

- Last known health score is displayed with normal formatting.
- A "stale" badge appears next to the score: "Score from {timestamp}" in muted text.
- Health band (GREEN/YELLOW/RED) uses the stale score for display but does NOT trigger auto-pass.

#### Identity Resolver (return_default)

- Entity field shows "Unresolved" with a dashed border.
- A "Resolve manually" button appears inline, opening the entity search dialog.
- Previously resolved entities retain their matches.

#### Otto AI Agent (return_error)

- The AI chat FAB shows a yellow warning icon overlay.
- Opening the context panel shows: "AI assistant is temporarily unavailable. The service is recovering and will resume automatically."
- Chat input is disabled (grayed out) with a tooltip explaining the outage.
- Previous conversation history remains visible and scrollable.

#### Batch Processor (skip_silently)

- Items in the queue remain in their current state (no items are dropped).
- The batch status panel shows: "Processing paused -- service recovering" with a progress bar frozen at its last position.
- New uploads are accepted and queued but not processed until the breaker closes.
- Admin is alerted via the System Health strip; end users see only the paused status.

---

## Audit Events

All failure detection state changes are recorded in the audit log (see [Admin / Settings](../Admin/overview.md) -- Audit Log section).

| Event Type | Logged Fields |
|---|---|
| `health.state_changed` | feature_key, previous_state, new_state, error_rate, error_count, timestamp |
| `flag.circuit_breaker_tripped` | feature_key, error_count, threshold, window, cooldown_until, last_error_message, timestamp |
| `flag.circuit_breaker_recovered` | feature_key, recovery_successes, time_in_open_state, timestamp |
| `flag.circuit_breaker_retripped` | feature_key, error_during_half_open, cooldown_until, timestamp |
| `flag.circuit_breaker_force_closed` | feature_key, actor, reason, timestamp |
| `flag.circuit_breaker_force_opened` | feature_key, actor, reason, timestamp |
| `failure.escalation_created` | feature_key, duration_erroring, error_count, escalation_task_id, timestamp |

---

## Implementation Notes

### Performance Constraints

- Health evaluation runs on a background timer, NOT on the hot path of feature execution.
- Error event writes to Redis are fire-and-forget (async, non-blocking to the calling feature).
- Health state reads are cached in memory with 5-second refresh for the Admin UI.
- Circuit breaker state checks ARE on the hot path but are a single Redis GET (sub-millisecond).

### Data Retention

| Data | Storage | Retention |
|---|---|---|
| Individual error events | Redis sorted sets | 1 hour (rolling window) |
| Error count summaries | Redis keys | 5 minutes (rolling) |
| Health state | Redis + PostgreSQL | Persistent (current state in Redis, history in PostgreSQL) |
| Circuit breaker state | Redis + PostgreSQL | Persistent |
| Audit events | PostgreSQL | Per workspace retention policy (default: 90 days) |
| Error rate history | PostgreSQL (time-series) | 30 days (for 24h chart in Admin UI) |

### Startup Behavior

- On server start, all features begin in **Unknown** state.
- After the first evaluation window (default 60s), features transition to Healthy (if no errors) or their appropriate state.
- Circuit breaker state is restored from Redis on startup (survives restarts).
- If Redis is unavailable at startup, all breakers default to **Closed** (fail-open -- features run normally).

---

## Related Specs

- [Feature Control Plane Overview](./overview.md) -- Parent spec listing all 14 engines, 27 flags, 32 calibration params
- [Toggle System](./toggle-system.md) -- DB-backed flags, HTTP toggle, per-workspace targeting, rollout %, circuit breaker parameters
- [Calibration](./calibration.md) -- Threshold and weight tuning via Admin UI
- [Admin / Settings](../Admin/overview.md) -- System Health strip and Failure Feed in the Workspace Admin dashboard
- [Notifications](../Notifications/overview.md) -- Novu templates for circuit breaker alerts, two-level notification architecture
- [Universal Task System](../TaskSystem/overview.md) -- URGENT and ACTION tasks created by failure detection alerts
