# CRM Lifecycle Flows -- Design Spec

> **Status:** SPECCED
> **Source:** crm-views-design-spec.md, overview.md, crm-comms-brainstorm.md, Roles/overview.md, WorkflowEngine/overview.md

## Summary

Complete state machines and trigger definitions for the CRM lead-to-customer lifecycle. Documents what causes data to move between screens, who can trigger transitions, and what side effects occur. Every transition answers four questions: WHO triggers it, WHAT causes it, WHAT side effects happen, and WHERE the user ends up.

**Key architectural principle:** There is no "conversion" event in Airlock. A vault starts in Discover and its `lifecycle_stage` field advances. The "lead" and the "opportunity" are the same vault row the entire time. CRM sub-panel sections are filtered views of the same hierarchy at different stages. Zero data migration, zero broken history.

---

## 1. Lead Lifecycle State Machine

### State Diagram

```
                    +-------+
                    |  New  |
                    +---+---+
                        |
            +-----------+-----------+
            |                       |
            v                       v
        +-------+              +---------+
        |  MQL  |              | Nurture |<----+
        +---+---+              +---------+     |
            |                       ^          |
            v                       |          |
        +-------+                   |          |
        |  SAL  +-------------------+          |
        +---+---+                              |
            |                                  |
            v                                  |
        +-------+                              |
        |  SQL  +------------------------------+
        +---+---+
            |
            v
  +-------------------+
  | Converted to Deal |
  +-------------------+
  (Creates Item vault
   at Prospecting stage)


  Any active state ---> [Disqualified]
                        (terminal, soft-delete)
```

### Transition Table

| # | From | To | Trigger | Who Can Trigger | Reversible? |
|---|------|----|---------|-----------------|-------------|
| T1 | New | MQL | Lead score reaches 40+ (auto) OR rep clicks "Qualify as MQL" | Any rep with `manage_tasks` permission, or automated via Workflow Engine lead scoring rule | Yes -- manager can demote back to New |
| T2 | MQL | SAL | Rep clicks "Accept" on Qualify funnel OR manager assigns lead to rep | Assigned rep or any rep with Builder+ module role in CRM | Yes -- rep can "Return to MQL" with reason |
| T3 | SAL | SQL | Discovery call completed AND lead score >= 70 (auto) OR manager manually promotes | Assigned rep or manager (Lead/Director org role) | Yes -- can return to SAL if re-evaluation needed |
| T4 | SQL | Converted to Deal | Rep clicks "Convert to Deal" on Qualify funnel or lead detail | Assigned rep or manager | No -- deal creation is permanent (lead record archived, deal vault persists) |
| T5 | Any active | Nurture | Rep clicks "Move to Nurture" or "Reject" with reason | Assigned rep or manager | Yes -- re-engagement triggers automatic re-scoring |
| T6 | Any active | Disqualified | Rep marks as disqualified (must select reason: bad data, wrong person, spam, duplicate, competitor) | Assigned rep, manager, or Gatekeeper+ | No -- soft-deleted from active views; only admin can restore |

### Transition Details

#### T1: New --> MQL

**What causes it:**
- **Automatic:** Workflow Engine `lead.scoring` workflow evaluates inbound signals and computes a `qualification_score`. When score crosses 40, the workflow fires a `lead.stage_change` event with `target_stage: 'mql'`.
- **Manual:** Rep views lead in New Leads table, clicks "Qualify as MQL" button in the lead detail panel.

**Scoring inputs (from Workflow Engine lead scoring workflow):**
- Firmographic fit (industry match, company size): 0-20 points
- Engagement signals (replied to message, opened email, visited web form): 0-30 points
- Source quality (referral = 15, contract upload = 12, web form = 8, unknown inbound = 5): 0-15 points
- Contact completeness (name known, role known, email known): 0-15 points
- Recency (contacted within 7 days = 10, 14 days = 5, 30 days = 2): 0-10 points
- Entity resolution confidence: 0-10 points

**Side effects:**
1. `qualification_score` updated on vault record
2. `lifecycle_stage` set to `'mql'`
3. Task created: "Complete MQL checklist" (type: `deal_task`, assigned to rep or pool)
   - Checklist items: Verify company identity, Confirm contact info, Review source data, Assess initial fit
4. Activity event logged: `lead.stage_changed` with `{from: 'new', to: 'mql', trigger: 'auto'|'manual', actor_id}`
5. Notification sent to assigned rep (Novu: in-app + optional push)
6. Lead card moves from New Leads table to Qualify funnel MQL column

**Where user ends up:** If triggered manually, the lead detail panel updates to show MQL badge. If automatic, rep sees notification badge on Qualify nav item in sub-panel.

---

#### T2: MQL --> SAL

**What causes it:**
- **Primary:** Rep clicks "Accept" button on the lead card in the Qualify funnel MQL column.
- **Alternative:** Manager opens unassigned MQL lead and clicks "Assign to [rep name]" -- assignment implicitly accepts.
- **Drag:** Rep drags card from MQL column to SAL column in Qualify Kanban.

**Side effects:**
1. `lifecycle_stage` set to `'sal'`
2. `assigned_to` set to accepting rep's `user_id` (if not already assigned)
3. Activity event logged: `lead.accepted` with `{actor_id, previous_stage: 'mql'}`
4. Sales team notification via Novu: "Lead [name] accepted by [rep]"
5. Lead card moves from MQL to SAL column in Qualify funnel
6. Task created: "Complete discovery preparation" (due: +3 business days)
   - Checklist: Research account background, Identify stakeholders, Prepare discovery questions
7. Timer starts: days-in-stage counter for SAL (tracked for velocity metrics)

**Where user ends up:** Qualify funnel with card now in SAL column. Card shows assigned rep avatar.

**Backward movement (SAL --> MQL):**
- Rep clicks "Return to MQL" -- requires reason selection (not ready, needs more nurturing, wrong assignment)
- Side effects: assignment cleared, task cancelled, activity event logged with reason
- Only the assigned rep or a manager can demote

---

#### T3: SAL --> SQL

**What causes it:**
- **Automatic:** Two conditions must BOTH be true:
  1. Task "Complete discovery call" marked as done (sets `discovery_call_completed: true`)
  2. `qualification_score >= 70` (score recalculated after discovery call activity)
- **Manual:** Manager clicks "Promote to SQL" on the lead detail panel (bypasses score requirement, logs override reason)

**Side effects:**
1. `lifecycle_stage` set to `'sql'`
2. `qualified_at` timestamp set (this marks the "conversion moment" for reporting)
3. Activity event logged: `lead.qualified` with `{score, discovery_completed: true}`
4. "Convert to Deal" button becomes prominent on the lead card (green, pulsing badge)
5. Task created: "Convert to Deal or document reason for hold" (due: +2 business days)
6. Pipeline notification to sales manager: "SQL lead ready for conversion: [name], score: [score]"
7. Workflow Engine fires `lead.qualified` event -- cross-module trigger available

