# Task Lifecycle Deep Dive: From Event to Resolution

> **Context:** The universal task table exists (see [overview.md](./overview.md)). This document answers: *What actually happens when something triggers a task?* How does a document annotation, an inbound message, or a deal stage change become a task with a builder, modifier, and signer? What does the task builder UX look like? How do automation rules fire?

> **Reference platforms studied:** ClickUp (custom task types, natural language automation), Salesforce (Flow-based stage triggers, Lead/Opportunity lifecycle automation), Asana (task templates, approval workflows), Aligned (AI-generated mutual action plans from calls, deal workspace task auto-generation).

---

## The Core Model: Every Task Has a Role Chain

Salesforce treats tasks as isolated to-do items on an object. Asana treats them as project steps. ClickUp adds custom types with unique fields. Aligned auto-generates them from AI call analysis.

Airlock combines all of these. Every task has a **role chain** — a sequence of people who must act on it:

```
BUILDER → MODIFIER(s) → SIGNER
```

| Role | What They Do | Maps To |
|------|-------------|---------|
| **Builder** | Creates the work product. Drafts the patch, writes the response, prepares the document. | Author, Creator, Drafter |
| **Modifier** | Refines the work. Edits, annotates, requests changes, adds context. Can be multiple rounds. | Reviewer, Editor, Commenter |
| **Signer** | Approves and finalizes. The authority that makes the output binding. | Approver, Gatekeeper, Admin |

This maps directly to the existing Patch Workflow (Author → Verifier → Admin) but generalizes across ALL task types — not just contract patches.

### Not Every Task Needs All Three Roles

| Task Type | Builder | Modifier | Signer | Example |
|-----------|---------|----------|--------|---------|
| `triage` | System (auto-created) | Builder (resolves) | Gatekeeper (verifies) | Extraction flagged low-confidence field |
| `review` | Builder (prepares vault) | — | Gatekeeper (approves gate) | Vault ready for preflight |
| `approval` | Builder (authors patch) | Verifier (clarifies) | Admin (final sign-off) | Contract field correction |
| `action` | Assigned user (does it) | — | — | "Follow up on Henderson call" |
| `inbound` | System (captures) | Rep (responds) | — | Customer texted the Smart Line |
| `deal_task` | System/AI (generates) | Rep (completes) | Manager (reviews) | "Send proposal to Nova" |
| `onboarding` | System (creates checklist) | Builder (executes steps) | Gatekeeper (confirms) | New customer setup |
| `annotation` | Analyst (flags section) | Builder (drafts resolution) | Gatekeeper (approves) | Document section flagged for review |

---

## Trigger → Task Pipeline

Every task in Airlock follows the same pipeline regardless of source:

```
EVENT (trigger)
    |
    v
CLASSIFY (what kind of task?)
    |
    v
CREATE (task row in master table)
    |
    v
ASSIGN (who's the builder? auto or manual?)
    |
    v
NOTIFY (Novu push to assigned user)
    |
    v
ROUTE (appears in correct module triage + Tasks inbox + Home widget)
```

### Trigger Source Map

| Source | Event | Task Type Created | Builder | Modifier | Signer |
|--------|-------|-------------------|---------|----------|--------|
| **Document annotation** | Analyst highlights section, clicks "Flag for Review" | `annotation` | Analyst (flags) → assigned builder | — | Gatekeeper |
| **Inbound iMessage/SMS** | Customer texts Smart Line | `inbound` | System (captures) | Rep (responds) | — |
| **Inbound call** | Customer calls, voicemail transcribed | `inbound` | System (captures) | Rep (follows up) | — |
| **AI intent from message** | Otto classifies: "wants to expand deal" | `deal_task` | System/AI (generates) | Rep (executes) | Manager (optional) |
| **Deal stage change** | Pipeline card dragged to "Proposal" | `deal_task` | System (auto-generates from stage rules) | Rep (completes) | — |
| **Contract extraction** | Low-confidence field detected | `triage` | System | Analyst (resolves) | Gatekeeper |
| **Preflight failure** | Gate check fails | `triage` | System | Builder (fixes) | Gatekeeper |
| **Patch submitted** | Author submits field correction | `approval` | Author | Verifier | Admin |
| **SLA approaching** | Deadline < 1 hour | `sla_warning` | System | Assigned user (resolves root task) | — |
| **Cross-module event** | Contract shipped → CRM update needed | `action` | System (via BullMQ) | Assigned user | — |
| **Manual creation** | User clicks "New Task" or uses /task | `manual` | User | Assignee (if different) | — |
| **AI call analysis** | Post-call AI generates action items (Aligned-style) | `deal_task` | AI (drafts) | Rep (confirms/edits) | — |
| **Mutual action plan** | Buyer + seller agree on next steps | `deal_task` | System (from MAP template) | Both parties | Manager |

