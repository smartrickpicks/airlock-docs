# RBAC Enforcement -- Security Spec

> **Status:** SPECCED
> **Source:** Security overview.md (Q1-Q5, A8), /Features/Roles/overview.md, /Features/Auth/overview.md, /Features/Auth/migrations.md
> **Resolves:** A8 (multi-tenant isolation model)

## Summary

This document formalizes the security properties of Airlock's Discord-style RBAC system. It defines how permissions are computed with cryptographic-strength guarantees, how self-approval prevention is enforced at multiple layers to be unforgeable, how tenant isolation works at both the application and database layers, and provides the complete API endpoint permission matrix. Every enforcement point is specified with enough detail for an engineer to implement without ambiguity.

---

## 1. Permission Computation Security Model

### Three-Layer Role Architecture

Airlock computes effective permissions through three layers, each with strict security boundaries:

```
Layer 1: Org Role (ceiling)
    Sets the maximum permission boundary for the user across the entire workspace.
    A Member can never acquire admin-level permissions regardless of other layers.

Layer 2: Module Role (functional) + Custom Roles (additive)
    Determines what the user can do within a specific module.
    Multiple custom roles are unioned (additive).
    Ad-hoc overrides grant or deny individual permissions.

Layer 3: Vault/Channel Overrides (fine-grained)
    Per-vault grants or denies for specific users or roles.
    Most specific scope wins.
```

### Computation Algorithm

The permission computation is deterministic and follows a strict evaluation order:

```
Step 1: Collect all roles for the user in this workspace
         roles = get_user_roles(user_id, workspace_id)
         Example: [Builder (system), Senior Analyst (custom)]

Step 2: Union all permissions from all roles
         base_permissions = union(role.permissions for role in roles)
         Example: {view_vaults, create_vaults, create_patches, ..., approve_low_risk}

Step 3: Apply module-level ad-hoc overrides
         module_overrides = get_active_overrides(user_id, workspace_id, 'module', module_id)
         for override in module_overrides:
             if override.effect == 'grant': base_permissions.add(override.permission)
             if override.effect == 'deny':  base_permissions.discard(override.permission)

Step 4: Apply vault/channel-level ad-hoc overrides (most specific)
         if channel_id:
             channel_overrides = get_active_overrides(user_id, workspace_id, 'channel', channel_id)
             for override in channel_overrides:
                 if override.effect == 'grant': base_permissions.add(override.permission)
                 if override.effect == 'deny':  base_permissions.discard(override.permission)

Step 5: Apply org role ceiling (subtract disallowed permissions)
         ceiling_exclusions = ORG_ROLE_CEILING[org_role]
         effective_permissions = base_permissions - ceiling_exclusions

Step 6: Return effective_permissions
         (SoD rules checked at action time, not here)
```

### Security Properties

| Property | Guarantee | Enforcement |
|----------|-----------|-------------|
| **Deny-always-wins** | If any source denies a permission, it is denied regardless of grants | `discard()` in steps 3-4 removes the permission entirely; subsequent grants in the same step cannot re-add it because overrides are processed in order: all grants first, then all denies |
| **Org ceiling is absolute** | Module/custom roles cannot exceed org role maximum | Step 5 subtracts ceiling exclusions after all grants are applied |
| **Deterministic** | Same inputs always produce same outputs | No randomness, no external state, pure function of role data |
| **Cache-safe** | Cached results are invalidated on any role/permission change | Redis cache with event-bus invalidation (see Caching section) |

### Deny-Always-Wins Implementation Detail

Within each scope (module or channel), overrides are processed in two passes to ensure deny always wins:

```python
def apply_overrides(permissions: set, overrides: list) -> set:
    """Apply overrides with deny-always-wins guarantee."""
    # Pass 1: Apply all grants
    for ov in overrides:
        if ov.effect == 'grant' and not is_expired(ov):
            permissions.add(ov.permission)

    # Pass 2: Apply all denies (overwrites any grants)
    for ov in overrides:
        if ov.effect == 'deny' and not is_expired(ov):
            permissions.discard(ov.permission)

    return permissions
```

