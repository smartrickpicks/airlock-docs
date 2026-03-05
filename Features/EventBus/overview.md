# Cross-Module Event Bus — Design Spec

> **Status:** SPECCED — BullMQ-powered event bus for server-side cross-module communication.

> **Key Insight:** The event bus is the nervous system of Airlock. When something happens in one module (contract shipped, entity resolved, SLA breached), the event bus routes that signal to every other module that cares. This is how the "flywheel" works — modules feed each other automatically.

> **Tech:** BullMQ (MIT, Redis-backed) for job queues + Trigger.dev (Apache-2.0) for complex multi-step workflows.

---

## Architecture

```
Module A (Contracts)                Event Bus                    Module B (CRM)
+-------------------+              +---------+                   +------------------+
| contract.shipped  | --publish--> | BullMQ  | --dispatch------> | handle_shipped() |
|                   |              | Router  |                   | create_vault()   |
+-------------------+              |         |                   +------------------+
                                   |         |
                                   |         |                   Module C (Tasks)
                                   |         | --dispatch------> +------------------+
                                   |         |                   | handle_shipped() |
                                   +---------+                   | create_task()    |
                                       |                         +------------------+
                                       |
                                       v
                                   +---------+
                                   | Audit   |  (every event logged)
                                   | Events  |
                                   +---------+
```

### Why BullMQ (Not Direct Function Calls)

| Concern | Direct calls | BullMQ |
|---------|-------------|--------|
| **Failure isolation** | Module B crash kills Module A's request | Module A completes; Module B retries independently |
| **Decoupling** | Module A must know about B, C, D | Module A publishes; bus routes to subscribers |
| **Retry** | Manual retry logic per caller | Built-in retry with backoff |
| **Observability** | Console logs | Job dashboard, timing metrics, failure counts |
| **Ordering** | No guarantees | FIFO per queue, priority support |
| **Rate limiting** | Manual | Built-in concurrency limits |

---

## Event Schema

Every cross-module event follows a standard schema:

```typescript
interface CrossModuleEvent {
  // Identity
  id: string;                    // UUID
  event_type: string;            // "contract.shipped", "entity.resolved", etc.
  workspace_id: string;

  // Source
  source_module: string;         // "contracts", "crm", "tasks"
  source_vault_id: string;       // vault that triggered this event
  source_event_id: string;       // channel_event ID that originated this

  // Payload
  payload: Record<string, any>;  // event-specific data

  // Metadata
  actor_id: string | null;       // user who triggered (null for system)
  actor_type: "human" | "system" | "ai";
  timestamp: string;             // ISO 8601
  priority: number;              // 0 (normal) to 10 (critical)
}
```

---

## Queue Design

### Queue Topology

One queue per target module. Events are dispatched by the router to the appropriate queue(s).

```
Queue: crm-events          (concurrency: 5)
Queue: tasks-events         (concurrency: 10)
Queue: calendar-events      (concurrency: 5)
Queue: notifications-events (concurrency: 20)
Queue: admin-events         (concurrency: 3)
Queue: analytics-events     (concurrency: 3)
```

### Router

The event router maps event types to target queues:

```typescript
const EVENT_ROUTES: Record<string, string[]> = {
  // Contracts module events
  "contract.shipped":           ["crm-events", "tasks-events", "calendar-events", "notifications-events"],
  "contract.failed":            ["admin-events", "notifications-events"],
  "entity.resolved":            ["crm-events"],
  "entity.new_customer":        ["crm-events", "tasks-events"],
  "extraction.completed":       ["analytics-events"],
  "health.updated":             ["notifications-events"],
  "gate.advanced":              ["notifications-events", "analytics-events"],
  "patch.submitted":            ["notifications-events"],
  "patch.approved":             ["notifications-events"],
  "batch.completed":            ["notifications-events", "analytics-events"],
  "batch.failed":               ["admin-events", "notifications-events"],

  // CRM module events
  "account.created":            ["notifications-events"],
  "account.health_changed":     ["notifications-events"],

  // Tasks module events
  "task.created":               ["notifications-events"],
  "task.overdue":               ["notifications-events", "admin-events"],

  // System events
  "sla.warning":                ["notifications-events"],
  "sla.breached":               ["notifications-events", "admin-events"],
  "feature.circuit_breaker":    ["admin-events", "notifications-events"],
};
```

---

## Event Catalog

### Contracts → Other Modules

| Event | Payload | Target Module(s) | Handler Action |
|-------|---------|-------------------|----------------|
| `contract.shipped` | `{ vault_id, entity_name, counterparty, contract_type, effective_date, term_end }` | CRM, Tasks, Calendar | CRM: update relationship. Tasks: create onboarding checklist. Calendar: create renewal date. |
| `contract.failed` | `{ vault_id, error, retry_count, batch_id }` | Admin, Notifications | Admin: increment failure count. Notifications: alert owner. |
| `entity.resolved` | `{ vault_id, entity_name, confidence, provider }` | CRM | CRM: link vault to existing account or create new. |
| `entity.new_customer` | `{ entity_name, source_vault_id, extracted_fields }` | CRM, Tasks | CRM: create parent + counterparty vault. Tasks: create "enrich account" task. |
| `extraction.completed` | `{ vault_id, field_count, pass, review, fail, confidence_avg }` | Analytics | Analytics: update extraction success metrics. |
| `health.updated` | `{ vault_id, old_score, new_score, band }` | Notifications | Notifications: alert if drop > 10%. |
| `gate.advanced` | `{ vault_id, from_gate, to_gate, actor_id }` | Notifications, Analytics | Notifications: alert assigned reviewer. Analytics: track gate velocity. |
| `patch.submitted` | `{ vault_id, patch_id, field, intent, author_id }` | Notifications | Notifications: alert assigned Gatekeeper. |
| `patch.approved` | `{ vault_id, patch_id, approver_id, stage }` | Notifications | Notifications: alert patch author. |
| `batch.completed` | `{ batch_id, total, passed, failed, duration_ms }` | Notifications, Analytics | Notifications: alert batch owner. Analytics: track batch metrics. |

