# Audit & Immutability -- Security Spec

> **Status:** SPECCED
> **Source:** Security overview.md (Q6, A14), FeatureControlPlane spec, /Features/Auth/overview.md
> **Resolves:** A14 (audit event immutability scope)

## Summary

This document defines which data stores in Airlock are append-only, how immutability is enforced at the database level, the complete security event schema, the optional hash chain for tamper evidence, and the decision lineage model that traces every change from AI extraction through human approval to baseline application. These are the tables that form Airlock's "source of truth" -- if they can be tampered with, the entire governance model breaks.

---

## 1. Append-Only Table Catalog (Resolves A14)

### Decision

**Three tables are append-only with database-enforced immutability. All other tables allow normal CRUD operations.**

| Table | Immutability Level | Enforcement | Rationale |
|-------|-------------------|-------------|-----------|
| `audit_events` | **Full** -- no UPDATE, no DELETE | PostgreSQL trigger | Security-critical decision lineage. If an attacker can modify audit events, they can rewrite who approved what, when, and why. This table is the authoritative record for compliance, legal disputes, and security investigations. |
| `channel_events` | **Full** -- no UPDATE, no DELETE | PostgreSQL trigger | Vault event history is the source of truth for all state changes within a vault. Every extraction, patch, approval, rejection, and promotion is recorded here. Modifying events would allow rewriting the history of a contract or deal. |
| `otto_messages` | **Soft-delete only** -- no UPDATE on content columns, DELETE sets `deleted_at` | PostgreSQL trigger (selective) | AI session history is needed for attribution (which model produced what output, what tool calls were made). However, GDPR Article 17 (right to erasure) requires a deletion path. Soft-delete preserves the event structure while allowing PII removal. |

### Why These Three (and Not Others)

**Included:**
- `audit_events` -- Security backbone. Cannot be compromised.
- `channel_events` -- Operational backbone. The "git log" of every vault.
- `otto_messages` -- AI attribution chain. Proves which model said what.

**Excluded (normal CRUD allowed):**
- `vaults` -- Metadata can be edited (name, type, status).
- `patches` -- Status transitions are tracked via `channel_events`, not by immutability of the patches table itself.
- `users` -- Profile updates are normal operations.
- `refresh_tokens` -- Must be marked as used/revoked (requires UPDATE).
- `custom_roles` -- Permissions can be modified by admins.
- `permission_overrides` -- Can be revoked (requires UPDATE to set `revoked_at`).
- All other tables -- State changes are tracked through `channel_events` and `audit_events`, so the source tables themselves do not need immutability.

---

## 2. Database Trigger Enforcement

### `audit_events` -- Full Immutability

```sql
-- Prevent UPDATE and DELETE on audit_events
CREATE OR REPLACE FUNCTION prevent_audit_mutation()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'UPDATE' THEN
        RAISE EXCEPTION 'audit_events table is append-only. UPDATE is prohibited. '
            'Event ID: %, Event Type: %',
            OLD.id, OLD.event_type
            USING ERRCODE = 'restrict_violation';
    ELSIF TG_OP = 'DELETE' THEN
        RAISE EXCEPTION 'audit_events table is append-only. DELETE is prohibited. '
            'Event ID: %, Event Type: %',
            OLD.id, OLD.event_type
            USING ERRCODE = 'restrict_violation';
    END IF;
    RETURN NULL;  -- Prevent the operation
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER audit_immutable_trigger
    BEFORE UPDATE OR DELETE ON audit_events
    FOR EACH ROW EXECUTE FUNCTION prevent_audit_mutation();

-- Prevent trigger removal by non-superusers
COMMENT ON TRIGGER audit_immutable_trigger ON audit_events IS
    'SECURITY: Append-only enforcement. Do not drop without security team approval.';
```

### `channel_events` -- Full Immutability

