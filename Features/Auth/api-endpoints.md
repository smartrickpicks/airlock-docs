# Auth API Endpoints — Complete Surface

> **Status:** SPECCED — Every auth and permission endpoint needed for Airlock.
>
> **Base path:** `/api/auth/` for authentication, `/api/permissions/` for authorization management.
>
> **Source code to port:** OrcestrateOS `server/routes/auth_google.py` (203 lines)

---

## Authentication Endpoints

### `POST /api/auth/google/verify`

Exchange a Google OAuth 2.0 ID token for Airlock access + refresh tokens.

| Property | Value |
|----------|-------|
| Auth required | No |
| Rate limit | 10/min per IP |
| Audit event | `auth.login.success` or `auth.login.failed` |

**Request:**
```json
{
  "credential": "eyJhbGciOi...",     // Google ID token from OAuth flow
  "workspace_id": "ws_abc123"         // Target workspace (optional if single-workspace user)
}
```

**Response (200):**
```json
{
  "access_token": "eyJhbGciOi...",    // Airlock JWT (15min expiry)
  "user": {
    "id": "usr_abc123",
    "email": "ana@company.com",
    "display_name": "Ana Chen",
    "avatar_url": "https://...",
    "org_role": "member",
    "workspace_id": "ws_abc123"
  }
}
// + Set-Cookie: refresh_token=...; HttpOnly; Secure; SameSite=Strict; Path=/api/auth; Max-Age=604800
```

**Response (403) — first-time user (invited):**
```json
{
  "requires_onboarding": true,
  "access_token": "eyJhbGciOi...",
  "user": { ... }
}
```

**Response (200) — multi-workspace user:**
```json
{
  "requires_workspace_selection": true,
  "workspaces": [
    { "id": "ws_abc", "name": "Acme Corp", "role": "member" },
    { "id": "ws_def", "name": "Nova LLC", "role": "lead" }
  ]
}
```

**Errors:**
- `400` — Missing credential or invalid JSON
- `401` — Invalid Google ID token
- `403` — User not in workspace / account inactive
- `429` — Rate limit exceeded
- `500` — Google OAuth not configured

---

### `POST /api/auth/refresh`

Exchange refresh token (from httpOnly cookie) for new access + refresh token pair.

| Property | Value |
|----------|-------|
| Auth required | Refresh token cookie |
| Rate limit | 30/min per user |
| Audit event | `auth.refresh.success` or `auth.refresh.failed` |

**Request:** No body. Refresh token sent via cookie.

**Response (200):**
```json
{
  "access_token": "eyJhbGciOi..."
}
// + Set-Cookie: refresh_token=<new_token>; HttpOnly; Secure; SameSite=Strict; Path=/api/auth; Max-Age=604800
```

**Errors:**
- `401` — No refresh token cookie, invalid token, expired token
- `401` — Token reuse detected (replay attack — entire family revoked, user must re-login)
- `429` — Rate limit exceeded

---

### `POST /api/auth/logout`

Revoke current refresh token family (single device logout).

| Property | Value |
|----------|-------|
| Auth required | Bearer token |
| Rate limit | 5/min per user |
| Audit event | `auth.logout` |

**Request:** No body.

**Response (200):**
```json
{ "status": "logged_out" }
// + Set-Cookie: refresh_token=; HttpOnly; Secure; SameSite=Strict; Path=/api/auth; Max-Age=0
```

---

### `POST /api/auth/logout-all`

Revoke ALL refresh tokens for the user across all devices.

| Property | Value |
|----------|-------|
| Auth required | Bearer token |
| Rate limit | 2/min per user |
| Audit event | `auth.logout_all` |

**Request:** No body.

**Response (200):**
```json
{
  "status": "logged_out_all",
  "families_revoked": 3
}
```

---

### `GET /api/auth/me`

Return current authenticated user's profile, org role, and module roles.

| Property | Value |
|----------|-------|
| Auth required | Bearer token |
| Rate limit | 60/min per user |
| Audit event | None |

