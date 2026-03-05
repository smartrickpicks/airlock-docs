# Admin / Settings — Design Spec

> **Status:** SPECCED — Admin is a system overlay, not a module.

> **Key Insight:** Admin doesn't follow the chamber pattern (Discover > Build > Review > Ship). There's no lifecycle for settings. It's a **system overlay** accessed from the user avatar, split into two distinct surfaces: Workspace Admin (for admins) and Personal Settings (for everyone).

> **Terminology:** See [Naming & Hierarchy](../Shell/naming-and-hierarchy.md). Admin is an **overlay**, not a module.

---

## Access Model

| Overlay | Triggered By | Required Role | Route |
|---------|-------------|--------------|-------|
| **Workspace Admin** | Gear icon in module bar (visible only to Admin+) | Admin (level 2) or Architect (level 3) | `/admin/*` |
| **Personal Settings** | User avatar click (bottom of module bar) | Any authenticated user | `/settings/*` |

Both overlays render **on top of** the current module view. The module bar, sub-panel, and main content are dimmed/blurred behind the overlay. Pressing Escape or clicking outside closes the overlay and returns to the previous view.

---

## Source Mapping (OrcestrateOS)

| OrcestrateOS File | Size | Airlock Target |
|---|---|---|
| `admin-demo.html` | 66 KB | Admin overlay UI |
| `server/auth.py` | 322 lines | Authentication + RBAC middleware |
| `server/feature_flags.py` | 246 lines | Feature Control Plane (major expansion) |
| `server/routes/admin_reporting.py` | 420 lines | System Health section |
| `server/routes/audit_events.py` | 140 lines | Audit Log section |
| `server/routes/members.py` | 325 lines | Members section |

---

## Workspace Admin

### Layout

Full-screen overlay with its own navigation sidebar:

```
+----------------------------------------------------------+
| WORKSPACE ADMIN                              [X Close]    |
|----------------------------------------------------------|
| > Dashboard     |                                         |
| > Members       |   [Active section content]              |
| > Roles         |                                         |
| > Feature Flags |                                         |
| > Calibration   |                                         |
| > Modules       |                                         |
| > Connectors    |                                         |
| > Audit Log     |                                         |
| > System Health |                                         |
+----------------------------------------------------------+
```

### Sections

#### Dashboard (Landing)

System health at a glance. The admin's daily starting point.

| Widget | Content |
|--------|---------|
| **System Health Strip** | One card per managed feature: status dot (green/amber/red), last error, requests/min, avg latency |
| **Active Users** | Online count, list of currently active users with role badges |
| **Recent Admin Actions** | Last 10 audit events from admin actions (flag toggles, role changes, calibration updates) |
| **Module Status** | Enabled/disabled modules with vault counts |
| **Error Feed** | Real-time stream of failures across all features, severity-coded |

#### Members

User management for the workspace.

| Feature | Description |
|---------|-------------|
| **User list** | Table: avatar, name, email, role badge, last active, status |
| **Invite user** | Email invite with role pre-assignment |
| **Edit role** | Dropdown: Analyst → Verifier → Admin → Architect |
| **Remove user** | Soft-remove with confirmation + audit event |
| **Bulk actions** | Multi-select: change role, deactivate, export list |

Role changes emit audit events and take effect immediately.

#### Roles & Permissions

| Role | Level | Airlock Name | Key Permissions |
|------|-------|-------------|----------------|
| **Analyst** | 0 | **Builder** | View vaults, create triage items, create patches, chat with AI agent |
| **Verifier** | 1 | **Gatekeeper** | All Builder + approve/reject patches, resolve triage items, advance gates |
| **Admin** | 2 | **Owner** | All Gatekeeper + manage members, toggle features, calibrate thresholds, export |
| **Architect** | 3 | **Architect** | All Owner + schema changes, destructive operations, system configuration |

**Enforcement:** Hidden, not disabled. If a user can't do it, the UI element doesn't exist in their DOM. Server validates every request independently.

#### Feature Flags

Toggle grid for every managed feature. See [Feature Control Plane](../FeatureControlPlane/overview.md).

| Feature | Description |
|---------|-------------|
| **Flag grid** | Grouped by category (Extractors, AI, Integrations, Export). Each row: flag name, toggle, rollout %, status dot |
| **Toggle** | Instant flip via HTTP PATCH. No restart. |
| **Rollout %** | Slider: 0-100% gradual deployment |
| **Workspace overrides** | Per-workspace enable/disable with reason text |
| **Circuit breaker status** | Shows if/when a flag was auto-disabled due to errors |

#### Calibration

Threshold and weight management. See [Calibration](../FeatureControlPlane/calibration.md).