```sql
-- Prevent UPDATE and DELETE on channel_events
CREATE OR REPLACE FUNCTION prevent_channel_event_mutation()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'UPDATE' THEN
        RAISE EXCEPTION 'channel_events table is append-only. UPDATE is prohibited. '
            'Event ID: %, Event Type: %',
            OLD.id, OLD.event_type
            USING ERRCODE = 'restrict_violation';
    ELSIF TG_OP = 'DELETE' THEN
        RAISE EXCEPTION 'channel_events table is append-only. DELETE is prohibited. '
            'Event ID: %, Event Type: %',
            OLD.id, OLD.event_type
            USING ERRCODE = 'restrict_violation';
    END IF;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER channel_events_immutable_trigger
    BEFORE UPDATE OR DELETE ON channel_events
    FOR EACH ROW EXECUTE FUNCTION prevent_channel_event_mutation();
```

### `otto_messages` -- Selective Immutability (Soft-Delete Only)

```sql
-- Allow setting deleted_at but prevent modifying content columns
CREATE OR REPLACE FUNCTION protect_otto_message_content()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'DELETE' THEN
        RAISE EXCEPTION 'otto_messages cannot be hard-deleted. Set deleted_at instead.'
            USING ERRCODE = 'restrict_violation';
    END IF;

    IF TG_OP = 'UPDATE' THEN
        -- These content columns are immutable once written
        IF OLD.user_message IS DISTINCT FROM NEW.user_message THEN
            RAISE EXCEPTION 'otto_messages.user_message is immutable after creation.'
                USING ERRCODE = 'restrict_violation';
        END IF;
        IF OLD.assistant_response IS DISTINCT FROM NEW.assistant_response THEN
            RAISE EXCEPTION 'otto_messages.assistant_response is immutable after creation.'
                USING ERRCODE = 'restrict_violation';
        END IF;
        IF OLD.tool_calls IS DISTINCT FROM NEW.tool_calls THEN
            RAISE EXCEPTION 'otto_messages.tool_calls is immutable after creation.'
                USING ERRCODE = 'restrict_violation';
        END IF;
        IF OLD.model_id IS DISTINCT FROM NEW.model_id THEN
            RAISE EXCEPTION 'otto_messages.model_id is immutable after creation.'
                USING ERRCODE = 'restrict_violation';
        END IF;
        IF OLD.tokens_in IS DISTINCT FROM NEW.tokens_in THEN
            RAISE EXCEPTION 'otto_messages.tokens_in is immutable after creation.'
                USING ERRCODE = 'restrict_violation';
        END IF;
        IF OLD.tokens_out IS DISTINCT FROM NEW.tokens_out THEN
            RAISE EXCEPTION 'otto_messages.tokens_out is immutable after creation.'
                USING ERRCODE = 'restrict_violation';
        END IF;

        -- These metadata columns CAN be updated:
        -- deleted_at (for GDPR soft-delete)
        -- deleted_by (who initiated the deletion)
        -- deleted_reason (why: 'gdpr_request', 'admin', 'user')
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER otto_messages_content_protection
    BEFORE UPDATE OR DELETE ON otto_messages
    FOR EACH ROW EXECUTE FUNCTION protect_otto_message_content();
```

### Trigger Security

PostgreSQL triggers can only be created or dropped by the table owner or a superuser. To prevent accidental or malicious trigger removal:

1. **Application database user** has `INSERT`, `SELECT`, and limited `UPDATE` privileges -- NOT `TRIGGER` or `DROP TRIGGER` privileges
2. **Migration user** (used only during schema migrations) has `TRIGGER` privilege
3. **Monitoring job** runs daily to verify trigger existence on all three tables:

```sql
-- Verification query (run by monitoring job)
SELECT
    t.tgname AS trigger_name,
    c.relname AS table_name,
    t.tgenabled AS enabled
FROM pg_trigger t
JOIN pg_class c ON t.tgrelid = c.oid
WHERE c.relname IN ('audit_events', 'channel_events', 'otto_messages')
AND t.tgname IN (
    'audit_immutable_trigger',
    'channel_events_immutable_trigger',
    'otto_messages_content_protection'
);

-- Expected: 3 rows, all enabled ('O' = origin, 'A' = always)
-- If any are missing or disabled: CRITICAL ALERT
```

