# CRM + Tasks Views — Design Spec

> **Purpose:** Concrete UI specs for every CRM and Tasks view, designed to match existing demo shell patterns (triage tables, Kanban boards, triptych layouts). Each view gets a wireframe, component breakdown, data mapping, and interaction spec. Ready to build as demo HTML pages.

> **Existing patterns to reuse:**
> - **Triage table** (triage-demo.html) — sortable columns, severity badges, filter bar, bulk select
> - **Kanban board** (tasks-demo.html) — status columns, draggable cards, toolbar with view toggle
> - **Vault tree** (crm-demo.html) — collapsible hierarchy, search, account detail panel
> - **Dossier/Triptych** (dossier-demo.html) — Signal | Orchestrate | Control panels
> - **Shell patterns** — sub-panel nav, chamber stats, badge counts, mini dashboard

---

## Updated Sub-Panel: CRM Module

Replaces the old chamber-labeled navigation with the business-story structure.

```
+-------------------------------+
| CRM                    MODULE |
|-------------------------------|
| > Quick Stats                 |
|   [42] Accounts  [8] Leads   |
|   [12] Deals     [3] Risk    |
|-------------------------------|
| > DISCOVER                    |
|   * Inbox               (5)  |  <- default view on module open
|   o New Leads            (8)  |
|   o Qualify              (3)  |
|                               |
| > DEALS                       |
|   o Pipeline                  |
|   o Deal Review          (2)  |
|   o Won Deals                 |
|                               |
| > ACCOUNTS                    |
|   o Accounts                  |
|   o Contacts                  |
|   o Activity Feed             |
|                               |
| > CUSTOMERS                   |
|   o Onboarding           (1)  |
|   o Health Monitor            |
|   o Renewals             (4)  |
|   o Expansion                 |
|-------------------------------|
| ACTIVE VAULTS                 |
|   * Acme Inc                  |
|   o Ostereo Music Group       |
|   o Summit Media              |
+-------------------------------+
```

### Quick Stats Implementation

Instead of the 4-chamber stat strip (Disc/Build/Rev/Ship), use business metrics:

```html
<div class="crm-quick-stats">
  <div class="quick-stat">
    <span class="quick-stat-count">42</span>
    <span class="quick-stat-label">Accounts</span>
  </div>
  <div class="quick-stat">
    <span class="quick-stat-count amber">8</span>
    <span class="quick-stat-label">Leads</span>
  </div>
  <div class="quick-stat">
    <span class="quick-stat-count blue">12</span>
    <span class="quick-stat-label">Deals</span>
  </div>
  <div class="quick-stat">
    <span class="quick-stat-count red">3</span>
    <span class="quick-stat-label">At Risk</span>
  </div>
</div>
```

CSS: Same grid as `.chamber-stats` but with semantic colors instead of chamber colors.

---

## View 1: CRM Inbox

**File:** `Features/CRM/crm-inbox-demo.html`
**Sub-panel nav:** DISCOVER > Inbox
**Pattern:** Triage table (reuse from triage-demo.html) + conversation preview panel

### Layout

```
+--Toolbar---------------------------------------------------------+
| Inbox (5 unread)  |  [All] [Texts] [Emails] [Calls]  | [Search] |
+---Table-------------------------------+---Preview Panel----------+
|  * | From         | Subject/Preview   | Channel | Time          |
|----+-----+--------------------------+---------+--------+        |
| [] | Jack Chen    | "Hey, we need to  | iMsg    | 10m    |  [Full|
|    | Nova Ent.    |  discuss renewal."|         |        |  conv |
|----+-----+--------------------------+---------+--------+  view  |
| [] | Sarah Kim    | "Can you send me  | Email   | 2h     |  here]|
|    | Acme Inc     |  the Q1 report?"  |         |        |       |
|----+-----+--------------------------+---------+--------+       |
| [] | +1-555-0199  | "Who handles      | SMS     | 4h     |       |
|    | (Unknown)    |  billing?"        |         |        |       |
|----+-----+--------------------------+---------+--------+       |
| [] | Mike Torres  | Missed call (2:14)| Call    | 1d     |       |
|    | Nova Ent.    |                   |         |        |       |
|----+-----+--------------------------+---------+--------+       |
| [] | Web Form     | Demo request:     | Form    | 1d     |       |
|    | "Lisa Park"  |  "Interested in.."| (web)   |        |       |
+----+-----+--------------------------+---------+--------+-------+
```

### Components