**Response (200):**
```json
{
  "user_id": "usr_abc123",
  "email": "ana@company.com",
  "display_name": "Ana Chen",
  "avatar_url": "https://...",
  "org_role": "member",
  "workspace_id": "ws_abc123",
  "module_roles": {
    "contracts": { "module_role": "owner", "agentic_role": "evidence_curator" },
    "crm": { "module_role": "viewer", "agentic_role": null },
    "tasks": { "module_role": "builder", "agentic_role": null }
  },
  "custom_roles": [
    { "id": "role_xyz", "name": "Senior Analyst", "color": "#00D1FF" }
  ],
  "active_overrides": 2
}
```

---

### `GET /api/auth/config`

Public endpoint — returns OAuth configuration for the login page.

| Property | Value |
|----------|-------|
| Auth required | No |
| Rate limit | None |

**Response (200):**
```json
{
  "google_client_id": "123456789.apps.googleusercontent.com",
  "configured": true,
  "dev_mode": false
}
```

---

### `POST /api/auth/dev/login` (Dev Mode Only)

Login as a seed dev user without Google OAuth. Returns 404 in production.

| Property | Value |
|----------|-------|
| Auth required | No |
| Rate limit | None |
| Availability | `AIRLOCK_DEV_MODE=true` only |

**Request:**
```json
{
  "user": "dev_builder"    // dev_builder | dev_gatekeeper | dev_owner | dev_architect
}
```

**Response (200):**
```json
{
  "access_token": "eyJhbGciOi...",
  "user": {
    "id": "dev_builder",
    "email": "builder@dev.airlock",
    "display_name": "Dev Builder",
    "org_role": "member",
    "workspace_id": "ws_dev"
  }
}
```

---

## Permission Management Endpoints

### `GET /api/permissions/effective`

Compute and return effective permissions for the current user in a given scope.

| Property | Value |
|----------|-------|
| Auth required | Bearer token |
| Rate limit | 30/min per user |

**Query params:**
- `module_id` (required) — which module
- `channel_id` (optional) — specific channel for channel-level overrides

**Response (200):**
```json
{
  "user_id": "usr_abc123",
  "module_id": "contracts",
  "channel_id": null,
  "org_role": "member",
  "module_role": "owner",
  "custom_roles": ["Senior Analyst"],
  "effective_permissions": [
    "view_vaults", "create_vaults", "edit_vault_metadata", "archive_vaults",
    "view_extraction", "run_extraction",
    "create_patches", "submit_patches",
    "approve_low_risk", "approve_medium_risk", "approve_high_risk", "apply_patches",
    "view_triage", "create_triage", "resolve_triage",
    "view_tasks", "manage_tasks",
    "chat_with_otto", "configure_otto",
    "export_csv", "export_pdf", "export_json", "export_docx",
    "manage_members", "manage_roles",
    "toggle_features", "calibrate_thresholds",
    "view_audit_log", "system_health", "manage_connectors"
  ],
  "override_sources": [
    { "permission": "export_json", "source": "ad-hoc grant by dev@company.com", "expires_at": null }
  ]
}
```

---

### `GET /api/permissions/check`

Quick boolean check — does the current user have a specific permission?

| Property | Value |
|----------|-------|
| Auth required | Bearer token |
| Rate limit | 120/min per user |

**Query params:**
- `permission` (required) — permission key to check
- `module_id` (required)
- `channel_id` (optional)

**Response (200):**
```json
{
  "permission": "approve_low_risk",
  "module_id": "contracts",
  "channel_id": "ch_abc123",
  "allowed": false,
  "reason": "Denied by channel-level override (conflict of interest)"
}
```

---

## Role Management Endpoints (Admin)

### `GET /api/workspaces/{ws_id}/members`

List all members with their org roles and module roles.

| Property | Value |
|----------|-------|
| Auth required | Bearer token |
| Permission | `manage_members` |

**Response (200):**
```json
{
  "members": [
    {
      "user_id": "usr_abc123",
      "email": "ana@company.com",
      "display_name": "Ana Chen",
      "avatar_url": "https://...",
      "org_role": "member",
      "module_roles": {
        "contracts": "owner",
        "crm": "viewer",
        "tasks": "builder"
      },
      "custom_roles": ["Senior Analyst"],
      "active_overrides": 2,
      "last_login": "2026-03-03T14:30:00Z"
    }
  ],
  "total": 4
}
```

