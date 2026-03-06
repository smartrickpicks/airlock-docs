# Auth & Identity -- Security Spec

> **Status:** SPECCED
> **Source:** Security overview.md (Q16, Q17, A9, A13), /Features/Auth/overview.md, /Features/Auth/migrations.md, /Features/Auth/api-endpoints.md
> **Resolves:** A9 (passkey timeline), A13 (JWT secret management)

## Summary

This document consolidates all authentication and identity security decisions into the Security context. It formalizes JWT lifecycle management, OAuth flow hardening, API key protection, session management, WebSocket authentication, and the passkey/WebAuthn roadmap. Every decision here is engineer-ready -- an implementer should be able to build the auth layer without asking questions.

---

## 1. Authentication Architecture Overview

### Design Decisions (Locked)

| Decision | Value | Rationale |
|----------|-------|-----------|
| MVP auth method | Google OAuth 2.0 (sole method) | Eliminates password storage, phishing risk, and credential management |
| Access token | JWT, 15 min TTL, in-memory only | Short-lived limits exposure window; in-memory prevents XSS exfiltration from localStorage |
| Refresh token | JWT, 7 day TTL, httpOnly cookie | Cookie-based prevents JavaScript access; rotation detects theft |
| Refresh rotation | Single-use with family-based replay detection | If a stolen token is used, the entire family is revoked |
| Dev mode | `AIRLOCK_DEV_MODE` env var with seed users | Explicit opt-in, local only, never auto-detected |

### Auth Flow (End-to-End)

```
User clicks "Sign in with Google"
    |
    v
Google OAuth 2.0 Authorization Code Flow (with PKCE)
    |
    v
Google returns authorization code to callback URL
    |
    v
POST /api/auth/google/verify (server-side token exchange)
    |
    v
Server verifies Google ID token via google.oauth2.id_token library
    |
    v
Look up user by email in users table
    |
    v
Check: user.status == 'active' (or 'invited' for first login)
    |
    v
Look up workspace role(s) in user_workspace_roles
    |
    v
Issue Airlock JWT access_token (15 min) + refresh_token (7 day)
    |
    v
Return { access_token, user, workspace } + Set-Cookie: refresh_token
    |
    v
Frontend stores access_token in JavaScript variable (NOT localStorage)
    |
    v
All subsequent requests: Authorization: Bearer <access_token>
```

---

## 2. JWT Lifecycle (Answers Q16, Resolves A13)

### Token Structure

**Access Token:**

| Component | Content |
|-----------|---------|
| Header | `{ "alg": "HS256", "typ": "JWT" }` (MVP) / `{ "alg": "RS256" }` (enterprise) |
| Payload | `sub` (user_id), `workspace_id`, `org_role`, `email`, `name`, `exp`, `iat`, `jti` |
| Signature | HMAC-SHA256 of header + payload using `JWT_SECRET` |

**Refresh Token:**

| Component | Content |
|-----------|---------|
| Header | `{ "alg": "HS256", "typ": "JWT" }` |
| Payload | `sub` (user_id), `workspace_id`, `token_family` (UUID), `jti` (unique token ID), `exp`, `iat` |
| Signature | HMAC-SHA256 using same `JWT_SECRET` (different prefix validation prevents type confusion) |

### Signing Key Management (Resolves A13)

**MVP:**

| Property | Value |
|----------|-------|
| Algorithm | HS256 (HMAC-SHA256) |
| Key source | `JWT_SECRET` environment variable |
| Minimum length | 256-bit (32 bytes) -- enforced at startup |
| Generation | `python -c "import secrets; print(secrets.token_urlsafe(32))"` |
| Startup validation | FastAPI `lifespan` event checks key length; refuses to start if < 32 bytes |

**Key Rotation (MVP-compatible):**

Dual-key verification enables zero-downtime rotation:

```
1. Admin generates new JWT_SECRET_NEW
2. Set both: JWT_SECRET (current) and JWT_SECRET_PREVIOUS (old)
3. New tokens signed with JWT_SECRET
4. Token verification tries JWT_SECRET first, falls back to JWT_SECRET_PREVIOUS
5. Grace period: 30 minutes (2x access token TTL)
6. After grace period: remove JWT_SECRET_PREVIOUS
7. All tokens signed with old key have expired naturally
```

**Implementation:**