**Toolbar:**
- Title: "Inbox" + unread count badge (red)
- Channel filter chips: All | Texts | Emails | Calls | Forms
- Search input (right-aligned)

**Table (left ~60%):**
- Columns: Checkbox | From (name + company) | Preview (message snippet) | Channel (icon+label) | Time (relative)
- Row click → loads conversation in preview panel
- Unread rows: bold text, blue left border accent
- Channel icons: blue bubble (iMessage), green bubble (SMS), envelope (email), phone (call), globe (web form)
- Unknown contacts: show phone number, amber "Unknown" badge

**Preview Panel (right ~40%):**
- Conversation thread (most recent at bottom)
- Each message bubble: sender name, timestamp, message text
- AI intent badge at top: "deal_discussion (92%)" with teal background
- Quick action bar at bottom: [Reply] [Create Task] [Assign] [Dismiss]
- Reply composer: text input + "Send via iMessage" button

### Data Mapping

| Column | Source |
|--------|--------|
| From | `vault_contacts.name` or raw phone number |
| Company | Parent vault name (via vault hierarchy lookup) |
| Preview | `communications.body` (truncated to 60 chars) |
| Channel | `communications.channel` — icon mapped: imessage→blue, sms→green, email→envelope, call→phone |
| Time | `communications.created_at` — relative ("10m", "2h", "1d") |
| Intent | `communications.intent_classification` — shown in preview panel |

### Interactions

| Action | Result |
|--------|--------|
| Click row | Load conversation thread in preview panel |
| Click "Reply" | Open reply composer, pre-set channel to original |
| Click "Create Task" | Open Task Builder modal (pre-populated with communication context) |
| Click "Assign" | Dropdown to assign conversation to a team member |
| Click unknown contact row | Show "Identify Contact" prompt in preview panel |
| Bulk select + "Assign" | Reassign selected conversations |
| Bulk select + "Dismiss" | Archive selected (move out of inbox) |

---

## View 2: New Leads

**File:** `Features/CRM/crm-leads-demo.html`
**Sub-panel nav:** DISCOVER > New Leads
**Pattern:** Triage table with lead-specific columns

### Layout

```
+--Toolbar---------------------------------------------------------+
| New Leads (8)  | [All] [Unworked] [Mine]  | [+ Add Lead] [Search]|
+--Table-----------------------------------------------------------+
| [] | Lead          | Source      | Score | Stage | Assigned | Age |
|----+---------------+------------+-------+-------+----------+-----|
| [] | TechFlow Inc  | Contract   |  72   | New   | —        | 3d  |
|    |               | Upload     |       |       |          |     |
|----+---------------+------------+-------+-------+----------+-----|
| [] | Lisa Park     | Web Form   |  —    | New   | —        | 1d  |
|    | (unmatched)   |            |       |       |          |     |
|----+---------------+------------+-------+-------+----------+-----|
| [] | +1-555-0199   | Smart Line |  —    | New   | —        | 4h  |
|    | (unknown)     | (inbound)  |       |       |          |     |
|----+---------------+------------+-------+-------+----------+-----|
| [] | MediaWorks    | Entity     |  65   | MQL   | Ana C.   | 5d  |
|    | LLC           | Resolution |       |       |          |     |
|----+---------------+------------+-------+-------+----------+-----|
| [] | Pinnacle      | Referral   |  81   | SAL   | Sarah M. | 7d  |
|    | Partners      | (Jack Chen)|       |       |          |     |
+----+---------------+------------+-------+-------+----------+-----+
```

### Components

**Toolbar:**
- Title + count badge
- Quick filters: All | Unworked (no assignee) | Mine (assigned to current user)
- [+ Add Lead] button (opens manual lead creation modal)
- Search

**Table:**
- Lead name: entity name or phone number; sub-line shows match status
- Source: how they entered (Contract Upload, Web Form, Smart Line, Entity Resolution, Referral, API)
- Score: qualification score (0-100), color-coded (red <40, amber 40-70, green >70), dash if unscored
- Stage: lifecycle badge — `New` (gray) | `MQL` (teal) | `SAL` (blue) | `SQL` (green)
- Assigned: avatar + initials, or dash if unassigned
- Age: time since created (relative)

**Row click → Lead detail panel** (slide-in from right, or navigate to vault):
- Contact info (if known)
- Communication history (if any)
- Source details (which contract, which form, which call)
- Quick actions: [Qualify as MQL] [Assign] [Create Task] [Convert to Deal]