---

## 3. Security Event Schema

### `audit_events` Table Schema

```sql
CREATE TABLE audit_events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_type TEXT NOT NULL,                      -- Dot-separated event type
    workspace_id UUID NOT NULL,                    -- Tenant isolation
    actor_id UUID,                                 -- User who performed the action (NULL for system events)
    actor_type TEXT NOT NULL DEFAULT 'user',        -- 'user' | 'system' | 'api_key' | 'otto'
    payload JSONB NOT NULL DEFAULT '{}',           -- Event-specific data
    ip_address INET,                               -- Client IP (if applicable)
    user_agent TEXT,                                -- Client user agent (if applicable)
    integrity_hash TEXT,                           -- SHA-256 chain hash (enterprise feature)
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Primary query: events by workspace, newest first
CREATE INDEX idx_audit_workspace_time ON audit_events(workspace_id, created_at DESC);

-- Security monitoring: auth events
CREATE INDEX idx_audit_auth_events ON audit_events(workspace_id, created_at DESC)
    WHERE event_type LIKE 'auth.%';

-- Security alerting: failed auth attempts
CREATE INDEX idx_audit_auth_failures ON audit_events(created_at DESC)
    WHERE event_type IN ('auth.login_failed', 'auth.login_blocked', 'auth.refresh_replay');

-- Security monitoring: self-approval attempts
CREATE INDEX idx_audit_self_approval ON audit_events(workspace_id, created_at DESC)
    WHERE event_type = 'security.self_approval_attempt';

-- Actor lookup: all actions by a specific user
CREATE INDEX idx_audit_actor ON audit_events(actor_id, created_at DESC);
```

### Complete Security Event Type Catalog

#### Authentication Events

| Event Type | Payload Fields | Trigger Condition |
|-----------|---------------|-------------------|
| `auth.login` | `user_id`, `email`, `workspace_id`, `ip_address`, `user_agent`, `method` (oauth/api_key/dev) | Successful login via any method |
| `auth.login_failed` | `email`, `ip_address`, `user_agent`, `reason` (invalid_token/inactive/unknown_email) | Failed login attempt |
| `auth.login_blocked` | `ip_address`, `attempt_count`, `window_seconds` | Rate limit exceeded on login endpoint |
| `auth.refresh` | `user_id`, `token_family`, `ip_address` | Successful token refresh |
| `auth.refresh_failed` | `user_id`, `token_family`, `reason` (expired/invalid/unknown) | Failed token refresh |
| `auth.refresh_replay` | `user_id`, `token_family`, `ip_address`, `original_ip`, `families_revoked` | Refresh token reuse detected -- entire family revoked |
| `auth.logout` | `user_id`, `token_family` | Single-device logout |
| `auth.logout_all` | `user_id`, `families_revoked` (count) | All-device logout |

#### Authorization Events

| Event Type | Payload Fields | Trigger Condition |
|-----------|---------------|-------------------|
| `security.permission_denied` | `user_id`, `endpoint`, `method`, `required_permission`, `module_id` | 403 response from `require_permission()` middleware |
| `security.self_approval_attempt` | `user_id`, `resource_id`, `resource_type`, `endpoint` | User attempted to approve their own resource |
| `security.rate_limit_exceeded` | `user_id` (if authenticated), `ip_address`, `endpoint`, `limit`, `window` | Rate limit hit on any endpoint |

#### Role & Permission Events

| Event Type | Payload Fields | Trigger Condition |
|-----------|---------------|-------------------|
| `security.role_changed` | `target_user_id`, `changed_by`, `old_role`, `new_role`, `scope` (org/module), `module_id` (if module scope) | Org or module role assignment changed |
| `security.permission_override` | `target_user_id`, `granted_by`/`revoked_by`, `permission`, `effect` (grant/deny), `scope_type`, `scope_id`, `expires_at`, `action` (created/revoked) | Ad-hoc permission override created or revoked |
| `security.role_simulation_start` | `user_id`, `simulated_role`, `timeout_minutes` | Admin entered role simulation mode |
| `security.role_simulation_end` | `user_id`, `simulated_role`, `duration_seconds`, `reason` (manual/timeout) | Admin exited role simulation mode |

