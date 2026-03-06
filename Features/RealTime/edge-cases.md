# Real-Time Edge Cases & Resilience -- Design Spec

> **Status:** SPECCED
> **Source:** RealTime overview.md, WebSocket RFC 6455, production incident patterns

## Summary

This spec catalogs the edge cases, failure modes, and resilience strategies required to make Airlock's real-time WebSocket layer production-grade. It covers authentication refresh during live connections, subscription limits, rate limiting, reconnection replay, message ordering, compression, and lifecycle corner cases that the parent overview intentionally leaves abstract. Each section defines exact thresholds, close codes, and client/server responsibilities so an engineer can implement without ambiguity.

---

## 1. Authentication During Active Connections

Access tokens expire every 15 minutes. WebSocket connections routinely outlast a single token.

**Refresh flow:**

1. Client tracks token expiry locally. At T-60 seconds before expiry, client sends:
   ```json
   { "type": "auth:refresh", "refresh_token": "<opaque_token>" }
   ```
2. Server validates the refresh token, issues a new access token, and updates the connection's `UserContext` in `ConnectionManager.contexts`.
3. Server responds:
   ```json
   { "type": "auth:refreshed", "expires_at": "2026-03-05T11:15:00Z" }
   ```
4. If the refresh token is invalid or expired, server responds:
   ```json
   { "type": "auth:expired", "reason": "refresh_token_invalid" }
   ```
   Then closes the connection with close code **4401** (Unauthorized). Client must re-authenticate via the login flow and establish a new WebSocket connection.

**Race condition handling:** If an event arrives between token expiry and a successful refresh, the server queues up to 50 events for that connection (held in memory, not Redis). After `auth:refreshed`, the server flushes the queued events in order. If the queue exceeds 50 events or the refresh does not complete within 10 seconds, the server drops the queue and sends `auth:expired`.

---

## 2. Per-Connection Subscription Limits

Each WebSocket connection may subscribe to a maximum number of topics to prevent resource exhaustion on the server.

| Connection Type | Max Subscriptions |
|----------------|-------------------|
| Standard user  | 50                |
| Admin (event bus monitoring) | 200 |

**Enforcement:**

- When a `subscribe` message would exceed the limit, the server responds:
  ```json
  { "type": "error:subscription_limit", "current": 50, "max": 50, "topic": "vault:overflow-example" }
  ```
  The subscription is **not** created.
- Client should unsubscribe from stale topics (vaults no longer visible, previous chamber views) before subscribing to new ones.
- `ConnectionManager` tracks subscription count per `connection_id`. The check is O(1) against a counter, not a set-size calculation.

---

## 3. Inbound Message Rate Limiting

Clients send `subscribe`, `unsubscribe`, `auth:refresh`, and `ping` messages. These are rate-limited to prevent abuse.

| Threshold | Value |
|-----------|-------|
| Max inbound messages per second | 10 |
| Warning budget before disconnect | 3 warnings |
| Close code for persistent violation | 4429 (Rate Limited) |

**Mechanism:**

- Sliding window counter per connection, tracked in-process memory (not Redis -- the overhead of a Redis call per inbound message defeats the purpose).
- First 3 violations in a 60-second window: server sends `{ "type": "warning:rate_limit", "remaining_warnings": N }`.
- Fourth violation within the same window: server closes the connection with close code **4429**.
- The counter resets after 60 seconds of no violations.

---

## 4. Subscription Replay (Catching Up After Reconnect)

When a client reconnects after a brief disconnect, it can request missed events rather than doing a full state refresh.

**Protocol:**

1. Client reconnects and re-authenticates.
2. For each previously subscribed topic, client sends:
   ```json
   { "type": "replay:request", "topic": "vault:henderson-msa", "last_event_id": "evt_abc123" }
   ```
3. Server queries the Redis STREAM for the topic using `XRANGE <topic_stream> <last_event_id> +`.
4. Server sends missed events in order, then:
   ```json
   { "type": "replay:complete", "topic": "vault:henderson-msa", "events_replayed": 12 }
   ```

**Limits:**

| Constraint | Value |
|-----------|-------|
| Max replay window | 5 minutes from disconnect |
| Max events per replay | 100 |
| Redis STREAM retention | 1 hour (MAXLEN ~10,000 per topic) |

- If the gap exceeds 5 minutes or 100 events, server responds:
  ```json
  { "type": "replay:expired", "topic": "vault:henderson-msa", "reason": "gap_too_large" }
  ```
  Client must perform a full state refresh via the REST API for that topic's data.

**Implementation note:** This requires a Redis STREAM alongside the existing Redis Pub/Sub. Pub/Sub remains the primary delivery path; the STREAM serves only as a short-term replay buffer. `EventEmitter.emit_event()` writes to both `PUBLISH` and `XADD` in a single pipeline call.

---

## 5. Message Ordering Guarantees

| Scope | Guarantee |
|-------|-----------|
| Within a single topic | Ordered by Redis STREAM ID (monotonic, lexicographic) |
| Across topics | No ordering guarantee -- events on different topics may arrive out of order |

**Client-side sequencing:**