**Where user ends up:** Qualify funnel with card in SQL column. Card shows green "Ready to Convert" badge.

---

#### T4: SQL --> Converted to Deal

**What causes it:**
- Rep clicks "Convert to Deal" on SQL card or lead detail panel.
- A modal appears: Deal Creation Modal.

**Deal Creation Modal fields:**
```
+--Convert to Deal-------------------------------+
| Deal Name:    [henderson-msa          ]        |
| Deal Value:   [$120,000              ]         |
| Close Date:   [Apr 15, 2026          ]         |
| Pipeline Stage: [Prospecting v]  (default)     |
| Assigned Rep:   [Sarah Miller v] (pre-filled)  |
|                                                 |
| Account: Nova Entertainment  (auto-linked)      |
| Contact: Jack Chen           (auto-linked)      |
| Source:  Referral             (from lead)        |
|                                                 |
| [Cancel]                    [Create Deal]       |
+-------------------------------------------------+
```

**Side effects:**
1. New Item vault (level 4) created under the Counterparty vault at `pipeline_stage: 'prospecting'`
2. `lifecycle_stage` set to `'opportunity'`
3. Lead record archived: `is_archived: true`, removed from active lead views but history preserved
4. Deal linked to Contact vault (level 3) and Account vault (level 1)
5. Account auto-created at level 1 if it does not already exist
6. Activity event logged: `lead.converted` with `{deal_vault_id, deal_value, pipeline_stage}`
7. Task batch auto-created for Prospecting stage (see Deal Pipeline section below)
8. Cross-module event: `crm.deal.created` -- available to Contracts, Tasks, Workflow Engine
9. Converted card appears in Qualify "Converted" column (grayed out, shows "-> Pipeline: Prospecting")
10. Deal card appears in Pipeline Kanban at Prospecting column

**Where user ends up:** Pipeline Kanban view with the new deal card highlighted. Optional: system prompts "View deal in Pipeline?" with a navigation link.

**Irreversibility:** Once converted, the deal exists as its own vault. The lead record is archived. To "undo," the deal vault must be manually deleted by an Owner+ and the lead manually un-archived -- there is no one-click revert.

---

#### T5: Any Active --> Nurture

**What causes it:**
- Rep clicks "Move to Nurture" or "Reject" on any lead card (New, MQL, SAL, or SQL).
- Reason required: "Not ready to buy", "No budget this quarter", "Needs education", "Bad timing", "Other: [free text]".

**Side effects:**
1. `lifecycle_stage` set to `'nurture'`
2. `nurture_reason` recorded on vault metadata
3. `qualification_score` preserved (not reset)
4. Activity event logged: `lead.nurtured` with `{from_stage, reason, actor_id}`
5. Workflow Engine `nurture.sequence.start` event fired -- triggers automated nurture email sequence
6. All pending tasks for this lead cancelled (marked `cancelled`, not deleted)
7. Lead removed from Qualify funnel, visible only via "Nurture" filter on New Leads screen
8. Notification to manager: "[Lead name] moved to Nurture by [rep] -- reason: [reason]"

**Where user ends up:** Lead disappears from Qualify funnel. If viewing New Leads with "Nurture" filter active, lead appears with Nurture badge.

**Re-engagement path:** See Section 8 (Nurture Queue) for the full re-engagement lifecycle.

---

#### T6: Any Active --> Disqualified

**What causes it:**
- Rep clicks "Disqualify" on any lead card.
- Reason required (select one): Bad data, Wrong person, Spam, Duplicate, Competitor, Do not contact.

**Side effects:**
1. `lifecycle_stage` set to `'disqualified'`
2. `disqualified_reason` recorded on vault metadata
3. `disqualified_at` timestamp set
4. `disqualified_by` set to actor's `user_id`
5. Record soft-deleted: `is_active: false` -- not shown in any active CRM views
6. All pending tasks cancelled
7. Activity event logged: `lead.disqualified` with `{reason, actor_id}`
8. Audit trail preserved: full history of the lead's lifecycle remains queryable
9. If reason is "Duplicate," system prompts: "Merge with existing record?" and offers entity resolution merge flow

**Where user ends up:** Lead removed from all active views. Confirmation toast: "Lead disqualified. [Undo within 30 seconds]" -- undo reverts to previous stage.

---

## 2. Deal Pipeline State Machine

### State Diagram

```
+-------------+     +-----------+     +----------+     +-------------+     +-------+
| Prospecting +---->+ Discovery +---->+ Proposal +---->+ Negotiation +---->+ Close |
+------+------+     +-----+-----+     +----+-----+     +------+------+     +---+---+
       |                   |                |                  |                 |
       v                   v                v                  v                 v
   +------+            +------+         +------+           +------+          +-----+
   | Lost |            | Lost |         | Lost |           | Lost |          | Won |
   +------+            +------+         +------+           +------+          +-----+

  Any stage -----> [On Hold]  (reversible pause)
```

### Pipeline Stage Configuration