### Org Role Ceiling Definitions

The ceiling defines which permissions are **excluded** for each org role. Everything not in the exclusion set is allowed:

| Org Role | Excluded Permissions | Rationale |
|----------|---------------------|-----------|
| `member` | `manage_members`, `manage_roles`, `toggle_features`, `system_health`, `manage_connectors`, `delete_vaults` | Members cannot perform admin operations regardless of module role |
| `lead` | `manage_roles`, `toggle_features`, `system_health`, `delete_vaults` | Leads can manage members but not roles, flags, or system health |
| `director` | `delete_vaults` | Directors can do almost everything except permanent deletion |
| `executive` | (empty set -- no exclusions) | Executives have no ceiling |

### Caching Strategy

Permission computation requires multiple database queries. Cache aggressively with targeted invalidation:

| Cache Layer | Key Pattern | TTL | Invalidation Trigger |
|------------|-------------|-----|---------------------|
| User roles | `perm:roles:{workspace_id}:{user_id}` | 5 min | Role assignment change, custom role permission change |
| Module overrides | `perm:mod_ov:{workspace_id}:{user_id}:{module_id}` | 5 min | Override grant/revoke for this user+module |
| Channel overrides | `perm:ch_ov:{workspace_id}:{user_id}:{channel_id}` | 2 min | Override grant/revoke for this user+channel |
| Org role | `perm:org:{workspace_id}:{user_id}` | 10 min | Org role change |
| Full computed permissions | `perm:computed:{workspace_id}:{user_id}:{module_id}:{channel_id}` | 2 min | Any of the above triggers |

**Invalidation mechanism:** When any role, override, or assignment changes, the write path publishes a Redis `PUBLISH` event on channel `perm_invalidate:{workspace_id}:{user_id}`. All API instances subscribe to this channel and evict matching cache keys.

```python
# Cache invalidation on role change
async def on_role_change(workspace_id: str, user_id: str):
    """Invalidate all permission caches for a user."""
    pattern = f"perm:*:{workspace_id}:{user_id}*"
    keys = await redis.keys(pattern)
    if keys:
        await redis.delete(*keys)
    await redis.publish(f"perm_invalidate:{workspace_id}:{user_id}", "invalidated")
```

### Permission Format

All permissions use the flat string format: `{action}` or `{entity}_{action}`. Examples:

| Permission | Description |
|-----------|-------------|
| `view_vaults` | Read vault data and events |
| `create_patches` | Draft data corrections |
| `approve_low_risk` | Approve patches classified as low risk |
| `manage_members` | Invite/remove workspace members |
| `configure_extraction` | Edit extraction anchors and config |
| `*` | Wildcard -- all permissions (Architect system role only) |

Wildcard check:

```python
def has_permission(user_permissions: set, required: str) -> bool:
    return '*' in user_permissions or required in user_permissions
```

---

## 2. Self-Approval Prevention (Security Principle #4)

### Rule

**A user CANNOT approve any artifact they created, own, or are assigned to as the primary author.** This is the fourth non-negotiable security principle: "Self-approval prevention is unforgeable."

### Enforcement Points (Defense in Depth)

Self-approval is prevented at three independent layers. An attacker must bypass all three to succeed:

#### Layer 1: API Middleware

```python
async def require_not_author(
    resource_id: str,
    resource_type: str,
    auth: AuthResult = Depends(require_auth()),
):
    """FastAPI dependency that blocks self-approval.

    This check runs BEFORE the endpoint handler.
    It cannot be skipped by any API parameter or header.
    """
    resource = await get_resource(resource_id, resource_type)
    if resource is None:
        raise HTTPException(404, "Resource not found")

    if resource.created_by == auth.user_id:
        await emit_audit_event("security.self_approval_attempt", {
            "user_id": auth.user_id,
            "resource_id": resource_id,
            "resource_type": resource_type,
            "endpoint": "approve",
        })
        raise HTTPException(
            403,
            "Self-approval is not permitted. A different user must approve this resource."
        )
```

Usage on approval endpoints:

