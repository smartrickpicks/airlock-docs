# Auth Migrations — Database Schema

> **Status:** SPECCED — All auth-related tables needed for Airlock that don't exist in OrcestrateOS.
>
> **Context:** OrcestrateOS has `users`, `user_workspace_roles`, and `api_keys` tables. Airlock needs additional tables for refresh tokens, module-level roles, permission overrides, custom roles, and auth audit events.

---

## Migration: `025_auth_refresh_tokens.sql`

Refresh token storage for JWT rotation with replay detection.

```sql
-- Refresh token storage (rotation + replay detection)
CREATE TABLE refresh_tokens (
    id TEXT PRIMARY KEY,                       -- jti (unique token ID, prefix: rt_)
    family_id UUID NOT NULL,                   -- token family for rotation detection
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    workspace_id UUID NOT NULL REFERENCES workspaces(id),
    issued_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at TIMESTAMPTZ NOT NULL,
    used_at TIMESTAMPTZ,                       -- NULL = unused; set when rotated to new token
    revoked_at TIMESTAMPTZ,                    -- NULL = active; set when explicitly revoked
    revoked_reason TEXT,                       -- 'rotation' | 'logout' | 'admin' | 'replay_detected' | 'logout_all'
    user_agent TEXT,                            -- browser/client UA string for audit
    ip_address INET,                           -- client IP for audit
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Active tokens per user (for logout, session management)
CREATE INDEX idx_refresh_user_active ON refresh_tokens(user_id, workspace_id)
    WHERE revoked_at IS NULL AND used_at IS NULL;

-- Family lookup (for replay detection)
CREATE INDEX idx_refresh_family ON refresh_tokens(family_id);

-- Expired token cleanup (background job)
CREATE INDEX idx_refresh_expired ON refresh_tokens(expires_at)
    WHERE revoked_at IS NULL;

-- Cleanup: auto-delete tokens older than 30 days
-- (Run via cron/BullMQ job, not a trigger)
-- DELETE FROM refresh_tokens WHERE expires_at < now() - INTERVAL '30 days';
```

---

## Migration: `026_module_roles.sql`

Module-level role assignments — extending the org-level `user_workspace_roles`.

```sql
-- Module-level role assignments
-- Each user can have a different role per module within a workspace
CREATE TABLE user_module_roles (
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    module_id TEXT NOT NULL,                    -- 'contracts' | 'crm' | 'tasks' | 'calendar' | 'documents'
    module_role TEXT NOT NULL DEFAULT 'viewer', -- 'builder' | 'gatekeeper' | 'owner' | 'designer' | 'viewer'
    agentic_role TEXT,                          -- optional: 'truth_keeper', 'evidence_curator', etc.
    pi_profile TEXT,                            -- optional: 'analyzer', 'guardian', 'maverick', etc.
    assigned_by UUID REFERENCES users(id),
    assigned_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, workspace_id, module_id)
);

CREATE INDEX idx_module_roles_workspace ON user_module_roles(workspace_id, module_id);

-- Validate module_role values
ALTER TABLE user_module_roles
    ADD CONSTRAINT chk_module_role
    CHECK (module_role IN ('builder', 'gatekeeper', 'owner', 'designer', 'viewer'));
```

---

## Migration: `027_permission_overrides.sql`

Ad-hoc permission grants and denies — Discord-style per-user overrides on top of base roles.

