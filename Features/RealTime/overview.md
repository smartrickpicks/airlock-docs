# Real-Time Event Architecture — Design Spec

> **Status:** SPECCED — WebSocket layer for live event streaming across all modules.

> **Key Insight:** Every vault and every chamber view subscribes to a real-time event stream. When something happens anywhere in the system (extraction completes, patch submitted, health score changes), every connected client seeing that vault or view gets the update instantly. No polling.

> **Tech:** Starlette WebSocket (built into FastAPI) + Redis Pub/Sub for multi-process fan-out.

---

## Architecture

```
Browser Tab                    API Server (FastAPI)              Redis
+-----------+                  +-----------------------+         +-------+
| Zustand   |   WebSocket      | ws.py                |         |       |
| Store     | <=============> | - authenticate        |  Sub    |       |
|           |   JSON frames    | - register subs      | <=====> | PubSub|
| EventFeed |                  | - heartbeat           |  Pub    |       |
| HealthCard|                  |                       |         |       |
| Badges    |                  | event_emitter.py      |         |       |
+-----------+                  | - emit_event()        | ======> |       |
                               | - publishes to Redis  |         |       |
                               +-----------------------+         +-------+
```

### Why Redis Pub/Sub

- FastAPI can run multiple worker processes (uvicorn workers)
- A WebSocket connection lands on ONE worker
- When worker A emits an event, worker B's connected clients also need it
- Redis Pub/Sub bridges workers: emit → Redis → all subscribers across all workers
- Zero persistence overhead (pub/sub is fire-and-forget; events are already stored in `channel_events` table)

---

## Connection Lifecycle

### 1. Connect

```
Client → ws://api.airlock.dev/ws?token=JWT
Server:
  1. Validate JWT → extract user_id, workspace_id
  2. Register connection in ConnectionManager
  3. Send ACK frame: { type: "connected", user_id, workspace_id }
  4. Start heartbeat (ping every 30s, timeout 90s)
```

### 2. Subscribe

After connecting, client sends subscription frames:

```json
{ "type": "subscribe", "topic": "vault:henderson-msa" }
{ "type": "subscribe", "topic": "view:contracts/triage-board" }
{ "type": "subscribe", "topic": "module:contracts" }
```

**Topic patterns:**

| Pattern | Receives events from |
|---------|---------------------|
| `vault:{vault_id}` | All events for a specific vault (triptych view) |
| `view:{module}/{view}` | Aggregated events for a chamber view (triage board, review queue) |
| `module:{module_id}` | Module-level events (badge counts, status changes) |
| `workspace` | Workspace-wide events (admin actions, system health) |
| `user:{user_id}` | Personal events (task assigned, mention, SLA warning) |

### 3. Receive Events

Server pushes events as JSON frames:

```json
{
  "type": "event",
  "topic": "vault:henderson-msa",
  "event": {
    "id": "evt_abc123",
    "event_type": "extraction.completed",
    "payload": { "field_count": 42, "pass": 35, "review": 5, "fail": 2 },
    "author_type": "system",
    "created_at": "2024-03-15T10:30:00Z"
  }
}
```

### 4. Unsubscribe

```json
{ "type": "unsubscribe", "topic": "vault:henderson-msa" }
```

Client unsubscribes when navigating away from a vault or view.

### 5. Disconnect

Connection closes on tab close, network loss, or explicit disconnect. Server cleans up all subscriptions for that connection.

---

## Server-Side Components

### ConnectionManager

```python
class ConnectionManager:
    """Manages all active WebSocket connections and their subscriptions."""

    # connection_id -> WebSocket
    connections: dict[str, WebSocket]

    # topic -> set of connection_ids
    subscriptions: dict[str, set[str]]

    # connection_id -> user context (user_id, workspace_id, role)
    contexts: dict[str, UserContext]

    async def connect(ws, user_context) -> str  # returns connection_id
    async def disconnect(connection_id)
    async def subscribe(connection_id, topic)
    async def unsubscribe(connection_id, topic)
    async def broadcast(topic, event)  # send to all subscribers of a topic
```

### EventEmitter

Every backend action that produces a `channel_events` row also publishes to Redis:

```python
async def emit_event(event: ChannelEvent):
    """Store event in DB and publish to Redis for real-time delivery."""
    # 1. INSERT into channel_events (already done by the caller)
    # 2. Publish to Redis pub/sub
    await redis.publish(f"vault:{event.channel_id}", event.to_json())

    # 3. Also publish to aggregate topics
    channel = await get_channel(event.channel_id)
    await redis.publish(f"module:{channel.module_type}", event.to_json())
    await redis.publish(f"view:{channel.module_type}/triage-board", event.to_json())
```

### Redis Subscriber (per worker)

Each FastAPI worker runs a background task that subscribes to Redis and forwards events to local WebSocket connections:

```python
async def redis_listener():
    """Background task: receive Redis pub/sub messages, forward to local WebSockets."""
    pubsub = redis.pubsub()
    # Subscribe to all active topics for this worker's connections
    while True:
        message = await pubsub.get_message()
        if message and message['type'] == 'message':
            topic = message['channel']
            event = json.loads(message['data'])
            await connection_manager.broadcast(topic, event)
```