### CRM → Other Modules

| Event | Payload | Target Module(s) | Handler Action |
|-------|---------|-------------------|----------------|
| `account.created` | `{ vault_id, account_name, source }` | Notifications | Notifications: alert CRM owner. |
| `account.health_changed` | `{ vault_id, old_score, new_score }` | Notifications | Notifications: alert if health declining. |

### Tasks → Other Modules

| Event | Payload | Target Module(s) | Handler Action |
|-------|---------|-------------------|----------------|
| `task.overdue` | `{ task_id, vault_id, assigned_to, overdue_by }` | Notifications, Admin | Notifications: urgent alert. Admin: SLA breach log. |

---

## Handler Pattern

Each module registers handlers for events it consumes:

```python
# server/events/handlers/crm_handler.py

class CRMEventHandler:
    """Handles cross-module events that affect the CRM module."""

    async def handle_contract_shipped(self, event: CrossModuleEvent):
        """When a contract ships, update the CRM relationship."""
        vault_id = event.payload["vault_id"]
        entity = event.payload["entity_name"]
        # Update counterparty vault aggregate metrics
        # Recalculate account health
        # Emit "account.health_changed" if score changed

    async def handle_entity_new_customer(self, event: CrossModuleEvent):
        """When a new entity is detected, create CRM vaults."""
        # Create parent vault (Level 1)
        # Create counterparty vault (Level 3)
        # Create "enrich account" task in Tasks module
        # Emit "account.created"

    async def handle_entity_resolved(self, event: CrossModuleEvent):
        """When an entity is resolved, link to existing CRM vault."""
        # Find existing counterparty vault
        # Link source vault to counterparty
        # Update aggregate metrics
```

### Handler Registration

```python
# server/events/registry.py

EVENT_HANDLERS = {
    "crm-events": CRMEventHandler(),
    "tasks-events": TasksEventHandler(),
    "calendar-events": CalendarEventHandler(),
    "notifications-events": NotificationsEventHandler(),
    "admin-events": AdminEventHandler(),
    "analytics-events": AnalyticsEventHandler(),
}
```

---

## Retry & Error Handling

| Parameter | Value |
|-----------|-------|
| **Max retries** | 3 per job |
| **Backoff** | Exponential: 5s, 30s, 120s |
| **Dead letter queue** | Failed jobs moved to `{queue}-dlq` after max retries |
| **Timeout** | 30s per handler execution |
| **Idempotency** | Handlers must be idempotent (use `event.id` as dedup key) |

### Dead Letter Queue Handling

Jobs in the DLQ are:
1. Visible in Admin > System Health > Error Feed
2. Manually retryable from Admin UI
3. Include full event payload + error context for debugging
4. Auto-purged after 30 days

---

## Observability

### Metrics (per queue)

| Metric | Description |
|--------|-------------|
| `queue.pending` | Jobs waiting to be processed |
| `queue.active` | Jobs currently being processed |
| `queue.completed` | Jobs completed (last hour) |
| `queue.failed` | Jobs failed (last hour) |
| `queue.avg_duration_ms` | Average handler execution time |
| `queue.dlq_size` | Dead letter queue depth |

### Admin Dashboard Integration

The event bus feeds the Admin > System Health section:

- **Queue Health Cards**: One card per queue showing pending/active/failed/DLQ counts
- **Event Flow Chart**: Sparkline of events processed per minute
- **Error Feed**: Failed events with error messages, retryable from the UI
- **Cross-Module Dependency Map**: Visual graph of which modules feed which

---

## Trigger.dev Integration (Complex Workflows)

For multi-step workflows that span multiple modules, use Trigger.dev instead of raw BullMQ:

| Workflow | Steps | Why Trigger.dev |
|----------|-------|-----------------|
| **Contract Onboarding** | Ship → Create CRM vault → Create tasks → Schedule dates → Notify team | 5 steps with dependencies and rollback |
| **Batch Failure Escalation** | 3 failures → Circuit breaker → Disable feature → Notify admin → Create incident task | Conditional branching |
| **SLA Breach Chain** | Warning → Notification → Escalation → Manager alert → Auto-reassign | Time-delayed steps |

Trigger.dev provides:
- Visual workflow editor (for debugging)
- Step-level retry and rollback
- Scheduled/delayed steps
- Workflow versioning

---

## Related Specs

- **[Real-Time Events](../RealTime/overview.md)** — Client-side event delivery (WebSocket). The event bus routes events server-side; WebSocket delivers them to browsers.
- **[Notifications](../Notifications/overview.md)** — Novu workflows triggered by event bus handlers.
- **[Feature Control Plane](../FeatureControlPlane/overview.md)** — Circuit breaker events flow through the bus.
- **[Admin / Settings](../Admin/overview.md)** — Admin System Health displays queue metrics.
- **[Tech Stack](../TechStack/overview.md)** — BullMQ + Trigger.dev choices.