---

## Deep Dive: Document Annotation → Task

This is the flow the user specifically asked about. Today, annotations live in the annotation layer and mark up the document. The missing piece: they need to CREATE tasks with full lifecycle.

### Current Flow (Annotation Only)

```
Analyst reads contract section
    → Highlights text
    → Adds annotation (comment, flag, question)
    → Annotation lives in the document layer
    → Other users see it in the document viewer
    ❌ No task created
    ❌ No assignment
    ❌ No tracking
    ❌ No SLA
```

### New Flow (Annotation → Task)

```
Analyst reads contract section
    |
    → Highlights text in document viewer
    |
    → Clicks "Create Task" (or "Flag for Review") in annotation toolbar
    |
    → Task Builder modal opens (pre-populated):
    |     Field: auto-detected from highlighted section
    |     Context: extracted text snippet (the evidence)
    |     Vault: current vault (auto-set)
    |     Module: contracts (auto-set)
    |     Severity: analyst selects (info / warning / blocker)
    |
    → Analyst fills in:
    |     Title: "Indemnification clause missing cap amount"
    |     Intent: "This section references liability cap but doesn't specify dollar amount"
    |     Task type: annotation (auto-set) or triage / review / approval
    |     Assign to: selects user or team, or leaves unassigned
    |     Due date: optional
    |
    → Task created in master table:
    |     task_type = 'annotation'
    |     source = 'document_annotation'
    |     field_code = 'indemnification.cap_amount'
    |     metadata = {
    |       annotation_id: "ann_123",
    |       document_id: "doc_456",
    |       page: 14,
    |       highlighted_text: "The Contractor shall indemnify...",
    |       section_path: "Article 8 > Section 8.2 > Indemnification"
    |     }
    |
    → Annotation in document viewer gets a task badge (linked)
    |     Clicking the badge opens the task in Focus Mode
    |
    → Task appears in:
    |     - Contracts triage board
    |     - Tasks module inbox
    |     - Home widget
    |     - Vault Signal panel (event: "Task created from annotation")
    |
    → Assigned builder gets Novu notification:
    |     "New task: Indemnification clause missing cap amount (henderson-msa)"
    |
    → Builder opens task → sees annotation in context:
    |     Document viewer scrolls to the exact highlighted section
    |     Task detail shows the evidence (highlighted text + section path)
    |     Builder can: draft a patch, add a comment, request clarification, resolve
    |
    → If builder drafts a patch:
    |     Task transitions to in_review
    |     Patch workflow kicks in (Author → Verifier → Admin)
    |     Task resolves when patch is applied
    |
    → If builder resolves directly:
    |     Adds resolution note: "Cap amount confirmed at $2M in Amendment 3"
    |     Task transitions to resolved
    |     Gatekeeper can verify if task was flagged as blocker
```

### The Annotation Toolbar

When a user highlights text in the document viewer, a floating toolbar appears:

```
+--[ Comment ]--[ Flag ]--[ Create Task ]--[ Link to Existing ]--+
```

