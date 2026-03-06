# Notification Triggers & Delivery -- Design Spec

> **Status:** SPECCED
> **Source:** Notifications overview.md, EventBus event catalog, RealTime overview.md, Novu self-hosted docs

## Summary

This spec maps every cross-module event type from the EventBus to the notification(s) it produces, defines recipient resolution rules, deduplication logic, rate limiting, and the end-to-end delivery pipeline through Novu self-hosted. It serves as the authoritative reference for which events generate notifications, who receives them, and how they are delivered.

---

## 1. Trigger Catalog

Every event routed to the `notifications-events` BullMQ queue produces one or more notifications. The tables below are the exhaustive mapping, organized by source module.

### Contracts Module Events

| Event Type | Notification Message | Priority | Recipients | Channels |
|---|---|---|---|---|
| `contract.shipped` | "{vault} shipped successfully" | INFO | Vault members | In-app |
| `contract.failed` | "{vault} processing failed: {error}" | URGENT | Vault owner + workspace admins | In-app + toast |
| `extraction.completed` | "Extraction complete: {field_count} fields ({pass} pass, {review} review, {fail} fail)" | INFO | Vault owner | In-app |
| `health.updated` (drop > 10%) | "Health score dropped from {old_score} to {new_score} on {vault}" | ACTION | Vault members | In-app + badge |
| `health.updated` (drop <= 10%) | No notification | -- | -- | -- |
| `gate.advanced` | "{vault} advanced to {to_gate}" | INFO | Vault members | In-app |
| `gate.failed` | "Gate failed on {vault}: {reason}" | ACTION | Vault owner | In-app + badge |
| `patch.submitted` | "Amendment needs review on {vault}" | ACTION | Assigned gatekeeper(s) | In-app + badge |
| `patch.approved` | "Amendment approved on {vault}" | INFO | Patch author | In-app |
| `patch.rejected` | "Amendment rejected on {vault}: {reason}" | URGENT | Patch author | In-app + toast |
| `batch.completed` | "Batch '{name}' complete: {passed}/{total} passed" | INFO | Batch creator | In-app |
| `batch.failed` | "Batch '{name}' failed: {error}" | URGENT | Batch creator + workspace admins | In-app + toast |

### CRM Module Events

| Event Type | Notification Message | Priority | Recipients | Channels |
|---|---|---|---|---|
| `account.created` | "New account created: {account_name}" | INFO | CRM module members with owner role | In-app |
| `account.health_changed` (declining) | "Account health declining: {account_name} ({old_score} -> {new_score})" | ACTION | Account owner | In-app + badge |
| `account.health_changed` (improving) | No notification | -- | -- | -- |
| `crm.deal_stage_changed` | "Deal '{deal_name}' moved to {stage}" | INFO | Deal owner | In-app |
| `crm.deal_closed_won` | "Deal closed-won: {deal_name}" | INFO | Deal owner + workspace admins | In-app |
| `crm.deal_closed_lost` | "Deal closed-lost: {deal_name}" | ACTION | Deal owner + sales manager | In-app + badge |

### Tasks Module Events

| Event Type | Notification Message | Priority | Recipients | Channels |
|---|---|---|---|---|
| `task.created` | "Task assigned: {title}" | ACTION | Assigned user | In-app |
| `task.overdue` | "Task overdue: {title} (by {overdue_by})" | URGENT | Assigned user + task creator | In-app + toast |
| `task.completed` | "Task completed: {title}" | INFO | Task creator (if different from completer) | In-app |
| `task.reassigned` | "Task reassigned to you: {title}" | ACTION | New assignee | In-app |

### SLA Events

| Event Type | Notification Message | Priority | Recipients | Channels |
|---|---|---|---|---|
| `sla.warning` | "SLA deadline in {hours}h on {vault}" | URGENT | Assigned reviewer | In-app + toast |
| `sla.breached` | "SLA breached on {vault} (overdue by {duration})" | URGENT | Workspace admins + vault members | In-app + toast |
| `sla.extended` | "SLA extended by {amount} on {vault}" | INFO | Assigned reviewer | In-app |
| `sla.reassigned` | "Review reassigned to you on {vault}" | ACTION | New reviewer | In-app + badge |

### System & Feature Control Plane Events