### Stage Progression

Dragging or clicking stage buttons advances the lifecycle:

```
New → MQL → SAL → SQL → [Convert to Deal]
                              |
                              v
                     Creates vault in DEALS > Pipeline
                     at "Prospecting" stage
```

"Convert to Deal" is the Airlock equivalent of Salesforce's Lead Conversion — but it just changes `lifecycle_stage` on the same vault row, no object migration.

---

## View 3: Qualify (MQL/SAL/SQL Funnel)

**File:** `Features/CRM/crm-qualify-demo.html`
**Sub-panel nav:** DISCOVER > Qualify
**Pattern:** 3-column Kanban (reuse tasks-demo.html pattern)

### Layout

```
+--Toolbar---------------------------------------------------------+
| Qualify (3 pending)  | [Funnel View] [Table View]  | [Filters]   |
+------+---MQL---+---+---SAL---+---+---SQL---+-----+---Converted--+
|      | MediaWrk|   | Pinnacl.|   |         |     |  Acme Inc    |
|      | Score:65|   | Score:81|   |         |     |  Score: 88   |
|      | Source: |   | Source: |   |         |     |  -> Pipeline |
|      | Entity  |   | Referral|   |         |     |              |
|      | 5 days  |   | 7 days  |   |         |     |  2 days ago  |
|      |         |   |         |   |         |     |              |
|      | [Accept]|   | [Accept]|   |         |     |              |
|      | [Reject]|   | [Reject]|   |         |     |              |
+------+---------+---+---------+---+---------+-----+--------------+
```

### Components

3-column Kanban board mimicking the tasks-demo.html pattern:

| Column | Color | What's Here |
|--------|-------|-------------|
| **MQL** | Teal | Leads that meet automated criteria. Rep must "Accept" (→ SAL) or "Reject" (→ Nurture) |
| **SAL** | Blue | Rep accepted, actively working. Must "Qualify" (→ SQL) or "Disqualify" (→ Nurture) |
| **SQL** | Green | Confirmed buying intent. [Convert to Deal] button creates opportunity in Pipeline |
| **Converted** | Gray | Recently converted (last 7 days). Shows where they landed in Pipeline |

**Card anatomy:**

```
+---------------------------+
| [Score dot] MediaWorks LLC|
| Source: Entity Resolution |
| Score: 65/100             |
| Age: 5 days               |
| Assigned: Ana Chen        |
|                           |
| [Accept →] [Reject ×]    |
+---------------------------+
```

Cards are draggable between columns (like Tasks Kanban). Drag from MQL → SAL = Accept. Drag backward = requires reason.

---

## View 4: Pipeline (Deal Kanban)

**File:** `Features/CRM/crm-pipeline-demo.html`
**Sub-panel nav:** DEALS > Pipeline
**Pattern:** Kanban board (reuse tasks-demo.html) with deal-specific cards

### Layout

```
+--Toolbar---------------------------------------------------------------+
| Pipeline  | [Kanban] [Table] [Forecast]  | [+ New Deal]  | [Filters]   |
+--Prospecting-+--Discovery--+--Proposal--+--Negotiation--+--Close------+
| Nova Ent.    | Acme Inc    | Summit     |               | Ostereo     |
| $120K        | $85K        | $240K      |               | $180K       |
| Sarah M.     | Ana C.      | Sarah M.   |               | David P.    |
| 3 tasks      | 2 tasks     | 4 tasks    |               | 1 task      |
| 5 days       | 12 days     | 8 days     |               | 21 days     |
|              |             |            |               |             |
| TechFlow     |             |            |               |             |
| $60K         |             |            |               |             |
| —            |             |            |               |             |
| 0 tasks      |             |            |               |             |
| 1 day        |             |            |               |             |
+--------------+-------------+------------+---------------+-------------+
| TOTAL: $180K | $85K        | $240K      | $0            | $180K       |
+--------------+-------------+------------+---------------+-------------+
```

### Components

**Toolbar:**
- Title
- View toggle: Kanban (default) | Table | Forecast (bar chart)
- [+ New Deal] button — opens deal creation modal
- Filters: assignee, value range, age, account

**Column headers:**
- Stage name
- Count of deals
- Total value
- Color-coded by stage progression (cool → warm: teal → blue → purple → amber → green)

**Deal card:**