| Action | What Happens |
|--------|-------------|
| **Comment** | Adds inline annotation (existing behavior, no task) |
| **Flag** | Quick-flag: creates a task with severity=warning, auto-title from section name, unassigned |
| **Create Task** | Opens the full Task Builder modal (see below) |
| **Link to Existing** | Associates this annotation with an already-existing task |

---

## Deep Dive: Inbound Message → Task

When a customer texts the Smart Line:

```
Customer texts: "Hey, we need to discuss the renewal terms for the Henderson contract"
    |
    v
iMessage Bridge → webhook → POST /api/comms/inbound
    |
    v
Phone # lookup: +1-555-0142 → Jack Chen, Nova Entertainment (vault_contact)
    |
    v
AI Intent Classification (Otto):
    intent = "deal_discussion"
    sub_intent = "renewal"
    confidence = 0.92
    entities_detected = ["Henderson", "renewal", "terms"]
    vault_match = henderson-msa (via entity "Henderson")
    |
    v
Smart Router Decision:
    Rule: deal_discussion + known contact + assigned rep → route to rep
    Action: route to Sarah Miller (relationship owner for Nova)
    |
    v
Three things happen simultaneously:
    |
    +--[1. Communication logged]
    |     INSERT INTO communications (
    |       vault_id = nova_entertainment,
    |       contact_id = jack_chen,
    |       channel = 'imessage',
    |       direction = 'inbound',
    |       body = "Hey, we need to discuss...",
    |       intent_classification = 'deal_discussion',
    |       routed_to = sarah_miller
    |     )
    |
    +--[2. Task created]
    |     INSERT INTO tasks (
    |       title = "Respond: Jack Chen (Nova) — renewal discussion",
    |       task_type = 'inbound',
    |       module_type = 'crm',
    |       vault_id = nova_entertainment,
    |       severity = 'warning',  -- deal discussion = elevated priority
    |       assigned_to = sarah_miller,
    |       source = 'smart_line',
    |       metadata = {
    |         communication_id: "comm_789",
    |         contact_name: "Jack Chen",
    |         intent: "deal_discussion",
    |         sub_intent: "renewal",
    |         linked_vault: "henderson-msa",
    |         message_preview: "Hey, we need to discuss the renewal..."
    |       }
    |     )
    |
    +--[3. Novu notification]
          Channel: push + in-app
          To: sarah_miller
          Message: "Jack Chen (Nova) texted about Henderson renewal"
          Action button: "Open in CRM Inbox"
```

### What Sarah Sees

Sarah's CRM Inbox shows a new item at the top. Clicking it opens a **conversation view**:

```
+-----Signal (Thread)------+----Orchestrate (Reply)----+----Control (Context)----+
|                           |                           |                         |
| Jack Chen (Nova)          | [Reply composer]          | Contact: Jack Chen      |
| 10:42 AM                  |                           | Role: Biz Dev           |
| "Hey, we need to discuss  | [Template suggestions]    | Account: Nova Ent.      |
|  the renewal terms for    |                           | Vault: henderson-msa    |
|  the Henderson contract"  | Quick replies:            | Last contact: 3 days ago|
|                           | [ ] "Sure, let me pull    | Deal stage: Negotiation |
| --- Earlier messages ---  |      up the terms"        | Renewal: 45 days out    |
| Mar 1: "Thanks for the    | [ ] "I'll have Sarah from | Health: 82/100          |
|  updated proposal"        |      legal reach out"     |                         |
| Feb 28: "Can you send     | [ ] [Custom reply]        | Tasks on this vault: 3  |
|  the Q1 report?"          |                           | Open patches: 1         |
|                           | [Send via iMessage]       |                         |
+---------------------------+---------------------------+-------------------------+
```

The task auto-resolves when Sarah sends a reply (status → resolved, resolution_note = "Replied via iMessage").

---

## Deep Dive: Deal Stage Change → Task Auto-Generation

This is the Aligned/Salesforce model: when a deal moves to a new pipeline stage, the system auto-generates the tasks required for that stage.

### Stage → Task Rules (Configurable per Workspace)