#### API Key Events

| Event Type | Payload Fields | Trigger Condition |
|-----------|---------------|-------------------|
| `security.api_key_created` | `user_id`, `key_id`, `key_name`, `scopes`, `expires_at` | New API key generated |
| `security.api_key_revoked` | `user_id`, `key_id`, `key_name`, `reason` | API key revoked |

#### Infrastructure Events

| Event Type | Payload Fields | Trigger Condition |
|-----------|---------------|-------------------|
| `security.workspace_created` | `user_id`, `workspace_id`, `workspace_name` | New workspace created |
| `security.feature_flag_changed` | `flag_name`, `old_value`, `new_value`, `changed_by`, `workspace_id` | Feature flag toggled or calibration parameter changed |
| `security.cipher_key_accessed` | `workspace_id`, `category`, `operation` (encrypt/decrypt), `count`, `duration_ms` | Encryption key used (batch reporting, not per-field) |

#### Data Movement Events

| Event Type | Payload Fields | Trigger Condition |
|-----------|---------------|-------------------|
| `security.data_export` | `user_id`, `export_type` (csv/pdf/json/docx), `vault_id`, `row_count`, `file_size_bytes` | Data exported from a vault |
| `security.data_import` | `user_id`, `import_type`, `vault_id`, `row_count`, `source_file`, `file_size_bytes` | Data imported into a vault |

#### AI/LLM Events

| Event Type | Payload Fields | Trigger Condition |
|-----------|---------------|-------------------|
| `security.llm_call` | `user_id`, `session_id`, `model` (claude-sonnet-4/gpt-4o/otto-local), `tokens_in`, `tokens_out`, `cost_usd`, `tool_calls_count`, `redaction_applied` (boolean), `duration_ms` | Every LLM API call made by Otto |
| `security.llm_circuit_break` | `workspace_id`, `error_count`, `window_seconds`, `cooldown_seconds` | Otto circuit breaker activated (5 errors in 60s) |
| `security.llm_pii_redaction` | `user_id`, `session_id`, `fields_redacted` (list), `redaction_method` | PII was redacted before sending context to LLM provider |

### Event Insertion Pattern

All security events are inserted through a single function to ensure consistency:

```python
async def emit_security_event(
    event_type: str,
    workspace_id: str,
    actor_id: str | None,
    payload: dict,
    actor_type: str = "user",
    ip_address: str | None = None,
    user_agent: str | None = None,
):
    """Insert an immutable audit event.

    This is the ONLY way to create audit events.
    Direct SQL INSERT is blocked by application-level convention
    and verified by code review.
    """
    await db.execute(
        text("""
            INSERT INTO audit_events
                (event_type, workspace_id, actor_id, actor_type, payload,
                 ip_address, user_agent)
            VALUES
                (:event_type, :workspace_id, :actor_id, :actor_type,
                 :payload::jsonb, :ip_address, :user_agent)
        """),
        {
            "event_type": event_type,
            "workspace_id": workspace_id,
            "actor_id": actor_id,
            "actor_type": actor_type,
            "payload": json.dumps(payload),
            "ip_address": ip_address,
            "user_agent": user_agent,
        }
    )
```

---

## 4. Hash Chain for Tamper Evidence (Enterprise)

### Purpose

An optional integrity verification layer that makes any tampering with audit events detectable. Each event's hash includes the previous event's hash, forming a chain. Breaking any link in the chain (modifying, deleting, or inserting events out of order) is detectable.

### Hash Computation

