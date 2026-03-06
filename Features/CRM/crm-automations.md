# CRM Automations & Scoring -- Design Spec

> **Status:** SPECCED
> **Source:** Gap analysis, crm-views-design-spec.md, workflow-engine overview

## Summary
Defines all automated rules, health scoring formulas, lead scoring formulas, SLA timers, and escalation chains for the CRM module.

### 1. Lead Scoring Formula

Define the complete lead scoring system (0-100):

**Input signals and weights:**

| Signal | Weight | Scoring Logic |
|--------|--------|---------------|
| Company size | 15 | Enterprise(15), Mid-Market(10), SMB(5), Unknown(3) |
| Role/seniority | 15 | C-suite(15), VP(12), Director(10), Manager(7), Individual(3) |
| Source quality | 10 | Referral(10), Smart Line inbound(8), Web Form(6), LinkedIn(5), Cold(2) |
| Engagement recency | 15 | Last 24h(15), Last 3d(12), Last 7d(8), Last 14d(4), 14d+(0) |
| Engagement depth | 15 | Multiple channels(15), Replied(12), Opened(8), Clicked(5), None(0) |
| Topic relevance | 10 | Deal discussion(10), General inquiry(6), Support(3), Spam(0) |
| Response time | 10 | <1h(10), <4h(8), <24h(5), <72h(2), >72h(0) |
| Website behavior | 10 | Pricing page(10), Demo request(10), Docs(6), Blog(3), None(0) |

**Thresholds:**
- 0-39: Cold (red) -- minimal engagement, low fit
- 40-69: Warm (amber) -- some signals, needs nurture
- 70-89: Hot (green) -- strong signals, prioritize
- 90-100: On Fire -- immediate attention, likely ready to buy

**Recalculation triggers:** New communication, form submission, website visit, manual adjustment, daily batch

### 2. Health Score Formula

**Account health score (0-100):**

| Component | Weight | Calculation |
|-----------|--------|-------------|
| Contact responsiveness | 30% | Based on: avg reply time (<4h=100, <24h=75, <72h=50, >72h=25), % of messages replied to, days since last inbound |
| Deal progress | 25% | Based on: stage task completion %, days_in_stage vs avg for that stage (penalty if >2x avg) |
| Task completion | 20% | Based on: % tasks on time, overdue task count (each overdue = -10 points) |
| Interaction recency | 15% | Based on: days since last human interaction (not automated). 0-3d=100, 4-7d=80, 8-14d=50, 15-30d=25, 30d+=0 |
| Renewal proximity | 10% | Multiplier: if renewal <30 days, any decline is amplified 2x |

**Status thresholds:**
- 71-100: Healthy (green)
- 31-70: Watch (amber)
- 0-30: At Risk (red)

**Trend calculation:** Compare current score to 7-day rolling average. +/- shown as trend arrow.

### 3. SLA Timer Rules

Define SLA rules for Deal Review:

| Deal Value | Review SLA | Escalation 1 | Escalation 2 |
|------------|-----------|--------------|--------------|
| < $50K | 8 hours | Manager (at 6h) | Skip-level (at 8h) |
| $50K-$200K | 4 hours | Manager (at 3h) | VP (at 4h) |
| > $200K | 2 hours | VP (at 1.5h) | CEO (at 2h) |

SLA timer pauses: weekends, holidays (configurable), when reviewer is OOO.

### 4. Stage Gate Rules

Define required tasks per pipeline stage:

| Stage | Required Tasks Before Advancing | Auto-Created on Entry |
|-------|--------------------------------|----------------------|
| Prospecting | Research account, Send intro email | "Research account background", "Send intro email", "Schedule discovery call" |
| Discovery | Complete discovery questionnaire, Identify decision makers | "Complete discovery questionnaire", "Identify decision makers", "Map stakeholder org chart" |
| Proposal | Draft proposal, Get pricing approval, Send proposal | "Draft proposal document", "Get pricing approval", "Send proposal to buyer", "Follow up on review" |
| Negotiation | Review counterparty redlines, Finalize terms | "Review counterparty redlines", "Finalize territory/scope", "Get legal sign-off" |
| Close | Generate contract, Internal sign-off, Buyer signature | "Generate final contract", "Internal sign-off", "Send for buyer signature" |

### 5. Notification Rules

| Event | Who Gets Notified | Channel | Priority |
|-------|-------------------|---------|----------|
| New lead assigned | Assigned rep | In-app + push | Action |
| Lead score crosses 70 | Assigned rep + manager | In-app | Info |
| Deal stage advanced | Deal owner + team | In-app | Info |
| Deal review pending | Reviewer | In-app + push | Urgent |
| SLA approaching (50% elapsed) | Reviewer | In-app | Action |
| SLA breached | Reviewer + manager | In-app + push + email | Urgent |
| Health drops below 30 | Account owner + VP | In-app + email | Urgent |
| Health rapid decline (15+ pts/7d) | Account owner + manager | In-app + push | Urgent |
| Renewal <30 days | Account owner | In-app | Action |
| Renewal overdue | Account owner + manager | In-app + email | Urgent |
| Inbound message (unread >1h) | Assigned rep | Push | Action |
| Task overdue | Assignee | In-app + push | Action |

### 6. Workflow Engine Integrations

List the pre-built CRM workflows:

1. **Lead Intake (Smart Line)**: Message received → AI classify intent → Score lead → Route (score >=80 to AE, 50-79 to SDR, <50 to nurture) → Create task
2. **Lead Intake (Web Form)**: Form submitted → Enrich from form fields → Score → Route → Create task
3. **Lead Nurture**: Rejected/cold lead → 5-email sequence over 30 days → Monthly check-in → Re-score on re-engagement
4. **Deal Stage Tasks**: Stage entered → Create required tasks → Assign to deal owner → Set due dates (relative to entry date)
5. **Onboarding Kickoff**: Deal won → Create onboarding checklist → Assign team → Set SLA → Notify customer
6. **Health Alert Response**: Score drops below threshold → Generate recommended actions → Assign to account owner → Create task with 48h SLA
7. **Renewal Reminder**: 90/30/7 days before renewal → Notify owner → Create renewal task → If auto-renew, just notify

---

## Related Specs

- [overview.md](./overview.md) -- CRM module overview
- [crm-views-design-spec.md](./crm-views-design-spec.md) -- View definitions referenced by automations
- [/Features/WorkflowEngine/overview.md](/Features/WorkflowEngine/overview.md) -- Workflow engine that executes these automations
- [/Features/Notifications/overview.md](/Features/Notifications/overview.md) -- Novu notification system for delivery