```
+---------------------------+
| Nova Entertainment        |
| henderson-msa             |  <- vault name (smaller, muted)
|                           |
| $120,000                  |  <- deal value (large, white)
|                           |
| [Avatar] Sarah Miller     |  <- assigned rep
| 3 tasks (1 overdue)       |  <- task count, red if overdue
| Stage: 5 days             |  <- time in current stage
|                           |
| [---====-------] 40%      |  <- progress bar (tasks complete %)
|                           |
| [*] Next: Send proposal   |  <- next task due
+---------------------------+
```

**Stage-gating indicator:** When a deal card has incomplete required tasks, a lock icon appears on the right column border. Hovering shows: "Complete 2 required tasks before advancing."

**Column footer:** Total deal value for that stage. Useful for pipeline forecasting.

### Interactions

| Action | Result |
|--------|--------|
| Drag card between columns | Advance/retreat deal stage. Gated stages require confirmation modal listing incomplete tasks |
| Click card | Navigate to vault in Accounts view (or open deal detail panel) |
| Click task count | Expand inline task list on the card |
| Click "+ New Deal" | Open deal creation modal (title, account, value, assignee, stage) |
| Hover column header | Show entry/exit criteria tooltip |

### Deal Detail Panel (Slide-In)

When a card is clicked, a right-side panel slides in (40% width):

```
+--Deal Detail: Nova Entertainment-----------+
| henderson-msa                               |
| Stage: Prospecting (5 days)                 |
| Value: $120,000                             |
| Probability: 20%                            |
| Close date: Apr 15, 2026                    |
| Owner: Sarah Miller                         |
|---------------------------------------------|
| STAGE TASKS                                 |
| [x] Research account background    Mar 1    |
| [x] Send intro email               Mar 2    |
| [ ] Schedule discovery call         Mar 7    |  <- overdue (red)
|---------------------------------------------|
| RECENT ACTIVITY                             |
| Mar 4  Jack Chen texted about renewal       |
| Mar 2  Sarah sent intro email               |
| Mar 1  Lead converted from MQL              |
|---------------------------------------------|
| CONTACTS                                    |
| Jack Chen — Biz Dev — last: 10m ago         |
| Sarah Kim — Legal — last: 3d ago            |
|---------------------------------------------|
| [Advance Stage] [Create Task] [Add Note]    |
+---------------------------------------------+
```

---

## View 5: Deal Review

**File:** `Features/CRM/crm-deal-review-demo.html`
**Sub-panel nav:** DEALS > Deal Review
**Pattern:** Triage table (reuse from triage-demo.html) filtered to review-stage deals

### Layout

```
+--Toolbar---------------------------------------------------------+
| Deal Review (2)  | [Pending] [Approved] [Rejected] | [Search]    |
+--Table-----------------------------------------------------------+
| [] | Deal          | Value    | Reviewer  | Status  | SLA       |
|----+---------------+----------+-----------+---------+-----------|
| [] | Summit Media  | $240K    | David P.  | Pending | 18h left  |
|    | henderson-msa | Proposal |           |         |           |
|----+---------------+----------+-----------+---------+-----------|
| [] | Ostereo Music | $180K    | David P.  | Pending | 6h left   |
|    | ostereo-msa   | Close    |           |         | (amber)   |
+----+---------------+----------+-----------+---------+-----------+
```

Same triage table pattern. Row click opens the deal in a triptych-style review view with the approval chain (reuse the Patch Workflow approval-chain pattern).

---

## View 6: Accounts

**File:** `Features/CRM/crm-demo.html` (existing — enhance)
**Sub-panel nav:** ACCOUNTS > Accounts
**Pattern:** Existing vault hierarchy tree + account cards

The existing `crm-demo.html` already has the vault hierarchy tree on the left and account detail on the right. Enhance with:

- **Health score** badge on each account row (green/amber/red dot)
- **Deal count** badge (number of active deals)
- **Last contact** timestamp
- **Segment** tag (Enterprise / Mid-Market / SMB)
- **Smart Line status** indicator (active / not provisioned)

### Enhanced Account Detail (Right Panel)