```
event.integrity_hash = SHA-256(
    event.id
    || event.event_type
    || event.workspace_id
    || event.actor_id
    || event.payload (canonical JSON, sorted keys)
    || event.created_at (ISO 8601 format)
    || previous_event.integrity_hash
)
```

**For the first event in a workspace:** `previous_event.integrity_hash` is a well-known genesis value:

```
genesis_hash = SHA-256("airlock:genesis:" + workspace_id)
```

### Implementation

```python
import hashlib
import json

def compute_integrity_hash(
    event_id: str,
    event_type: str,
    workspace_id: str,
    actor_id: str | None,
    payload: dict,
    created_at: str,  # ISO 8601
    previous_hash: str,
) -> str:
    """Compute the integrity hash for an audit event."""
    # Canonical JSON: sorted keys, no whitespace, null for None
    canonical_payload = json.dumps(payload, sort_keys=True, separators=(',', ':'))

    hash_input = (
        f"{event_id}"
        f"{event_type}"
        f"{workspace_id}"
        f"{actor_id or 'null'}"
        f"{canonical_payload}"
        f"{created_at}"
        f"{previous_hash}"
    )

    return hashlib.sha256(hash_input.encode('utf-8')).hexdigest()


def compute_genesis_hash(workspace_id: str) -> str:
    """Compute the genesis hash for a workspace's audit chain."""
    return hashlib.sha256(
        f"airlock:genesis:{workspace_id}".encode('utf-8')
    ).hexdigest()
```

### Chain Properties

| Property | Value |
|----------|-------|
| Hash algorithm | SHA-256 |
| Chain scope | Per-workspace (each workspace has its own independent chain) |
| Genesis value | `SHA-256("airlock:genesis:" + workspace_id)` |
| Ordering | `created_at` ASC within workspace; ties broken by `id` |
| Storage | `integrity_hash` column on `audit_events` table (nullable for MVP, required for enterprise) |

### Verification

A daily verification job checks the integrity of the entire chain:

```python
async def verify_audit_chain(workspace_id: str) -> VerificationResult:
    """Verify the integrity hash chain for a workspace.

    Returns: VerificationResult with status, break_point (if any),
             and total events verified.
    """
    events = await db.execute(
        text("""
            SELECT id, event_type, workspace_id, actor_id, payload,
                   created_at, integrity_hash
            FROM audit_events
            WHERE workspace_id = :ws_id
            ORDER BY created_at ASC, id ASC
        """),
        {"ws_id": workspace_id}
    )

    previous_hash = compute_genesis_hash(workspace_id)
    verified_count = 0

    for event in events:
        expected_hash = compute_integrity_hash(
            event_id=str(event.id),
            event_type=event.event_type,
            workspace_id=str(event.workspace_id),
            actor_id=str(event.actor_id) if event.actor_id else None,
            payload=event.payload,
            created_at=event.created_at.isoformat(),
            previous_hash=previous_hash,
        )

        if event.integrity_hash != expected_hash:
            return VerificationResult(
                status="CHAIN_BROKEN",
                break_point=event.id,
                break_event_type=event.event_type,
                break_created_at=event.created_at,
                expected_hash=expected_hash,
                actual_hash=event.integrity_hash,
                verified_count=verified_count,
            )

        previous_hash = event.integrity_hash
        verified_count += 1

    return VerificationResult(
        status="VERIFIED",
        verified_count=verified_count,
    )
```

### Chain Break Response

If the verification job detects a broken chain:

| Severity | Condition | Action |
|----------|-----------|--------|
| Critical | Chain break detected | Immediate alert to all workspace admins + security team email |
| Critical | Trigger missing from `audit_events` | Immediate alert + system health dashboard warning |
| Warning | Hash chain not enabled (enterprise not activated) | Logged only; no alert |
| Info | Verification completed successfully | Logged with `verified_count` |

### Performance Considerations

- **Write-time cost:** One SHA-256 computation per audit event insertion (~1 microsecond). Negligible.
- **Verification cost:** Full chain scan. For a workspace with 1M events: ~30 seconds. Run during off-peak hours.
- **Previous hash lookup:** Last event's hash is cached in Redis (`audit_chain:last_hash:{workspace_id}`) and updated on each insert. If cache miss, query the last event from the database.