```python
@router.post("/api/vaults/{vault_id}/patches/{patch_id}/approve")
async def approve_patch(
    vault_id: str,
    patch_id: str,
    auth: AuthResult = Depends(require_auth()),
    _perm = Depends(require_permission("approve_low_risk", module_id="contracts")),
    _not_author = Depends(require_not_author(patch_id, "patch")),
):
    # This code only runs if the user is NOT the patch author
    ...
```

#### Layer 2: Database Constraint

```sql
-- On the patches table
ALTER TABLE patches
    ADD CONSTRAINT chk_no_self_approval
    CHECK (approved_by IS NULL OR approved_by != created_by);

-- On any table with approval workflow
ALTER TABLE contract_releases
    ADD CONSTRAINT chk_no_self_approval_release
    CHECK (approved_by IS NULL OR approved_by != created_by);
```

This constraint is enforced by PostgreSQL itself. Even if the API middleware is bypassed (e.g., direct SQL access), the database rejects the operation.

#### Layer 3: UI Defense (Non-Primary)

```typescript
// Frontend: hide/disable approve button for own items
const canApprove = (patch: Patch, currentUser: User) => {
    // Primary defense is server-side; this is UX optimization
    return patch.created_by !== currentUser.id
        && hasPermission('approve_low_risk');
};
```

The UI layer is defense-in-depth only. It is **never** relied upon as the primary control because client-side code can be modified.

### No Bypass Paths

The following bypass attempts are explicitly blocked:

| Bypass Attempt | Why It Fails |
|---------------|-------------|
| Admin tries to self-approve | `require_not_author` checks `created_by`, not role. Admins are not exempt. |
| API flag `?skip_sod=true` | No such parameter exists. The middleware does not accept any override flags. |
| Environment variable `SKIP_SOD_CHECK` | No such variable exists. The check is hardcoded, not configurable. |
| Direct database UPDATE | PostgreSQL `CHECK` constraint rejects `approved_by = created_by` at the database level. |
| Dev mode bypass | Dev mode bypasses OAuth, not SoD. Self-approval prevention is active in all modes including dev. |
| Service account | Service accounts (`system@airlock.local`) can create resources but cannot approve them. Another human must approve. |
| Permission override granting self-approval | SoD is checked at action time, after permission computation. Having `approve_*` permission does not bypass the author check. |

### Audit Trail

Every self-approval attempt is logged as a security event:

```json
{
    "event_type": "security.self_approval_attempt",
    "workspace_id": "ws_abc123",
    "actor_id": "usr_abc123",
    "payload": {
        "user_id": "usr_abc123",
        "resource_id": "patch_xyz",
        "resource_type": "patch",
        "endpoint": "/api/vaults/v1/patches/patch_xyz/approve"
    },
    "created_at": "2026-03-05T10:30:00Z"
}
```

Workspace admins receive a notification on `security.self_approval_attempt` events. Repeated attempts (3+ in 1 hour) escalate to a high-priority alert.

---

## 3. Tenant Isolation (Answers Q5, Resolves A8)

### Decision

**Application-layer `workspace_id` filtering for MVP, with PostgreSQL Row-Level Security (RLS) as the enterprise hardening path.**

### MVP: Application-Layer Enforcement

Every API query includes a `workspace_id` filter derived from the authenticated user's JWT:

```python
# FastAPI dependency: extract workspace_id from JWT
async def get_workspace_id(auth: AuthResult = Depends(require_auth())) -> str:
    """Returns the workspace_id from the authenticated user's token.

    This is the SOLE source of workspace_id for all queries.
    It cannot be overridden by query parameters or request body.
    """
    return auth.workspace_id
```

#### Repository Base Class

All database repositories inherit from a base class that enforces workspace filtering:

```python
class BaseRepository:
    """Base repository with mandatory workspace_id filtering.

    Every query method includes workspace_id in its WHERE clause.
    There is no method to query without workspace_id.
    """

    def __init__(self, db: AsyncSession, workspace_id: str):
        self.db = db
        self.workspace_id = workspace_id

    async def get_by_id(self, table, id: str):
        """Get a record by ID, scoped to workspace."""
        result = await self.db.execute(
            select(table).where(
                table.c.id == id,
                table.c.workspace_id == self.workspace_id,  # Always present
            )
        )
        return result.first()

    async def list_all(self, table, **filters):
        """List records, always scoped to workspace."""
        query = select(table).where(
            table.c.workspace_id == self.workspace_id,  # Always present
        )
        for key, value in filters.items():
            query = query.where(getattr(table.c, key) == value)
        return (await self.db.execute(query)).all()

    async def create(self, table, **data):
        """Create a record, always setting workspace_id."""
        data["workspace_id"] = self.workspace_id  # Always set
        result = await self.db.execute(
            insert(table).values(**data).returning(table)
        )
        return result.first()
```

#### Integration Test Requirements

Cross-workspace data leakage is verified by integration tests:

```python
async def test_workspace_isolation():
    """Verify that workspace A cannot access workspace B data."""
    # Setup: create vault in workspace A
    vault_a = await create_vault(workspace_id="ws_A", name="Acme Contract")

    # Test: query from workspace B should not find it
    repo_b = VaultRepository(db, workspace_id="ws_B")
    result = await repo_b.get_by_id(vaults, vault_a.id)
    assert result is None  # Must not leak across workspaces

    # Test: list from workspace B should not include it
    results = await repo_b.list_all(vaults)
    assert vault_a.id not in [r.id for r in results]

    # Test: direct ID access from workspace B should fail
    response = await client_b.get(f"/api/vaults/{vault_a.id}")
    assert response.status_code == 404  # Not 403 (no information leak)
```

### Enterprise: PostgreSQL Row-Level Security (RLS)

RLS adds database-enforced isolation as a second layer on top of application filtering:

```sql
-- Enable RLS on all tenant-scoped tables
ALTER TABLE vaults ENABLE ROW LEVEL SECURITY;
ALTER TABLE patches ENABLE ROW LEVEL SECURITY;
ALTER TABLE channel_events ENABLE ROW LEVEL SECURITY;
ALTER TABLE audit_events ENABLE ROW LEVEL SECURITY;
ALTER TABLE otto_sessions ENABLE ROW LEVEL SECURITY;
-- ... (all tables with workspace_id)

-- Create isolation policy
CREATE POLICY workspace_isolation ON vaults
    USING (workspace_id = current_setting('app.workspace_id')::uuid);

CREATE POLICY workspace_isolation ON patches
    USING (workspace_id = current_setting('app.workspace_id')::uuid);

-- ... (same policy on every table)
```

**Connection-time setup:**

```python
# Set workspace_id at connection time from JWT
async def setup_rls(db: AsyncSession, workspace_id: str):
    """Set RLS context for this database connection."""
    await db.execute(
        text("SET app.workspace_id = :ws_id"),
        {"ws_id": workspace_id}
    )
```

**Double enforcement:** With RLS enabled, even if a bug in the application layer omits the `WHERE workspace_id = ...` clause, the database silently filters rows. Even SQL injection attacks cannot access other workspaces because RLS is enforced at the PostgreSQL engine level, below the query parser.

### Blast Radius Analysis

| Scenario | MVP (App-Layer) | Enterprise (App + RLS) |
|----------|----------------|----------------------|
| Bug omits workspace_id filter | Data leak possible | RLS prevents leak |
| SQL injection in API parameter | Data leak possible | RLS prevents cross-workspace access |
| Compromised API server | Full workspace access | Full workspace access (RLS trusts app-set context) |
| Compromised database credentials | All workspace access | All workspace access (RLS bypassed by superuser) |
| Stolen database backup | All data visible (unless encrypted) | All data visible (RLS is runtime-only) |

**Key insight:** RLS protects against application bugs and SQL injection but not against database-level compromise. That is the job of the cipher model (see `data-cipher-model.md`).

---

## 4. API Endpoint Permission Matrix

Every API endpoint with its required permissions, additional checks, and audit events:

### Authentication Endpoints