```
+--Account: Nova Entertainment-----------------+
| [Health: 82] [Segment: Enterprise] [Active]  |
|                                               |
| HIERARCHY                                     |
|   Nova Entertainment (Parent)                 |
|   +-- Entertainment Division                  |
|   |   +-- Henderson MSA (Item) — $120K        |
|   |   +-- Q2 Expansion (Item) — $60K          |
|   +-- Music Division                          |
|       +-- Ostereo MSA (Item) — $180K          |
|                                               |
| CONTACTS (3)                                  |
|   Jack Chen — Biz Dev — 12 interactions       |
|   Sarah Kim — Legal — 5 interactions          |
|   Mike Torres — CFO — 1 interaction           |
|                                               |
| SMART LINE                                    |
|   Number: +1-555-0100 (active)                |
|   Messages this month: 24                     |
|   Last inbound: 10 minutes ago                |
|                                               |
| DEALS (2 active)                              |
|   henderson-msa — Prospecting — $120K         |
|   ostereo-msa — Close — $180K                 |
|                                               |
| HEALTH TREND                                  |
|   [===== 82 ====---]  +3 this week            |
|   [mini sparkline chart]                      |
+-----------------------------------------------+
```

---

## View 7: Contacts (Org Tree)

**File:** `Features/CRM/crm-contacts-demo.html`
**Sub-panel nav:** ACCOUNTS > Contacts
**Pattern:** Directory list + org tree visualization

### Layout

```
+--Toolbar---------------------------------------------------------+
| Contacts (47) | [Directory] [Org Tree]  | [+ Add Contact] [Search]|
+--Directory (left 50%)---+--Org Tree (right 50%)-----------------+
|                          |                                        |
| NOVA ENTERTAINMENT       |        Nova Entertainment              |
|   Jack Chen              |           /    |    \                  |
|   Biz Dev | 12 msgs      |      Jack    Sarah   Mike              |
|   last: 10m ago          |     BizDev   Legal    CFO              |
|                          |      12msg   5msg    1msg              |
|   Sarah Kim              |                                        |
|   Legal | 5 msgs         |                                        |
|   last: 3d ago           |  ACME INC                              |
|                          |        Acme Inc                        |
|   Mike Torres            |          /       \                     |
|   CFO | 1 msg            |     Sarah K.   Tom B.                  |
|   last: 1d ago           |     Billing   Ops                      |
|                          |      3msg     1msg                     |
| ACME INC                 |                                        |
|   Sarah Kim              |                                        |
|   Billing | 3 msgs       |                                        |
|   last: 2d ago           |                                        |
+---------------------------+---------------------------------------+
```

### Components

**Directory (left):**
- Grouped by account (parent vault)
- Each contact: name, role, interaction count, last interaction
- Contact source icon: manual (person), inbound (phone), entity resolution (robot)
- Click contact → highlights in org tree

**Org Tree (right):**
- Visual tree diagram per account
- Nodes: name, role, message count
- Node size proportional to interaction frequency
- Lines show reporting relationships (if detected from conversations)
- Click node → selects contact in directory, shows detail below

---

## View 8: Activity Feed

**File:** `Features/CRM/crm-activity-demo.html`
**Sub-panel nav:** ACCOUNTS > Activity Feed
**Pattern:** Chronological event stream (like Signal panel, but cross-account)

### Layout

```
+--Toolbar---------------------------------------------------------+
| Activity Feed  | [All] [Texts] [Calls] [Deals] [Tasks] | [Search]|
+--Feed------------------------------------------------------------+
|                                                                   |
|  TODAY                                                            |
|  10:42 AM  [iMsg]  Jack Chen (Nova) texted about Henderson        |
|            renewal. Intent: deal_discussion (92%)                 |
|            [View Thread] [Create Task]                            |
|                                                                   |
|  10:15 AM  [Deal]  Summit Media moved to Proposal stage           |
|            by Sarah Miller. 4 stage tasks auto-created.           |
|            [View Deal] [View Tasks]                               |
|                                                                   |
|  9:30 AM   [Task]  "Send Q1 report to Acme" resolved             |
|            by Ana Chen.                                           |
|                                                                   |
|  YESTERDAY                                                        |
|  4:15 PM   [Call]  Missed call from +1-555-0199 (Unknown)         |
|            Voicemail: "Hi, calling about partnership..."          |
|            [Identify Contact] [Create Lead]                       |
|                                                                   |
|  2:30 PM   [Deal]  Ostereo Music Group moved to Close             |
|            Verbal agreement confirmed. Contract generation        |
|            task auto-created.                                     |
|                                                                   |
|  11:00 AM  [Lead]  TechFlow Inc created from contract upload      |
|            Entity resolution confidence: 72%. Needs review.       |
|            [View Lead] [Qualify]                                   |
|                                                                   |
+-------------------------------------------------------------------+
```

### Components

