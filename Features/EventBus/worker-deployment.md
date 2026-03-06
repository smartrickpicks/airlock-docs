# Worker Deployment & Operations — Design Spec

> **Status:** SPECCED
> **Source:** EventBus overview.md, BullMQ Python SDK docs, Docker Compose patterns

## Summary

Operational specification for deploying and scaling BullMQ workers in Airlock. Workers run as dedicated processes outside FastAPI, providing crash isolation, independent scaling, and dedicated CPU for event processing across all six module queues defined in the EventBus overview. This document covers the worker process model, Docker Compose configuration, idempotency enforcement, cross-workspace security boundaries, the DLQ admin interface, performance targets, monitoring, and scaling strategy.

---

## 1. Worker Deployment Model

### Why Separate Worker Processes

Workers MUST run as standalone processes, not as FastAPI `BackgroundTasks`. The reasons:

| Concern | BackgroundTasks | Dedicated Worker Process |
|---------|----------------|--------------------------|
| **Scaling** | Tied to API replica count | Scale workers independently of API |
| **Crash isolation** | Worker crash kills the API request cycle | Worker crash has zero impact on API availability |
| **CPU contention** | Handler computation competes with request handling | Dedicated CPU/memory per worker |
| **Deployment** | Redeploying workers requires API downtime | Workers roll independently |
| **Observability** | Metrics mixed with API metrics | Isolated metrics, clear resource attribution |

### Process Architecture

```
Docker Compose
├── api          (FastAPI — publishes events, serves HTTP)
├── worker-1     (BullMQ worker — consumes queues, runs handlers)
├── worker-2     (BullMQ worker — replica for throughput)
├── redis        (Queue backend + idempotency store)
└── postgres     (Persistent storage — handlers query here)
```

Each worker process registers handlers for all six queues from the EventBus overview: `crm-events`, `tasks-events`, `calendar-events`, `notifications-events`, `admin-events`, and `analytics-events`. Queue-specific concurrency limits are enforced per-worker via BullMQ configuration.

---

## 2. BullMQ Python SDK Setup

### Worker Bootstrap

```python
# src/workers/main.py

import asyncio
import signal
from bullmq import Worker
from redis.asyncio import ConnectionPool, Redis

from src.workers.registry import HANDLER_REGISTRY
from src.workers.config import WORKER_CONFIG
from src.core.logging import get_logger

logger = get_logger("worker")

# Connection pool shared across all queue workers
redis_pool = ConnectionPool.from_url(
    WORKER_CONFIG.redis_url,
    max_connections=20,
    decode_responses=True,
)

async def process_job(job):
    """Route a job to its registered handler by event_type."""
    event_type = job.data.get("event_type")
    handler = HANDLER_REGISTRY.get(event_type)
    if not handler:
        logger.warning(f"No handler for event_type={event_type}, skipping")
        return

    workspace_id = job.data.get("workspace_id")
    logger.info(f"Processing {event_type} for workspace={workspace_id}")
    await handler.handle(job.data)

# Queue definitions mirror the six queues from EventBus overview
QUEUE_CONCURRENCY = {
    "crm-events":           5,
    "tasks-events":         10,
    "calendar-events":      5,
    "notifications-events": 20,
    "admin-events":         3,
    "analytics-events":     3,
}

async def main():
    redis = Redis(connection_pool=redis_pool)
    workers = []

    for queue_name, concurrency in QUEUE_CONCURRENCY.items():
        w = Worker(queue_name, process_job, {
            "connection": redis,
            "concurrency": concurrency,
        })
        workers.append(w)
        logger.info(f"Started worker for {queue_name} (concurrency={concurrency})")

    # Graceful shutdown: drain current jobs, then exit
    shutdown_event = asyncio.Event()

    def handle_signal():
        logger.info("SIGTERM received — draining active jobs")
        shutdown_event.set()

    loop = asyncio.get_running_loop()
    loop.add_signal_handler(signal.SIGTERM, handle_signal)
    loop.add_signal_handler(signal.SIGINT, handle_signal)

    await shutdown_event.wait()

    for w in workers:
        await w.close()
    await redis_pool.disconnect()
    logger.info("All workers shut down cleanly")

if __name__ == "__main__":
    asyncio.run(main())
```

### Graceful Shutdown Behavior

1. Kubernetes or Docker sends `SIGTERM` to the worker process.
2. Worker stops accepting new jobs from all queues.
3. Active jobs (currently mid-handler) are allowed to complete up to the 30-second timeout.
4. After all active jobs finish (or timeout), the process exits with code 0.
5. Docker Compose `stop_grace_period: 45s` ensures the container is not killed before drain completes.