| Endpoint | Method | Required Permission | Additional Checks | Audit Event |
|----------|--------|-------------------|-------------------|-------------|
| `/api/auth/google/verify` | POST | None (public) | Rate limit: 10/min/IP | `auth.login` or `auth.login_failed` |
| `/api/auth/refresh` | POST | Refresh token cookie | Rate limit: 30/min/user | `auth.refresh` |
| `/api/auth/logout` | POST | Bearer token | Rate limit: 5/min/user | `auth.logout` |
| `/api/auth/logout-all` | POST | Bearer token | Rate limit: 2/min/user | `auth.logout_all` |
| `/api/auth/me` | GET | Bearer token | Rate limit: 60/min/user | None |
| `/api/auth/config` | GET | None (public) | None | None |
| `/api/auth/dev/login` | POST | None | `AIRLOCK_DEV_MODE=true` only; returns 404 in production | `auth.login` |
| `/api/auth/sessions` | GET | Bearer token | None | None |
| `/api/auth/sessions/{family_id}` | DELETE | Bearer token | Can only revoke own sessions | `auth.session.revoked` |

### Vault Endpoints

| Endpoint | Method | Required Permission | Additional Checks | Audit Event |
|----------|--------|-------------------|-------------------|-------------|
| `/api/vaults` | GET | `view_vaults` | workspace_id filter (mandatory) | None |
| `/api/vaults` | POST | `create_vaults` | workspace_id set from JWT | `vault.created` |
| `/api/vaults/{id}` | GET | `view_vaults` | workspace_id filter | None |
| `/api/vaults/{id}` | PATCH | `edit_vault_metadata` | workspace_id filter; owner or admin | `vault.updated` |
| `/api/vaults/{id}` | DELETE | `delete_vaults` | Architect only; workspace_id filter | `vault.deleted` |
| `/api/vaults/{id}/archive` | POST | `archive_vaults` | Owner+; workspace_id filter | `vault.archived` |
| `/api/vaults/{id}/members` | GET | `view_vaults` | workspace_id filter | None |
| `/api/vaults/{id}/members` | POST | `manage_members` | Module Owner+ | `vault.member_added` |

### Patch Workflow Endpoints

| Endpoint | Method | Required Permission | Additional Checks | Audit Event |
|----------|--------|-------------------|-------------------|-------------|
| `/api/vaults/{id}/patches` | GET | `view_vaults` | workspace_id filter | None |
| `/api/vaults/{id}/patches` | POST | `create_patches` | Vault member; workspace_id filter | `patch.created` |
| `/api/vaults/{id}/patches/{pid}` | GET | `view_vaults` | workspace_id filter | None |
| `/api/vaults/{id}/patches/{pid}/submit` | POST | `submit_patches` | Author only; workspace_id filter | `patch.submitted` |
| `/api/vaults/{id}/patches/{pid}/approve` | POST | `approve_low_risk` / `approve_medium_risk` / `approve_high_risk` (by risk level) | **NOT author** (`require_not_author`); workspace_id filter | `patch.approved` |
| `/api/vaults/{id}/patches/{pid}/reject` | POST | `approve_low_risk`+ | NOT author; workspace_id filter | `patch.rejected` |
| `/api/vaults/{id}/patches/{pid}/apply` | POST | `apply_patches` | Owner+; patch must be approved; workspace_id filter | `patch.applied` |

### Extraction Endpoints

| Endpoint | Method | Required Permission | Additional Checks | Audit Event |
|----------|--------|-------------------|-------------------|-------------|
| `/api/vaults/{id}/extractions` | GET | `view_extraction` | workspace_id filter | None |
| `/api/vaults/{id}/extractions` | POST | `run_extraction` | workspace_id filter; rate limit | `extraction.started` |
| `/api/extraction/config` | GET | `configure_extraction` | Designer+ | None |
| `/api/extraction/config` | PUT | `configure_extraction` | Designer+; workspace_id filter | `extraction.config_changed` |

### Triage Endpoints