**Event card anatomy:**
```
+--[icon]--[timestamp]--[event type badge]--[description]-----------+
|          [sub-detail line]                                         |
|          [Action buttons]                                          |
+-------------------------------------------------------------------+
```

**Event types and icons:**
| Type | Icon | Color | Badge |
|------|------|-------|-------|
| iMessage/SMS | Chat bubble | Blue/Green | iMsg / SMS |
| Email | Envelope | Gray | Email |
| Call | Phone | Purple | Call |
| Deal stage change | Arrow right | Teal | Deal |
| Task completed | Checkmark | Green | Task |
| Task created | Plus | Blue | Task |
| Lead created | Person+ | Amber | Lead |
| Health alert | Warning | Red | Alert |

**Filters:** Channel type chips at top. Click to toggle visibility.

---

## View 9: Onboarding

**File:** `Features/CRM/crm-onboarding-demo.html`
**Sub-panel nav:** CUSTOMERS > Onboarding
**Pattern:** Checklist cards per customer (like batch progress cards in triage)

### Layout

```
+--Toolbar---------------------------------------------------------+
| Onboarding (1 active)  | [Active] [Completed]  | [Search]        |
+--Cards-----------------------------------------------------------+
|                                                                   |
| +--Acme Inc — Onboarding Started Mar 2-----------------------+   |
| | Progress: [========--------] 60% (3/5 complete)             |   |
| |                                                             |   |
| | [x] Create account in system               Ana C.  Mar 2   |   |
| | [x] Provision Smart Line number             System  Mar 2   |   |
| |     +1-555-0142 (active)                                    |   |
| | [x] Send welcome packet                    Sarah M. Mar 3  |   |
| | [ ] Assign team members                    —        Mar 5   |   |
| | [ ] Schedule kickoff call                  —        Mar 7   |   |
| |                                                             |   |
| | [Assign All] [Send Reminder]                                |   |
| +-------------------------------------------------------------+   |
|                                                                   |
+-------------------------------------------------------------------+
```

---

## View 10: Health Monitor

**File:** `Features/CRM/crm-health-demo.html`
**Sub-panel nav:** CUSTOMERS > Health Monitor
**Pattern:** Dashboard cards (like entity health cards in triage) + risk table

### Layout

```
+--Toolbar---------------------------------------------------------+
| Health Monitor  | [At Risk] [Declining] [All]  | [Search]         |
+--Risk Cards (horizontal scroll)------+--Detail Panel-------------+
| +--Nova Ent.--+  +--Summit---+      |                            |
| | Health: 82  |  | Health: 54|      | Selected: Summit Media     |
| | Trend: +3   |  | Trend: -12|      | Health Score: 54 (-12)     |
| | Status: OK  |  | Status:   |      |                            |
| | 2 deals     |  | AT RISK   |      | RISK FACTORS               |
| | Last: 10m   |  | 1 deal    |      | - No contact in 14 days    |
| +-------------+  | Last: 14d |      | - Deal stalled at Proposal |
|                   +----------+      | - 1 overdue task           |
| +--Acme Inc--+  +--Ostereo--+      |                            |
| | Health: 91  |  | Health: 78|      | RECOMMENDED ACTIONS         |
| | Trend: +1   |  | Trend: -2 |      | [x] Send check-in text     |
| | Status: OK  |  | Status: OK|      | [ ] Follow up on proposal  |
| | 1 deal      |  | 1 deal    |      | [ ] Escalate to manager    |
| +-------------+  +----------+      |                            |
+--------------------------------------+----------------------------+
```

---

## View 11: Renewals

**File:** `Features/CRM/crm-renewals-demo.html`
**Sub-panel nav:** CUSTOMERS > Renewals
**Pattern:** Timeline/table hybrid (like calendar but focused on renewal dates)

### Layout

```
+--Toolbar---------------------------------------------------------+
| Renewals (4 upcoming)  | [30 days] [90 days] [All]  | [Search]   |
+--Timeline Table--------------------------------------------------+
| Due Date    | Account       | Contract      | Value  | Status    |
|-------------+---------------+---------------+--------+-----------|
| Mar 15      | Henderson Co  | henderson-msa | $120K  | Due in 11d|
| (11 days)   |               |               |        | (amber)   |
|-------------+---------------+---------------+--------+-----------|
| Apr 1       | Summit Media  | summit-msa    | $240K  | Due in 28d|
| (28 days)   |               |               |        | (green)   |
|-------------+---------------+---------------+--------+-----------|
| Apr 15      | Warner Bros   | warner-amend  | $95K   | Due in 42d|
| (42 days)   |               |               |        | (green)   |
|-------------+---------------+---------------+--------+-----------|
| Jun 1       | Ostereo Music | ostereo-msa   | $180K  | Due in 89d|
| (89 days)   |               |               |        | (green)   |
+-------------+---------------+---------------+--------+-----------+
| TOTAL RENEWAL VALUE: $635K                                        |
+-------------------------------------------------------------------+
```