- Each event carries a `sequence_id` (integer, monotonically increasing per topic, assigned by the server at `XADD` time).
- Client tracks `last_sequence_id` per topic. If a received event has `sequence_id <= last_sequence_id`, the client drops it silently.
- This handles duplicates from reconnect replays or multi-path delivery (e.g., event arrives via both `vault:{id}` and `module:{id}` topics).

**Duplicate detection:**

- Client maintains a bounded Set of the last 100 event IDs per topic.
- Any event whose `id` is already in the Set is ignored.
- The Set evicts the oldest entry when it exceeds 100 (FIFO).

---

## 6. Compression Strategy

WebSocket per-message compression (RFC 7692, `permessage-deflate`) reduces bandwidth for JSON-heavy event payloads.

| Setting | Value |
|---------|-------|
| Extension | `permessage-deflate` |
| Compression level | 6 (zlib, balances CPU vs. bandwidth) |
| Minimum message size for compression | 128 bytes |
| Expected bandwidth reduction | 60-70% for typical JSON payloads |

- Messages smaller than 128 bytes skip compression (the deflate header overhead exceeds savings).
- Server advertises `permessage-deflate` in the WebSocket handshake. Clients that do not support it receive uncompressed frames.
- Starlette's WebSocket implementation supports `permessage-deflate` via `uvicorn --ws websockets` (the `websockets` library, not `wsproto`).

---

## 7. Connection Lifecycle Edge Cases

| Scenario | Server Behavior | Client Behavior |
|----------|----------------|-----------------|
| Server restart (deploy) | Close all connections with code **1012** (Service Restart) | Auto-reconnect with exponential backoff; replay missed events |
| Client tab hidden (Page Visibility API) | No change -- connection persists | Pause DOM rendering; queue received events in memory |
| Client tab restored | No change | Flush queued events to Zustand store; re-render UI |
| Network switch (WiFi to cellular) | Connection drops silently (no close frame) | Detect via missed server ping; trigger reconnect |
| Redis Pub/Sub failure | Log error; continue serving already-connected clients with no new events | No visible impact unless events stop for > heartbeat interval |
| Worker OOM kill | OS terminates process; TCP connection hangs until keepalive timeout | Detect via heartbeat timeout (90s); trigger reconnect |
| Browser crash / hard close | No close frame sent | On next app load, fresh connect (no replay -- treat as cold start) |

**Tab visibility optimization:** When `document.visibilityState === "hidden"`, the client continues receiving WebSocket frames but does not update React state or trigger re-renders. Events are buffered in a memory queue (max 500 events). When the tab becomes visible again, the queue is flushed into the Zustand store in a single batch update to avoid cascading re-renders.

---

## 8. Heartbeat Protocol Detail

| Parameter | Value |
|-----------|-------|
| Server ping interval | 30 seconds |
| Client pong deadline | 10 seconds after server ping |
| Missed pongs before server closes | 3 consecutive |
| Client-side ping interval | 30 seconds (independent timer) |
| Missed server pings before client reconnects | 2 consecutive |

**Server-side:** Uses WebSocket protocol-level ping/pong frames (opcode 0x9 / 0xA), not application-level JSON messages. This keeps heartbeat traffic minimal and works even if the application message handler is blocked.

**Client-side:** The browser's WebSocket API automatically responds to protocol-level pings with pongs. The client also runs its own 30-second timer. If 2 expected server pings are missed (60 seconds of silence), the client proactively closes the connection and begins the reconnect sequence from the backoff table in the parent spec.

---

## 9. Testing Strategy

**Unit tests:** Use the `websockets` library test client to mock WebSocket connections against the FastAPI app. Test cases:
- Token refresh during active connection (happy path and expired refresh token)
- Subscription limit enforcement (50th and 51st subscribe)
- Rate limit escalation (warnings then disconnect at 4429)
- Replay request with valid and expired gaps

**Integration tests:** Docker Compose with Redis 7 + FastAPI. Real WebSocket connections verify:
- Multi-worker fan-out (emit on worker A, receive on worker B)
- Redis STREAM replay after simulated disconnect
- Heartbeat timeout and automatic cleanup

**Chaos tests:** Use `toxiproxy` to simulate failure conditions:
- Kill Redis mid-stream -- verify graceful degradation (no crash, events resume after Redis recovery)
- Restart FastAPI server during active connections -- verify clients reconnect and replay
- Inject 5-second network latency -- verify heartbeat timeout fires correctly
- Simulate 50% packet loss -- verify message ordering holds via `sequence_id`

**Load tests:** Target thresholds:
- 1,000 concurrent WebSocket connections per worker
- 100 events/second aggregate throughput across all topics
- P50 delivery latency < 50ms, P99 < 200ms
- Zero dropped events under sustained load (verified via `sequence_id` gap detection)

---

## Related Specs

- [RealTime Overview](./overview.md) -- Parent spec with architecture, topic patterns, and connection lifecycle
- [EventBus](../EventBus/overview.md) -- Server-side event routing (BullMQ + Trigger.dev) that feeds the real-time layer
- [Auth](../Auth/overview.md) -- JWT token lifecycle and refresh token strategy
- [Security](../Security/overview.md) -- Encryption requirements for event payloads in transit and at rest