| Endpoint | Method | Required Permission | Additional Checks | Audit Event |
|----------|--------|-------------------|-------------------|-------------|
| `/api/triage` | GET | `view_triage` | workspace_id filter | None |
| `/api/triage` | POST | `create_triage` | workspace_id filter | `triage.created` |
| `/api/triage/{id}/resolve` | POST | `resolve_triage` | Gatekeeper+; workspace_id filter | `triage.resolved` |
| `/api/triage/{id}/dismiss` | POST | `resolve_triage` | Gatekeeper+; workspace_id filter | `triage.dismissed` |

### Task Endpoints

| Endpoint | Method | Required Permission | Additional Checks | Audit Event |
|----------|--------|-------------------|-------------------|-------------|
| `/api/tasks` | GET | `view_tasks` | workspace_id filter | None |
| `/api/tasks` | POST | `manage_tasks` | workspace_id filter | `task.created` |
| `/api/tasks/{id}` | PATCH | `manage_tasks` | Assignee or admin; workspace_id filter | `task.updated` |
| `/api/tasks/{id}/complete` | POST | `manage_tasks` | Assignee or admin; workspace_id filter | `task.completed` |

### AI Agent (Otto) Endpoints

| Endpoint | Method | Required Permission | Additional Checks | Audit Event |
|----------|--------|-------------------|-------------------|-------------|
| `/api/otto/chat` | POST | `chat_with_otto` | Rate limit per user; circuit breaker | `security.llm_call` |
| `/api/otto/sessions` | GET | `chat_with_otto` | workspace_id filter; own sessions only | None |
| `/api/otto/sessions/{id}` | GET | `chat_with_otto` | Own session only; workspace_id filter | None |
| `/api/otto/config` | GET | `configure_otto` | Owner+ | None |
| `/api/otto/config` | PUT | `configure_otto` | Owner+; workspace_id filter | `otto.config_changed` |

### Admin Endpoints

| Endpoint | Method | Required Permission | Additional Checks | Audit Event |
|----------|--------|-------------------|-------------------|-------------|
| `/api/workspaces/{ws_id}/members` | GET | `manage_members` | workspace_id from JWT must match | None |
| `/api/workspaces/{ws_id}/members/{uid}/org-role` | PUT | `manage_members` | Cannot change own role; cannot promote above own level | `auth.role.changed` |
| `/api/workspaces/{ws_id}/members/{uid}/module-role` | PUT | `manage_roles` or `manage_members` | Cannot assign role higher than own in that module | `auth.role.changed` |
| `/api/workspaces/{ws_id}/overrides` | GET | `manage_roles` | workspace_id filter | None |
| `/api/workspaces/{ws_id}/overrides` | POST | `manage_roles` | Cannot grant beyond own ceiling; SoD is unbypassable | `auth.permission.override` |
| `/api/workspaces/{ws_id}/overrides/{id}` | DELETE | `manage_roles` | workspace_id filter | `auth.permission.override` |
| `/api/workspaces/{ws_id}/roles` | GET | `manage_roles` | workspace_id filter | None |
| `/api/workspaces/{ws_id}/roles` | POST | `manage_roles` | Cannot create role above own hierarchy; reserved names blocked | `role.created` |
| `/api/workspaces/{ws_id}/roles/{id}` | PUT | `manage_roles` | System roles immutable; cannot modify above own level | `role.modified` |
| `/api/workspaces/{ws_id}/roles/{id}` | DELETE | `manage_roles` | System roles cannot be deleted | `role.deleted` |
| `/api/workspaces/{ws_id}/roles/{id}/assign` | POST | `manage_roles` | workspace_id filter | `role.assigned` |
| `/api/workspaces/{ws_id}/roles/{id}/assign/{uid}` | DELETE | `manage_roles` | workspace_id filter | `role.revoked` |
| `/api/workspaces/{ws_id}/roles/reorder` | PUT | `manage_roles` | System role positions pinned; cannot move above own | `role.modified` |
| `/api/workspaces/{ws_id}/api-keys` | GET | `manage_connectors` | workspace_id filter | None |
| `/api/workspaces/{ws_id}/api-keys` | POST | `manage_connectors` | workspace_id filter | `auth.api_key.created` |
| `/api/workspaces/{ws_id}/api-keys/{id}` | DELETE | `manage_connectors` | workspace_id filter | `auth.api_key.revoked` |