| Feature | Description |
|---------|-------------|
| **Parameter groups** | By feature: Extractors, Preflight, Identity Resolver, Health Scoring, Batch Processing |
| **Slider + numeric input** | Each parameter: slider with min/max/step |
| **Last calibrated** | Timestamp + actor name below each parameter |
| **Reset to default** | One-click restore to factory default |

#### Modules

| Feature | Description |
|---------|-------------|
| **Module list** | All available modules with enable/disable toggle |
| **Sort order** | Drag-reorder to control module bar icon order |
| **Module config** | Per-module settings (e.g., default grouping, pipeline stage names) |
| **Module health** | Vault counts per chamber, active user count per module |

#### Connectors

| Feature | Description |
|---------|-------------|
| **Connector list** | Configured integrations: Google Drive, Salesforce, Slack, Jira |
| **Add connector** | OAuth flow or API key configuration |
| **Status** | Health: active (green), error (red), pending (amber), disabled (gray) |
| **Sync log** | Per-connector history with success/failure counts |
| **Test connection** | Manual connectivity test button |

#### Audit Log

| Feature | Description |
|---------|-------------|
| **Event stream** | All audit events, most recent first |
| **Filters** | By: feature, user, event type, date range, severity |
| **Event detail** | Expandable: full payload, before/after state, timing data |
| **Export** | Download as CSV or JSON |

Event types: `flag.toggled`, `calibration.changed`, `member.role_changed`, `module.enabled`, `connector.sync_completed`, `lifecycle.transition`, `patch.applied`, `export.completed`

#### System Health

| Feature | Description |
|---------|-------------|
| **Health strip** | Per-feature card: status, last error, throughput, latency |
| **Pipeline dashboard** | Live pipeline: queued → downloading → extracting → preflight → ready |
| **Error rate charts** | Per-feature error rate over time (sparkline) |
| **Queue depths** | BullMQ queues: pending, active, completed, failed |
| **Uptime** | Per-service: API, WebSocket, Database, Redis |

---

## Personal Settings

### Layout

Centered modal overlay:

```
+------------------------------------------+
| PERSONAL SETTINGS              [X Close] |
|------------------------------------------|
| > Profile        |                       |
| > Notifications  |  [Active section]     |
| > Appearance     |                       |
| > Keybindings    |                       |
| > Connected Accts|                       |
+------------------------------------------+
```

### Sections

#### Profile

| Field | Editable | Notes |
|-------|----------|-------|
| Display name | Yes | |
| Email | Read-only | Set by auth provider |
| Avatar | Yes | Upload or Gravatar |
| Role | Read-only | Assigned by admin |
| Timezone | Yes | Used for SLA calculations |

#### Notifications

Per-user preferences (backed by Novu subscriber preferences).

| Setting | Options |
|---------|---------|
| **Priority filter** | Which priorities: Urgent, Action, Info, AI (checkboxes) |
| **Per-module toggle** | Enable/disable notifications per module |
| **Vault muting** | List of muted vaults |
| **Email digest** (future) | Daily / Weekly / Off |
| **Push** (future) | On / Off |

#### Appearance

| Setting | Options |
|---------|---------|
| **Theme** | Base / Airlock Dark (OLED) |
| **Sidebar width** | Compact (200px) / Standard (240px) / Wide (280px) |
| **Font size** | Small / Medium / Large |
| **Reduced motion** | On / Off |
| **High contrast** | On / Off |

#### Keybindings

| Default | Action |
|---------|--------|
| `Cmd+1-4` | Triptych states: Overview / Inspect / Edit / Approve |
| `Cmd+K` | Search / command palette |
| `Cmd+E` | Toggle heatmap mode |
| `Escape` | Close overlay / return to default |

#### Connected Accounts

OAuth connections for external services (Google, Salesforce, Slack, etc.).

---

## Key Design Decisions

1. **Overlay, not module** — Admin doesn't produce vaults, doesn't have chambers, doesn't follow the universal lifecycle.
2. **Two overlays** — Workspace Admin (power users) vs Personal Settings (everyone). Different access, different content.
3. **Hidden, not disabled** — Non-admin users never see admin controls.
4. **Audit everything** — Every admin action logged with before/after state.
5. **Instant effect** — Flag toggles, calibration changes, role changes all take effect immediately.

---

## Related Specs

- **[Feature Control Plane](../FeatureControlPlane/overview.md)** — Toggle, calibrate, audit, failure flag
- **[Toggle System](../FeatureControlPlane/toggle-system.md)** — DB-backed flags, HTTP toggle
- **[Calibration](../FeatureControlPlane/calibration.md)** — Admin-editable thresholds
- **[Roles](../Roles/overview.md)** — RBAC hierarchy and permission model
- **[Naming & Hierarchy](../Shell/naming-and-hierarchy.md)** — Admin as system overlay
- **[Notifications](../Notifications/overview.md)** — Novu preferences in Personal Settings