---

## 3. Docker Compose Addition

```yaml
# docker-compose.yml (additions to existing services)

worker:
  build:
    context: ./apps/api
    dockerfile: Dockerfile
  command: python -m src.workers.main
  environment:
    - REDIS_URL=redis://redis:6379/0
    - DATABASE_URL=postgresql+asyncpg://airlock:${DB_PASSWORD}@postgres:5432/airlock
    - WORKER_CONCURRENCY=5
    - LOG_LEVEL=info
  depends_on:
    redis:
      condition: service_healthy
    postgres:
      condition: service_healthy
  deploy:
    replicas: 2
    resources:
      limits:
        memory: 256M
        cpus: "0.5"
      reservations:
        memory: 128M
        cpus: "0.25"
  stop_grace_period: 45s
  restart: unless-stopped
  healthcheck:
    test: ["CMD", "python", "-c", "import redis; redis.Redis(host='redis').ping()"]
    interval: 15s
    timeout: 5s
    retries: 3
```

Workers share the same Docker image as the API (`./apps/api`) but run a different entrypoint (`python -m src.workers.main` instead of `uvicorn`). This keeps the build pipeline simple and ensures workers have access to the same handler code and database models.

---

## 4. Idempotency Key Pattern

Every event is processed at-most-once via a deterministic idempotency key stored in Redis.

### Key Format

```
idempotency:{event_type}:{source_vault_id}:{timestamp_ms}
```

Example: `idempotency:contract.shipped:vault_abc123:1709654400000`

### Enforcement Flow

```python
# src/workers/idempotency.py

IDEMPOTENCY_TTL = 86400  # 24 hours

async def is_duplicate(redis: Redis, event: dict) -> bool:
    key = f"idempotency:{event['event_type']}:{event['source_vault_id']}:{event['timestamp']}"
    was_set = await redis.set(key, "1", nx=True, ex=IDEMPOTENCY_TTL)
    return was_set is None  # None means key already existed
```

The `process_job` function checks `is_duplicate()` before dispatching to the handler. If the event is a duplicate, the job completes successfully without re-executing the handler. The 24-hour TTL ensures keys do not accumulate indefinitely while providing protection across the retry window (max retry span is ~2.5 minutes with the 5s/30s/120s backoff).

---

## 5. Cross-Workspace Isolation

Airlock enforces strict workspace boundaries. The event bus is a potential cross-workspace leak vector and must be hardened.

### Rules

1. **Every event payload includes `workspace_id`** — this is set at publish time by the API layer and is immutable.
2. **Workers validate `workspace_id` on every job** — the handler confirms the source vault belongs to the claimed workspace before processing.
3. **No cross-workspace event routing** — the BullMQ router does not fan out events to queues belonging to other workspaces. All six queues are workspace-agnostic (shared infrastructure), but handler logic is workspace-scoped.
4. **RLS enforced at the database layer** — every SQL query within a handler sets the `workspace_id` context variable, and PostgreSQL Row-Level Security policies filter results. A handler cannot read or write data belonging to another workspace even if a malformed event contains a wrong `workspace_id`.
5. **Audit trail** — workspace boundary violations (vault lookup returning a different `workspace_id` than the event claims) are logged as security events and the job is rejected to the DLQ.

```python
# Inside process_job, before handler dispatch:
vault = await get_vault(job.data["source_vault_id"])
if vault.workspace_id != job.data["workspace_id"]:
    logger.error(f"Workspace mismatch: event={job.data['workspace_id']}, vault={vault.workspace_id}")
    raise WorkspaceViolationError("Cross-workspace event rejected")
```

---

## 6. DLQ Admin UI

The Dead Letter Queue admin interface lives at `/admin/event-bus`, gated behind the `ENABLE_EVENT_BUS_ADMIN` feature flag (see Feature Control Plane spec).

### Visibility

| Column | Source |
|--------|--------|
| Queue name | BullMQ queue identifier |
| Job ID | BullMQ job ID |
| Event type | `event_type` from payload |
| Source vault | `source_vault_id` with link to vault |
| Error message | Last error from failed attempt |
| Retry count | Number of attempts (out of 3 max) |
| Failed at | Timestamp of final failure |
| Payload preview | Truncated JSON (first 200 chars), expandable |

### Actions

- **Retry single job** — re-enqueue one job from the DLQ back to its source queue.
- **Retry all** — re-enqueue all DLQ jobs for a given queue (confirmation dialog required).
- **Purge DLQ** — delete all jobs from a queue's DLQ (confirmation dialog, audit logged).
- **Export failures** — download DLQ contents as JSON for offline debugging.