| Event Type | Notification Message | Priority | Recipients | Channels |
|---|---|---|---|---|
| `feature.circuit_breaker` | "Circuit breaker tripped: {feature_name}" | URGENT | Workspace admins | In-app + toast |
| `feature.health_degraded` | "{feature_name} health degraded (error rate {rate}%)" | ACTION | Workspace admins | In-app + badge |
| `feature.health_recovered` | "{feature_name} recovered to healthy" | INFO | Workspace admins | In-app |
| `sync.completed` | "Sync complete: {system_name}" | INFO | Sync initiator | In-app |
| `sync.failed` | "Sync failed: {system_name} -- {error}" | URGENT | Sync initiator + workspace admins | In-app + toast |
| `calibration.changed` | "Calibration updated: {parameter} set to {new_value}" | INFO | Workspace admins | In-app |

### AI Events

| Event Type | Notification Message | Priority | Recipients | Channels |
|---|---|---|---|---|
| `otto.suggestion` | "Otto has a suggestion for {vault}" | AI | Vault members | In-app |
| `otto.patch_suggested` | "Otto drafted an amendment for {vault}" | AI | Vault owner | In-app |
| `otto.injection_detected` | "Potential prompt injection blocked in {vault}" | URGENT | Workspace admins | In-app + toast |

### Collaboration Events

| Event Type | Notification Message | Priority | Recipients | Channels |
|---|---|---|---|---|
| `comment.added` | "{actor} commented on {vault}" | INFO | Vault members (excluding commenter) | In-app |
| `mention.created` | "{actor} mentioned you in {vault}" | ACTION | Mentioned user | In-app + badge |
| `document.uploaded` | "New document uploaded to {vault}: {filename}" | INFO | Vault members (excluding uploader) | In-app |
| `review.assigned` | "Review assigned to you: {vault}" | ACTION | Assigned gatekeeper | In-app + badge |

### Entity Resolution Events

| Event Type | Notification Message | Priority | Recipients | Channels |
|---|---|---|---|---|
| `entity.resolved` | "Entity resolved: {entity_name} (confidence {confidence}%)" | INFO | Vault owner | In-app |
| `entity.needs_resolution` | "Ambiguous entity match: {entity_name}" | ACTION | Assigned builder | In-app + badge |
| `entity.new_customer` | "New customer detected: {entity_name}" | INFO | CRM module members | In-app |

### Lifecycle Events

| Event Type | Notification Message | Priority | Recipients | Channels |
|---|---|---|---|---|
| `lifecycle.transition` | "{vault} moved from {from_state} to {to_state}" | INFO | Vault members | In-app |

---

## 2. Recipient Resolution

Every notification must resolve its target audience before delivery. Resolution follows a hierarchy of lookup strategies.

### Resolution Strategies

| Strategy | Lookup | Source Table |
|---|---|---|
| **Vault members** | All users with an active role on the vault | `vault_member` |
| **Vault owner** | User with `owner` role on the vault | `vault_member WHERE role = 'owner'` |
| **Module members** | All users with access to the module containing the vault | `user_module_role` |
| **Workspace admins** | Users with `admin` role at workspace level | `workspace_member WHERE role = 'admin'` |
| **Assigned gatekeeper** | User assigned as gatekeeper on the specific patch gate | `patch_gate WHERE assignee_id = ?` |
| **Assigned builder** | User assigned as builder on the vault | `vault_member WHERE role = 'builder'` |
| **Specific user** | Directly referenced user (e.g., patch author, task assignee) | Event payload (`author_id`, `assigned_to`) |
| **Deal owner** | User who owns the CRM deal vault | `vault_member WHERE role = 'owner' AND module = 'crm'` |
| **Batch creator** | User who initiated the batch operation | Event payload (`actor_id`) |
| **Sync initiator** | User who triggered the sync | Event payload (`actor_id`) |

### Resolution Rules

1. **Self-exclusion**: The actor who triggered the event is never included in the recipient list. If a gatekeeper approves a patch, they do not receive the `patch.approved` notification.

2. **Mute check**: After resolving recipients, filter out any user who has `muted = true` for the notification type in `notification_preference`, or who has muted the vault in `vault_mute`. Exception: URGENT notifications ignore the mute flag.