---

## Client-Side Components

### WebSocket Manager (`lib/websocket.ts`)

```typescript
class AirlockWebSocket {
  private ws: WebSocket | null = null;
  private subscriptions: Set<string> = new Set();
  private reconnectAttempts = 0;
  private maxReconnectAttempts = 10;

  connect(token: string): void
  subscribe(topic: string): void
  unsubscribe(topic: string): void
  onEvent(handler: (topic: string, event: ChannelEvent) => void): void
  disconnect(): void
}
```

### Auto-Reconnect

| Attempt | Delay | Strategy |
|---------|-------|----------|
| 1-3 | 1s, 2s, 4s | Exponential backoff |
| 4-7 | 8s, 16s, 32s, 60s | Cap at 60s |
| 8-10 | 60s | Steady retry |
| 10+ | Stop | Show "Connection lost" banner with manual retry button |

On reconnect: re-authenticate, re-subscribe to all previously subscribed topics.

### React Hook (`hooks/useRealtimeEvents.ts`)

```typescript
function useRealtimeEvents(topic: string, handler: (event: ChannelEvent) => void) {
  // Subscribes on mount, unsubscribes on unmount
  // Calls handler for each received event
  // Auto-reconnects on connection loss
}
```

---

## What Subscribes to What

| UI Component | Topic | What it updates |
|-------------|-------|-----------------|
| **Vault triptych (Signal)** | `vault:{id}` | Append new events to feed |
| **Vault triptych (Control)** | `vault:{id}` | Update health card, SLA timer, approval chain |
| **Triage Board** | `view:contracts/triage-board` | Update table rows, metrics strip |
| **Review Queue** | `view:contracts/review-queue` | Update entity cards, alert counts, feed |
| **Home triage widgets** | `module:{id}` for each enabled module | Update widget counts and rows |
| **Module bar badges** | `module:{id}` for each module | Update notification badges |
| **CRM pipeline** | `view:crm/pipeline` | Update pipeline card positions |
| **Tasks Kanban** | `view:tasks/board` | Update card positions, new cards |
| **Admin System Health** | `workspace` | Update health strip, error feed |

---

## Event Payload Conventions

All WebSocket event payloads follow the same structure as `channel_events` rows:

```typescript
interface RealtimeEvent {
  id: string;
  event_type: string;          // e.g., "extraction.completed"
  channel_id: string;          // vault ID
  payload: Record<string, any>;
  author_id: string | null;
  author_type: "human" | "ai" | "system" | "connector";
  gate: string | null;
  confidence: number | null;
  created_at: string;          // ISO 8601
}
```

### Event Types That Trigger Real-Time Updates

| Event Type | Module | Subscribers |
|-----------|--------|-------------|
| `extraction.completed` | Contracts | Vault triptych, Triage Board |
| `preflight.completed` | Contracts | Vault triptych, Triage Board, Review Queue |
| `health.updated` | Contracts | Vault triptych, Review Queue, Home widgets |
| `patch.submitted` | Contracts | Vault triptych, Review Queue, Patch Approvals |
| `patch.approved` | Contracts | Vault triptych, Review Queue, Patch Approvals |
| `gate.advanced` | Contracts | Vault triptych, Triage Board, Review Queue |
| `task.created` | Tasks | Tasks Kanban, Home widgets, module badge |
| `task.status_changed` | Tasks | Tasks Kanban, Home widgets |
| `entity.resolved` | CRM | CRM pipeline, vault triptych |
| `sla.warning` | Any | User notification, vault triptych |
| `system.health_changed` | Admin | Admin System Health |

---

## Performance Considerations

| Concern | Mitigation |
|---------|-----------|
| **Too many connections** | Connection limit per user (3 tabs max), shared connection via BroadcastChannel API |
| **Event flood** | Server-side throttling: max 10 events/second per topic. Batch low-priority events. |
| **Large payloads** | Events carry IDs and summaries, not full data. Client fetches details on-demand. |
| **Stale subscriptions** | Heartbeat timeout (90s) cleans up dead connections |
| **Redis memory** | Pub/sub is transient (no storage). Only active subscriptions consume memory. |

---

## Graceful Degradation

If WebSocket is unavailable:

| Scenario | Fallback |
|----------|----------|
| **Connection refused** | Poll `/api/events?since=last_event_id` every 5s |
| **Intermittent drops** | Auto-reconnect with exponential backoff, replay missed events via REST |
| **Redis down** | Direct worker delivery (single-worker mode, no fan-out) |

The UI always works — real-time just makes it faster.

---

## Related Specs

- **[Cross-Module Event Bus](../EventBus/overview.md)** — Server-side event routing (BullMQ). WebSocket delivers events to clients; the event bus routes events between modules.
- **[Notifications](../Notifications/overview.md)** — Notification events delivered via WebSocket to in-app notification center.
- **[Universal Task System](../TaskSystem/overview.md)** — Task events drive Home widget updates.
- **[Edge Cases & Resilience](./edge-cases.md)** — Auth refresh, subscription limits, rate limiting, replay, ordering, compression, and chaos testing.