### Real-Time Updates

The DLQ admin view subscribes to a WebSocket topic `admin:event-bus` to receive live updates when jobs enter or leave the DLQ. This uses the same WebSocket infrastructure defined in the RealTime spec. Only users with the `admin` role receive these events.

---

## 7. Performance Targets

| Metric | Target | Monitoring | Alert Threshold |
|--------|--------|------------|-----------------|
| Events/second throughput | 100 events/sec per worker | Prometheus counter `event_bus_processed_total` | < 50 events/sec sustained 5 min |
| P99 processing latency | < 500ms for standard events | Prometheus histogram `event_bus_duration_seconds` | P99 > 1s for 5 min |
| DLQ rate | < 0.1% of total events | Ratio of `event_bus_failed_total` / `event_bus_processed_total` | > 1% in any 5-min window |
| Worker memory | < 256MB per worker process | Docker memory limit + `process_resident_memory_bytes` | > 200MB (warning) |
| Queue depth | < 1,000 pending jobs (normal) | Prometheus gauge `event_bus_queue_depth` | > 5,000 pending jobs |
| Graceful shutdown drain | < 45 seconds | Docker `stop_grace_period` | Process killed by SIGKILL (logged) |

---

## 8. Monitoring & Alerting

### Prometheus Metrics

Workers expose metrics at `http://worker:{METRICS_PORT}/metrics` via `prometheus_client`:

| Metric Name | Type | Labels | Description |
|-------------|------|--------|-------------|
| `event_bus_processed_total` | Counter | `queue`, `event_type`, `status` | Total jobs processed (success/failure) |
| `event_bus_failed_total` | Counter | `queue`, `event_type`, `error_class` | Total jobs that exhausted retries |
| `event_bus_duration_seconds` | Histogram | `queue`, `event_type` | Handler execution time (buckets: 10ms to 30s) |
| `event_bus_queue_depth` | Gauge | `queue` | Current pending + active jobs per queue |
| `event_bus_dlq_depth` | Gauge | `queue` | Current DLQ size per queue |
| `event_bus_idempotency_hits` | Counter | `queue` | Duplicate events skipped |

### Development Tooling

- **bull-board** — visual queue inspection dashboard, enabled only when `AIRLOCK_ENV=development`. Mounted at `/dev/queues` on the API process (not the worker) for convenience.

### Grafana Dashboard (Future)

A pre-built Grafana dashboard template will visualize: queue throughput over time, P50/P95/P99 latency distributions, DLQ accumulation trends, and per-module event flow. This is deferred to the infrastructure buildout chamber.

---

## 9. Scaling Strategy

### Horizontal Scaling

Increase worker replicas in Docker Compose:

```yaml
deploy:
  replicas: 4  # doubled from default 2
```

BullMQ distributes jobs across all workers competing on the same queue. No coordination is needed — Redis handles job locking.

### Vertical Scaling

Increase per-worker concurrency via environment variable:

```yaml
environment:
  - WORKER_CONCURRENCY=10  # doubled from default 5
```

This increases the number of jobs a single worker process handles in parallel. Monitor memory usage when increasing — each concurrent handler holds its database connection and payload in memory.

### Queue-Specific Dedicated Workers

For high-throughput queues (e.g., `notifications-events` at concurrency 20), deploy dedicated worker instances that only consume that queue. This prevents notification processing from starving lower-concurrency queues:

```yaml
worker-notifications:
  extends: worker
  command: python -m src.workers.main --queues notifications-events
  deploy:
    replicas: 3
```

### Auto-Scaling Trigger

When queue depth exceeds 5,000 pending jobs for more than 2 minutes, the monitoring system fires an alert. In container orchestration environments (Kubernetes), this maps to an HPA rule scaling worker replicas based on the `event_bus_queue_depth` metric.

---

## Related Specs

- [EventBus Overview](./overview.md) — Parent spec with event schema, routing, handler patterns, and retry policy
- [RealTime](../RealTime/overview.md) — WebSocket delivery of processed events to browser clients
- [Notifications](../Notifications/overview.md) — Novu notification workflows triggered by event bus handlers
- [BatchProcessing](../BatchProcessing/overview.md) — Batch events route through the EventBus for completion/failure signaling
- [Feature Control Plane](../FeatureControlPlane/overview.md) — `ENABLE_EVENT_BUS_ADMIN` flag gates the DLQ admin UI
- [Security](../Security/overview.md) — Workspace isolation, RLS enforcement, audit trail requirements