### Feature Control Plane Endpoints

| Endpoint | Method | Required Permission | Additional Checks | Audit Event |
|----------|--------|-------------------|-------------------|-------------|
| `/api/features` | GET | `toggle_features` | workspace_id filter | None |
| `/api/features/{flag}` | PUT | `toggle_features` | Owner+; workspace_id filter | `security.feature_flag_changed` |
| `/api/features/{flag}/calibrate` | PUT | `calibrate_thresholds` | Owner+; within allowed range | `feature.calibrated` |
| `/api/system/health` | GET | `system_health` | Owner+ | None |

### Permission Endpoints

| Endpoint | Method | Required Permission | Additional Checks | Audit Event |
|----------|--------|-------------------|-------------------|-------------|
| `/api/permissions/effective` | GET | Bearer token | Own permissions only; workspace_id filter | None |
| `/api/permissions/check` | GET | Bearer token | Own permissions only | None |
| `/api/permissions/catalog` | GET | `manage_roles` | None | None |

### Export Endpoints

| Endpoint | Method | Required Permission | Additional Checks | Audit Event |
|----------|--------|-------------------|-------------------|-------------|
| `/api/vaults/{id}/export/csv` | GET | `export_csv` | workspace_id filter | `security.data_export` |
| `/api/vaults/{id}/export/pdf` | GET | `export_pdf` | workspace_id filter | `security.data_export` |
| `/api/vaults/{id}/export/json` | GET | `export_json` | workspace_id filter | `security.data_export` |
| `/api/vaults/{id}/export/docx` | GET | `export_docx` | workspace_id filter | `security.data_export` |

---

## 5. Role Simulation Sandbox

### Purpose

Admins can "simulate" another role to verify that permission configuration works as expected before assigning it to a real user. This is a diagnostic tool, not a privilege escalation mechanism.

### Security Boundaries

| Property | Constraint |
|----------|-----------|
| **Access** | Only users with `manage_roles` permission can enter simulation mode |
| **Read-only** | All state mutations are blocked. No POST, PUT, PATCH, DELETE operations succeed. |
| **Cannot simulate up** | Users cannot simulate a role with higher hierarchy_position than their own org role allows |
| **Timeout** | Maximum 30 minutes, then automatically reverts to real role |
| **Audit trail** | Entry and exit from simulation mode are logged as `security.role_simulation_start` and `security.role_simulation_end` |
| **Visual indicator** | UI shows a persistent banner: "Simulating [Role Name] -- Read-Only Mode" |
| **No cascading** | Cannot enter simulation while already in simulation |

### Implementation

```python
@router.post("/api/admin/simulate-role")
async def start_role_simulation(
    request: SimulateRoleRequest,
    auth: AuthResult = Depends(require_auth()),
    _perm = Depends(require_permission("manage_roles")),
):
    """Enter role simulation mode. Sets a simulation flag on the session."""
    target_role = await get_role(request.role_id)

    # Cannot simulate higher-privilege role
    if target_role.hierarchy_position > get_max_hierarchy(auth.user_id, auth.workspace_id):
        raise HTTPException(403, "Cannot simulate a role above your own privilege level")

    # Set simulation state (stored in Redis, keyed by user_id)
    await redis.setex(
        f"sim:{auth.workspace_id}:{auth.user_id}",
        1800,  # 30 minute timeout
        json.dumps({
            "role_id": request.role_id,
            "role_name": target_role.name,
            "started_at": datetime.utcnow().isoformat(),
        })
    )

    await emit_audit_event("security.role_simulation_start", {
        "user_id": auth.user_id,
        "simulated_role": target_role.name,
        "timeout_minutes": 30,
    })

    return {"status": "simulation_active", "role": target_role.name, "expires_in": 1800}
```

**Mutation blocking middleware:**