| Pipeline Stage | Auto-Generated Tasks | Builder | Due |
|---------------|---------------------|---------|-----|
| **Prospecting** | "Research account background" | Assigned rep | +2 days |
| | "Send intro email/text" | Assigned rep | +1 day |
| **Discovery** | "Schedule discovery call" | Assigned rep | +3 days |
| | "Prepare discovery questions" | Assigned rep | +2 days |
| | "Identify decision makers" | Assigned rep | +5 days |
| **Proposal** | "Draft proposal document" | Assigned rep | +5 days |
| | "Get pricing approval" | Rep → Manager (signer) | +3 days |
| | "Send proposal to buyer" | Assigned rep | +7 days |
| **Negotiation** | "Review buyer's redlines" | Legal (modifier) | +3 days |
| | "Prepare counteroffer" | Assigned rep | +5 days |
| | "Update CRM with negotiation notes" | Assigned rep | +1 day |
| **Close** | "Generate final contract" | System → Legal (builder) | +3 days |
| | "Obtain internal sign-off" | Rep → Gatekeeper (signer) | +2 days |
| | "Send for buyer signature" | Assigned rep | +5 days |
| **Won** | "Create onboarding checklist" | System (auto) | immediate |
| | "Provision Smart Line" | System (auto) | immediate |
| | "Send welcome packet" | Assigned rep | +1 day |
| | "Schedule kickoff call" | Assigned rep | +3 days |

### How It Works

```
Rep drags deal card from "Discovery" to "Proposal" on Pipeline Kanban
    |
    v
BullMQ job: deal_stage_changed
    payload: { vault_id, from_stage: "discovery", to_stage: "proposal", rep_id }
    |
    v
Task Rule Engine looks up rules for "proposal" stage:
    workspace_id → custom rules (if configured) || default rules
    |
    v
For each rule:
    CREATE task in master table:
        title = rule.title
        task_type = 'deal_task'
        module_type = 'crm'
        vault_id = deal.vault_id
        assigned_to = rule.assignee (rep, manager, legal, or system)
        due_at = now() + rule.due_offset
        metadata = {
            stage_trigger: "proposal",
            rule_id: "rule_123",
            auto_generated: true
        }
    |
    v
Each task appears in:
    - CRM Deals > Pipeline (linked to deal card as subtasks)
    - Tasks module inbox
    - Home widget
    - Vault Signal panel
    |
    v
Rep sees a "Stage tasks" panel on the deal card:
    [ ] Draft proposal document          Due: Mar 7
    [ ] Get pricing approval             Due: Mar 5
    [ ] Send proposal to buyer           Due: Mar 9
```

### Task Rule Editor (Admin)

Workspace admins can customize which tasks auto-generate per stage. The rule editor:

```
+-----------------------------------------------------------+
| TASK RULES: Pipeline Stage → Auto-Generated Tasks          |
|                                                            |
| Stage: [Proposal ▾]                                       |
|                                                            |
| +-------------------------------------------------------+ |
| | Task 1                                                 | |
| | Title: [Draft proposal document                      ] | |
| | Assign to: [Assigned Rep ▾]                            | |
| | Due offset: [5] days after stage entry                 | |
| | Severity: [info ▾]                                     | |
| | Required: [✓] Must complete before advancing to next   | |
| +-------------------------------------------------------+ |
| | Task 2                                                 | |
| | Title: [Get pricing approval                         ] | |
| | Assign to: [Rep's Manager ▾]                           | |
| | Role chain: Builder=[Rep] → Signer=[Manager]           | |
| | Due offset: [3] days after stage entry                 | |
| | Severity: [warning ▾]                                  | |
| | Required: [✓]                                          | |
| +-------------------------------------------------------+ |
|                                                            |
| [+ Add Task Rule]                                         |
|                                                            |
| Gate behavior:                                             |
| [✓] Block stage advancement until all required tasks done  |
| [ ] Allow advancement with warning if tasks incomplete     |
+-----------------------------------------------------------+
```