3. **Module access check**: Recipients must have active access to the module. A user without CRM access does not receive `account.created` notifications even if they are a workspace admin. Exception: `feature.circuit_breaker` and `sla.breached` bypass module access checks for workspace admins.

4. **Workspace scope**: Recipient resolution is always scoped to the workspace where the event originated. No cross-workspace notification delivery.

5. **Role inheritance**: Workspace admins implicitly receive all URGENT-priority notifications for their workspace, regardless of module membership.

---

## 3. Deduplication Rules

Duplicate notifications degrade trust. The deduplication system prevents redundant alerts using Redis-backed checks.

### Time-Window Deduplication

- **Key format**: `dedup:{event_type}:{vault_id}:{recipient_id}`
- **TTL**: 60 seconds
- **Storage**: Redis SET with automatic expiry
- **Logic**: Before creating a notification, check if the dedup key exists. If it does, discard the duplicate. If it does not, set the key with 60s TTL and proceed.

### Batch Collapsing

When 5 or more notifications of the same type arrive for the same recipient within a 5-minute window, collapse them into a single grouped notification:

| Collapsed Notification | Example |
|---|---|
| Multiple extractions | "5 extractions completed across your vaults" |
| Multiple comments | "3 new comments on {vault}" |
| Multiple health updates | "Health changed on 4 vaults" |
| Multiple task assignments | "4 tasks assigned to you" |

**Implementation**: Use a Redis sorted set keyed by `batch:{event_type}:{recipient_id}` with event timestamps as scores. A scheduled job checks every 30 seconds for sets with 5+ members. When found, it deletes individual pending notifications and creates one grouped notification.

### SLA Stacking

Multiple SLA warnings for the same vault collapse to the most urgent (shortest remaining time):

- `sla.warning` at 4h remaining, then another at 1h remaining -> keep only the 1h warning
- Key: `sla:{vault_id}:{recipient_id}` -- overwrite previous SLA notification

### Event Supersession

Certain events make previous notifications obsolete:

| Superseding Event | Cancels |
|---|---|
| `patch.approved` | Pending `patch.submitted` notification for the same patch |
| `patch.rejected` | Pending `patch.submitted` notification for the same patch |
| `gate.advanced` | Pending `gate.failed` notification for the same vault + gate |
| `task.completed` | Pending `task.overdue` notification for the same task |
| `feature.health_recovered` | Pending `feature.health_degraded` for the same feature |

---

## 4. Rate Limiting Per User

Notification fatigue is a primary risk. Rate limits protect users from event storms.

### Limits

| Tier | Max Per Hour | Behavior When Exceeded | Bypass |
|---|---|---|---|
| Default | 30 | Queue remaining; deliver in next hour | URGENT priority |
| Light | 15 | Queue remaining; deliver in next hour | URGENT priority |
| Heavy | 60 | Queue remaining; deliver in next hour | URGENT priority |
| Unlimited | No cap | All delivered | N/A |

- Users configure their personal rate limit tier in Settings > Notifications.
- Default tier applies when no preference is set.
- URGENT priority notifications (`sla.breached`, `batch.failed`, `contract.failed`, `feature.circuit_breaker`, `otto.injection_detected`) always bypass rate limits.

### Rate Limit Implementation

- **Key**: `ratelimit:{user_id}:{hour_bucket}` where `hour_bucket = floor(unix_timestamp / 3600)`
- **Storage**: Redis counter with 1-hour TTL
- **Check**: Increment counter via `INCR`. If counter > limit, push notification to `deferred:{user_id}` sorted set with timestamp score.
- **Drain**: A scheduled job runs every 5 minutes. For each user with deferred notifications, check if the current hour's counter is below the limit. If so, deliver deferred notifications (oldest first) until the limit is reached again.

---

## 5. Novu Self-Hosted Deployment

Novu provides the notification delivery infrastructure. It runs alongside Airlock as Docker services.

### Docker Compose Services