Row color: Red if <7 days, Amber if <30 days, Green if >30 days. Click row → opens vault.

---

## Task Builder Modal

Shared across all modules. Opens when "Create Task" is clicked from any context.

**File:** Component inside `demo-shell/index.html` (not an iframe — overlay modal in the shell)

### Layout

```
+--Create Task-------------------------------------------+
|                                                [x]     |
| Title*                                                 |
| [Indemnification clause missing cap amount     ]       |
|                                                        |
| Description                                            |
| [This section references liability cap but     ]       |
| [doesn't specify dollar amount                 ]       |
|                                                        |
| +--Context (auto-populated)-------------------------+  |
| | Module: Contracts                                 |  |
| | Vault: henderson-msa                              |  |
| | Source: Document annotation (page 14, Art. 8.2)   |  |
| +---------------------------------------------------+  |
|                                                        |
| Type       [annotation v]    Severity  (o)info         |
|                                        ( )warning      |
|                                        ( )blocker      |
|                                                        |
| +--Role Chain--------------------------------------+   |
| | Builder:  [Ana Chen v]                           |   |
| | Modifier: [+ Add]                               |   |
| | Signer:   [David Park v]  Required: [x]         |   |
| +--------------------------------------------------+   |
|                                                        |
| Due Date  [Mar 7, 2026]  [5:00 PM]                    |
|                                                        |
| +--Linked Evidence---------------------------------+   |
| | [PDF icon] henderson-msa-v3.pdf — Page 14        |   |
| | "The Contractor shall indemnify..."              |   |
| | [View in Document]                               |   |
| +--------------------------------------------------+   |
|                                                        |
|                          [Cancel]  [Create Task]       |
+--------------------------------------------------------+
```

### Context Auto-Population

The modal pre-fills based on where it was opened:

| Opened From | Auto-Populated Fields |
|-------------|----------------------|
| Document annotation | Module=Contracts, Vault=current, Source=annotation, page/section, highlighted text |
| CRM Inbox message | Module=CRM, Vault=sender's account, Source=smart_line, message preview, contact info |
| Pipeline deal card | Module=CRM, Vault=deal vault, Source=deal_stage, stage name, deal value |
| Triage table row | Module=Contracts, Vault=triage item's vault, Source=triage, field code, severity |
| Tasks module "New Task" | Module=Tasks, Vault=none, Source=manual |
| Keyboard shortcut (Cmd+T) | Module=current, Vault=current (if in a vault view) |

### Type-Specific Sections

When the user changes the Type dropdown, additional sections appear:

- **annotation:** Document link, page, section path, highlighted text preview
- **approval:** Approval chain editor (Verifier → Admin), self-approval prevention toggle
- **deal_task:** Pipeline stage selector, stage-gating toggle, linked contact
- **inbound:** Communication thread link, channel, AI intent classification

---

## Task Card Anatomy (Shared Component)

Used in Tasks Kanban, Pipeline deal cards (inline), Home widgets, and Activity Feed.

### Standard Card

```
+--[Severity dot]--[Module badge]---------------+
| Indemnification clause missing cap amount     |
| henderson-msa                                  |
|                                                |
| [Avatar] Ana Chen          Due: 2h 15m (amber)|
| Type: annotation           Source: doc viewer  |
|                                                |
| Role chain: Ana → [review] → David            |
| [===========--------] in_progress              |
+------------------------------------------------+
```

### Compact Card (Home Widget)

```
+--[*]--Missing $ cap--henderson--blocker--2h 15m--+
```

Single row: severity dot, title (truncated), vault, severity badge, due countdown.

### Inline Task List (Pipeline Card)

```
| [x] Research account background    Mar 1    |
| [x] Send intro email               Mar 2    |
| [ ] Schedule discovery call         Mar 7    | <- overdue (red)
```

Checkbox + title + due date. Overdue items in red. Checked items strikethrough.

---

## Updated Sub-Panel: Tasks Module

The Tasks module sub-panel also gets a refresh to match the business-story pattern:

```
+-------------------------------+
| Tasks                  MODULE |
|-------------------------------|
| > Quick Stats                 |
|   [24] Open    [8] Progress   |
|   [5] Review   [12] Today    |
|-------------------------------|
| > INBOX                       |
|   * All Tasks            (24) |  <- default view
|   o My Tasks             (8)  |
|   o Unassigned           (6)  |
|   o Overdue              (3)  |
|                               |
| > WORK                        |
|   o Kanban Board              |
|   o Focus Mode                |
|                               |
| > REVIEW                      |
|   o Pending Review       (5)  |
|   o Recently Completed        |
|                               |
| > REPORTS                     |
|   o Productivity              |
|   o Workload                  |
|-------------------------------|
| PINNED GROUPS                 |
|   * Contracts Triage          |
|   o CRM Follow-ups           |
|   o Onboarding Tasks          |
+-------------------------------+
```

---

## Demo Build Priority

What to build first as demo HTML pages:

| Priority | View | File | Reuses Pattern From |
|----------|------|------|---------------------|
| 1 | **Pipeline Kanban** | crm-pipeline-demo.html | tasks-demo.html (Kanban) |
| 2 | **CRM Inbox** | crm-inbox-demo.html | triage-demo.html (table) + new preview panel |
| 3 | **Accounts** (enhance existing) | crm-demo.html | Already exists, add health/deals/smartline |
| 4 | **Activity Feed** | crm-activity-demo.html | New (event stream) |
| 5 | **Contacts + Org Tree** | crm-contacts-demo.html | New (directory + tree viz) |
| 6 | **New Leads** | crm-leads-demo.html | triage-demo.html (table) |
| 7 | **Qualify Funnel** | crm-qualify-demo.html | tasks-demo.html (Kanban, 3 columns) |
| 8 | **Health Monitor** | crm-health-demo.html | New (dashboard cards) |
| 9 | **Renewals** | crm-renewals-demo.html | triage-demo.html (table) |
| 10 | **Deal Review** | crm-deal-review-demo.html | triage-demo.html (table) |
| 11 | **Onboarding** | crm-onboarding-demo.html | New (checklist cards) |
| 12 | **Task Builder Modal** | In demo-shell/index.html | New (modal overlay) |

### Also Update

- **demo-shell/index.html** — Replace CRM sub-panel with new Discover/Deals/Accounts/Customers structure
- **demo-shell/index.html** — Replace Tasks sub-panel with new Inbox/Work/Review/Reports structure
- **demo-shell/index.html** — Add Task Builder modal as shell-level overlay (Cmd+T shortcut)
- **demo-shell/index.html** — Wire CRM nav items to new demo HTML files

---

## Design Tokens (CRM-Specific)

| Token | Value | Usage |
|-------|-------|-------|
| `--crm-discover` | `var(--teal)` | Discover section headers, MQL badges |
| `--crm-deals` | `var(--accent-blue)` | Deals section, Pipeline headers |
| `--crm-accounts` | `var(--purple)` | Accounts section |
| `--crm-customers` | `var(--green)` | Customers section, onboarding |
| `--stage-prospecting` | `#4dd0e1` | Pipeline column: Prospecting |
| `--stage-discovery` | `#29b6f6` | Pipeline column: Discovery |
| `--stage-proposal` | `#7e57c2` | Pipeline column: Proposal |
| `--stage-negotiation` | `#ffa726` | Pipeline column: Negotiation |
| `--stage-close` | `#66bb6a` | Pipeline column: Close |
| `--channel-imessage` | `#0091ea` | iMessage bubble icon |
| `--channel-sms` | `#4caf50` | SMS bubble icon |
| `--channel-email` | `#9e9e9e` | Email icon |
| `--channel-call` | `#7b1fa2` | Phone call icon |
| `--channel-form` | `#ff9800` | Web form icon |

---

## Related Specs

- **[CRM Comms Brainstorm](./crm-comms-brainstorm.md)** — Smart Line, communications architecture, sub-panel structure
- **[Task Lifecycle Deep Dive](../TaskSystem/task-lifecycle-deep-dive.md)** — Builder/Modifier/Signer roles, trigger → task pipeline
- **[Universal Task System](../TaskSystem/overview.md)** — Master task table schema
- **[Tasks Module](../Tasks/overview.md)** — Tasks module views
- **[Triage Dashboard](../Triage/overview.md)** — Table pattern reference
- **[Record Inspector](../Record%20Inspector/)** — Document annotation → task flow
