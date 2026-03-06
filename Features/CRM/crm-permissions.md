# CRM Permissions Model -- Design Spec

> **Status:** SPECCED
> **Source:** /Features/Roles/overview.md, gap analysis

## Summary
Role-based access control matrix for all CRM views and actions. Based on Airlock's Discord-style permission model (Builder/Gatekeeper/Owner + custom roles).

### 1. CRM-Relevant Roles

| Role | Description | Typical User |
|------|-------------|-------------|
| **Owner** | Full access, can configure CRM module, manage Smart Lines, approve any deal | Founder, CEO |
| **Gatekeeper** | Can review/approve deals, view all accounts, manage health alerts | VP Sales, Sales Manager |
| **Builder** | Can create/edit deals, manage own leads/contacts, view own pipeline | Sales Rep, SDR, AE |
| **Viewer** | Read-only access to dashboards and reports | Finance, Operations |
| **Designer** | Can configure CRM layouts, import templates, set up workflows | CRM Admin |

### 2. Permission Matrix

| Action | Owner | Gatekeeper | Builder | Viewer | Designer |
|--------|-------|-----------|---------|--------|----------|
| **DISCOVER** |
| View Inbox | All | All | Own + pool | - | - |
| Reply to messages | All | All | Own assigned | - | - |
| View New Leads | All | All | All | All | All |
| Create lead manually | Yes | Yes | Yes | - | - |
| Assign lead | Any rep | Any rep | Self only | - | - |
| Qualify lead (MQL→SAL→SQL) | Yes | Yes | Own leads | - | - |
| Reject/disqualify lead | Yes | Yes | Own leads | - | - |
| **DEALS** |
| View Pipeline | All | All | All | All | All |
| Create deal | Yes | Yes | Yes | - | - |
| Edit deal (own) | Yes | Yes | Yes | - | - |
| Edit deal (others') | Yes | Yes | - | - | - |
| Advance deal stage | Yes | Yes | Own deals | - | - |
| Mark deal Won/Lost | Yes | Yes | Own deals | - | - |
| Delete deal | Yes | - | - | - | - |
| Approve deal review | Yes | Yes | - | - | - |
| Self-approve own deal | **NO** | **NO** | **NO** | **NO** | **NO** |
| **ACCOUNTS** |
| View accounts | All | All | Own + assigned | All | All |
| Create account | Yes | Yes | Yes | - | - |
| Edit account | Yes | Yes | Own accounts | - | - |
| Merge accounts | Yes | Yes | - | - | - |
| Delete/archive account | Yes | - | - | - | - |
| **CONTACTS** |
| View contacts | All | All | Own accounts | All | All |
| Create contact | Yes | Yes | Yes | - | - |
| Edit contact | Yes | Yes | Own contacts | - | - |
| Delete contact | Yes | Yes | - | - | - |
| **CUSTOMERS** |
| View Onboarding | All | All | Own customers | All | - |
| Complete onboarding tasks | Yes | Yes | Assigned tasks | - | - |
| View Health Monitor | All | All | Own customers | All | - |
| Action health recommendations | Yes | Yes | Own customers | - | - |
| View Renewals | All | All | Own | All | - |
| Process renewal | Yes | Yes | Own | - | - |
| **ADMIN** |
| Configure Smart Line | Yes | - | - | - | Yes |
| Import data (CSV) | Yes | Yes | - | - | Yes |
| Export data | Yes | Yes | - | - | Yes |
| Configure workflows | Yes | - | - | - | Yes |
| Manage CRM settings | Yes | - | - | - | Yes |

### 3. Self-Approval Prevention

The self-approval block is **unforgeable** (security principle #4):
- A user CANNOT approve a deal review they created or own
- Enforced at API level: `require_permission('deal.approve')` + `deal.owner_id != current_user.id`
- No API path, admin override, or workflow can bypass this
- Audit logged if attempted

### 4. Data Visibility Rules

| Data | Owner | Gatekeeper | Builder | Viewer |
|------|-------|-----------|---------|--------|
| All accounts | Yes | Yes | Assigned only | Yes (read) |
| All deals | Yes | Yes | Own + team | Yes (read) |
| Deal values | Yes | Yes | Own | Aggregates only |
| Contact PII (email/phone) | Yes | Yes | Own contacts | Redacted |
| Communication content | Yes | Yes | Own threads | - |
| Health scores | Yes | Yes | Own accounts | Yes |
| Import history | Yes | Yes | - | - |
| Audit logs | Yes | - | - | - |

### 5. Permission Computation

Follows Discord-style additive model:
1. Start with base role permissions (from table above)
2. Add permissions from any custom roles assigned to user
3. Apply channel-level overrides (per-account permission overrides)
4. Cache computed permissions in Redis (invalidate on role change)
5. Check via `require_permission('crm.deal.create')` middleware

Permission format: `module.entity.action` (e.g., `crm.deal.create`, `crm.lead.qualify`, `crm.account.merge`)

---

## Related Specs

- [/Features/Roles/overview.md](/Features/Roles/overview.md) -- Base role definitions and Discord-style permission model
- [/Features/Auth/overview.md](/Features/Auth/overview.md) -- Authentication and session management
- [crm-data-model.md](./crm-data-model.md) -- Entity schemas that permissions protect
