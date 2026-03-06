# Authentication & Authorization — Design Spec

> **Status:** SPECCED — Complete auth pipeline from login to per-endpoint permission checks.
>
> **Key Insight:** Auth in Airlock has two distinct concerns: (1) **Authentication** — who is this person? (JWT + Google OAuth) and (2) **Authorization** — what can they do? (Discord-style computed permissions from Roles spec). This document covers the full pipeline, the gaps that exist in OrcestrateOS, and the implementation decisions for Airlock.
>
> **Source code:** OrcestrateOS `server/auth.py` (323 lines), `server/jwt_utils.py` (83 lines), `server/routes/auth_google.py` (203 lines)

---

## Authentication Pipeline

### Overview

```
User clicks "Sign in with Google"
    ↓
Google OAuth 2.0 → Google ID token (credential)
    ↓
POST /api/auth/google/verify  ← Airlock API
    ↓
Verify token via google.oauth2.id_token  ← Google's library
    ↓
Look up user by email in users table
    ↓
Check user status (active/inactive)
    ↓
Look up workspace role in user_workspace_roles
    ↓
Issue Airlock JWT (access_token + refresh_token)  ← NEW: refresh token
    ↓
Return { access_token, refresh_token, user, workspace }
    ↓
Frontend stores access_token in memory, refresh_token in httpOnly cookie
    ↓
All subsequent requests: Authorization: Bearer <access_token>
```

### What Ports from OrcestrateOS (As-Is)

| Component | File | Lines | Status |
|-----------|------|-------|--------|
| JWT signing/verification | `jwt_utils.py` | 83 | Copy as-is |
| Bearer token resolution | `auth.py` → `_resolve_bearer()` | 50 | Copy, remove fallback-by-email |
| API key resolution | `auth.py` → `_resolve_api_key()` | 40 | Copy as-is |
| `AuthResult` class | `auth.py` | 20 | Copy, extend with module_roles |
| `require_auth()` dependency | `auth.py` | 30 | Copy as-is |
| `require_role()` check | `auth.py` | 20 | Replace with `require_permission()` |
| Google OAuth verify | `auth_google.py` | 170 | Copy, add refresh token issuance |
| `/auth/me` endpoint | `auth_google.py` | 20 | Copy, extend response with module roles |

### What's New in Airlock