```python
# JWT verification with dual-key support
def verify_jwt(token: str, token_type: str = "access") -> dict | None:
    """Verify JWT with dual-key fallback for rotation."""
    secrets = [JWT_SECRET]
    if JWT_SECRET_PREVIOUS:
        secrets.append(JWT_SECRET_PREVIOUS)

    for secret in secrets:
        try:
            payload = jwt.decode(token, secret, algorithms=["HS256"])
            # Validate token_type prefix to prevent type confusion
            if token_type == "refresh" and not payload.get("token_family"):
                continue  # Not a refresh token
            if token_type == "access" and payload.get("token_family"):
                continue  # Not an access token
            return payload
        except jwt.ExpiredSignatureError:
            return None  # Expired -- do not try other keys
        except jwt.InvalidTokenError:
            continue  # Try next key

    return None  # No key could verify
```

**Enterprise Path:**

| Property | Value |
|----------|-------|
| Algorithm | RS256 (RSA-SHA256) |
| Private key | Stored in KMS (AWS KMS, HashiCorp Vault, or GCP Cloud KMS) |
| Public key | Distributed to all services via JWKS endpoint (`/.well-known/jwks.json`) |
| Rotation | Generate new RSA key pair in KMS, publish new public key to JWKS, old key remains for grace period |
| Key ID (`kid`) | Included in JWT header to select the correct verification key |

### Token Revocation

**Access tokens** are stateless and cannot be individually revoked. Their short 15-minute TTL limits the exposure window. For immediate revocation scenarios (e.g., user fired), revoke all refresh token families -- the access token expires naturally within 15 minutes.

**Refresh tokens** are stored server-side in the `refresh_tokens` table. Revocation modes:

| Mode | Trigger | Effect |
|------|---------|--------|
| Single-family revocation | `POST /api/auth/logout` | Revokes one token family (one device) |
| All-family revocation | `POST /api/auth/logout-all` | Revokes all families for a user (all devices) |
| Replay-triggered revocation | Refresh token reuse detected | Revokes entire family + emits `auth.refresh_replay` alert |
| Admin-initiated revocation | Admin revokes user session | Revokes specified family with `revoked_reason: 'admin'` |

### Refresh Token Family Rotation

Each login creates a new `token_family` (UUID). Every refresh creates a new token in the same family:

```
Login -> token_family: fam_abc123
    |
    v
refresh_token_1 (jti: rt_001) -- issued at login
    |
    v (used to refresh)
refresh_token_2 (jti: rt_002) -- rt_001 marked as used_at
    |
    v (used to refresh)
refresh_token_3 (jti: rt_003) -- rt_002 marked as used_at
```

**Replay detection:** If `rt_001` is used again after it was already consumed:

```
rt_001 used again -> used_at IS NOT NULL -> REPLAY DETECTED
    |
    v
Revoke entire family fam_abc123 (all tokens: rt_001, rt_002, rt_003)
    |
    v
Emit audit event: auth.refresh_replay { user_id, family_id, ip_address }
    |
    v
User must re-authenticate via Google OAuth on all devices
```

---

## 3. OAuth Flow Security

### PKCE (Proof Key for Code Exchange)

PKCE prevents authorization code interception attacks:

```
1. Frontend generates code_verifier (43-128 char random string)
2. Frontend computes code_challenge = BASE64URL(SHA256(code_verifier))
3. Authorization request includes code_challenge + code_challenge_method=S256
4. Google returns authorization code to callback URL
5. Token exchange includes code_verifier (server-side)
6. Google verifies: SHA256(code_verifier) == code_challenge
```

### State Parameter (CSRF Protection)

```
1. Frontend generates random state value (256-bit)
2. State stored in sessionStorage (cleared on tab close)
3. State included in authorization request
4. Google returns state in callback
5. Frontend verifies: callback_state == stored_state
6. Mismatch -> reject (CSRF attempt)
```

### Nonce (ID Token Replay Prevention)

```
1. Frontend generates random nonce (256-bit)
2. Nonce included in authorization request
3. Google embeds nonce in ID token payload
4. Server verifies: id_token.nonce == expected_nonce
5. Mismatch -> reject (replay attempt)
```

### Server-Side Token Exchange

The authorization code is exchanged for tokens **server-side only**. The client never sees the Google access token or ID token directly from Google's token endpoint. This prevents token leakage through browser history, referrer headers, or client-side JavaScript vulnerabilities.

### Callback URL Restrictions

| Environment | Allowed Callback URLs |
|-------------|----------------------|
| Production | `https://app.airlock.io/auth/callback` (exact match) |
| Staging | `https://staging.airlock.io/auth/callback` |
| Local dev | `http://localhost:3000/auth/callback` |

Callback URLs are registered in Google Cloud Console and validated server-side. No wildcard or partial matching.

---

## 4. API Key Security (Answers Q17)

### Key Generation