```sql
-- Ad-hoc permission overrides (per-user, on top of base role)
-- Admins can grant or deny specific permissions to individual users
CREATE TABLE permission_overrides (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    scope_type TEXT NOT NULL,                   -- 'module' | 'channel'
    scope_id TEXT NOT NULL,                     -- module_id (e.g., 'contracts') or channel_id (UUID)
    permission TEXT NOT NULL,                   -- permission key (e.g., 'approve_low_risk', 'export_csv')
    effect TEXT NOT NULL DEFAULT 'grant',       -- 'grant' | 'deny'
    reason TEXT,                                -- why this override was granted (free text)
    expires_at TIMESTAMPTZ,                     -- NULL = permanent; date = temporary escalation
    granted_by UUID NOT NULL REFERENCES users(id),
    granted_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    revoked_at TIMESTAMPTZ,                    -- NULL = active; set when revoked
    revoked_by UUID REFERENCES users(id),
    revoked_reason TEXT                         -- why it was revoked
);

-- Active overrides for a user (permission computation)
CREATE INDEX idx_overrides_user_active ON permission_overrides(user_id, workspace_id)
    WHERE revoked_at IS NULL;

-- Active overrides by scope (for admin views)
CREATE INDEX idx_overrides_scope ON permission_overrides(scope_type, scope_id)
    WHERE revoked_at IS NULL;

-- Validate effect values
ALTER TABLE permission_overrides
    ADD CONSTRAINT chk_override_effect
    CHECK (effect IN ('grant', 'deny'));

ALTER TABLE permission_overrides
    ADD CONSTRAINT chk_override_scope
    CHECK (scope_type IN ('module', 'channel'));

-- Unique constraint: one override per user+scope+permission (no duplicates)
CREATE UNIQUE INDEX idx_overrides_unique ON permission_overrides(user_id, workspace_id, scope_type, scope_id, permission)
    WHERE revoked_at IS NULL;
```

---

## Migration: `028_custom_roles.sql`

Discord-style custom roles with granular permission assignment and drag-to-reorder hierarchy.

```sql
-- Custom role definitions
-- 5 system roles (Builder, Gatekeeper, Owner, Designer, Viewer) are seeded as is_system=true
-- Admins can create unlimited custom roles on top of these
CREATE TABLE custom_roles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    color TEXT DEFAULT '#64748B',                -- Hex color for UI badges
    description TEXT,
    base_template TEXT,                          -- which system role this was cloned from (or NULL)
    permissions TEXT[] NOT NULL DEFAULT '{}',    -- array of permission strings
    hierarchy_position INT NOT NULL,             -- higher = more authority; used for ordering
    is_system BOOLEAN NOT NULL DEFAULT false,    -- true for the 5 base roles (can't be deleted/renamed)
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(workspace_id, name)
);

-- Hierarchy ordering lookup
CREATE INDEX idx_custom_roles_hierarchy ON custom_roles(workspace_id, hierarchy_position);

-- User ↔ custom role assignments (many-to-many, Discord-style multiple roles per user)
CREATE TABLE user_custom_roles (
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    role_id UUID NOT NULL REFERENCES custom_roles(id) ON DELETE CASCADE,
    assigned_by UUID REFERENCES users(id),
    assigned_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, workspace_id, role_id)
);

-- Lookup: all roles for a user
CREATE INDEX idx_user_roles_lookup ON user_custom_roles(user_id, workspace_id);

-- Lookup: all users with a specific role
CREATE INDEX idx_role_users_lookup ON user_custom_roles(role_id);

-- Seed system roles for every new workspace
-- (Run after workspace creation via application code, not a trigger)
--
-- INSERT INTO custom_roles (workspace_id, name, color, permissions, hierarchy_position, is_system)
-- VALUES
--   (:ws, 'Architect', '#A855F7', ARRAY['*'], 500, true),
--   (:ws, 'Owner',     '#22C55E', ARRAY[<owner_perms>], 400, true),
--   (:ws, 'Designer',  '#00D1FF', ARRAY[<designer_perms>], 300, true),
--   (:ws, 'Gatekeeper','#EAB308', ARRAY[<gatekeeper_perms>], 200, true),
--   (:ws, 'Builder',   '#3B82F6', ARRAY[<builder_perms>], 100, true),
--   (:ws, 'Viewer',    '#64748B', ARRAY[<viewer_perms>], 0, true);
```

---