| Component | Why | Spec Section |
|-----------|-----|-------------|
| Refresh token flow | OrcestrateOS has no refresh; 24hr hard cutoff | [Refresh Tokens](#refresh-tokens) |
| `require_permission()` middleware | OrcestrateOS only checks role hierarchy; Airlock needs granular permissions | [Permission Middleware](#permission-check-middleware) |
| Auth event audit logging | Login attempts, token refresh, role changes not logged | [Auth Audit Events](#auth-audit-events) |
| Rate limiting on auth endpoints | No brute-force protection | [Rate Limiting](#rate-limiting) |
| API key scope enforcement | Scopes stored but never validated | [API Key Scopes](#api-key-scope-enforcement) |
| Dev mode auth bypass | Replace REPLIT_DEV_DOMAIN sandbox logic | [Dev Mode](#dev-mode-auth-bypass) |

### What's Removed from OrcestrateOS

| Component | Why |
|-----------|-----|
| `_SANDBOX_AUTH_ALLOWED` (Replit check) | Airlock has its own dev mode |
| `_is_local_request()` localhost bypass | Security risk; use explicit dev mode instead |
| Fallback bearer resolution by email/UUID | Security risk; only accept valid JWTs |
| `_apply_role_simulation()` | Replace with proper dev tools in Admin |
| `CONTRACT_AUTHOR` role | Replaced by module-level Builder role |

---

## JWT Token Strategy

### Access Token

| Property | Value |
|----------|-------|
| Algorithm | HS256 |
| Expiry | **15 minutes** (down from OrcestrateOS's 24 hours) |
| Storage | In-memory only (JavaScript variable, NOT localStorage) |
| Payload | `sub`, `email`, `name`, `workspace_id`, `org_role`, `iat`, `exp` |

**Why 15 minutes:** Shorter-lived access tokens reduce the window of exposure if a token is stolen. Combined with refresh tokens, the UX is seamless.

### Refresh Token

| Property | Value |
|----------|-------|
| Algorithm | HS256 (same secret, different prefix) |
| Expiry | **7 days** |
| Storage | `httpOnly`, `secure`, `sameSite=strict` cookie |
| Payload | `sub`, `workspace_id`, `token_family`, `iat`, `exp` |
| Rotation | Single-use — each refresh issues a new refresh token |
| Revocation | Stored in `refresh_tokens` DB table; revocable per-user or per-family |

### Token Family (Refresh Token Rotation)

Each refresh token belongs to a `token_family` (UUID). When a refresh token is used:

1. Look up the family in `refresh_tokens` table
2. If the token is valid and not used: issue new access + refresh token, mark old as used
3. If the token was already used (replay attack): **revoke entire family** — force re-login
4. New refresh token gets same `token_family` but new `jti` (token ID)

This prevents refresh token theft while allowing seamless rotation.

```python
# Refresh token rotation logic
async def refresh_access_token(refresh_token: str):
    payload = verify_jwt(refresh_token, token_type="refresh")
    if not payload:
        raise HTTPException(401, "Invalid refresh token")

    family_id = payload["token_family"]
    token_id = payload["jti"]

    # Check if this specific token was already used
    token_record = await get_refresh_token(token_id)
    if token_record is None:
        raise HTTPException(401, "Unknown refresh token")

    if token_record.used_at is not None:
        # REPLAY DETECTED — revoke entire family
        await revoke_token_family(family_id)
        await emit_auth_event("refresh_token.replay_detected", {
            "user_id": payload["sub"],
            "family_id": family_id,
        })
        raise HTTPException(401, "Token reuse detected — logged out for security")

    # Mark this token as used
    await mark_token_used(token_id)

    # Issue new pair
    new_access = sign_jwt({
        "sub": payload["sub"],
        "email": user.email,
        "name": user.display_name,
        "workspace_id": payload["workspace_id"],
        "org_role": user.org_role,
    }, expiry=900)  # 15 minutes

    new_refresh = sign_jwt({
        "sub": payload["sub"],
        "workspace_id": payload["workspace_id"],
        "token_family": family_id,
        "jti": generate_token_id(),
    }, expiry=604800, token_type="refresh")  # 7 days

    await store_refresh_token(new_refresh_jti, family_id, payload["sub"])

    return new_access, new_refresh
```

### Token Lifecycle

```
Login (Google OAuth)
    ↓
Issue access_token (15min) + refresh_token (7d)
    ↓
Frontend uses access_token for API calls
    ↓
access_token expires (15min)
    ↓
Frontend calls POST /api/auth/refresh with refresh_token cookie
    ↓
Server: validate → rotate → issue new pair
    ↓
Frontend gets new access_token, continues seamlessly
    ↓
After 7 days of inactivity (no refresh): force re-login
```

---

## Refresh Tokens

### Database Table

```sql
CREATE TABLE refresh_tokens (
    id TEXT PRIMARY KEY,                     -- jti (unique token ID)
    family_id UUID NOT NULL,                 -- token family for rotation detection
    user_id UUID NOT NULL REFERENCES users(id),
    workspace_id UUID NOT NULL REFERENCES workspaces(id),
    issued_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at TIMESTAMPTZ NOT NULL,
    used_at TIMESTAMPTZ,                     -- NULL = unused, set when rotated
    revoked_at TIMESTAMPTZ,                  -- NULL = active, set when revoked
    revoked_reason TEXT,                     -- 'rotation', 'logout', 'admin', 'replay_detected'
    user_agent TEXT,                         -- browser/client info for audit
    ip_address INET                          -- client IP for audit
);

CREATE INDEX idx_refresh_user ON refresh_tokens(user_id) WHERE revoked_at IS NULL;
CREATE INDEX idx_refresh_family ON refresh_tokens(family_id);
```

### API Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/auth/refresh` | Exchange refresh token for new access + refresh pair |
| `POST` | `/api/auth/logout` | Revoke current refresh token family |
| `POST` | `/api/auth/logout-all` | Revoke ALL refresh tokens for user (all devices) |

### Frontend Integration

```typescript
// lib/auth.ts — Token management

let accessToken: string | null = null;

export function setAccessToken(token: string) {
  accessToken = token;
}

export function getAccessToken(): string | null {
  return accessToken;
}

// Automatic refresh on 401
export async function authFetch(path: string, options?: RequestInit): Promise<Response> {
  let res = await fetch(`${API_URL}${path}`, {
    ...options,
    headers: {
      ...options?.headers,
      Authorization: `Bearer ${accessToken}`,
    },
  });

  if (res.status === 401 && accessToken) {
    // Try refresh
    const refreshRes = await fetch(`${API_URL}/api/auth/refresh`, {
      method: 'POST',
      credentials: 'include',  // sends httpOnly cookie
    });

    if (refreshRes.ok) {
      const { access_token } = await refreshRes.json();
      accessToken = access_token;

      // Retry original request
      res = await fetch(`${API_URL}${path}`, {
        ...options,
        headers: {
          ...options?.headers,
          Authorization: `Bearer ${accessToken}`,
        },
      });
    } else {
      // Refresh failed — force re-login
      accessToken = null;
      window.location.href = '/login';
    }
  }

  return res;
}
```

---

## Permission Check Middleware

### The Problem

OrcestrateOS uses `require_role(workspace_id, auth_result, min_role)` — a flat hierarchy check (analyst < verifier < admin < architect). This doesn't support:

- Different permissions per module
- Ad-hoc overrides (Builder + approve_low_risk)
- Channel-level access control
- Custom roles with arbitrary permission sets

### The Solution: `require_permission()`

```python
from functools import wraps
from fastapi import Request, HTTPException

def require_permission(permission: str, module_id: str = None, channel_id: str = None):
    """FastAPI dependency that checks computed permissions.

    Usage:
        @router.post("/vaults/{vault_id}/patches")
        async def create_patch(
            vault_id: str,
            auth: AuthResult = Depends(require_auth(AuthClass.BEARER)),
            _perm = Depends(require_permission("create_patches", module_id="contracts")),
        ):
            ...
    """
    def dependency(request: Request):
        auth_result = getattr(request.state, 'auth', None)
        if auth_result is None:
            raise HTTPException(401, "Authentication required")

        workspace_id = auth_result.workspace_id

        # Resolve module_id from request if not hardcoded
        resolved_module = module_id or request.path_params.get("module_id")
        resolved_channel = channel_id or request.path_params.get("channel_id")

        if not check_permission(
            auth_result.user_id,
            workspace_id,
            resolved_module,
            permission,
            resolved_channel,
        ):
            raise HTTPException(
                403,
                f"Missing permission: {permission} in {resolved_module or 'workspace'}"
            )

        return True

    return dependency
```

### Permission Resolution Flow

```
require_permission("approve_low_risk", module_id="contracts", channel_id="ch_abc123")
    ↓
1. get_user_roles(user_id, workspace_id)
   → Returns: [Builder, Senior Analyst (custom)]
    ↓
2. Union all permissions from all roles
   → Builder: {view_vaults, create_vaults, create_patches, ...}
   → Senior Analyst: {approve_low_risk, ...}
   → Combined: {view_vaults, create_vaults, create_patches, approve_low_risk, ...}
    ↓
3. get_active_overrides(user_id, workspace_id, 'module', 'contracts')
   → Grant: export_json (ad-hoc override from admin)
   → Combined: {... + export_json}
    ↓
4. get_active_overrides(user_id, workspace_id, 'channel', 'ch_abc123')
   → Deny: approve_low_risk (conflict of interest on this specific contract)
   → Combined: {... - approve_low_risk}
    ↓
5. Enforce org_role_ceiling(org_role)
   → User is Member → ceiling removes admin-only permissions
    ↓
6. Check: "approve_low_risk" in final_permissions?
   → FALSE (denied at channel level) → 403 Forbidden
```

### Caching Strategy

Permission computation is expensive (multiple DB queries). Cache aggressively:

| Cache | TTL | Invalidation |
|-------|-----|-------------|
| User roles | 5 minutes | On role assignment change |
| Module overrides | 5 minutes | On override grant/revoke |
| Channel overrides | 2 minutes | On override grant/revoke |
| Org role | 10 minutes | On org role change |

Cache key: `perm:{workspace_id}:{user_id}:{module_id}:{channel_id}`

Use Redis for shared cache across API instances. Invalidate on write via event bus.

---

## Rate Limiting

### Auth Endpoint Limits

| Endpoint | Limit | Window | Action on Exceed |
|----------|-------|--------|-----------------|
| `POST /api/auth/google/verify` | 10 requests | per minute per IP | 429 + 60s cooldown |
| `POST /api/auth/refresh` | 30 requests | per minute per user | 429 + 30s cooldown |
| `POST /api/auth/logout` | 5 requests | per minute per user | 429 |
| `GET /api/auth/me` | 60 requests | per minute per user | 429 |

### Implementation

```python
# Rate limiter using Redis sliding window
from fastapi import Request, HTTPException
import time

async def rate_limit(key: str, max_requests: int, window_seconds: int):
    """Sliding window rate limiter using Redis sorted sets."""
    now = time.time()
    window_start = now - window_seconds

    pipe = redis.pipeline()
    pipe.zremrangebyscore(key, 0, window_start)  # Remove expired entries
    pipe.zadd(key, {str(now): now})               # Add current request
    pipe.zcard(key)                                # Count requests in window
    pipe.expire(key, window_seconds)               # Auto-cleanup key
    _, _, count, _ = await pipe.execute()

    if count > max_requests:
        raise HTTPException(
            429,
            detail=f"Rate limit exceeded. Try again in {window_seconds}s.",
            headers={"Retry-After": str(window_seconds)},
        )
```

### IP-Based vs User-Based

- **Pre-auth endpoints** (`/google/verify`): rate limit by IP address
- **Post-auth endpoints** (`/refresh`, `/me`): rate limit by user_id (from token)
- **Both**: apply IP limit as a backstop even on authenticated endpoints

---

## API Key Scope Enforcement

### The Problem

OrcestrateOS stores `scopes` in the `api_keys` table but never validates them. A key with `scopes: ["read"]` can still call write endpoints.

### The Solution

Define scope → permission mapping:

```python
# Scope definitions
API_KEY_SCOPES = {
    "read": {
        "view_vaults", "view_extraction", "view_triage",
        "view_tasks", "view_audit_log",
    },
    "write": {
        "create_vaults", "edit_vault_metadata", "create_patches",
        "submit_patches", "create_triage", "manage_tasks",
    },
    "admin": {
        "manage_members", "manage_roles", "toggle_features",
        "calibrate_thresholds", "system_health",
    },
    "export": {
        "export_csv", "export_pdf", "export_json", "export_docx",
    },
}

def resolve_api_key_permissions(scopes: list[str]) -> set[str]:
    """Expand API key scopes into permission set."""
    permissions = set()
    for scope in scopes:
        permissions |= API_KEY_SCOPES.get(scope, set())
    return permissions
```

### Enforcement Point

In `require_permission()`, when `auth_result.is_api_key`:

```python
if auth_result.is_api_key:
    api_permissions = resolve_api_key_permissions(auth_result.api_key_scopes)
    if permission not in api_permissions:
        raise HTTPException(403, f"API key scope insufficient for: {permission}")
```

---

## Auth Audit Events

### Event Types

Every auth-related action emits an audit event to the `audit_events` table:

| Event Type | Trigger | Payload |
|-----------|---------|---------|
| `auth.login.success` | Successful Google OAuth | `{user_id, email, workspace_id, ip, user_agent}` |
| `auth.login.failed` | Invalid token or inactive user | `{email, reason, ip, user_agent}` |
| `auth.login.blocked` | Rate limit exceeded | `{ip, attempt_count}` |
| `auth.refresh.success` | Token refreshed | `{user_id, token_family}` |
| `auth.refresh.failed` | Invalid/expired refresh token | `{user_id, reason}` |
| `auth.refresh.replay` | Refresh token reuse detected | `{user_id, family_id, ip}` |
| `auth.logout` | User logout (single device) | `{user_id, family_id}` |
| `auth.logout_all` | User logout all devices | `{user_id, families_revoked}` |
| `auth.role.changed` | Org or module role assignment | `{user_id, changed_by, old_role, new_role, scope}` |
| `auth.override.granted` | Ad-hoc permission granted | `{user_id, granted_by, permission, scope, expires_at}` |
| `auth.override.revoked` | Ad-hoc permission revoked | `{user_id, revoked_by, permission, scope}` |
| `auth.apikey.created` | API key created | `{key_id, created_by, scopes}` |
| `auth.apikey.revoked` | API key revoked | `{key_id, revoked_by}` |
| `auth.apikey.used` | API key used for request | `{key_id, endpoint, ip}` |

### Audit Event Schema

These use the existing `audit_events` table (append-only, trigger-protected):

```sql
INSERT INTO audit_events (event_type, workspace_id, actor_id, payload)
VALUES ('auth.login.success', :ws_id, :user_id, :payload::jsonb);
```

---

## Dev Mode Auth Bypass

### Replacing OrcestrateOS's Sandbox Logic

OrcestrateOS has three dev-mode paths that need cleanup:

1. `_SANDBOX_AUTH_ALLOWED` — checks for `REPLIT_DEV_DOMAIN` env var
2. `_is_local_request()` — trusts localhost connections
3. `_AUTH_DISABLED` — global auth kill switch

**Airlock replaces all three with a single, explicit dev mode:**

```python
# Environment variable: AIRLOCK_DEV_MODE=true
DEV_MODE = os.environ.get("AIRLOCK_DEV_MODE", "").strip().lower() in ("true", "1")

# Dev mode seed users (never in production)
DEV_USERS = {
    "dev_builder": {"email": "builder@dev.airlock", "org_role": "member", "display_name": "Dev Builder"},
    "dev_gatekeeper": {"email": "gatekeeper@dev.airlock", "org_role": "lead", "display_name": "Dev Gatekeeper"},
    "dev_owner": {"email": "owner@dev.airlock", "org_role": "director", "display_name": "Dev Owner"},
    "dev_architect": {"email": "architect@dev.airlock", "org_role": "executive", "display_name": "Dev Architect"},
}
```

### Dev Mode Behavior

| Feature | Behavior |
|---------|----------|
| Google OAuth | Bypassed — dev users available via `POST /api/auth/dev/login` |
| JWT tokens | Issued normally (same code path, just skips Google verification) |
| Refresh tokens | Work normally |
| Permissions | Work normally (dev users have real roles) |
| Rate limiting | Disabled |
| Audit logging | Still active (useful for debugging) |

### Dev Login Endpoint (Dev Mode Only)

```python
@router.post("/api/auth/dev/login")
async def dev_login(request: Request):
    """Dev-only: login as a seed user without Google OAuth."""
    if not DEV_MODE:
        raise HTTPException(404)  # Endpoint doesn't exist in production

    body = await request.json()
    user_key = body.get("user", "dev_builder")

    if user_key not in DEV_USERS:
        raise HTTPException(400, f"Unknown dev user: {user_key}")

    # Issue real JWT for the dev user
    dev_user = DEV_USERS[user_key]
    access_token = sign_jwt({...})
    refresh_token = sign_jwt({...}, token_type="refresh")

    return {"access_token": access_token, "user": dev_user}
```

### Safety Guards

- `AIRLOCK_DEV_MODE` must be explicitly set — no implicit detection
- Dev login endpoint returns 404 (not 403) in production — no information leak
- Docker Compose sets `AIRLOCK_DEV_MODE=true` only in `docker-compose.override.yml` (not base)
- CI/CD pipeline checks: `AIRLOCK_DEV_MODE` must NOT be set in production deployments

---

## CSRF Protection

### Strategy

Since Airlock uses JWT bearer tokens (not cookies) for API authentication, traditional CSRF attacks don't apply — the attacker can't forge the `Authorization` header from a cross-origin request.

**However**, the refresh token IS stored in a cookie. CSRF protection for the refresh endpoint:

| Protection | Applied To |
|-----------|-----------|
| `SameSite=Strict` on refresh cookie | Prevents cross-origin cookie sending |
| `httpOnly` on refresh cookie | Prevents JavaScript access |
| `Secure` flag on refresh cookie | HTTPS only |
| Origin header validation | Verify `Origin` or `Referer` matches allowed origins |

```python
ALLOWED_ORIGINS = {
    "http://localhost:3000",          # Dev
    "https://app.airlock.io",         # Production
}

async def validate_origin(request: Request):
    """CSRF protection for cookie-authenticated endpoints."""
    origin = request.headers.get("Origin") or request.headers.get("Referer", "").split("/")[2] if "Referer" in request.headers else None
    if origin and origin not in ALLOWED_ORIGINS:
        raise HTTPException(403, "Invalid origin")
```

---

## First-Login Flow (Integration with Onboarding)

### New User (Invited)

```
Admin invites user@company.com with role "Builder" in Contracts
    ↓
System creates users row (status=invited, email=user@company.com)
System creates user_workspace_roles row (role=member)
System creates user_module_roles row (module=contracts, role=builder)
    ↓
User receives invite email with link: https://app.airlock.io/invite/{token}
    ↓
User clicks link → redirected to Google OAuth
    ↓
POST /api/auth/google/verify
    ↓
Server: finds user by email → status=invited → update to active
    ↓
Issue tokens → redirect to onboarding wizard (Onboarding spec)
```

### Existing User (Returning)

```
User visits https://app.airlock.io
    ↓
Frontend checks: access_token in memory? No.
    ↓
Frontend calls POST /api/auth/refresh (sends cookie)
    ↓
If refresh succeeds: user is logged in (no Google prompt)
If refresh fails: redirect to /login → Google OAuth
```

### Workspace Selection (Multi-Workspace Users)

A user can belong to multiple workspaces. After Google OAuth:

```python
# If user has multiple workspaces, return the list
workspaces = get_user_workspaces(user_id)
if len(workspaces) == 1:
    # Auto-select single workspace
    workspace_id = workspaces[0].id
elif len(workspaces) > 1:
    # Return workspace list — frontend shows picker
    return {"requires_workspace_selection": True, "workspaces": workspaces}
```

Frontend shows a workspace picker before issuing the final JWT.

---

## Configuration

### Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `JWT_SECRET` | Yes | — | HS256 signing key (min 32 chars) |
| `GOOGLE_OAUTH_CLIENT_ID` | Yes (prod) | — | Google OAuth 2.0 client ID |
| `AIRLOCK_DEV_MODE` | No | `false` | Enable dev mode auth bypass |
| `ACCESS_TOKEN_EXPIRY` | No | `900` | Access token TTL in seconds (15min) |
| `REFRESH_TOKEN_EXPIRY` | No | `604800` | Refresh token TTL in seconds (7 days) |
| `RATE_LIMIT_ENABLED` | No | `true` | Enable/disable rate limiting |
| `ALLOWED_ORIGINS` | No | `http://localhost:3000` | Comma-separated allowed origins |

### Feature Control Plane Integration

| Calibration Key | Default | Range | Description |
|----------------|---------|-------|-------------|
| `auth.access_token_expiry` | 900 | 300–3600 | Access token TTL (seconds) |
| `auth.refresh_token_expiry` | 604800 | 86400–2592000 | Refresh token TTL (seconds) |
| `auth.rate_limit.login` | 10 | 1–100 | Login attempts per minute per IP |
| `auth.rate_limit.refresh` | 30 | 1–100 | Refresh attempts per minute per user |
| `auth.permission_cache_ttl` | 300 | 60–600 | Permission cache TTL (seconds) |

---

## Migration from OrcestrateOS Role Hierarchy

### OrcestrateOS Roles → Airlock Roles

| OrcestrateOS | Airlock Org Role | Airlock Default Module Role |
|-------------|-----------------|---------------------------|
| `analyst` | `member` | `builder` |
| `verifier` | `lead` | `gatekeeper` |
| `contract_author` | `member` | `builder` (with ad-hoc `generate_contracts` permission) |
| `admin` | `director` | `owner` |
| `architect` | `executive` | `designer` (or `owner` depending on module) |

### Backward Compatibility

During migration, the old `has_minimum_role()` function can be shimmed:

```python
# Temporary compatibility shim — remove after full migration
def has_minimum_role_compat(user_role: str, required_role: str) -> bool:
    """Maps old hierarchical role check to new permission check."""
    ROLE_TO_PERMISSIONS = {
        "analyst": {"view_vaults", "create_patches", "create_triage"},
        "verifier": {"approve_low_risk", "approve_medium_risk", "resolve_triage"},
        "admin": {"manage_members", "manage_roles", "toggle_features"},
        "architect": {"*"},  # all permissions
    }
    # ... expand and check
```

---

## Related Specs

- **[Roles & Permissions](../Roles/overview.md)** — Permission model, module roles, custom roles, permission catalog
- **[Security & Architecture](../Security/overview.md)** — Encryption, threat model, key management
- **[Onboarding](../Onboarding/overview.md)** — First-login flow, workspace setup wizard
- **[Admin / Settings](../Admin/overview.md)** — Members page, role management UI
- **[Feature Control Plane](../FeatureControlPlane/overview.md)** — Auth calibration parameters
- **[Migrations](./migrations.md)** — SQL schema for refresh tokens, auth events
- **[API Endpoints](./api-endpoints.md)** — Complete auth API surface