```python
async def block_simulation_writes(request: Request):
    """Middleware: block all write operations during role simulation."""
    auth = getattr(request.state, 'auth', None)
    if auth is None:
        return

    sim_key = f"sim:{auth.workspace_id}:{auth.user_id}"
    if await redis.exists(sim_key):
        if request.method in ("POST", "PUT", "PATCH", "DELETE"):
            # Allow simulation exit endpoint
            if request.url.path == "/api/admin/simulate-role/exit":
                return
            raise HTTPException(
                403,
                "Write operations are blocked during role simulation. Exit simulation first."
            )
```

---

## 6. Vault Member Inheritance

### Membership Model

Vaults in Airlock follow a hierarchical structure (Parent > Division > Counterparty > Item). Membership cascades from parent to child vaults with explicit tracking:

| Property | Value |
|----------|-------|
| Cascade direction | Parent -> Child (downward only) |
| Tracking | `inherited` boolean flag on membership records |
| Override | Direct membership on child vault overrides inherited (can grant more, not less) |
| Removal cascade | Removing parent membership cascades removal to inherited children |
| Direct preservation | Direct child membership is preserved even if parent membership is removed |

### Inheritance Rules

```
Scenario: Ana has direct membership on Parent Vault "Acme Corp"

1. Ana automatically has inherited membership on:
   - Division: "Acme Corp / Music"
   - Counterparty: "Acme Corp / Music / Sub-Label A"
   - Item: "Acme Corp / Music / Sub-Label A / Distribution Agreement #1"

2. If admin adds DIRECT membership for Ana on "Sub-Label A":
   - Ana now has DIRECT membership on "Sub-Label A" (overrides inherited)
   - Inherited membership on children of "Sub-Label A" remains

3. If admin removes Ana from Parent "Acme Corp":
   - Inherited memberships on "Acme Corp / Music" are removed
   - DIRECT membership on "Sub-Label A" is PRESERVED
   - Children of "Sub-Label A" still inherit from Ana's direct membership
```

### Database Schema

```sql
-- Vault membership with inheritance tracking
CREATE TABLE vault_members (
    vault_id UUID NOT NULL REFERENCES vaults(id) ON DELETE CASCADE,
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    workspace_id UUID NOT NULL REFERENCES workspaces(id),
    membership_type TEXT NOT NULL DEFAULT 'direct',  -- 'direct' | 'inherited'
    inherited_from UUID REFERENCES vaults(id),       -- parent vault ID (NULL if direct)
    granted_by UUID REFERENCES users(id),
    granted_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (vault_id, user_id)
);

CREATE INDEX idx_vault_members_user ON vault_members(user_id, workspace_id);
CREATE INDEX idx_vault_members_inherited ON vault_members(inherited_from)
    WHERE membership_type = 'inherited';
```

### Access Check with Inheritance

```python
async def check_vault_access(user_id: str, vault_id: str, workspace_id: str) -> bool:
    """Check if user has access to a vault (direct or inherited)."""
    member = await db.execute(
        select(vault_members).where(
            vault_members.c.vault_id == vault_id,
            vault_members.c.user_id == user_id,
            vault_members.c.workspace_id == workspace_id,
        )
    )
    return member.first() is not None
    # Both 'direct' and 'inherited' membership_type grant access
```

---

## Related Specs

- [overview.md](./overview.md) -- Security architecture anchor, 70-question inventory
- [auth-and-identity.md](./auth-and-identity.md) -- JWT lifecycle, OAuth security, API key management
- [audit-and-immutability.md](./audit-and-immutability.md) -- Append-only audit enforcement, security event schema
- [data-cipher-model.md](./data-cipher-model.md) -- Encryption at rest, tenant isolation at storage layer
- [Auth Overview](../Auth/overview.md) -- Permission middleware implementation, rate limiting
- [Auth Migrations](../Auth/migrations.md) -- Database schema for roles, overrides, custom roles
- [Auth API Endpoints](../Auth/api-endpoints.md) -- Complete auth + permission + role management API surface
- [Roles Overview](../Roles/overview.md) -- Role architecture, custom roles, permission catalog, Discord parallels
- [FeatureControlPlane Overview](../FeatureControlPlane/overview.md) -- Feature flags, circuit breaker, calibration