## Migration: `029_linked_roles.sql`

Automated role assignment from external connections (Salesforce, HR systems, PI assessments).

```sql
-- External connections that can trigger role assignment
CREATE TABLE linked_role_connections (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    provider TEXT NOT NULL,                      -- 'salesforce' | 'hr_system' | 'pi_assessment' | 'google_workspace'
    provider_user_id TEXT,                       -- external system's user ID
    metadata JSONB NOT NULL DEFAULT '{}',        -- provider-specific data (department, title, etc.)
    connected_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    disconnected_at TIMESTAMPTZ,                 -- set when connection revoked
    last_synced_at TIMESTAMPTZ
);

CREATE INDEX idx_linked_connections_user ON linked_role_connections(user_id, workspace_id)
    WHERE disconnected_at IS NULL;

-- Admin-configured mappings: "if provider X has condition Y, grant role Z"
CREATE TABLE linked_role_mappings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id) ON DELETE CASCADE,
    provider TEXT NOT NULL,                      -- which provider this mapping applies to
    condition JSONB NOT NULL,                    -- {"field": "department", "equals": "Legal"}
    target_module TEXT,                          -- NULL = org-level, or specific module_id
    target_role TEXT NOT NULL,                   -- role name to assign
    is_active BOOLEAN NOT NULL DEFAULT true,
    created_by UUID NOT NULL REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_linked_mappings_workspace ON linked_role_mappings(workspace_id, provider)
    WHERE is_active = true;
```

---

## Migration: `030_auth_audit_events.sql`

Auth-specific audit events. These use the existing `audit_events` table pattern but ensure the auth event types are indexed efficiently.

```sql
-- Partial index for auth events (fast lookup for security monitoring)
CREATE INDEX idx_audit_auth_events ON audit_events(workspace_id, created_at DESC)
    WHERE event_type LIKE 'auth.%';

-- Partial index for failed auth attempts (security alerting)
CREATE INDEX idx_audit_auth_failures ON audit_events(created_at DESC)
    WHERE event_type IN ('auth.login.failed', 'auth.login.blocked', 'auth.refresh.replay');

-- Rate limit tracking table (Redis is primary, this is fallback/audit)
CREATE TABLE auth_rate_limits (
    key TEXT NOT NULL,                           -- 'ip:{ip}:login' or 'user:{user_id}:refresh'
    window_start TIMESTAMPTZ NOT NULL,
    request_count INT NOT NULL DEFAULT 1,
    PRIMARY KEY (key, window_start)
);

-- Auto-cleanup old rate limit entries
-- DELETE FROM auth_rate_limits WHERE window_start < now() - INTERVAL '1 hour';
```

---

## Migration: `031_update_org_roles.sql`

Update the existing `user_workspace_roles` table to use Airlock's org role vocabulary.

```sql
-- Update org role values from OrcestrateOS to Airlock vocabulary
-- OrcestrateOS: analyst, verifier, admin, architect
-- Airlock:      member, lead, director, executive

-- Add new role constraint (allowing both during migration)
ALTER TABLE user_workspace_roles
    DROP CONSTRAINT IF EXISTS chk_workspace_role;

ALTER TABLE user_workspace_roles
    ADD CONSTRAINT chk_workspace_role
    CHECK (role IN ('member', 'lead', 'director', 'executive',
                    'analyst', 'verifier', 'admin', 'architect'));  -- old values allowed during migration

-- Migration function: update old role values to new
-- Run once, then remove old values from constraint
--
-- UPDATE user_workspace_roles SET role = 'member'    WHERE role = 'analyst';
-- UPDATE user_workspace_roles SET role = 'lead'      WHERE role = 'verifier';
-- UPDATE user_workspace_roles SET role = 'director'  WHERE role = 'admin';
-- UPDATE user_workspace_roles SET role = 'executive' WHERE role = 'architect';
--
-- Then tighten constraint:
-- ALTER TABLE user_workspace_roles
--     DROP CONSTRAINT chk_workspace_role,
--     ADD CONSTRAINT chk_workspace_role
--     CHECK (role IN ('member', 'lead', 'director', 'executive'));
```