This is where ClickUp's "custom task types with per-type fields" and Salesforce's "Flow-based stage triggers" merge. The rules define WHAT tasks, WHO does them, and WHETHER they gate advancement.

---

## Deep Dive: AI-Generated Tasks (Aligned Model)

Aligned auto-builds deal resources after every call — executive summary, action items, follow-ups. We do the same via Otto.

### Post-Call Task Generation

```
Rep finishes call with Jack from Nova (logged via Smart Line)
    |
    v
Call recording → Whisper transcription (Phase 3+)
    |
    v
Otto analyzes transcript:
    |
    +--[Executive Summary]
    |     "Jack wants to expand the Henderson deal to include Q2 deliverables.
    |      He mentioned budget approval is pending from their CFO (Mike Torres).
    |      Jack asked for updated pricing by Friday."
    |
    +--[Action Items Extracted]:
    |     1. "Send updated pricing to Jack" — due: Friday
    |     2. "Identify Mike Torres (CFO) as decision maker" — due: +1 day
    |     3. "Follow up on budget approval status" — due: +5 days
    |     4. "Update Henderson deal to include Q2 scope" — due: +1 day
    |
    +--[Contact Enrichment]:
    |     Mike Torres → new vault_contact (role: CFO, source: call_transcript)
    |
    v
For each action item:
    CREATE task in master table:
        title = action_item.text
        task_type = 'deal_task'
        module_type = 'crm'
        vault_id = henderson-msa
        assigned_to = sarah_miller (rep on the call)
        due_at = action_item.due
        source = 'ai'
        metadata = {
            call_id: "call_456",
            transcript_excerpt: "...",
            ai_confidence: 0.88,
            auto_generated: true
        }
    |
    v
Rep gets a notification with the AI-generated task list:
    "Otto generated 4 action items from your call with Jack Chen."
    [Review & Confirm] [Edit] [Dismiss]
```

### The Confirmation Step (Positive Friction)

AI-generated tasks don't auto-commit. The rep sees a **review panel**:

```
+-----------------------------------------------------------+
| POST-CALL ACTION ITEMS                     [Jack Chen call]|
|                                                            |
| Generated by Otto from 18-minute call transcript           |
|                                                            |
| [✓] Send updated pricing to Jack           Due: Mar 7     |
|     Confidence: 92%                                        |
|                                                            |
| [✓] Identify Mike Torres as decision maker Due: Mar 5     |
|     Confidence: 88%                                        |
|     → New contact will be created: Mike Torres (CFO)       |
|                                                            |
| [✓] Follow up on budget approval status    Due: Mar 9     |
|     Confidence: 85%                                        |
|                                                            |
| [✓] Update Henderson deal — add Q2 scope   Due: Mar 5     |
|     Confidence: 78%                                        |
|                                                            |
| [Confirm All]  [Edit Selected]  [Dismiss All]             |
+-----------------------------------------------------------+
```

This is the evidence-first, positive-friction pattern: Otto surfaces evidence (the action items with confidence scores), but the human confirms. Nothing happens without deliberate action.

---

## The Task Builder Modal

When a user creates a task (from annotation toolbar, from "New Task" button, from Kanban quick-add), they get a modal with type-aware fields.

### Base Fields (All Task Types)

```
+-----------------------------------------------------------+
| CREATE TASK                                                |
|                                                            |
| Title*: [_________________________________]                |
|                                                            |
| Description: [TipTap rich text editor      ]               |
|              [with markdown support         ]               |
|                                                            |
| Type: [ annotation ▾ ]                                     |
|   annotation | triage | review | approval | deal_task |    |
|   inbound | action | manual                                |
|                                                            |
| Module: [ contracts ▾ ]  Vault: [ henderson-msa ▾ ]       |
|                                                            |
| Severity: ( ) info  (•) warning  ( ) blocker              |
|                                                            |
| Due date: [ Mar 7, 2026 ]  [ 5:00 PM ]                    |
|                                                            |
+-----------------------------------------------------------+
```

### Role Chain Section (Appears for types that need it)