```yaml
# Notification infrastructure (Novu self-hosted)
novu-api:
  image: ghcr.io/novuhq/novu/api:latest
  environment:
    - NODE_ENV=production
    - MONGO_URL=mongodb://novu-mongo:27017/novu
    - REDIS_HOST=redis
    - REDIS_PORT=6379
    - JWT_SECRET=${NOVU_JWT_SECRET}
    - STORE_ENCRYPTION_KEY=${NOVU_ENCRYPTION_KEY}
  depends_on:
    - novu-mongo
    - redis

novu-worker:
  image: ghcr.io/novuhq/novu/worker:latest
  environment:
    - NODE_ENV=production
    - MONGO_URL=mongodb://novu-mongo:27017/novu
    - REDIS_HOST=redis
    - REDIS_PORT=6379
  depends_on:
    - novu-mongo
    - redis

novu-web:
  image: ghcr.io/novuhq/novu/web:latest
  ports:
    - "4200:4200"
  environment:
    - REACT_APP_API_URL=http://novu-api:3000

novu-mongo:
  image: mongo:7
  volumes:
    - novu_mongo_data:/data/db
  # Novu requires MongoDB -- separate from Airlock's PostgreSQL.
  # Novu stores workflow templates, subscriber preferences, and
  # notification history in MongoDB. Airlock data stays in PostgreSQL.
```

### Novu Workflow Templates

Each notification type maps to a Novu workflow template, configured via the Novu API at deployment time:

| Template ID | Trigger Event | Steps | Channel |
|---|---|---|---|
| `sla-warning` | `sla.warning` | Digest (5 min) -> In-App | In-app |
| `sla-breached` | `sla.breached` | In-App (immediate) | In-app (V2: + email) |
| `patch-submitted` | `patch.submitted` | In-App (immediate) | In-app |
| `patch-approved` | `patch.approved` | In-App (immediate) | In-app |
| `patch-rejected` | `patch.rejected` | In-App (immediate) | In-app |
| `contract-failed` | `contract.failed` | In-App (immediate) | In-app (V2: + email) |
| `batch-complete` | `batch.completed` | Digest (5 min) -> In-App | In-app |
| `batch-failed` | `batch.failed` | In-App (immediate) | In-app (V2: + email) |
| `task-assigned` | `task.created` | In-App (immediate) | In-app |
| `task-overdue` | `task.overdue` | In-App (immediate) | In-app |
| `entity-needs-resolution` | `entity.needs_resolution` | In-App (immediate) | In-app |
| `health-dropped` | `health.updated` (drop > 10%) | In-App (immediate) | In-app |
| `circuit-breaker` | `feature.circuit_breaker` | In-App (immediate) | In-app (V2: + email) |
| `otto-suggestion` | `otto.suggestion` | Digest (10 min) -> In-App | In-app |
| `review-assigned` | `review.assigned` | In-App (immediate) | In-app |
| `comment-added` | `comment.added` | Digest (5 min) -> In-App | In-app |
| `daily-digest` | Scheduled (cron) | Digest (24h) -> Email | Email (V2 only) |

### Novu Subscriber Sync

Airlock users are synced to Novu as subscribers when they are created or updated:

```python
# server/notifications/subscriber_sync.py
async def sync_subscriber(user: User, workspace_id: str):
    """Sync Airlock user to Novu subscriber."""
    await novu_client.subscribers.identify(
        subscriber_id=f"{workspace_id}:{user.id}",
        email=user.email,
        first_name=user.first_name,
        last_name=user.last_name,
        data={"workspace_id": workspace_id, "role": user.workspace_role},
    )
```

---

## 6. Notification Preference Schema

User preferences control which notifications are delivered and through which channels.

```sql
CREATE TABLE notification_preference (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  workspace_id    UUID NOT NULL REFERENCES workspace(id),
  user_id         UUID NOT NULL REFERENCES "user"(id),
  notification_type TEXT NOT NULL,     -- matches event_type from trigger catalog
  in_app          BOOLEAN DEFAULT true,
  email           BOOLEAN DEFAULT false,    -- future (V2)
  push            BOOLEAN DEFAULT false,    -- future (V3)
  muted           BOOLEAN DEFAULT false,
  rate_limit_tier TEXT DEFAULT 'default',   -- 'light', 'default', 'heavy', 'unlimited'
  created_at      TIMESTAMPTZ DEFAULT now(),
  updated_at      TIMESTAMPTZ DEFAULT now(),
  UNIQUE (workspace_id, user_id, notification_type)
);

-- Index for fast lookup during recipient resolution
CREATE INDEX idx_notif_pref_user_type
  ON notification_preference (workspace_id, user_id, notification_type)
  WHERE muted = false;

-- Per-vault mute (supplements notification_preference)
CREATE TABLE vault_mute (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  workspace_id    UUID NOT NULL REFERENCES workspace(id),
  user_id         UUID NOT NULL REFERENCES "user"(id),
  vault_id        UUID NOT NULL,
  muted_at        TIMESTAMPTZ DEFAULT now(),
  UNIQUE (workspace_id, user_id, vault_id)
);
```