---

### `PUT /api/workspaces/{ws_id}/members/{user_id}/org-role`

Change a user's org-level role.

| Property | Value |
|----------|-------|
| Auth required | Bearer token |
| Permission | `manage_members` |
| Audit event | `auth.role.changed` |

**Request:**
```json
{
  "role": "lead",
  "reason": "Promoted to team lead"
}
```

**Constraints:**
- Cannot change your own org role
- Cannot promote someone above your own org role
- `executive` role can only be assigned by another `executive`

---

### `PUT /api/workspaces/{ws_id}/members/{user_id}/module-role`

Change a user's module-level role.

| Property | Value |
|----------|-------|
| Auth required | Bearer token |
| Permission | `manage_roles` in the target module, or `manage_members` at org level |
| Audit event | `auth.role.changed` |

**Request:**
```json
{
  "module_id": "contracts",
  "module_role": "gatekeeper",
  "agentic_role": "verifier",
  "reason": "Transitioning from Builder to Gatekeeper"
}
```

**Constraints:**
- Cannot assign a module role higher than your own in that module
- Owner can assign any role in their module
- Org-level `manage_members` permission can assign any module role

---

### `POST /api/workspaces/{ws_id}/overrides`

Create an ad-hoc permission override for a user.

| Property | Value |
|----------|-------|
| Auth required | Bearer token |
| Permission | `manage_roles` (module scope) or `manage_members` (channel scope) |
| Audit event | `auth.override.granted` |

**Request:**
```json
{
  "user_id": "usr_def456",
  "scope_type": "module",
  "scope_id": "contracts",
  "permission": "approve_low_risk",
  "effect": "grant",
  "reason": "Covering for Dev Patel during PTO",
  "expires_at": "2026-03-10T00:00:00Z"
}
```

**Response (201):**
```json
{
  "id": "ovr_xyz789",
  "user_id": "usr_def456",
  "scope_type": "module",
  "scope_id": "contracts",
  "permission": "approve_low_risk",
  "effect": "grant",
  "expires_at": "2026-03-10T00:00:00Z",
  "granted_by": "usr_abc123",
  "granted_at": "2026-03-04T10:00:00Z"
}
```

**Constraints:**
- Cannot override self-approval prevention (SoD is hardcoded)
- Cannot grant permissions beyond your own org role ceiling
- Deny overrides cannot be overridden by other grants

---

### `DELETE /api/workspaces/{ws_id}/overrides/{override_id}`

Revoke an ad-hoc permission override.

| Property | Value |
|----------|-------|
| Auth required | Bearer token |
| Permission | `manage_roles` |
| Audit event | `auth.override.revoked` |

**Request:**
```json
{
  "reason": "PTO coverage period ended"
}
```

---

### `GET /api/workspaces/{ws_id}/overrides`

List all active overrides in the workspace (for admin dashboard).

| Property | Value |
|----------|-------|
| Auth required | Bearer token |
| Permission | `manage_roles` |

**Query params:**
- `user_id` (optional) — filter by user
- `scope_type` (optional) — `module` or `channel`
- `scope_id` (optional) — specific module or channel
- `include_expired` (optional) — include expired/revoked overrides

---

## Custom Role Endpoints (Admin)

### `GET /api/workspaces/{ws_id}/roles`

List all roles (system + custom) ordered by hierarchy position.

| Property | Value |
|----------|-------|
| Auth required | Bearer token |
| Permission | `manage_roles` |