---

## Default Permission Sets

These are the base permissions for each system role. Stored in `custom_roles.permissions` array for system roles.

```python
# Default permissions per system role
SYSTEM_ROLE_PERMISSIONS = {
    "viewer": [
        "view_vaults", "view_extraction", "view_triage", "view_tasks",
    ],
    "builder": [
        # Viewer permissions +
        "view_vaults", "view_extraction", "view_triage", "view_tasks",
        "create_vaults", "edit_vault_metadata",
        "run_extraction",
        "create_patches", "submit_patches",
        "create_triage",
        "manage_tasks",
        "chat_with_otto",
        "export_csv", "export_pdf", "export_docx",
    ],
    "gatekeeper": [
        # Builder permissions +
        "view_vaults", "view_extraction", "view_triage", "view_tasks",
        "create_vaults", "edit_vault_metadata",
        "run_extraction",
        "create_patches", "submit_patches",
        "approve_low_risk", "approve_medium_risk",
        "create_triage", "resolve_triage",
        "manage_tasks",
        "chat_with_otto",
        "export_csv", "export_pdf", "export_docx",
        "view_audit_log",
    ],
    "owner": [
        # Gatekeeper permissions +
        "view_vaults", "view_extraction", "view_triage", "view_tasks",
        "create_vaults", "edit_vault_metadata", "archive_vaults",
        "run_extraction",
        "create_patches", "submit_patches",
        "approve_low_risk", "approve_medium_risk", "approve_high_risk", "apply_patches",
        "create_triage", "resolve_triage",
        "manage_tasks",
        "chat_with_otto", "configure_otto",
        "export_csv", "export_pdf", "export_json", "export_docx",
        "manage_members", "manage_roles",
        "toggle_features", "calibrate_thresholds",
        "view_audit_log", "system_health",
        "manage_connectors",
    ],
    "designer": [
        # Specialized role — schema and config focus
        "view_vaults", "view_extraction", "view_triage", "view_tasks",
        "create_vaults", "edit_vault_metadata",
        "run_extraction", "configure_extraction",
        "create_patches", "submit_patches",
        "create_triage",
        "manage_tasks",
        "chat_with_otto", "configure_otto",
        "export_csv", "export_pdf", "export_json", "export_docx",
        "calibrate_thresholds",
        "view_audit_log",
    ],
    "architect": [
        "*",  # Wildcard — all permissions. Checked as: if '*' in permissions, allow.
    ],
}
```

---

## Org Role Ceiling

The org role sets an absolute ceiling on what module-level permissions can be effective:

```python
# Org role ceiling — max permissions regardless of module role
ORG_ROLE_CEILING = {
    "member": {
        # Can have any module-level permissions EXCEPT admin-only ones
        "manage_members", "manage_roles", "toggle_features",
        "system_health", "manage_connectors",
        "delete_vaults",  # permanent delete is admin-only
    },  # These are EXCLUDED — member can't have these even with overrides
    "lead": {
        "manage_roles", "toggle_features",
        "system_health",
        "delete_vaults",
    },  # Excluded set
    "director": {
        "delete_vaults",
    },  # Only permanent delete excluded
    "executive": set(),  # No ceiling — can have everything
}

# In permission computation:
# effective = computed_permissions - ORG_ROLE_CEILING[org_role]
```

---

## Related Specs

- **[Auth Overview](./overview.md)** — Authentication pipeline, JWT strategy, middleware
- **[API Endpoints](./api-endpoints.md)** — Complete auth API surface
- **[Roles & Permissions](../Roles/overview.md)** — Permission model, module roles, custom roles
- **[Security & Architecture](../Security/overview.md)** — Encryption, threat model