---

## 5. Decision Lineage

### What Is Decision Lineage?

Every change in Airlock has a traceable chain of events from discovery to baseline application. This chain cannot be altered after the fact because it spans two immutable tables (`channel_events` and `audit_events`).

### Lineage Example: Contract Field Correction

```
Step 1: [channel_event] AI Extraction
        event_type: "extraction.completed"
        vault_id: "vault_henderson_msa"
        payload: {
            field: "OPP_EFFECTIVE_DATE",
            value: "2026-01-15",
            confidence: 0.87,
            extractor: "date_extractor",
            model: "claude-sonnet-4"
        }
        author_type: "system"
        created_at: "2026-03-01T10:00:00Z"

Step 2: [channel_event] Otto Suggests Patch
        event_type: "otto.patch_suggested"
        vault_id: "vault_henderson_msa"
        payload: {
            field: "OPP_EFFECTIVE_DATE",
            suggested_value: "2026-02-01",
            reason: "Document clause 3.1 states February 1, 2026",
            session_id: "otto_session_xyz",
            model: "claude-sonnet-4"
        }
        author_type: "ai_assisted"
        created_at: "2026-03-01T10:05:00Z"

Step 3: [audit_event] Builder Creates Patch
        event_type: "patch.created"
        actor_id: "usr_jane_builder"
        payload: {
            patch_id: "patch_001",
            vault_id: "vault_henderson_msa",
            field: "OPP_EFFECTIVE_DATE",
            before_value: "2026-01-15",
            after_value: "2026-02-01",
            risk_level: "low",
            source: "otto_suggestion"
        }
        created_at: "2026-03-01T10:10:00Z"

Step 4: [audit_event] Builder Submits Patch
        event_type: "patch.submitted"
        actor_id: "usr_jane_builder"
        payload: {
            patch_id: "patch_001",
            submitted_to: "gatekeeper_review"
        }
        created_at: "2026-03-01T10:11:00Z"

Step 5: [audit_event] Gatekeeper Approves Patch
        event_type: "patch.approved"
        actor_id: "usr_bob_gatekeeper"     -- DIFFERENT from Jane (SoD enforced)
        payload: {
            patch_id: "patch_001",
            approval_level: "low_risk",
            evidence_reviewed: true,
            notes: "Verified against source document page 3"
        }
        created_at: "2026-03-01T14:30:00Z"

Step 6: [channel_event] Patch Applied to Vault
        event_type: "patch.applied"
        vault_id: "vault_henderson_msa"
        payload: {
            patch_id: "patch_001",
            field: "OPP_EFFECTIVE_DATE",
            before_value: "2026-01-15",
            after_value: "2026-02-01",
            applied_by: "usr_bob_gatekeeper"
        }
        author_type: "user"
        created_at: "2026-03-01T14:31:00Z"
```

### Lineage Query

To reconstruct the full lineage of any field change:

```sql
-- Get all events related to a specific patch
(
    SELECT 'channel_event' AS source, id, event_type, created_at, payload
    FROM channel_events
    WHERE vault_id = :vault_id
    AND (
        payload->>'patch_id' = :patch_id
        OR (event_type = 'extraction.completed' AND payload->>'field' = :field_name)
    )
)
UNION ALL
(
    SELECT 'audit_event' AS source, id, event_type, created_at, payload
    FROM audit_events
    WHERE workspace_id = :workspace_id
    AND payload->>'patch_id' = :patch_id
)
ORDER BY created_at ASC;
```

### Lineage Properties

| Property | Guarantee |
|----------|-----------|
| **Complete** | Every step from extraction to application is recorded |
| **Immutable** | Both `channel_events` and `audit_events` are append-only |
| **Attributable** | Every step has an `actor_id` (user) or `author_type` (system/AI) |
| **Timestamped** | `created_at` is set by the database (`DEFAULT now()`), not the application |
| **Separation of Duties** | Steps 3 (create) and 5 (approve) must have different `actor_id` values |
| **AI-Transparent** | Steps 1-2 record the model, confidence, and session for AI actions |