**Response (200):**
```json
{
  "roles": [
    { "id": "role_sys_architect", "name": "Architect", "color": "#A855F7", "hierarchy_position": 500, "is_system": true, "permissions": ["*"], "user_count": 1 },
    { "id": "role_sys_owner", "name": "Owner", "color": "#22C55E", "hierarchy_position": 400, "is_system": true, "permissions": ["..."], "user_count": 2 },
    { "id": "role_custom_1", "name": "Department Head", "color": "#00D1FF", "hierarchy_position": 350, "is_system": false, "permissions": ["..."], "user_count": 1 },
    { "id": "role_sys_gatekeeper", "name": "Gatekeeper", "color": "#EAB308", "hierarchy_position": 200, "is_system": true, "permissions": ["..."], "user_count": 3 },
    { "id": "role_sys_builder", "name": "Builder", "color": "#3B82F6", "hierarchy_position": 100, "is_system": true, "permissions": ["..."], "user_count": 5 },
    { "id": "role_sys_viewer", "name": "Viewer", "color": "#64748B", "hierarchy_position": 0, "is_system": true, "permissions": ["..."], "user_count": 2 }
  ]
}
```

---

### `POST /api/workspaces/{ws_id}/roles`

Create a custom role.

| Property | Value |
|----------|-------|
| Auth required | Bearer token |
| Permission | `manage_roles` |
| Audit event | `role.created` |

**Request:**
```json
{
  "name": "Senior Analyst",
  "color": "#00D1FF",
  "description": "Builder with low-risk approval privileges",
  "base_template": "builder",
  "permissions": [
    "view_vaults", "create_vaults", "edit_vault_metadata",
    "view_extraction", "run_extraction",
    "create_patches", "submit_patches", "approve_low_risk",
    "view_triage", "create_triage",
    "view_tasks", "manage_tasks",
    "chat_with_otto",
    "export_csv", "export_pdf"
  ],
  "hierarchy_position": 150
}
```