| Stage | Probability | Color Token | Chamber Mapping | Avg Days (Target) |
|-------|------------|-------------|-----------------|-------------------|
| Prospecting | 10% | `--stage-prospecting` (#4dd0e1) | Build | 7 |
| Discovery | 25% | `--stage-discovery` (#29b6f6) | Build | 14 |
| Proposal | 50% | `--stage-proposal` (#7e57c2) | Build | 10 |
| Negotiation | 75% | `--stage-negotiation` (#ffa726) | Review | 14 |
| Close | 90% | `--stage-close` (#66bb6a) | Review | 7 |
| Won | 100% | green | Ship | -- (terminal) |
| Lost | 0% | gray | archived | -- (terminal) |
| On Hold | preserved | amber | preserved | -- (paused) |

### Stage Transition Details

#### Prospecting --> Discovery

**Gate requirements (ALL must be true):**
1. Task "Research account background" marked complete
2. Task "Schedule discovery call" marked complete (or discovery call event logged)

**Trigger:**
- Rep drags deal card from Prospecting to Discovery column in Pipeline Kanban
- OR rep clicks "Advance Stage" in deal detail panel
- If gate requirements not met: modal appears listing incomplete tasks with a "Complete these first" prompt. Manager can override with reason.

**Side effects:**
1. `pipeline_stage` set to `'discovery'`
2. `probability` updated from 10% to 25%
3. `stage_entered_at` timestamp reset (for days-in-stage tracking)
4. Activity event logged: `deal.stage_changed` with `{from: 'prospecting', to: 'discovery', gate_tasks_completed: [list]}`
5. Auto-created tasks for Discovery stage:
   - "Complete discovery questionnaire" (due: +5 business days)
   - "Identify decision makers" (due: +7 business days)
   - "Document buyer pain points" (due: +7 business days)
6. Stage-entry event recorded for pipeline velocity analytics

**Who:** Assigned rep or manager. Gatekeeper+ can also advance if they have CRM Builder+ module role.

**Backward movement (Discovery --> Prospecting):**
- Allowed with reason. All Discovery tasks cancelled. Stage timer resets.

---

#### Discovery --> Proposal

**Gate requirements (ALL must be true):**
1. Task "Complete discovery questionnaire" marked complete
2. Task "Identify decision makers" marked complete

**Trigger:** Drag card or "Advance Stage" button.

**Side effects:**
1. `pipeline_stage` set to `'proposal'`
2. `probability` updated from 25% to 50%
3. `stage_entered_at` reset
4. Activity event logged: `deal.stage_changed`
5. Auto-created tasks for Proposal stage:
   - "Draft proposal" (due: +3 business days)
   - "Get pricing approval" (due: +5 business days)
   - "Send proposal to buyer" (due: +7 business days)
   - "Schedule proposal review call" (due: +10 business days)
6. If deal value > $100K: manager notification via Novu -- "High-value deal entering Proposal: [name] ($[value])"
7. Forecast recalculation triggered for pipeline dashboard

**Who:** Assigned rep or manager.

---

#### Proposal --> Negotiation

**Gate requirements (ALL must be true):**
1. Task "Draft proposal" marked complete
2. Task "Get pricing approval" marked complete
3. Task "Send proposal" marked complete

**Trigger:** Drag card or "Advance Stage" button.

**Side effects:**
1. `pipeline_stage` set to `'negotiation'`
2. `probability` updated from 50% to 75%
3. `stage_entered_at` reset
4. Activity event logged: `deal.stage_changed`
5. Auto-created tasks for Negotiation stage:
   - "Address buyer objections" (due: +5 business days)
   - "Finalize pricing and terms" (due: +7 business days)
   - "Prepare contract draft" (due: +10 business days)
6. Deal Review entry auto-created (see Section 3)
7. SLA timer starts: 4-hour response time for Deal Review
8. Legal review task created if deal value > $50K
9. Cross-module event: `crm.deal.negotiation_entered` -- triggers Contracts module contract generation preparation

**Who:** Assigned rep or manager.

---

#### Negotiation --> Close

**Gate requirements (ALL must be true):**
1. All Negotiation stage tasks marked complete
2. No unresolved blockers (blocker-severity tasks must be resolved)
3. Deal Review approved (if review was triggered -- see Section 3)

**Trigger:** Drag card or "Advance Stage" button.

**Side effects:**
1. `pipeline_stage` set to `'close'`
2. `probability` updated from 75% to 90%
3. `stage_entered_at` reset
4. Activity event logged: `deal.stage_changed`
5. Auto-created tasks for Close stage:
   - "Generate final contract" (due: +2 business days)
   - "Obtain buyer signature" (due: +5 business days)
   - "Obtain internal sign-off" (due: +5 business days)
6. Contract generation prompt shown: "Ready to generate contract? [Open Contract Generator]"
7. Final approval workflow triggered if deal value > $200K (Owner approval required)
8. Cross-module event: `crm.deal.close_entered` -- Contracts module alerted

**Who:** Assigned rep or manager. If Deal Review was required, advancement blocked until review is Approved.

---

#### Close --> Won

**Gate requirements (ALL must be true):**
1. Task "Generate final contract" complete (contract vault exists in Contracts module)
2. Task "Obtain buyer signature" complete
3. Task "Obtain internal sign-off" complete (self-approval prevention: deal owner cannot be the internal signer)

**Trigger:** Rep clicks "Mark as Won" button (prominent green button in deal detail panel). Confirmation modal: "Confirm this deal as Won? This will trigger onboarding workflows."

**Side effects:**
1. `pipeline_stage` set to `'won'`
2. `lifecycle_status` set to `'won'`
3. `probability` set to 100%
4. `closed_at` timestamp set
5. `closed_by` set to actor's `user_id`
6. Activity event logged: `deal.won` with `{deal_value, days_in_pipeline, actor_id}`
7. Celebration event: confetti animation on Pipeline board, "Deal Won!" toast notification
8. Cross-module events fired:
   - `crm.deal.won` -- Contracts module vault advances to Ship chamber
   - `crm.onboarding.trigger` -- Onboarding checklist creation (see Section 5)
   - `crm.health_monitor.initialize` -- Health monitoring begins for this customer
9. Renewal date computed: `closed_at + term_length` (from contract extraction) and stored
10. All remaining Close stage tasks auto-resolved as "N/A -- deal won"
11. Won deal appears in "Won Deals" view under DEALS section
12. Customer appears in CUSTOMERS section views (Onboarding, Health Monitor, Renewals)
13. Notification to sales manager + executive stakeholders: "Deal Won: [name] -- $[value]"

**Who:** Assigned rep. Manager can also trigger. Self-approval prevention: the rep who marks Won cannot also be the internal sign-off approver.

---

#### Any Stage --> Lost

**Trigger:** Rep clicks "Mark as Lost" button on deal card or detail panel. Loss reason is required.

**Loss reason categories:**
- Price/budget (competitor was cheaper, budget cut, timing)
- Product fit (missing feature, wrong use case, integration gap)
- Decision maker (champion left, reorg, no executive sponsor)
- Competition (chose competitor, incumbent renewed)
- No decision (went dark, indefinite hold, deprioritized)
- Other (free text)

**Side effects:**
1. `pipeline_stage` set to `'lost'`
2. `lifecycle_status` set to `'lost'`
3. `probability` set to 0%
4. `closed_at` timestamp set
5. `loss_reason` and `loss_reason_detail` recorded
6. All pending tasks cancelled (marked `cancelled`)
7. Activity event logged: `deal.lost` with `{loss_reason, stage_at_loss, days_in_pipeline, deal_value}`
8. Deal removed from Pipeline Kanban (does not appear in active columns)
9. Deal visible in Pipeline Table view with "Lost" filter active
10. Notification to manager: "Deal Lost: [name] ($[value]) -- [reason]"
11. Loss analytics updated: loss-by-stage, loss-by-reason, loss-by-rep dashboards
12. If at Negotiation or Close stage: post-mortem task auto-created for rep + manager

**Where user ends up:** Deal disappears from Pipeline Kanban. Toast: "Deal marked as Lost. [Undo within 60 seconds]"

**Reopening a Lost deal:**
- Manager clicks "Reopen Deal" on the lost deal record (accessible from Table view with Lost filter)
- Deal returns to the stage it was at when lost
- All previously cancelled tasks remain cancelled; new stage tasks are not auto-created
- Activity event logged: `deal.reopened` with `{reason, reopened_by}`
- Only managers (Lead/Director org role) or CRM Owner module role can reopen

---

#### Any Stage --> On Hold

**Trigger:** Rep clicks "Put on Hold" with reason (buyer requested pause, internal restructuring, seasonal, awaiting budget cycle).

**Side effects:**
1. `pipeline_stage` preserved (stored in `stage_before_hold`)
2. `is_on_hold` set to `true`
3. `hold_reason` recorded
4. SLA timers paused
5. Days-in-stage counter paused
6. Deal card shows amber "ON HOLD" badge, moves to a collapsed "On Hold" section at the bottom of Pipeline Kanban
7. All pending task due dates suspended (timers freeze)
8. Activity event logged: `deal.on_hold`
9. Auto-task created: "Review hold status" (due: +14 days, recurring weekly)

**Resuming from Hold:**
- Rep clicks "Resume Deal" -- deal returns to `stage_before_hold`
- All timers resume from where they paused
- Activity event logged: `deal.resumed`

---

## 3. Deal Review Flow

### When Deal Review Is Triggered

| Condition | Trigger Point | Review Level |
|-----------|--------------|--------------|
| Deal enters Negotiation stage | Automatic | Standard review (1 reviewer) |
| Deal value > $200K at any stage change | Automatic | Manager approval required |
| Deal value > $500K | Automatic | Manager + executive approval (multi-approver) |
| Rep manually requests review | "Request Review" button | Standard review |
| Deal has blocker-severity task | Automatic when blocker task created | Assigned to blocker resolver |

### Deal Review State Machine

```
                    +----------------+
                    | Pending Review |
                    +-------+--------+
                            |
              +-------------+-------------+
              |             |             |
              v             v             v
        +---------+  +------------+  +----------+
        | Approved|  | Changes    |  | Rejected |
        |         |  | Requested  |  |          |
        +---------+  +-----+------+  +----------+
                            |
                            v
                   (Rep makes changes)
                            |
                            v
                    +----------------+
                    | Re-submitted   |
                    | (Pending again)|
                    +----------------+
```

### Review Details

**Pending Review:**
- Created automatically when deal enters Negotiation (or when value threshold exceeded)
- Appears in DEALS > Deal Review triage table
- Assigned to: module Gatekeeper, or manager if Gatekeeper not configured
- Self-approval prevention: deal owner (the rep) CANNOT be the reviewer. If the only available Gatekeeper is the deal owner, review escalates to manager's manager.
- SLA: 4-hour response time during business hours
- Notification via Novu: in-app badge on "Deal Review" nav item + push notification to reviewer

**SLA Escalation:**
- 4 hours: initial notification to assigned reviewer
- 8 hours: reminder notification + "OVERDUE" badge on review item
- 24 hours: escalation to reviewer's manager + "ESCALATED" badge
- 48 hours: escalation to module Owner + red "CRITICAL" badge

**Approved:**
- Reviewer clicks "Approve" in the Deal Review detail view
- Side effects: deal unblocked for next stage advancement, activity event `deal.review.approved`, reviewer's name recorded on approval chain
- Reviewer can add approval notes visible on the deal timeline

**Changes Requested:**
- Reviewer clicks "Request Changes" with specific change items
- Side effects: deal blocked from advancement, notification to rep, task created per change item
- Rep addresses changes, then clicks "Re-submit for Review"
- Review returns to Pending Review state with "Re-submitted" flag
- Reviewer sees diff of what changed since last review

**Rejected:**
- Reviewer clicks "Reject" with detailed reason
- Side effects: deal blocked, notification to rep + manager, activity event logged
- Rep can appeal: "Request Re-review" which creates a new review assigned to a different reviewer
- Two rejections from different reviewers = deal auto-moved to Lost with reason "Failed review"

### High-Value Deal Escalation

```
Deal value > $200K:
  Standard review + Manager must approve
  Manager approval is a SEPARATE approval step (not the same as Gatekeeper review)

Deal value > $500K:
  Standard review + Manager + Executive approval
  Three-step chain: Gatekeeper approve -> Manager approve -> Executive approve
  All three must approve before deal can advance

Each step has independent SLA timers
Self-approval prevention applies at every step
```

---

## 4. Health Score Lifecycle

### Health Score Calculation

Health scores are computed per Account (level 1), per Counterparty (level 3), and per Item vault (level 4). Parent scores aggregate from children using a weighted average.

#### Input Weights

| Factor | Weight | Measurement | Score Range |
|--------|--------|-------------|-------------|
| Contact responsiveness | 30% | Avg response time to outbound messages; days since last inbound communication | 0-100 |
| Deal progress | 25% | Percentage of stage tasks complete; days in current stage vs. average for that stage | 0-100 |
| Task completion | 20% | Overdue task count (penalty: -5 per overdue); overall completion rate | 0-100 |
| Interaction recency | 15% | Days since last meaningful interaction (not automated emails/system messages) | 0-100 |
| Renewal proximity | 10% | Urgency multiplier that increases as renewal date approaches | 0-100 |

#### Factor Scoring Details

**Contact Responsiveness (30%):**
```
score = 100

-- Penalize slow responses
IF avg_response_time > 48h THEN score -= 30
ELIF avg_response_time > 24h THEN score -= 15
ELIF avg_response_time > 8h THEN score -= 5

-- Penalize stale relationships
IF days_since_last_inbound > 30 THEN score -= 40
ELIF days_since_last_inbound > 14 THEN score -= 20
ELIF days_since_last_inbound > 7 THEN score -= 10
```

**Deal Progress (25%):**
```
score = 100

-- Task completion within stage
task_completion_pct = completed_stage_tasks / total_stage_tasks
score = task_completion_pct * 60  -- up to 60 points from task completion

-- Days in stage vs average
IF days_in_stage <= avg_days_for_stage THEN score += 40
ELIF days_in_stage <= avg_days * 1.5 THEN score += 25
ELIF days_in_stage <= avg_days * 2 THEN score += 10
ELSE score += 0  -- stalled deal, no bonus
```

**Task Completion (20%):**
```
score = 100
score -= (overdue_task_count * 5)  -- -5 per overdue task
score *= completion_rate            -- multiply by overall completion %
score = max(0, score)
```

**Interaction Recency (15%):**
```
IF days_since_meaningful_interaction <= 3 THEN score = 100
ELIF days <= 7 THEN score = 85
ELIF days <= 14 THEN score = 60
ELIF days <= 30 THEN score = 30
ELSE score = 10
```
"Meaningful interaction" = manual communication (call, text, email reply). Automated emails, system notifications, and bot messages do NOT count.

**Renewal Proximity (10%):**
```
IF no_renewal_date THEN score = 80  -- neutral
ELIF days_until_renewal > 90 THEN score = 100
ELIF days_until_renewal > 30 THEN score = 70
ELIF days_until_renewal > 7 THEN score = 40
ELSE score = 10  -- urgent
```

#### Final Score Computation

```
health_score = (
    contact_responsiveness * 0.30 +
    deal_progress * 0.25 +
    task_completion * 0.20 +
    interaction_recency * 0.15 +
    renewal_proximity * 0.10
)

-- Round to integer, clamp to 0-100
health_score = clamp(round(health_score), 0, 100)
```

### Health Score Recalculation Triggers

| Trigger Event | Source | Recalculation Scope |
|--------------|--------|-------------------|
| Communication received (inbound) | Smart Line, email, web form | Vault + parent chain |
| Communication sent (outbound) | CRM Inbox reply | Vault + parent chain |
| Task completed | Tasks module | Vault + parent chain |
| Task becomes overdue | Tasks module (midnight batch) | Vault + parent chain |
| Deal stage changed | Pipeline | Vault + parent chain |
| Manual health check | Health Monitor "Recalculate" button | Single vault + parent chain |
| Daily batch recalculation | Cron job (midnight UTC) | ALL active vaults |

"Parent chain" means: when an Item vault (L4) health changes, its Counterparty (L3) and Account (L1) aggregates are recomputed.

### Health Status Thresholds and Alerts

| Condition | Status | Badge Color | Alert |
|-----------|--------|-------------|-------|
| Score >= 70 | Healthy | Green | None |
| Score 30-69 | Watch | Amber | Manager notified, appears in Health Monitor "Declining" filter |
| Score < 30 | At Risk | Red | VP/Director notified, recommended actions generated, appears in Health Monitor "At Risk" filter |
| Score drops 15+ points in 7 days | Rapid Decline | Red (pulsing) | Immediate alert to rep + manager regardless of absolute score |
| Score rises above 70 from Watch/At Risk | Recovery | Green (with arrow-up icon) | Auto-resolves alert, activity event logged |

### Alert Side Effects

**Watch (score drops below 70):**
1. `health_status` set to `'watch'`
2. Activity event: `health.status_changed` with `{from: 'healthy', to: 'watch', score, delta}`
3. Notification to assigned rep: "Customer health declining: [name] (score: [score])"
4. Notification to manager: "[name] health dropped to Watch"
5. Customer highlighted amber in Health Monitor
6. Recommended actions generated (AI-driven from health factor analysis):
   - "Send check-in message to [primary contact]"
   - "Follow up on overdue task: [task name]"
   - "Schedule review call"

**At Risk (score drops below 30):**
1. `health_status` set to `'at_risk'`
2. Activity event: `health.critical` with score breakdown
3. Notification to rep + manager + VP (Novu: in-app + email + push)
4. Task auto-created: "Urgent: Address at-risk customer [name]" (blocker severity, due: +1 business day)
5. Customer highlighted red in Health Monitor, sorted to top
6. Expanded recommended actions with severity indicators

**Rapid Decline (15+ point drop in 7 days):**
1. Activity event: `health.rapid_decline` with `{score, previous_score, delta, period: '7d'}`
2. Immediate notification to rep + manager (push notification, not just in-app)
3. "Rapid Decline" badge on customer card in Health Monitor
4. Automatic task: "Investigate rapid health decline for [name]" (blocker severity)

**Recovery (score rises above 70 from Watch/At Risk):**
1. `health_status` set to `'healthy'`
2. Activity event: `health.recovered` with `{from_status, score}`
3. Alert auto-resolved, removed from Health Monitor risk views
4. Notification to rep: "[name] health recovered to Healthy (score: [score])"

---

## 5. Onboarding Lifecycle

### Trigger

When a deal is marked Won (Close --> Won transition), the `crm.onboarding.trigger` cross-module event fires.

### Onboarding State Machine

```
+------------------+     +--------------+     +-------------+
| Not Started      +---->+ In Progress  +---->+ Completed   |
| (checklist       |     | (items being |     | (all items  |
|  created)        |     |  worked)     |     |  done)      |
+------------------+     +------+-------+     +-------------+
                                |
                                v
                         +-----------+
                         | Stalled   |
                         | (overdue  |
                         |  items)   |
                         +-----------+
```

### Default Onboarding Checklist

When `crm.onboarding.trigger` fires, the Workflow Engine creates the following tasks:

| # | Task | Assigned To | Due | Auto/Manual |
|---|------|------------|-----|-------------|
| 1 | Send welcome email | System (auto) | Immediate | Auto -- Workflow Engine sends templated email via Novu |
| 2 | Account setup call | Assigned rep | `won_date + 3 business days` | Manual -- rep schedules and completes |
| 3 | Document upload training | Assigned trainer (or rep) | `won_date + 5 business days` | Manual -- walk customer through doc upload |
| 4 | Workflow configuration | Assigned engineer (or rep) | `won_date + 7 business days` | Manual -- configure customer-specific workflows |
| 5 | Go-live review call | Assigned rep | `won_date + 14 business days` | Manual -- final check before full activation |

**Customization:** Workspace Owners can modify the default onboarding checklist template in Admin > Workflows > Onboarding Template. Custom checklists can be created per deal type or customer segment.

### Onboarding View Behavior

- Customer appears in CUSTOMERS > Onboarding with a progress bar showing completion percentage
- Each checklist item shows: status (done/pending/overdue), assigned person, due date
- Overdue items highlighted red, trigger escalation notifications
- Progress bar color: green (on track), amber (1+ items overdue < 3 days), red (items overdue > 3 days)

### Onboarding Completion

When ALL checklist items are marked complete:
1. `onboarding_status` set to `'completed'`
2. `onboarding_completed_at` timestamp set
3. Customer `lifecycle_status` changes from `'onboarding'` to `'active'`
4. Activity event: `onboarding.completed` with `{duration_days, on_time: boolean}`
5. Customer removed from Onboarding view, appears as "Active" in Health Monitor
6. Health monitoring begins (daily recalculation includes this customer)
7. Notification to rep + manager: "Onboarding complete for [name]"
8. If renewal date is set: customer appears in Renewals view

### Stalled Onboarding

If any onboarding item is overdue by 3+ business days:
1. `onboarding_status` set to `'stalled'`
2. Escalation notification to manager: "Onboarding stalled for [name] -- [N] items overdue"
3. Daily reminder to assigned persons for overdue items
4. After 14 days stalled: escalation to VP/Director

---

## 6. Renewal Lifecycle

### Renewal Date Sources

Renewal dates are computed from contract data, not manually entered (though manual override is available):

| Source | Computation | Priority |
|--------|------------|----------|
| Contract extraction | `effective_date + term_length` from extraction pipeline | Highest |
| Manual entry | Rep sets renewal date on deal or account detail | Medium |
| Default | `closed_at + 365 days` if no contract term found | Lowest |

### Renewal State Machine

```
                   90 days before
+----------+      renewal date        +-----------+
| Inactive |  ------------------->    | On Track  |
| (>90 days|                          | (green)   |
|  out)    |                          +-----+-----+
+----------+                                |
                                            | 30 days before
                                            v
                                      +-----------+
                                      | Due Soon  |
                                      | (amber)   |
                                      +-----+-----+
                                            |
                                            | 7 days before
                                            v
                                      +-----------+
                                      | Urgent    |
                                      | (red)     |
                                      +-----+-----+
                                            |
                                            | past due date
                                            v
                                      +-----------+
                                      | OVERDUE   |
                                      | (red,     |
                                      |  pulsing) |
                                      +-----+-----+
                                            |
                              +-------------+-------------+
                              |                           |
                              v                           v
                       +-----------+              +-------------+
                       | Renewed   |              | Churned     |
                       | (new deal |              | (customer   |
                       |  or ext.) |              |  lost)      |
                       +-----------+              +-------------+
```

### Renewal Transitions

**Inactive --> On Track (90 days before renewal):**
- Trigger: Daily batch job checks all active customers' renewal dates
- Side effects: Customer appears in Renewals view (green "On Track" badge), renewal prep task created for rep
- Notification to assigned rep: "Renewal approaching for [name] -- [date]"

**On Track --> Due Soon (30 days before renewal):**
- Trigger: Daily batch job
- Side effects: Badge changes to amber "Due Soon", notification to rep + manager
- Task created: "Initiate renewal conversation with [name]" (due: immediately)

**Due Soon --> Urgent (7 days before renewal):**
- Trigger: Daily batch job
- Side effects: Badge changes to red "Urgent", daily reminders to rep
- Escalation notification to manager if no renewal activity logged

**Urgent --> OVERDUE (past due date):**
- Trigger: Daily batch job
- Side effects: Badge changes to red pulsing "OVERDUE", notification to rep + manager + VP
- Task created: "URGENT: Renewal overdue for [name]" (blocker severity)
- Health score penalty applied (renewal proximity factor drops to 0)

**Any --> Renewed:**
- Trigger: Rep clicks "Mark as Renewed" or new deal created with type "Renewal"
- Side effects: New deal vault created (or existing deal extended), renewal counter resets, next renewal date computed, activity event logged, notification to team
- Customer stays in Health Monitor as Active

**Any --> Churned:**
- Trigger: Rep clicks "Mark as Churned" with churn reason
- Side effects: `lifecycle_status` set to `'churned'`, customer removed from active views, all tasks cancelled, health monitoring stopped, activity event logged, churn analytics updated

### Auto-Renew Contracts

For contracts flagged as auto-renewing (from extraction: `auto_renew: true`):
- Customer still appears in Renewals view with "Auto-Renew" badge
- No action required from rep unless customer objects
- 30 days before: notification to rep: "Auto-renewal for [name] on [date] -- confirm no objections"
- If customer objects: rep clicks "Cancel Auto-Renewal" -- deal enters normal renewal pipeline
- If no objection: auto-renewal processed on date, new contract period logged, renewal counter resets

---

## 7. Cross-Module Event Triggers

### Events FROM Other Modules --> CRM

| Source Module | Event | CRM Action | Vault Level Affected |
|---|---|---|---|
| Contracts | `entity.new_customer` | Create Account vault (L1) + Counterparty vault (L3). Set `lifecycle_stage: 'new'`. Appears in New Leads. | L1, L3 |
| Contracts | `entity.resolved` | Link Item vault (L4) to existing Counterparty (L3). Update deal count on Account. | L3, L4 |
| Contracts | `gate.advanced(review)` | Flag linked deal for Deal Review. Create review entry if value > threshold. | L4 |
| Contracts | `gate.advanced(ship)` | Update Account + Counterparty health aggregates. If deal won: trigger onboarding. | L1, L3, L4 |
| Contracts | `health.updated` | Recalculate Account + Counterparty health scores (aggregate from children). | L1, L3 |
| Contracts | `contract.signed` | Mark Close stage "Obtain buyer signature" task as complete. | L4 |
| Tasks | `task.completed` | Update deal task progress percentage. Recalculate health score (task completion factor). Check if stage gate requirements now met. | L4 |
| Tasks | `task.overdue` | Health score penalty (-5 per overdue task). Notification to deal owner. Flag in Health Monitor if repeated. | L4 |
| Tasks | `task.created` | Update task count on deal card in Pipeline Kanban. | L4 |
| Workflow Engine | `lead.qualified` | Move lead to appropriate stage in Qualify funnel. Update `lifecycle_stage`. | L3 |
| Workflow Engine | `lead.scoring.complete` | Update `qualification_score` on vault. Check if score crossed MQL/SQL threshold. | L3 |
| Workflow Engine | `onboarding.triggered` | Create onboarding checklist in CUSTOMERS > Onboarding view. | L1 |
| Workflow Engine | `nurture.sequence.complete` | Log completion. If lead re-engagement detected, fire `lead.re_engaged`. | L3 |
| Smart Line | `message.received` | Create/update Activity event in Activity Feed. Update contact `last_interaction` timestamp. Recalculate health (interaction recency factor). | L3 |
| Smart Line | `message.sent` | Log outbound in Activity Feed. Update contact interaction count. | L3 |
| Smart Line | `lead.intake.completed` | Create lead in New Leads with `qualification_score` + `source: 'smart_line'`. Trigger lead scoring workflow. | L3 |
| Smart Line | `contact.identified` | Update vault_contacts with name, role. Enrich org tree for Account. | L3 |
| Calendar | `meeting.completed` | Log activity event in Activity Feed. Update contact interaction count. Recalculate health (interaction recency). | L3 |
| Calendar | `meeting.no_show` | Log activity event. Health score penalty. Task created: "Follow up on no-show." | L3 |
| Notifications | `alert.escalated` | Flag in Health Monitor recommended actions panel. Create escalation task. | L1, L3 |

### Events FROM CRM --> Other Modules

| CRM Event | Target Module | Action |
|---|---|---|
| `crm.deal.created` | Contracts | Prepare vault for contract generation (if applicable) |
| `crm.deal.created` | Tasks | Create initial stage task batch |
| `crm.deal.won` | Contracts | Advance contract vault to Ship chamber |
| `crm.deal.won` | Workflow Engine | Trigger onboarding workflow |
| `crm.deal.won` | Notifications | Send celebration notifications to team |
| `crm.deal.lost` | Tasks | Cancel all pending deal-related tasks |
| `crm.deal.lost` | Notifications | Send loss notification to manager |
| `crm.deal.stage_changed` | Tasks | Create auto-tasks for new stage, cancel incomplete tasks from previous stage |
| `crm.deal.stage_changed` | Notifications | Notify relevant stakeholders |
| `crm.deal.negotiation_entered` | Contracts | Alert legal review queue, prepare contract generation |
| `crm.lead.converted` | Tasks | Create Prospecting stage task batch |
| `crm.health.critical` | Notifications | Send escalation alerts to manager + VP |
| `crm.health.critical` | Tasks | Create urgent remediation task |
| `crm.renewal.overdue` | Notifications | Escalation chain: rep -> manager -> VP |
| `crm.onboarding.completed` | Notifications | Send completion notification to team |
| `crm.account.created` | Smart Line | Provision dedicated line if auto-provision enabled |

### Event Schema

All cross-module events follow the standard Airlock event format:

```json
{
  "event_type": "crm.deal.stage_changed",
  "workspace_id": "uuid",
  "vault_id": "uuid",
  "vault_level": 4,
  "actor_id": "uuid",
  "actor_type": "user",
  "timestamp": "2026-03-05T14:30:00Z",
  "payload": {
    "from_stage": "proposal",
    "to_stage": "negotiation",
    "deal_value": 120000,
    "gate_tasks_completed": ["draft_proposal", "get_pricing_approval", "send_proposal"]
  },
  "metadata": {
    "module": "crm",
    "chamber": "review",
    "correlation_id": "uuid"
  }
}
```

Events are published to BullMQ via the Event Bus. Subscribers (other modules) consume events and process them independently. Events are immutable and append-only (Security principle #5).

---

## 8. Nurture Queue

### Nurture Sequence Lifecycle

```
Lead moved to Nurture
        |
        v
+------------------+
| Sequence Active  |  (5 emails over 30 days)
+--------+---------+
         |
         +-- Email 1: Day 1 (welcome / value prop)
         +-- Email 2: Day 5 (case study)
         +-- Email 3: Day 12 (resource / guide)
         +-- Email 4: Day 20 (check-in)
         +-- Email 5: Day 30 (last attempt)
         |
         v
+-------------------+
| Monthly Check-In  |  (1 email per month, indefinite)
+--------+----------+
         |
         +-- IF lead re-engages --> Re-score
         |        |
         |        +-- Score >= MQL threshold (40)?
         |        |        |
         |        |        YES --> Auto-move to New Leads
         |        |                with "Re-engaged" tag
         |        |
         |        NO --> Stay in Nurture
         |
         +-- IF 90 days no engagement --> Auto-archive
                  |
                  v
           +-------------+
           | Archived     |
           | (removed from|
           |  active views)|
           +-------------+
```

### Nurture Visibility

- Nurture leads are NOT a separate view. They appear on the New Leads screen when the "Nurture" filter is toggled on.
- Filter options on New Leads: `All | Unworked | Mine | Nurture`
- Nurture leads show a "Nurture" badge (amber) and nurture sequence status ("Email 3/5 sent", "Monthly check-in")

### Re-Engagement Detection

A nurtured lead is considered "re-engaged" when ANY of these occur:
1. Lead replies to a nurture email
2. Lead visits a tracked web page (webhook from analytics)
3. Lead fills out a web form
4. Lead sends inbound text to Smart Line
5. Lead is mentioned in a new contract upload (entity resolution match)

**Re-engagement flow:**
1. Re-engagement event detected
2. `qualification_score` recalculated with fresh signals
3. If score >= 40 (MQL threshold):
   - `lifecycle_stage` reverts to `'new'`
   - `re_engaged: true` flag set
   - `re_engaged_at` timestamp set
   - Lead appears in New Leads with green "Re-engaged" badge
   - Notification to original assigned rep (or pool): "Lead re-engaged: [name] (score: [score])"
   - Nurture sequence halted
4. If score < 40:
   - Lead stays in Nurture
   - Activity event logged: `lead.re_engagement_attempt` with `{score, below_threshold: true}`
   - Nurture sequence continues

### Auto-Archive

After 90 consecutive days with no engagement (no email opens, no replies, no inbound activity):
1. `lifecycle_stage` set to `'archived'`
2. Lead removed from all active views (including Nurture filter)
3. Activity event logged: `lead.auto_archived` with `{days_inactive: 90}`
4. Nurture sequence stopped
5. Lead still queryable via Admin search or analytics (soft-archive, not deletion)

---

## 9. Screen-to-Screen Navigation Triggers

### Navigation Map

| # | From Screen | User Action | Destination Screen | Context Passed | Deep Link Format |
|---|------------|-------------|-------------------|----------------|-----------------|
| N1 | New Leads | Click lead row | Lead Detail slide-over | `vault_id`, shows contact info + communication history + source details | `#crm/leads/{vault_id}` |
| N2 | New Leads | Click "Convert to Deal" | Deal Creation Modal -> Pipeline (Prospecting) | Lead data pre-fills deal fields (name, account, contact, source, score) | `#crm/pipeline?new_deal={vault_id}` |
| N3 | Qualify | Drag card to Converted column | Pipeline (Prospecting) | Lead + score + contacts carry over; deal creation modal opens | `#crm/pipeline?new_deal={vault_id}` |
| N4 | Qualify | Click lead card | Lead Detail slide-over | `vault_id`, same as N1 but within Qualify context | `#crm/qualify/{vault_id}` |
| N5 | Pipeline | Click deal card | Deal Detail slide-over | `vault_id`, shows tasks + activity + contacts + stage progress | `#crm/pipeline/{vault_id}` |
| N6 | Pipeline | Click "View Contract" in deal detail | Contracts module triptych | `vault_id` of linked contract vault, opens in Orchestrate panel | `#contracts/{vault_id}` |
| N7 | Pipeline | Click account name on deal card | Accounts view with account selected | `account_vault_id`, highlights in hierarchy tree | `#crm/accounts/{vault_id}` |
| N8 | Deal Review | Click deal row | Deal Review detail (triptych-style) | `vault_id`, shows approval chain + evidence + timeline | `#crm/review/{vault_id}` |
| N9 | Deal Review | Click "Approve" | Pipeline (deal advances to next stage) | Deal unblocked, review resolved, pipeline updates | `#crm/pipeline/{vault_id}` |
| N10 | Health Monitor | Click customer card | Health Detail panel (right side) | `account_vault_id`, shows risk factors + recommended actions + score breakdown | `#crm/health/{vault_id}` |
| N11 | Health Monitor | Click "View Account" | Accounts view with account selected | `account_vault_id`, tree expanded to show customer hierarchy | `#crm/accounts/{vault_id}` |
| N12 | Health Monitor | Click "View Deal" in detail | Pipeline with deal highlighted | `deal_vault_id`, scrolls to deal card in Pipeline Kanban | `#crm/pipeline/{vault_id}` |
| N13 | Renewals | Click renewal row | Renewal Detail slide-over | `vault_id`, shows contract details + renewal history + action buttons | `#crm/renewals/{vault_id}` |
| N14 | Renewals | Click "View Contract" | Contracts module triptych | Item vault opened in Contracts module | `#contracts/{vault_id}` |
| N15 | Activity Feed | Click event row | Source view with context | Routes to originating screen: deal event -> Pipeline, message event -> Inbox, task event -> Tasks | varies by event type |
| N16 | Accounts | Click Account row in tree | Account Detail (right panel) | `vault_id`, shows hierarchy + contacts + deals + health + Smart Line | `#crm/accounts/{vault_id}` |
| N17 | Accounts | Click Item vault row in account detail | Contracts module triptych | Opens vault triptych in Contracts module | `#contracts/{vault_id}` |
| N18 | Accounts | Click Counterparty row | Counterparty Detail (right panel) | `vault_id`, shows deals + interactions + entity resolution confidence | `#crm/accounts/{vault_id}` |
| N19 | Contacts | Click contact in directory | Contact Detail + org tree highlight | Contact `vault_id` + `contact_id`, highlights node in org tree visualization | `#crm/contacts/{contact_id}` |
| N20 | Contacts | Click "View Account" in contact detail | Accounts view | `account_vault_id`, expands tree to show contact's position | `#crm/accounts/{vault_id}` |
| N21 | Inbox | Click message row | Message thread in preview panel | `communication_id`, loads full conversation thread + AI intent badge | `#crm/inbox/{communication_id}` |
| N22 | Inbox | Click "Create Task" in preview panel | Task Builder Modal | Pre-fills: module=CRM, vault=sender's account, source=smart_line, message preview | modal overlay |
| N23 | Inbox | Click "View Account" in preview panel | Accounts view | `account_vault_id` from sender lookup | `#crm/accounts/{vault_id}` |
| N24 | Onboarding | Click customer card | Onboarding Detail (expanded checklist) | `vault_id`, shows full checklist with assignees + due dates + progress | `#crm/onboarding/{vault_id}` |
| N25 | Onboarding | Click "View Deal" | Pipeline (Won Deals filter) | `deal_vault_id`, shows the deal that triggered onboarding | `#crm/pipeline/{vault_id}` |
| N26 | Any CRM view | Click notification badge | Source view with highlighted item | `vault_id` + `event_type`, navigates to the screen where the notification originated | varies |

### Navigation Context Object

When navigating between screens, a context object is passed to maintain state:

```typescript
interface NavigationContext {
  source_view: string;           // e.g., 'pipeline', 'health_monitor', 'inbox'
  vault_id: string;              // primary vault being viewed
  vault_level: number;           // 1-4
  highlight_id?: string;         // specific element to highlight (task, event, card)
  scroll_to?: string;            // element to scroll into view
  filter_preset?: string;        // filter to apply on destination ('nurture', 'lost', 'overdue')
  modal?: string;                // modal to open on arrival ('deal_creation', 'task_builder')
  prefill_data?: Record<string, unknown>;  // data to pre-fill in modals/forms
}
```

### Back Navigation

Every navigation stores the previous context in a stack. The user can:
- Press `Esc` or click the back arrow to return to the previous view
- Browser back button works (deep links stored in URL hash)
- Maximum stack depth: 10 (prevents infinite nesting)

---

## 10. Lifecycle Stage --> Screen Mapping Summary

A single reference showing where each lifecycle stage appears across all CRM screens.

| `lifecycle_stage` | `pipeline_stage` | Visible In | Sub-Panel Section | Badge Color |
|---|---|---|---|---|
| `new` | -- | New Leads, Activity Feed | DISCOVER | Gray |
| `mql` | -- | Qualify (MQL column), New Leads (filtered), Activity Feed | DISCOVER | Teal |
| `sal` | -- | Qualify (SAL column), Activity Feed | DISCOVER | Blue |
| `sql` | -- | Qualify (SQL column), Activity Feed | DISCOVER | Green |
| `nurture` | -- | New Leads (Nurture filter), Activity Feed | DISCOVER | Amber |
| `opportunity` | `prospecting` | Pipeline, Deal Detail, Activity Feed | DEALS | `--stage-prospecting` |
| `opportunity` | `discovery` | Pipeline, Deal Detail, Activity Feed | DEALS | `--stage-discovery` |
| `opportunity` | `proposal` | Pipeline, Deal Detail, Deal Review, Activity Feed | DEALS | `--stage-proposal` |
| `opportunity` | `negotiation` | Pipeline, Deal Detail, Deal Review, Activity Feed | DEALS | `--stage-negotiation` |
| `opportunity` | `close` | Pipeline, Deal Detail, Deal Review, Activity Feed | DEALS | `--stage-close` |
| `opportunity` | `won` | Won Deals, Onboarding, Health Monitor, Renewals, Activity Feed | DEALS + CUSTOMERS | Green |
| `opportunity` | `lost` | Pipeline (Lost filter, Table view only), Activity Feed | DEALS | Gray |
| `customer` | -- | Health Monitor, Renewals, Expansion, Accounts, Activity Feed | CUSTOMERS | Green |
| `churned` | -- | Accounts (Churned filter), Activity Feed | ACCOUNTS | Red |
| `disqualified` | -- | Not visible in active views. Queryable via Admin search. | -- | -- |
| `archived` | -- | Not visible in active views. Queryable via Admin search. | -- | -- |

---

## Related Specs

- [overview.md](./overview.md) -- CRM module identity, vault hierarchy mapping, chamber views, react-admin integration
- [crm-views-design-spec.md](./crm-views-design-spec.md) -- Concrete UI wireframes for every CRM view, component breakdowns, interaction specs
- [crm-comms-brainstorm.md](./crm-comms-brainstorm.md) -- Smart Line architecture, iMessage bridge, communications layer, lead scoring inputs, pipeline stage mapping
- [../Roles/overview.md](../Roles/overview.md) -- Role architecture, permission model, self-approval prevention rules
- [../WorkflowEngine/overview.md](../WorkflowEngine/overview.md) -- Visual automation builder powering lead scoring, nurture sequences, onboarding workflows
- [../TaskSystem/overview.md](../TaskSystem/overview.md) -- Universal task table backing all CRM task creation
- [../Notifications/overview.md](../Notifications/overview.md) -- Novu notification delivery for health alerts, SLA escalations, deal notifications
- [../PatchWorkflow/overview.md](../PatchWorkflow/overview.md) -- Approval chain pattern reused in Deal Review flow
- [../Security/overview.md](../Security/overview.md) -- Self-approval prevention is unforgeable (Security principle #4), append-only audit (principle #5)