```
+-----------------------------------------------------------+
| ROLE CHAIN                                                 |
|                                                            |
| Builder*:  [ Sarah Miller ▾ ]  (who creates the work)     |
|                                                            |
| Modifier:  [ + Add modifier ]  (who reviews/edits)        |
|            [ Ana Chen ▾ ] [×]                              |
|                                                            |
| Signer:    [ David Park ▾ ]    (who approves)             |
|            Required: [✓]                                   |
|                                                            |
| Self-approval prevention: [✓] (builder cannot be signer)  |
+-----------------------------------------------------------+
```

### Type-Specific Fields

When the user selects a task type, additional fields appear:

**For `annotation` type:**
```
| ANNOTATION CONTEXT                                         |
| Document: [henderson-msa-v3.pdf]                           |
| Page: [14]  Section: [Article 8 > 8.2 > Indemnification] |
| Highlighted text: "The Contractor shall indemnify..."      |
| [View in Document ↗]                                       |
```

**For `approval` type:**
```
| APPROVAL CHAIN                                             |
| Step 1: Verifier → [ Ana Chen ▾ ]                         |
| Step 2: Admin → [ David Park ▾ ]                          |
| Escalation: [ Auto-escalate after 48h of no action ]      |
```

**For `deal_task` type:**
```
| DEAL CONTEXT                                               |
| Pipeline stage: [ Proposal ▾ ]                             |
| Deal value: [ $240,000 ]                                   |
| Linked contact: [ Jack Chen (Nova) ▾ ]                    |
| Stage-gating: [✓] Must complete before stage advance      |
```

**For `inbound` type:**
```
| COMMUNICATION CONTEXT                                      |
| Channel: iMessage                                          |
| From: Jack Chen (+1-555-0142)                              |
| Message: "Hey, we need to discuss the renewal terms..."    |
| Intent: deal_discussion (92% confidence)                   |
| [View Full Thread ↗]                                       |
```

---

## Task Automation Rules

Inspired by Salesforce Flows and ClickUp's natural language automation. Rules define: WHEN something happens → THEN create/modify tasks.

### Rule Structure

```sql
CREATE TABLE task_rules (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id),

    -- Trigger
    trigger_type TEXT NOT NULL,        -- 'stage_change' | 'task_status' | 'sla_breach' | 'communication' | 'annotation'
    trigger_conditions JSONB NOT NULL, -- { "stage_from": "discovery", "stage_to": "proposal" }

    -- Action
    action_type TEXT NOT NULL,         -- 'create_task' | 'assign_task' | 'notify' | 'update_field'
    action_config JSONB NOT NULL,      -- { "title": "Draft proposal", "assign_to": "rep", "due_offset_days": 5 }

    -- Scope
    module_type TEXT,                  -- null = all modules
    pipeline_id UUID,                  -- null = all pipelines

    -- Config
    is_active BOOLEAN DEFAULT true,
    is_gating BOOLEAN DEFAULT false,   -- blocks advancement until task completes
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Example Rules

| Trigger | Condition | Action | Gating? |
|---------|-----------|--------|---------|
| `stage_change` | to = "proposal" | Create "Draft proposal" task, assign to rep, due +5d | Yes |
| `stage_change` | to = "close" | Create "Generate contract" task, assign to legal | Yes |
| `stage_change` | to = "won" | Create onboarding checklist (4 tasks), provision Smart Line | No |
| `task_status` | type=inbound, status=open, age > 2h | Escalate: reassign to manager, severity → urgent | No |
| `sla_breach` | any task overdue | Create "SLA breach" warning task, notify manager | No |
| `communication` | intent = "cancellation" | Create "Retention risk" task, assign to account manager, severity = blocker | No |
| `annotation` | severity = "blocker" | Auto-assign to vault gatekeeper, set SLA = 24h | No |
| `communication` | new_contact detected | Create "Enrich contact" task, assign to CRM rep | No |

---

## Mutual Action Plans (Aligned Model)

For larger deals, the task list becomes a **shared plan** between buyer and seller. Aligned calls these "Mutual Action Plans." In Airlock, this is a task group attached to a vault.

### How It Works

```
Rep creates MAP from Deal > Pipeline > deal card > "Create Action Plan"
    |
    v