| Property | Value |
|----------|-------|
| Algorithm | `secrets.token_urlsafe(32)` -- 256-bit random |
| Format | `airlock_sk_live_{base64url_random}` (live) or `airlock_sk_test_{base64url_random}` (dev) |
| Display | Shown once at creation, never retrievable after |
| Clipboard | Frontend copies to clipboard on creation, shows "copied" confirmation |

### Key Storage

| Property | Value |
|----------|-------|
| Hash algorithm | SHA-256 |
| Rationale | 256-bit random keys are not susceptible to brute force (2^256 search space). SHA-256 is sufficient. bcrypt/Argon2 add unnecessary latency for high-entropy keys |
| Stored as | `sha256(key_value)` in `api_keys.key_hash` column |
| Lookup | Hash the incoming key, query by hash. O(1) with index |

**Why not bcrypt?** bcrypt is designed to slow down brute-force attacks on low-entropy secrets (passwords). API keys generated with `secrets.token_urlsafe(32)` have 256 bits of entropy -- brute-forcing SHA-256 of a 256-bit key is computationally infeasible (equivalent to breaking SHA-256 itself). bcrypt would add ~100ms per validation, which is unacceptable for API key authentication on every request.

### Key Scopes

Each API key has a set of scopes that map to permission sets:

| Scope | Permissions Granted |
|-------|-------------------|
| `read` | `view_vaults`, `view_extraction`, `view_triage`, `view_tasks`, `view_audit_log` |
| `write` | `create_vaults`, `edit_vault_metadata`, `create_patches`, `submit_patches`, `create_triage`, `manage_tasks` |
| `admin` | `manage_members`, `manage_roles`, `toggle_features`, `calibrate_thresholds`, `system_health` |
| `export` | `export_csv`, `export_pdf`, `export_json`, `export_docx` |

Scopes are per-workspace. An API key for workspace `ws_abc` cannot access workspace `ws_def`.

### Key Lifecycle

| Action | Process |
|--------|---------|
| **Creation** | Admin creates via `POST /api/workspaces/{ws_id}/api-keys`. Key value returned once. Hash stored. |
| **Validation** | On each request: hash incoming key, lookup by hash, check scopes against required permission |
| **Rotation** | Create new key -> migrate integrations -> verify new key works -> delete old key. No atomic rotation (two keys can coexist) |
| **Revocation** | `DELETE /api/workspaces/{ws_id}/api-keys/{key_id}`. Immediate effect. All subsequent requests with this key return 401 |
| **Expiration** | Optional `expires_at` field. Expired keys rejected at validation time |

### Rate Limiting

| Scope | Default Limit | Configurable |
|-------|--------------|-------------|
| Per API key | 100 requests/minute | Yes, via admin settings |
| Per workspace (all keys) | 1000 requests/minute | Yes |
| Burst allowance | 20 requests in 1 second | No |

Rate limit headers returned on every response:
- `X-RateLimit-Limit: 100`
- `X-RateLimit-Remaining: 87`
- `X-RateLimit-Reset: 1709571600` (Unix timestamp)

---

## 5. Session Management

### Stateless Access Tokens

Access tokens are stateless -- no server-side session store is required. The JWT payload contains all information needed for authorization:

```python
# Access token payload (decoded)
{
    "sub": "usr_abc123",           # user_id
    "email": "ana@company.com",
    "name": "Ana Chen",
    "workspace_id": "ws_abc123",
    "org_role": "member",
    "iat": 1709571600,             # issued at
    "exp": 1709572500,             # expires at (15 min later)
    "jti": "at_unique_id"          # unique token ID
}
```

### Server-Side Refresh Token Store

Refresh tokens are stored in the `refresh_tokens` table (see `/Features/Auth/migrations.md` for schema). This enables:

- **Per-device session tracking:** Each device has its own token family
- **Active session listing:** User can see all active sessions via `GET /api/auth/sessions`
- **Selective revocation:** Revoke a single device without affecting others
- **Replay detection:** Server-side state required to detect token reuse

### Cookie Security

| Attribute | Value | Rationale |
|-----------|-------|-----------|
| `HttpOnly` | `true` | Prevents JavaScript access (XSS defense) |
| `Secure` | `true` | HTTPS only (no plaintext transmission) |
| `SameSite` | `Strict` | Prevents cross-origin cookie sending (CSRF defense) |
| `Path` | `/api/auth` | Cookie only sent to auth endpoints (minimizes exposure) |
| `Max-Age` | `604800` (7 days) | Matches refresh token TTL |
| `Domain` | Not set (defaults to exact origin) | Prevents subdomain cookie theft |

### Multi-Device Sessions

Each device login creates an independent token family:

```
Ana's laptop  -> family: fam_001 -> refresh chain: rt_001 -> rt_002 -> rt_003
Ana's phone   -> family: fam_002 -> refresh chain: rt_004 -> rt_005
Ana's tablet  -> family: fam_003 -> refresh chain: rt_006
```

Session listing (`GET /api/auth/sessions`) returns:

| Family | Device | IP | Last Used | Current? |
|--------|--------|-----|-----------|----------|
| fam_001 | Chrome on macOS | 192.168.1.100 | 2 min ago | Yes |
| fam_002 | Safari on iPhone | 10.0.0.50 | 1 day ago | No |
| fam_003 | Chrome on iPad | 10.0.0.51 | 3 days ago | No |

Individual session revocation: `DELETE /api/auth/sessions/{family_id}` revokes that family.

---

## 6. WebSocket Authentication

### Connection Flow

```
1. Client obtains valid access_token via REST auth
2. Client opens WebSocket: wss://api.airlock.io/ws?token={access_token}
3. Server extracts token from query parameter
4. Server verifies JWT (same verify_jwt function as REST)
5. If valid: complete WebSocket upgrade, subscribe to workspace channels
6. If invalid: reject upgrade with 401 status
```

### Why Query Parameter (Not Header)?

The WebSocket API (`new WebSocket(url)`) does not support custom headers. The `Authorization: Bearer` pattern cannot be used. The token is passed as a query parameter. This is standard practice for WebSocket authentication.

**Mitigations for query parameter exposure:**

| Risk | Mitigation |
|------|-----------|
| Token in server logs | Configure reverse proxy to strip `?token=` from access logs |
| Token in browser history | WebSocket URLs are not stored in browser history |
| Token in referrer headers | WebSocket connections do not send referrer headers |
| Token lifetime | 15-minute TTL limits exposure window |

### Re-Authentication on Expiry

```
1. Access token expires (15 min)
2. Server detects expired token on next message or heartbeat
3. Server sends: { "type": "auth_expired", "message": "Token expired" }
4. Client refreshes access token via POST /api/auth/refresh
5. Client closes current WebSocket
6. Client opens new WebSocket with fresh token
7. Server restores subscriptions from previous connection (keyed by user_id + workspace_id)
```

### Heartbeat Protocol

| Property | Value |
|----------|-------|
| Interval | 30 seconds |
| Direction | Server sends ping, client responds with pong |
| Missed threshold | 3 consecutive missed pongs |
| Action on threshold | Server closes connection, cleans up subscriptions |
| Reconnection | Client implements exponential backoff: 1s, 2s, 4s, 8s, max 30s |

---

## 7. Auth Audit Events

Every authentication-related action emits an immutable audit event to the `audit_events` table. These events use the append-only enforcement defined in `audit-and-immutability.md`.

### Event Catalog

| Event Type | Trigger | Payload Fields |
|-----------|---------|---------------|
| `auth.login` | Successful Google OAuth or dev login | `user_id`, `email`, `workspace_id`, `ip_address`, `user_agent`, `method` (oauth/api_key/dev) |
| `auth.login_failed` | Invalid token, inactive user, unknown email | `email`, `ip_address`, `user_agent`, `reason` |
| `auth.login_blocked` | Rate limit exceeded on login endpoint | `ip_address`, `attempt_count`, `window_seconds` |
| `auth.refresh` | Successful token refresh | `user_id`, `token_family`, `ip_address` |
| `auth.refresh_failed` | Invalid/expired refresh token | `user_id`, `token_family`, `reason` |
| `auth.refresh_replay` | Refresh token reuse detected (theft indicator) | `user_id`, `token_family`, `ip_address`, `original_ip` |
| `auth.logout` | Single-device logout | `user_id`, `token_family` |
| `auth.logout_all` | All-device logout | `user_id`, `families_revoked` (count) |
| `auth.session.revoked` | Admin or user revokes a specific session | `user_id`, `target_family`, `revoked_by` |
| `auth.role.changed` | Org or module role assignment changed | `target_user_id`, `changed_by`, `old_role`, `new_role`, `scope` (org/module/channel) |
| `auth.permission.override` | Ad-hoc permission grant or deny created | `target_user_id`, `granted_by`, `permission`, `effect` (grant/deny), `scope_type`, `scope_id`, `expires_at` |
| `auth.api_key.created` | New API key generated | `user_id`, `key_id`, `scopes`, `name`, `expires_at` |
| `auth.api_key.rotated` | API key replaced (old revoked, new created) | `user_id`, `old_key_id`, `new_key_id` |
| `auth.api_key.revoked` | API key revoked | `user_id`, `key_id`, `reason` |
| `auth.mfa.enrolled` | WebAuthn credential registered (enterprise) | `user_id`, `credential_id`, `authenticator_type` |
| `auth.mfa.verified` | WebAuthn assertion verified (enterprise) | `user_id`, `credential_id` |