**Constraints:**
- Name must be unique within workspace
- Cannot set hierarchy_position above your own role's position
- Cannot assign permissions you don't have yourself
- System role names are reserved (can't create a role named "Builder")

---

### `PUT /api/workspaces/{ws_id}/roles/{role_id}`

Update a custom role's permissions, name, color, or hierarchy position.

| Property | Value |
|----------|-------|
| Auth required | Bearer token |
| Permission | `manage_roles` |
| Audit event | `role.modified` |

**Constraints:**
- System roles cannot be renamed, deleted, or have hierarchy changed
- System role permissions CAN be viewed but NOT modified

---

### `DELETE /api/workspaces/{ws_id}/roles/{role_id}`

Delete a custom role. Users with this role lose it (permissions revert to remaining roles).

| Property | Value |
|----------|-------|
| Auth required | Bearer token |
| Permission | `manage_roles` |
| Audit event | `role.deleted` |

**Constraints:**
- System roles cannot be deleted
- Confirmation required (frontend enforces)
- Users with this role are notified

---

### `POST /api/workspaces/{ws_id}/roles/{role_id}/assign`

Assign a custom role to a user (additive — they keep existing roles).

| Property | Value |
|----------|-------|
| Auth required | Bearer token |
| Permission | `manage_roles` |
| Audit event | `role.assigned` |

**Request:**
```json
{
  "user_id": "usr_def456"
}
```

---

### `DELETE /api/workspaces/{ws_id}/roles/{role_id}/assign/{user_id}`

Remove a custom role from a user.

| Property | Value |
|----------|-------|
| Auth required | Bearer token |
| Permission | `manage_roles` |
| Audit event | `role.revoked` |

---

### `PUT /api/workspaces/{ws_id}/roles/reorder`

Reorder role hierarchy positions (drag-to-reorder UI).

| Property | Value |
|----------|-------|
| Auth required | Bearer token |
| Permission | `manage_roles` |
| Audit event | `role.modified` (for each changed position) |

**Request:**
```json
{
  "order": [
    { "role_id": "role_sys_architect", "position": 500 },
    { "role_id": "role_sys_owner", "position": 400 },
    { "role_id": "role_custom_1", "position": 350 },
    { "role_id": "role_sys_gatekeeper", "position": 200 },
    { "role_id": "role_custom_2", "position": 150 },
    { "role_id": "role_sys_builder", "position": 100 },
    { "role_id": "role_sys_viewer", "position": 0 }
  ]
}
```

**Constraints:**
- System role positions are pinned (Architect at top, Viewer at bottom)
- Custom roles can only be positioned between system roles
- Cannot move a role above your own position

---

## API Key Management Endpoints

### `POST /api/workspaces/{ws_id}/api-keys`

Create a new API key with specific scopes.

| Property | Value |
|----------|-------|
| Auth required | Bearer token |
| Permission | `manage_connectors` |
| Audit event | `auth.apikey.created` |

**Request:**
```json
{
  "name": "CI/CD Pipeline",
  "scopes": ["read", "export"],
  "expires_at": "2026-06-01T00:00:00Z"
}
```

**Response (201):**
```json
{
  "key_id": "ak_abc123",
  "key_value": "airlock_sk_live_abc123def456...",   // Only shown ONCE
  "name": "CI/CD Pipeline",
  "scopes": ["read", "export"],
  "expires_at": "2026-06-01T00:00:00Z",
  "created_at": "2026-03-04T10:00:00Z"
}
```

---

### `GET /api/workspaces/{ws_id}/api-keys`

List all API keys (key_value is never returned after creation).

---

### `DELETE /api/workspaces/{ws_id}/api-keys/{key_id}`

Revoke an API key. Immediate effect.

| Property | Value |
|----------|-------|
| Audit event | `auth.apikey.revoked` |

---

## Auth Sessions Endpoint

### `GET /api/auth/sessions`

List active refresh token families (active sessions/devices) for the current user.

| Property | Value |
|----------|-------|
| Auth required | Bearer token |

**Response (200):**
```json
{
  "sessions": [
    {
      "family_id": "fam_abc123",
      "user_agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)...",
      "ip_address": "192.168.1.100",
      "created_at": "2026-03-01T09:00:00Z",
      "last_used_at": "2026-03-04T14:30:00Z",
      "is_current": true
    },
    {
      "family_id": "fam_def456",
      "user_agent": "Mozilla/5.0 (iPhone; CPU iPhone OS 17_0)...",
      "ip_address": "10.0.0.50",
      "created_at": "2026-03-02T20:00:00Z",
      "last_used_at": "2026-03-03T08:15:00Z",
      "is_current": false
    }
  ]
}
```

### `DELETE /api/auth/sessions/{family_id}`

Revoke a specific session (logout that device).

| Property | Value |
|----------|-------|
| Auth required | Bearer token |
| Audit event | `auth.logout` |

---

## Permission Catalog Endpoint

### `GET /api/permissions/catalog`

List all available permissions with descriptions. Used by the role editor UI.

| Property | Value |
|----------|-------|
| Auth required | Bearer token |
| Permission | `manage_roles` |

**Response (200):**
```json
{
  "categories": [
    {
      "name": "Vaults",
      "permissions": [
        { "key": "view_vaults", "label": "View vault data and events", "default_roles": ["viewer", "builder", "gatekeeper", "owner", "designer"] },
        { "key": "create_vaults", "label": "Create new vaults", "default_roles": ["builder", "gatekeeper", "owner", "designer"] },
        { "key": "edit_vault_metadata", "label": "Edit vault metadata fields", "default_roles": ["builder", "gatekeeper", "owner", "designer"] },
        { "key": "archive_vaults", "label": "Archive/soft-delete vaults", "default_roles": ["owner"] },
        { "key": "delete_vaults", "label": "Permanently delete vaults", "default_roles": ["architect"] }
      ]
    },
    {
      "name": "Extraction",
      "permissions": [
        { "key": "view_extraction", "label": "View extraction results", "default_roles": ["viewer", "builder", "gatekeeper", "owner", "designer"] },
        { "key": "run_extraction", "label": "Trigger extraction on documents", "default_roles": ["builder", "gatekeeper", "owner", "designer"] },
        { "key": "configure_extraction", "label": "Edit extraction anchors/config", "default_roles": ["designer"] }
      ]
    }
  ]
}
```

---

## Related Specs

- **[Auth Overview](./overview.md)** — JWT strategy, middleware, rate limiting
- **[Auth Migrations](./migrations.md)** — Database schema for all auth tables
- **[Roles & Permissions](../Roles/overview.md)** — Permission model, Discord parallels