System generates default tasks from pipeline stage rules
    + Rep adds custom tasks
    + Rep can mark tasks as "buyer-facing" (visible in customer portal)
    |
    v
MAP becomes a shared checklist:

+-----------------------------------------------------------+
| MUTUAL ACTION PLAN: Nova Entertainment - Henderson MSA     |
| Stage: Proposal                          Progress: 3/8     |
|                                                            |
| SELLER TASKS                                               |
| [✓] Research account background            Sarah   Mar 1  |
| [✓] Schedule discovery call                Sarah   Mar 2  |
| [✓] Prepare discovery questions            Sarah   Mar 3  |
| [ ] Draft proposal document                Sarah   Mar 7  |
| [ ] Get pricing approval                   Manager Mar 5  |
|                                                            |
| BUYER TASKS (shared with customer)                         |
| [✓] Provide current vendor contract        Jack    Mar 3  |
| [ ] Confirm budget with CFO                Jack    Mar 8  |
| [ ] Schedule legal review                  Nova    Mar 10 |
|                                                            |
| [+ Add Task]  [Share with Buyer]  [Export PDF]            |
+-----------------------------------------------------------+
```

Buyer tasks can be shared via the Smart Line:
- Rep sends the MAP link via iMessage
- Buyer can view and check off their items via a web link
- Completion status syncs back to Airlock in real-time
- Otto sends reminders for overdue buyer tasks

---

## How This Differs From Salesforce/ClickUp/Asana/Aligned

### vs Salesforce

| Aspect | Salesforce | Airlock |
|--------|-----------|---------|
| Task isolation | Tasks belong to ONE object (Lead OR Contact OR Opportunity) | Tasks link to a vault but appear in ALL relevant views simultaneously |
| Stage automation | Requires custom Flows or Process Builder | Built-in task rule engine with stage triggers |
| Role chain | No built-in concept | Builder → Modifier → Signer on every task |
| Communication tasks | Bolt-on (Pardot, Sales Engagement) | Native — inbound messages auto-create tasks |
| Cross-module | Tasks stay in their object silo | Universal task table, cross-module propagation via BullMQ |

### vs ClickUp

| Aspect | ClickUp | Airlock |
|--------|---------|---------|
| Custom task types | Yes — custom fields and statuses per type | Yes — type-specific fields in metadata JSONB |
| Automation | Natural language rule builder | Stage-based rules + AI intent triggers |
| Document integration | Docs are separate from tasks | Annotations directly create tasks with document context |
| CRM integration | Separate CRM feature | Vault hierarchy IS the CRM — tasks inherit relationship context |

### vs Aligned

| Aspect | Aligned | Airlock |
|--------|---------|---------|
| AI task generation | After calls, auto-generates action items | Same, but from ALL sources: calls, texts, annotations, stage changes |
| Mutual action plans | Shared buyer/seller checklists | Same, but shared via iMessage (not a portal) |
| Deal workspace | Centralized deal room | Vault IS the deal room — triptych layout with Signal/Orchestrate/Control |
| Buyer engagement | Tracks content views | Tracks content views + communication patterns + task completion |

### The Airlock Advantage (Summary)

1. **One table, everywhere**: A task created from a document annotation in Contracts shows up in the Tasks module, the Home widget, and the vault's Signal panel. No duplication, no sync.

2. **Communication-native**: Inbound messages auto-create tasks. No bolt-on integration. The Smart Line IS the task creation layer for customer interactions.

3. **Role chain generalizes Patch Workflow**: The Builder → Modifier → Signer pattern applies to everything, not just contract patches. A CRM task, a document annotation task, and a deal stage task all follow the same governance pattern.

4. **AI generates, humans confirm**: Otto extracts action items from calls, messages, and document analysis. But nothing commits without human confirmation. Evidence-first, positive friction.

5. **Stage rules are configurable**: Admins define which tasks auto-generate per pipeline stage, whether they gate advancement, and who they assign to. No code, no Flow builder — just a rule editor.

---

## Data Model Additions

### Role Chain Fields (Added to Master Task Table)

```sql
ALTER TABLE tasks ADD COLUMN builder_id UUID REFERENCES users(id);
ALTER TABLE tasks ADD COLUMN signer_id UUID REFERENCES users(id);
ALTER TABLE tasks ADD COLUMN modifiers UUID[] DEFAULT '{}';  -- Array of user IDs
ALTER TABLE tasks ADD COLUMN requires_sign_off BOOLEAN DEFAULT false;
ALTER TABLE tasks ADD COLUMN sign_off_at TIMESTAMPTZ;
ALTER TABLE tasks ADD COLUMN sign_off_by UUID REFERENCES users(id);
```

### Annotation Link Fields

```sql
ALTER TABLE tasks ADD COLUMN annotation_id UUID;       -- Link to document annotation
ALTER TABLE tasks ADD COLUMN document_id UUID;          -- Source document
ALTER TABLE tasks ADD COLUMN page_number INT;           -- Page reference
ALTER TABLE tasks ADD COLUMN section_path TEXT;         -- e.g., "Article 8 > 8.2"
ALTER TABLE tasks ADD COLUMN highlighted_text TEXT;     -- Extracted text snippet
```

### Communication Link Fields

```sql
ALTER TABLE tasks ADD COLUMN communication_id UUID REFERENCES communications(id);
ALTER TABLE tasks ADD COLUMN contact_id UUID REFERENCES vault_contacts(id);
ALTER TABLE tasks ADD COLUMN intent_classification TEXT;
```

### AI Generation Fields

```sql
ALTER TABLE tasks ADD COLUMN ai_generated BOOLEAN DEFAULT false;
ALTER TABLE tasks ADD COLUMN ai_confidence FLOAT;
ALTER TABLE tasks ADD COLUMN ai_confirmed BOOLEAN;     -- null = pending, true = confirmed, false = dismissed
ALTER TABLE tasks ADD COLUMN ai_confirmed_at TIMESTAMPTZ;
ALTER TABLE tasks ADD COLUMN ai_source TEXT;            -- 'call_transcript' | 'message_intent' | 'annotation_analysis' | 'stage_rule'
```

---

## Related Specs

- **[Universal Task System](./overview.md)** — Master table schema and module triage views
- **[Tasks Module](../Tasks/overview.md)** — Universal work queue UI
- **[Patch Workflow](../PatchWorkflow/overview.md)** — Approval chain (the original Builder → Verifier → Admin pattern)
- **[Action Focus](../PatchWorkflow/action-focus.md)** — Full editor experience for patch authoring
- **[Triage Dashboard](../Triage/overview.md)** — Contracts-specific triage view
- **[CRM Comms Brainstorm](../CRM/crm-comms-brainstorm.md)** — Smart Line, inbound message routing, AI intent classification
- **[Record Inspector](../Record%20Inspector/)** — Document viewer where annotations create tasks

## Sources

- [Salesforce Task Management Guide](https://www.getweflow.com/blog/salesforce-tasks) — Task activities, deal stage automation
- [Salesforce Flow: Stage-Based Task Automation](https://trailhead.salesforce.com/content/learn/modules/record-triggered-flows/add-a-scheduled-task-to-your-flow) — Scheduled task creation from stage changes
- [ClickUp Custom Task Types](https://clickup.com/features/custom-task-types) — Per-type fields, statuses, automations
- [Aligned AI Deal Workspace](https://alignedup.com/) — AI-generated action items, mutual action plans, buyer collaboration
- [Aligned Launch: AI Deal Workspace](https://www.prnewswire.com/il/news-releases/aligned-launches-the-ai-deal-workspace--the-missing-execution-layer-for-modern-sales-302684951.html) — Post-call task auto-generation, stakeholder mapping