---

## 6. Retention and Archival

### Retention Policy

| Table | Hot Storage (PostgreSQL) | Archive Trigger | Archive Destination | Deletion Policy |
|-------|------------------------|-----------------|--------------------|-----------------|
| `audit_events` | Indefinite (never deleted from hot storage during active workspace lifecycle) | Events older than 2 years moved to archive | Cold storage (encrypted S3 or equivalent) | Never deleted. Crypto-shredding of workspace key destroys all data including archives. |
| `channel_events` | Indefinite | Events older than 1 year archived | Cold storage (encrypted) | Never deleted. Crypto-shredding applies. |
| `otto_messages` | Configurable (default: indefinite) | Per workspace admin setting | Cold storage (encrypted) | GDPR purge: soft-delete sets `deleted_at`. Full purge after configurable retention period (default: 90 days after soft-delete). |

### Archival Process

```
1. Archive job runs weekly (BullMQ scheduled job)
2. Query: SELECT events WHERE created_at < (now - retention_threshold)
3. Export to encrypted Parquet files (preserving integrity hashes)
4. Upload to cold storage with workspace_id-scoped paths
5. Verify: downloaded archive matches source hash
6. Mark archived events with archived_at timestamp (UPDATE is allowed on this metadata column only)
7. Optionally: DELETE archived events from hot storage after verification
   (Only for channel_events after 2+ years; audit_events are NEVER deleted from hot storage)
```

### Archival and Hash Chain Integrity

Archived events preserve their `integrity_hash` values. The archive file format includes:

```json
{
    "workspace_id": "ws_abc123",
    "archive_range": {
        "start": "2024-01-01T00:00:00Z",
        "end": "2024-12-31T23:59:59Z"
    },
    "event_count": 45230,
    "first_event_hash": "a1b2c3...",
    "last_event_hash": "d4e5f6...",
    "archive_hash": "SHA-256 of entire archive file",
    "events": [ ... ]
}
```

The `last_event_hash` of one archive must match the `previous_hash` input of the first event in the next archive (or the next event in hot storage). This ensures the chain is continuous even across archive boundaries.

### GDPR Compliance for `otto_messages`

When a user exercises their right to erasure (GDPR Article 17):

```
1. User or admin requests deletion of AI session history
2. System sets deleted_at = now() on matching otto_messages rows
3. Content columns (user_message, assistant_response) remain temporarily
   (for audit trail continuity)
4. After retention period (default: 90 days), purge job runs:
   - Replaces user_message with "[REDACTED - GDPR]"
   - Replaces assistant_response with "[REDACTED - GDPR]"
   - Replaces tool_calls with "[]"
   - Preserves: id, session_id, model_id, tokens_in, tokens_out, cost,
     created_at, deleted_at (for billing and usage analytics)
5. Audit event recorded: security.gdpr_purge { user_id, message_count, session_ids }
```

This approach balances GDPR compliance (PII removed) with billing accuracy (usage metrics preserved) and audit integrity (event structure maintained).

---

## Related Specs

- [overview.md](./overview.md) -- Security architecture anchor, 70-question inventory (Q6, A14)
- [auth-and-identity.md](./auth-and-identity.md) -- Auth audit event definitions, JWT lifecycle events
- [rbac-enforcement.md](./rbac-enforcement.md) -- Permission-denied events, self-approval attempt events
- [data-cipher-model.md](./data-cipher-model.md) -- Cipher key access events, crypto-shredding interaction with retention
- [Auth Overview](../Auth/overview.md) -- Auth event types, refresh token replay detection
- [FeatureControlPlane Overview](../FeatureControlPlane/overview.md) -- Feature flag change events, circuit breaker events
- [Roles Overview](../Roles/overview.md) -- Role change audit events