### Default Preferences

When no `notification_preference` row exists for a user + event type combination, defaults apply:

| Priority | In-App Default | Email Default (V2) | Push Default (V3) |
|---|---|---|---|
| URGENT | ON | ON | ON |
| ACTION | ON | OFF | OFF |
| INFO | ON | OFF | OFF |
| AI | ON | OFF | OFF |

Users cannot mute URGENT notifications. The `muted` flag is ignored for URGENT-priority events.

---

## 7. Delivery Pipeline

The end-to-end path from event to user-visible notification:

```
1. Event fires (any module)
       |
2. EventBus routes to notifications-events queue (BullMQ)
       |
3. NotificationsEventHandler picks up job
       |
4. Recipient resolution
       |  - Look up recipients by strategy (vault members, admins, etc.)
       |  - Apply self-exclusion (remove actor)
       |  - Apply mute check (notification_preference + vault_mute)
       |  - Apply module access check
       |
5. Per-recipient processing:
       |
       +-- 5a. Deduplication check (Redis dedup key)
       |        - If duplicate -> discard
       |
       +-- 5b. Supersession check
       |        - If superseded by newer event -> discard
       |
       +-- 5c. Rate limit check (Redis counter)
       |        - If over limit -> push to deferred queue
       |        - If URGENT -> bypass limit
       |
       +-- 5d. Batch collapsing check
       |        - Add to batch window set
       |        - If batch threshold met -> collapse
       |
6. Novu trigger (novu_client.trigger)
       |  - Template ID from trigger catalog
       |  - Subscriber ID = {workspace_id}:{user_id}
       |  - Payload = notification message + metadata
       |
7. Novu processes workflow
       |  - Applies digest step if configured
       |  - Renders in-app notification
       |
8. WebSocket delivery
       |  - Novu in-app notification stored
       |  - Real-time push via user:{user_id} WebSocket topic
       |  - Badge count increment on module bar
       |  - Blue dot on vault in sub-panel
       |
9. Audit log
       - Every notification (delivered, deferred, discarded) logged
       - Fields: event_type, recipient_id, channel, status, timestamp
```

### Error Handling

| Failure Point | Behavior |
|---|---|
| Recipient resolution fails | Log error; skip notification; do not retry (data issue) |
| Redis unavailable (dedup/rate limit) | Fail open -- deliver the notification without dedup/rate checks |
| Novu API unavailable | Retry 3x with exponential backoff (2s, 8s, 32s) |
| Novu delivery fails after retries | Move to notification dead-letter queue (`notifications-dlq`) |
| WebSocket disconnected | Novu stores in-app notification; user sees it on next page load or reconnect |

### Dead-Letter Queue

Failed notifications land in `notifications-dlq`:

- Visible in Admin > System Health > Notification Errors
- Include full event payload, recipient, error context, and retry count
- Manually retryable from Admin UI
- Auto-purged after 14 days

---

## Related Specs

- [Notifications Overview](./overview.md) -- Parent spec with badge system, priority levels, two-level architecture
- [Cross-Module Event Bus](../EventBus/overview.md) -- Event catalog and BullMQ routing that feeds notification triggers
- [Real-Time Events](../RealTime/overview.md) -- WebSocket delivery of in-app notifications to connected clients
- [Admin / Settings](../Admin/overview.md) -- User preference settings UI for notification configuration
- [Patch Workflow / SLA Timer](../PatchWorkflow/sla-timer.md) -- Source of SLA-related notification events
- [Feature Control Plane / Failure Detection](../FeatureControlPlane/failure-detection.md) -- Circuit breaker events that trigger admin alerts
- [Workflow Engine](../WorkflowEngine/overview.md) -- Workflow-triggered notifications and delivery channel routing