### Security Alerting Thresholds

These events trigger immediate alerts to workspace admins:

| Condition | Alert Level | Action |
|-----------|------------|--------|
| `auth.refresh_replay` | Critical | Email admin + in-app notification |
| 5+ `auth.login_failed` from same IP in 5 min | High | Log + rate limit escalation |
| `auth.role.changed` to `executive` | Medium | Notification to all existing executives |
| `auth.api_key.created` with `admin` scope | Medium | Notification to workspace owner |
| `auth.login` from new IP/device for a user | Low | Logged only (no notification by default) |

---

## 8. Passkey/WebAuthn Roadmap (Resolves A9)

### Decision

**Passkeys are NOT required for MVP.** Google OAuth is the sole authentication method for initial release. Passkey support is planned for the enterprise phase in three stages.

### Roadmap

| Phase | Timeline | Scope | Details |
|-------|----------|-------|---------|
| **MVP** | Now | Google OAuth only | No passkey support. No TOTP. Single auth method. |
| **Enterprise Phase 1** | After MVP launch | WebAuthn as optional second factor | Users can register a passkey (platform or roaming authenticator) as 2FA alongside Google OAuth. TOTP (Google Authenticator, Authy) offered as alternative 2FA. Admin can require 2FA for elevated roles (Owner, Gatekeeper). |
| **Enterprise Phase 2** | After Phase 1 adoption | WebAuthn as primary auth (passwordless) | Users can log in with passkey alone (no Google OAuth required). Passkey + OAuth both valid primary auth methods. Enterprise SSO (SAML 2.0, OIDC) added as third option. |
| **Enterprise Phase 3** | Long-term | Passkey + DID integration | Passkey credential linked to DID document. Cross-workspace portable identity. Verifiable Credentials for role attestations. |

### Phase 1 Technical Requirements (Pre-Planning)

| Component | Technology | Notes |
|-----------|-----------|-------|
| WebAuthn library | `py_webauthn` (Python server) | Handles registration + authentication ceremonies |
| Credential storage | New `webauthn_credentials` table | `credential_id`, `public_key`, `sign_count`, `transports`, `user_id` |
| Authenticator types | Platform (TouchID, FaceID, Windows Hello) + Roaming (YubiKey, security keys) | Both supported |
| Recovery | Recovery codes (8 single-use codes, generated at enrollment) | Stored as bcrypt hashes |
| Admin controls | `auth.mfa.required_for_roles` calibration parameter | Array of role names requiring MFA |

### Phase 1 Database Schema (Preview)

```sql
-- WebAuthn credential storage (Enterprise Phase 1)
CREATE TABLE webauthn_credentials (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    credential_id BYTEA NOT NULL UNIQUE,        -- WebAuthn credential ID
    public_key BYTEA NOT NULL,                   -- COSE public key
    sign_count INT NOT NULL DEFAULT 0,           -- Monotonic counter for clone detection
    transports TEXT[],                            -- ['usb', 'nfc', 'ble', 'internal']
    authenticator_type TEXT NOT NULL,             -- 'platform' | 'cross-platform'
    name TEXT,                                    -- User-friendly name ("My YubiKey")
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_used_at TIMESTAMPTZ
);

CREATE INDEX idx_webauthn_user ON webauthn_credentials(user_id);

-- MFA recovery codes
CREATE TABLE mfa_recovery_codes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    code_hash TEXT NOT NULL,                     -- bcrypt hash of recovery code
    used_at TIMESTAMPTZ,                         -- NULL = unused
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Related Specs

- [overview.md](./overview.md) -- Security architecture anchor document, 70-question inventory
- [rbac-enforcement.md](./rbac-enforcement.md) -- Permission computation, tenant isolation, self-approval prevention
- [audit-and-immutability.md](./audit-and-immutability.md) -- Append-only audit event enforcement
- [data-cipher-model.md](./data-cipher-model.md) -- Encryption at rest (JWT_SECRET storage implications)
- [Auth Overview](../Auth/overview.md) -- Full auth pipeline, refresh tokens, permission middleware
- [Auth Migrations](../Auth/migrations.md) -- SQL schema for refresh tokens, module roles
- [Auth API Endpoints](../Auth/api-endpoints.md) -- Complete auth API surface
- [Roles Overview](../Roles/overview.md) -- Role model, custom roles, permission catalog
