# Workflow Engine: Visual Automation Builder

> **Status:** BRAINSTORM — Defining the visual workflow builder that powers lead qualification, communications routing, contract generation triggers, task automation, and conversational intake funnels.

> **Key Insight:** Every automation in Airlock — lead qualification, iMessage routing, contract generation triggers, deal stage tasks, notification workflows — is the same thing: an event enters, conditions are evaluated, actions fire. One visual builder handles all of them.

> **Tech Stack:** React Flow (`@xyflow/react` v12) for the visual canvas. Zustand for workflow state. ELKjs for auto-layout. BullMQ for execution. Stored as JSON in PostgreSQL.

> **Reference platforms studied:** Knock (workflow orchestration, batching, channel routing), HubSpot (250-branch workflows, enrollment triggers), Salesforce Flow (Record-Triggered Flows, Decision elements), ActiveCampaign (Goals as checkpoints, AI automation builder), SendBlue (iMessage webhooks, attachment handling), Discord (onboarding flows, role assignment, embed workflows), **BotGhost** (tree-based visual builder, block categories, variable scoping, sharing/templates), **Discord Bot Studio** (flowchart editor, code export, command/event node separation).

---

## Why One Workflow Engine

Every feature we've specced eventually says "and then this triggers that":

| Feature | The Trigger | The Automation |
|---------|------------|----------------|
| Lead qualification | Form submitted, message received | Score lead, route to pool or rep, create task |
| iMessage routing | Customer texts Smart Line | Classify intent, route to agent or AI, log communication |
| Contract generation | Template selected, clauses chosen | Validate inputs, generate draft, create review task |
| Deal stage change | Card dragged to new column | Create stage-specific tasks, notify team, check gate requirements |
| Patch workflow | Patch submitted | Route to verifier, start SLA timer, notify admin |
| Notification delivery | Any event | Batch, delay, choose channel (iMessage > SMS > email), deliver |
| Customer onboarding | Deal closed-won | Provision Smart Line, create checklist, assign team, send welcome |
| Attachment handling | File received via iMessage | Classify type, extract data, route to appropriate module |

**All of these are the same pattern:** EVENT → CONDITION → ACTION. Instead of hardcoding each one, we build a visual workflow builder that lets admins create, modify, and version these automations through a drag-and-drop canvas.

---

## The Discord Parallel

Discord's onboarding system is the closest UX analog to what we're building:

### Discord Onboarding Flows
- New member joins a server → onboarding flow activates
- **Custom questions** define what roles the user gets, what channels they see
- **Embed workflows** process form responses and assign roles automatically
- **Progressive disclosure** — users only see what's relevant to their answers
- **Bot-driven flows** (like MEE6, Carl-bot) extend this with custom sequences: welcome DM → role selection → channel access → verification

### How This Maps to Airlock

| Discord Concept | Airlock Equivalent |
|----------------|-------------------|
| Server | Workspace |
| Onboarding flow | Conversational Intake Workflow |
| Custom questions | AI-guided qualification questions |
| Role assignment | Route to rep, assign to pool, set lifecycle_stage |
| Channel visibility | Module/vault access based on qualification result |
| Bot-driven sequences | Workflow engine automation steps |
| Embed forms | Smart Line conversational forms |

### The Conversational Intake Funnel

Instead of a static "Contact Us" form, the workflow engine powers a **conversational intake** — whether via web widget, Smart Line (iMessage), or in-app:

```
Prospect reaches out (any channel)
    |
    v
Workflow activates: "New Contact Intake"
    |
    +-- AI asks: "Hi! What brings you to [Company]?"
    |
    +-- Prospect: "I need help with contract management"
    |
    +-- AI classifies: intent = contracts, sentiment = positive
    |
    +-- Workflow branch: intent == "contracts"?
    |     |
    |     +-- AI asks: "Great! How many contracts do you manage monthly?"
    |     |     |
    |     |     +-- "Under 50" → Route to self-serve onboarding
    |     |     +-- "50-500" → Route to SDR pool (Heymarket model)
    |     |     +-- "500+" → Route to enterprise AE directly
    |     |
    |     +-- AI asks: "Who should we follow up with? Name and role?"
    |           |
    |           +-- Creates vault_contact with role + context
    |           +-- Creates lead with lifecycle_stage = 'new'
    |           +-- Assigns to pool or rep based on volume answer
    |           +-- Sends confirmation: "Sarah from our team will reach out within 1 hour"
    |
    +-- All answers logged as communications + enrichment data
    +-- Full conversation becomes the lead's context in CRM
```

**The key insight from the user:** Instead of asking people to fill out a form, the workflow asks guiding questions that *define* the form dynamically. The conversation IS the qualification. And it routes them to the right person based on their actual needs, not a generic "sales@" inbox.

---

## What We're Stealing from BotGhost + Bot Studio

BotGhost and Discord Bot Studio are the closest working examples of visual workflow builders for Discord. We're adopting their proven patterns and adapting them for enterprise CRM/contract operations.

### BotGhost's Tree Architecture (What We're Taking)

BotGhost uses a **tree-based builder** with a yellow "root block" as the entry point. Everything flows downward from there. Blocks connect via circles (top = input, bottom = output). The sidebar has 6 sections for blocks, variables, errors, templates.

| BotGhost Pattern | Airlock Adaptation |
|-----------------|-------------------|
| **Yellow root block** (trigger) | Our Trigger node — the entry point. Same concept, different color per trigger type |
| **Blue action blocks** (do things) | Our Action nodes — Send Message, Create Task, Assign, etc. |
| **Purple option blocks** (user inputs) | Our Conversational Ask node — collects input from contact mid-workflow |
| **Condition blocks** (if/else branching) | Our Branch node — evaluate conditions, split into multiple paths |
| **Variable system** (global/server/channel/user scoping) | Our workflow scope — trigger_data, contact, vault, workspace, variables |
| **Block labels** (rename for clarity) | Node labels — rename nodes for readability in complex workflows |
| **Sidebar block library** (categorized) | Our left panel — Triggers / Functions / Actions categorized |
| **Shift+drag multi-select** | Same — bulk operations for duplicating, deleting, templating |
| **Save as Template** (reusable block sequences) | Our sharing system — export workflow segments as reusable templates |
| **Auto-save with intervals** | Same — draft auto-save, manual commit for versioning |
| **100-step undo/redo** | Same — full undo/redo history in the builder |
| **19 event categories** (triggers) | Our 10 trigger types — more focused on CRM events vs Discord events |
| **10 action categories** (Message, API, Role, Channel, etc.) | Our 11 action types — adapted for CRM (Create Task, Route to Pool, etc.) |
| **Object variables with dot notation** `{User[ID].username}` | Our Liquid templates `{{contact.name}}`, `{{vault.deal_value}}` |
| **Collection/array functions** | Our Loop node + Transform node for data manipulation |

### Bot Studio's Flowchart Model (What We're Taking)

Discord Bot Studio takes a different approach — a **flowchart editor** with:
- **Command nodes** (dark blue) — the trigger
- **Response nodes** (light blue) — the actions
- **Event nodes** (pink) — server-triggered automations
- **Double-click to edit** node details in a popup
- **Code export** — download the generated bot code

| Bot Studio Pattern | Airlock Adaptation |
|-------------------|-------------------|
| **Color-coded node types** | Our nodes are color-coded by category: teal (triggers), blue (functions), green (actions) |
| **Double-click to edit popup** | Our right-panel detail editor (slide-in on node click, same concept) |
| **Separate Commands vs Events pages** | Our workflow categories (Lead Qualification, Communications, Deal Automation, etc.) |
| **Code export / downloadable** | Our workflow JSON export + version history (for backup and migration) |
| **Flowchart-style connections** | React Flow handles this natively — drag from handle to handle |

### What We're Adding That Neither Has

| Airlock Unique Feature | Why |
|----------------------|-----|
| **AI nodes** (Classify, Generate) | Neither BotGhost nor Bot Studio has built-in AI classification. We run Claude on message content, attachments, call transcripts |
| **Conversational Ask with AI parsing** | BotGhost's option blocks are static choices. Ours uses AI to parse free-text responses in natural language |
| **Workflow versioning with environments** | BotGhost has auto-save. We have Draft → Commit → Dev → Staging → Prod promotion (Knock model) |
| **Execution logging with replay** | Full audit trail of every node visited, every branch decision, every action result. Critical for enterprise compliance |
| **Pool routing** | Neither has shared inbox/pool concepts. We have Heymarket-style pools with claim, presence, round-robin |
| **Cross-module triggers** | BotGhost triggers are Discord-only. Ours span CRM, Contracts, Tasks, Calendar — one engine for all modules |
| **SLA timers** | Time-bound actions with escalation workflows on expiry. Enterprise requirement |
| **Batch/throttle nodes** | Aggregate multiple events before acting. Prevent notification spam |
| **iMessage attachment classification** | Deep content analysis: is that PDF a contract or an invoice? Voice memo transcription. vCard parsing |

### The Builder Canvas: Unified Design Language

Combining BotGhost's tree clarity with React Flow's power:

```
+------------------------------------------------------------------+
| Workflow: "Smart Line Lead Qualifier"         [Test] [Commit] [v] |
|------------------------------------------------------------------|
| BLOCKS          |   CANVAS                                       |
| +-----------+   |                                                 |
| | TRIGGERS  |   |   +====================+                       |
| |  Inbound  |   |   || INBOUND MESSAGE  ||  ← Root (teal)       |
| |  Form     |   |   || Smart Line #s    ||                       |
| |  Stage    |   |   +=========+=========+                        |
| |  Webhook  |   |             |                                  |
| |  Schedule |   |   +---------v---------+                        |
| +-----------+   |   |  FETCH            |  ← Function (blue)    |
| | FUNCTIONS |   |   |  Phone # lookup   |                        |
| |  Branch   |   |   +---------+---------+                        |
| |  Delay    |   |             |                                  |
| |  AI       |   |   +---------v---------+                        |
| |  Fetch    |   |   |  BRANCH           |  ← Function (blue)    |
| |  Transform|   |   |  Known contact?   |                        |
| +-----------+   |   +----+--------+-----+                        |
| | ACTIONS   |   |        |        |                              |
| |  Send Msg |   |     YES|        |NO                            |
| |  Task     |   |   +----v---+ +--v--------+                    |
| |  Assign   |   |   |ASSIGN  | |CONV. ASK  |  ← Action (green) |
| |  Route    |   |   |To rep  | |"What      |                    |
| |  Notify   |   |   +----+---+ |brings you"|                    |
| |  Pool     |   |        |     +--+--------+                    |
| +-----------+   |   +----v---+    |                              |
|                 |   |NOTIFY  | +--v--------+                    |
| [MiniMap ]      |   |Via Novu| |AI CLASSIFY |                    |
| [Vars    ]      |   +-------+ |Score lead  |                    |
| [Logs    ]      |             +--+---------+                    |
| [Templates]     |                |                              |
|                 |   +------------v--------+                      |
|                 |   |  BRANCH             |                      |
|                 |   |  Score >= 80?       |                      |
|                 |   +---+-------+---------+                      |
|                 |    YES|       |NO                              |
|                 |   +---v---+ +-v--------+                      |
|                 |   |CREATE | |ROUTE     |                      |
|                 |   |VAULT  | |To SDR    |                      |
|                 |   |AE     | |Pool      |                      |
|                 |   +---+---+ +----------+                      |
|                 |       |                                        |
|                 |   +---v-------+                                |
|                 |   |SEND MSG   |                                |
|                 |   |"Sarah     |                                |
|                 |   |will be in |                                |
|                 |   |touch..."  |                                |
|                 |   +-----------+                                |
+------------------------------------------------------------------+
| Run Log | Last: 4m ago | 12 contacts today | 0 errors | v2.3    |
+------------------------------------------------------------------+
```

### The Sidebar (BotGhost's 6-Section Model, Adapted)

| Section | BotGhost | Airlock |
|---------|---------|--------|
| 1. Block Library | Actions, conditions, options | Triggers, Functions, Actions (categorized, searchable, draggable) |
| 2. Variables | Custom variable viewer/editor | Workflow scope inspector: trigger_data, contact, vault, variables |
| 3. Errors | Red icon, error logs | Execution errors + validation warnings (red/yellow badges) |
| 4. List | Command/event list | Workflow list (all workflows in this category) |
| 5. Timed Events | Scheduled task manager | Schedule node config (cron, daily, interval) |
| 6. Templates | Reusable block templates | Template library (pre-built + user-saved segments) |

---

## Node Types (The Building Blocks)

Inspired by BotGhost's block categories + Knock's 8 function types + HubSpot's action categories + Salesforce's Flow elements. Organized into three categories: Triggers, Functions, and Actions.

### Triggers (Entry Points)

Every workflow starts with exactly one trigger. The trigger defines what event activates the workflow.

| Node | Icon | Description | Config |
|------|------|-------------|--------|
| **Inbound Message** | Message bubble | iMessage, SMS, email, or web chat received | Channel filter, sender filter, keyword match |
| **Form Submitted** | Clipboard | Web form, conversational intake, or webhook | Form ID, field mapping |
| **Stage Change** | Arrow right | Vault lifecycle_stage or pipeline stage changes | From stage, to stage, module filter |
| **Task Created** | Check square | New task appears in universal task table | Task type, module, severity filter |
| **Schedule** | Clock | Time-based: daily, weekly, or cron expression | Schedule config, timezone |
| **Record Updated** | Pencil | Vault field changes, health score changes | Field name, old/new value conditions |
| **Webhook** | Zap | External system sends HTTP POST | URL path, payload schema, auth |
| **Manual** | Hand | User clicks "Run Workflow" button | Requires user_id context |
| **Attachment Received** | Paperclip | File received via any channel | MIME type filter, size filter |
| **Threshold Crossed** | Gauge | Score crosses a defined boundary | Score field, threshold value, direction (up/down) |

### Functions (Logic & Data)

Functions manipulate data and control flow. They don't produce external side effects.

| Node | Icon | Description | Config |
|------|------|-------------|--------|
| **Branch** | Git branch | If/else conditional routing (up to 10 paths) | Conditions using workflow scope data, AND/OR operators |
| **Delay** | Timer | Pause execution for a duration | Duration (minutes/hours/days), or "until" a date field |
| **Batch** | Stack | Aggregate multiple trigger events into one | Batch key, window duration, max batch size |
| **Throttle** | Speed limit | Rate-limit executions per recipient | Max per time window |
| **Fetch** | Globe | HTTP request to enrich data from external API | URL (with Liquid templating), method, headers, response path |
| **AI Classify** | Brain | Run AI model on input data | Prompt template, model selection, output schema, confidence threshold |
| **AI Generate** | Sparkles | Generate content using AI | Prompt template, model, output format (text/JSON) |
| **Transform** | Shuffle | Map/reshape data in workflow scope | Field mapping, expressions, computed values |
| **Loop** | Repeat | Iterate over a list in workflow scope | Collection path, variable name, max iterations |
| **Experiment** | Flask | A/B test with percentage-based cohort split | Variant names, percentages |

### Actions (Side Effects)

Actions produce external effects — they create records, send messages, notify people.

| Node | Icon | Description | Config |
|------|------|-------------|--------|
| **Send Message** | Send | Send via Smart Line (iMessage > SMS > email fallback) | Recipient, channel preference, template, media attachments |
| **Create Task** | Plus square | Create task in universal task table | Task type, title template, severity, module, role chain assignment |
| **Create Vault** | Folder plus | Create a new vault (lead, deal, account) | Vault type, parent, field mapping from workflow scope |
| **Update Record** | Save | Update vault fields, lifecycle_stage, tags | Record type, field mapping, merge strategy |
| **Assign** | User plus | Assign to user, pool, or round-robin group | Assignment strategy (direct/pool/round-robin/territory), user/group ID |
| **Notify** | Bell | Send notification via Novu | Channel (push/in-app/email/Slack), template, urgency |
| **Route to Pool** | Users | Route conversation to Heymarket-style shared inbox | Pool ID, assignment rules, priority |
| **Start Workflow** | Play | Trigger another workflow (nested) | Workflow ID, data passthrough |
| **Set SLA** | Stopwatch | Start an SLA countdown timer | Duration, escalation workflow on expiry |
| **Log Event** | File text | Write to audit trail / communications table | Event type, payload template |
| **Conversational Ask** | Message circle | Ask the contact a question, wait for response | Question text, expected response type, timeout, follow-up workflow |

---

## Workflow Execution Model

### How a Workflow Runs

```
1. TRIGGER fires (event received by BullMQ)
      |
2. Workflow definition loaded from DB (JSON)
      |
3. Execution context created:
      {
        trigger_data: { ... },      // The event payload
        contact: { ... },           // Resolved contact (if applicable)
        vault: { ... },             // Linked vault (if applicable)
        user: { ... },              // Acting user (if applicable)
        workspace: { ... },         // Workspace config
        variables: {},              // Mutable state set by Transform/Fetch nodes
      }
      |
4. Walk the node graph:
      - Each node reads from context, may write to context
      - Branch nodes evaluate conditions against context
      - Action nodes execute side effects
      - Delay nodes suspend execution (BullMQ delayed job)
      - Conversational Ask nodes suspend and wait for response event
      |
5. Execution log written:
      - Every node visited, with input/output snapshot
      - Branch decisions recorded (which path, why)
      - Action results captured (success/failure)
      - Total duration, node count
```

### Execution States

| State | Description |
|-------|-------------|
| `running` | Actively processing nodes |
| `waiting` | Paused at a Delay or Conversational Ask node |
| `completed` | All paths finished successfully |
| `failed` | A node threw an error (with retry info) |
| `cancelled` | Manually cancelled or superseded |

### Retry & Error Handling

- **Fetch nodes:** 3 retries with exponential backoff (1s, 5s, 30s). On final failure, take the "error" output path if defined, otherwise fail the workflow.
- **AI nodes:** 2 retries. On failure, log the error and continue with a default classification (configurable).
- **Action nodes:** 1 retry. On failure, create a triage task for manual resolution.
- **Entire workflow:** If uncaught error, mark as `failed`, create an alert in Admin Feature Control Plane.

### Versioning & Environments

Following Knock's model:

| Concept | How It Works |
|---------|-------------|
| **Draft** | Edits in the visual builder are auto-saved as drafts |
| **Commit** | User clicks "Commit" to snapshot the current version |
| **Environments** | Dev → Staging → Production promotion path |
| **Active/Inactive** | Toggle workflows on/off without changing version |
| **Test Runner** | Run a workflow against sample data before committing |
| **Rollback** | Revert to any previous committed version |

---

## The Visual Builder UI

### Canvas Layout

Built on React Flow (`@xyflow/react` v12) using their Workflow Editor template as the starting point.

```
+------------------------------------------------------------------+
| Workflow: "Lead Qualification"                    [Test] [Commit] |
|------------------------------------------------------------------|
| NODES          |                                                 |
| +----------+   |    +--[Trigger]--+                             |
| | Triggers |   |    | Inbound    |                             |
| |  Message |   |    | Message    |                             |
| |  Form    |   |    +-----+------+                             |
| |  Stage   |   |          |                                     |
| |  Webhook |   |    +-----v------+                             |
| +----------+   |    | AI Classify |                             |
| | Functions|   |    | Intent     |                             |
| |  Branch  |   |    +-----+------+                             |
| |  Delay   |   |          |                                     |
| |  Fetch   |   |     +----+----+                               |
| |  AI      |   |     |         |                               |
| +----------+   |  +--v---+  +--v---+                           |
| | Actions  |   |  |Sales |  |Supp. |                           |
| |  Send    |   |  |Branch|  |Branch|                           |
| |  Task    |   |  +--+---+  +--+---+                           |
| |  Assign  |   |     |         |                               |
| |  Notify  |   |  +--v---+  +--v---+                           |
| |  Route   |   |  |Assign |  |Route |                           |
| +----------+   |  |to Rep |  |to    |                           |
|                |  +--+---+  |Pool  |                           |
| [MiniMap]      |     |      +--+---+                           |
|                |  +--v---+     |                               |
|                |  |Create |  +--v---+                           |
|                |  |Task   |  |Notify|                           |
|                |  +------+  +------+                           |
+------------------------------------------------------------------+
| Execution Log: Last run 2m ago | 3 contacts processed | 0 errors |
+------------------------------------------------------------------+
```

### Node Sidebar (Left Panel)

- Categorized: Triggers / Functions / Actions
- Drag from sidebar onto canvas
- Search/filter nodes by name
- Recently used nodes at top
- Custom nodes (user-defined) in separate section

### Node Editor (Right Panel — opens on node click)

When you click a node on the canvas, a detail panel slides in from the right:

```
+--------------------------------+
| AI Classify                    |
| Intent Classification          |
+--------------------------------+
| PROMPT TEMPLATE                |
| "Classify the following        |
|  message intent:               |
|  {{trigger_data.body}}         |
|                                |
|  Categories: sales, support,   |
|  legal, billing, general"      |
+--------------------------------+
| MODEL         [Claude Haiku v] |
| CONFIDENCE    [0.7        ]    |
+--------------------------------+
| OUTPUT MAPPING                 |
| intent    -> variables.intent  |
| confidence -> variables.conf   |
+--------------------------------+
| CONDITIONS (skip this step if) |
| [+ Add condition]              |
+--------------------------------+
| TEST                           |
| [Run with sample data]         |
| Last result: "sales" (0.92)    |
+--------------------------------+
```

### Branch Editor (Special Node — Inline Conditions)

```
+----------------------------------+
| Branch: Route by Intent          |
+----------------------------------+
| PATH 1: "Sales"                  |
|   intent == "sales"              |
|   AND confidence >= 0.7          |
|   [+ Add condition]             |
+----------------------------------+
| PATH 2: "Support"               |
|   intent == "support"            |
|   [+ Add condition]             |
+----------------------------------+
| PATH 3: "Legal"                  |
|   intent == "legal"              |
|   [+ Add condition]             |
+----------------------------------+
| DEFAULT: "General"              |
|   (no conditions — catch-all)    |
+----------------------------------+
| [+ Add Path] (max 10)           |
+----------------------------------+
```

### Conversational Ask Node (The Discord Funnel)

This is the unique node that enables the conversational intake pattern:

```
+----------------------------------+
| Conversational Ask               |
| "What brings you here?"          |
+----------------------------------+
| QUESTION                         |
| "What brings you to [Company]?   |
|  I can help with:                |
|  1. Contract management          |
|  2. Deal flow & pipeline         |
|  3. General inquiry"             |
+----------------------------------+
| EXPECTED RESPONSE                |
| Type: [Free text     v]          |
|       [Multiple choice]          |
|       [Number        ]           |
|       [Yes/No        ]           |
+----------------------------------+
| AI PARSE RESPONSE                |
| [x] Use AI to classify response |
| Categories: contracts, deals,    |
|             general              |
+----------------------------------+
| TIMEOUT                          |
| Wait: [24 hours] then:           |
| [Send reminder    v]             |
|  "Still interested? Just reply   |
|   and we'll pick up where we     |
|   left off!"                     |
+----------------------------------+
| OUTPUT                           |
| response -> variables.interest   |
| parsed   -> variables.category   |
+----------------------------------+
```

When this node executes:
1. Sends the question via the active channel (iMessage, SMS, web chat)
2. Suspends the workflow execution
3. Waits for the contact's response (or timeout)
4. On response: AI parses it, stores in variables, continues to next node
5. On timeout: executes the timeout action (reminder, escalate, or end)

This is what turns a static form into a conversation. Chain multiple Conversational Ask nodes together and you get a Discord-style onboarding flow that qualifies and routes dynamically.

---

## Pre-Built Workflow Templates

Ship these as starter templates that admins can clone and customize:

### 1. Lead Qualification (Conversational)

```
Trigger: Inbound Message (any channel)
  → AI Classify (intent + sentiment)
  → Branch: Is this a known contact?
      YES → Fetch (load vault context)
           → Branch: What's the intent?
               Sales → Assign to account owner, Create follow-up task
               Support → Route to support pool, Create support task
               Legal → Route to legal team, Set SLA (4hr)
      NO → Conversational Ask: "What brings you to [Company]?"
          → Conversational Ask: "How many [items] do you manage?"
          → Conversational Ask: "Who should we follow up with?"
          → AI Classify (score lead from conversation)
          → Branch: Lead score
              High (80+) → Create vault, Assign to AE, Notify manager
              Medium (50-79) → Create vault, Route to SDR pool
              Low (<50) → Create vault, Enroll in nurture sequence
          → Send Message: "Thanks! [Rep name] will be in touch within [SLA]"
```

### 2. Smart Line Router

```
Trigger: Inbound Message (Smart Line number)
  → Fetch (phone # lookup against vault_contacts)
  → Branch: Contact found?
      YES → Branch: Has assigned rep?
              YES → Route to rep inbox, Notify via Novu
              NO → Route to account pool
      NO → AI Classify (is this spam, legitimate, or unknown?)
          → Branch: Classification
              Spam → Log, do not respond
              Legitimate → Start "Lead Qualification" workflow
              Unknown → Conversational Ask: "Hi! Who am I speaking with?"
                       → Create vault_contact from response
                       → Route to general intake pool
```

### 3. Attachment Handler

```
Trigger: Attachment Received (any channel)
  → AI Classify (attachment type from MIME + filename)
  → Branch: Attachment type
      PDF/Document →
          Branch: Is it a contract? (AI content analysis)
              YES → Create vault (type: contract), Start extraction pipeline
              NO → Attach to vault, Create review task
      Image →
          Log to communications, Attach to active conversation
      Contact Card (vCard) →
          Parse vCard fields, Create/update vault_contact
      Voice Memo →
          Transcribe (Whisper API), Log transcript, Create follow-up task
      Other →
          Attach to vault, Notify rep: "Received file: [filename]"
```

### 4. Deal Stage Automation

```
Trigger: Stage Change (CRM Pipeline)
  → Branch: New stage
      Prospecting →
          Create task: "Schedule discovery call"
          Create task: "Research account background"
          Set SLA: 48hr response
      Discovery →
          Create task: "Map stakeholder org chart"
          Create task: "Identify decision maker"
          Conversational Ask (to rep): "What's the estimated deal value?"
          Update vault: deal_value from response
      Proposal →
          Create task: "Send proposal"
          Create task: "Follow up on proposal (3 days)"
          Notify manager if deal_value > $100K
      Negotiation →
          Create task: "Get pricing approval"
          Branch: deal_value > $50K?
              YES → Create task: "Legal review required" (gated)
              NO → Skip legal review
      Close →
          Branch: Won or Lost?
              Won → Start "Customer Onboarding" workflow
              Lost → Conversational Ask: "Loss reason?"
                    → Log reason, Update vault, Archive
```

### 5. Customer Onboarding

```
Trigger: Deal stage = "Won"
  → Create tasks (checklist):
      "Provision Smart Line number"
      "Send welcome packet"
      "Schedule kickoff call"
      "Assign customer success manager"
      "Set up recurring check-in (monthly)"
  → Send Message: Welcome message via Smart Line
  → Update vault: lifecycle_stage = 'customer'
  → Notify team: "New customer: [account name]!"
  → Delay: 7 days
  → Conversational Ask (to customer): "How's the onboarding going? Any questions?"
  → Branch: Positive/Negative sentiment
      Positive → Log, continue onboarding
      Negative → Create escalation task, Notify CSM
```

### 6. Notification Workflow (Knock-style)

```
Trigger: Any event
  → Batch (5-minute window, group by recipient + event_type)
  → Branch: Urgency
      Urgent → Deliver immediately
      Normal → Delay 5 minutes (batching window)
  → Branch: Recipient preferences
      Has iMessage → Send via Smart Line
      Has push token → Send push notification
      Has email → Send email
      Fallback → In-app notification only
  → Log delivery status
```

---

## The Pool Model (Heymarket-Inspired)

### What It Is

A pool is a group of agents who share a single queue. Messages/leads entering the pool are visible to all pool members. Any member can claim and respond.

### Data Model

```sql
CREATE TABLE pools (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id),
    name TEXT NOT NULL,                    -- "Sales Team", "Support", "Legal"
    description TEXT,
    assignment_strategy TEXT NOT NULL DEFAULT 'manual',
    -- manual: agents claim from queue
    -- round_robin: auto-assign in rotation
    -- least_busy: assign to agent with fewest active conversations
    -- territory: assign based on geographic/segment rules
    max_concurrent INT DEFAULT 10,        -- max active conversations per agent
    sla_minutes INT,                       -- response time SLA
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE pool_members (
    pool_id UUID NOT NULL REFERENCES pools(id),
    user_id UUID NOT NULL REFERENCES users(id),
    role TEXT NOT NULL DEFAULT 'member',   -- owner | admin | member
    is_available BOOLEAN DEFAULT true,     -- presence/availability toggle
    active_count INT DEFAULT 0,            -- current active conversations
    PRIMARY KEY (pool_id, user_id)
);

CREATE TABLE pool_conversations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    pool_id UUID NOT NULL REFERENCES pools(id),
    communication_id UUID NOT NULL REFERENCES communications(id),
    vault_id UUID REFERENCES vaults(id),
    contact_id UUID REFERENCES vault_contacts(id),
    assigned_to UUID REFERENCES users(id), -- NULL = unassigned (in queue)
    status TEXT NOT NULL DEFAULT 'open',    -- open | claimed | resolved | escalated
    priority TEXT NOT NULL DEFAULT 'normal', -- low | normal | high | urgent
    claimed_at TIMESTAMPTZ,
    resolved_at TIMESTAMPTZ,
    internal_notes JSONB DEFAULT '[]',     -- private comments (agent-only)
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Pool UX in the CRM Inbox

```
+------------------------------------------------------------------+
| CRM INBOX                                          [Pool: Sales v]|
|------------------------------------------------------------------|
| QUEUE (3 unclaimed)        | CONVERSATION                        |
| +------------------------+ | +----------------------------------+|
| | * Nova Entertainment   | | | Jack Chen · Nova Entertainment   ||
| |   "Need help with      | | | Business Dev · +1 (555) 123-4567||
| |    contract terms"     | | | via iMessage · 12m ago           ||
| |   12m ago · UNCLAIMED  | | |----------------------------------||
| +------------------------+ | | Jack: Need help with the contract||
| | * TechFlow Inc         | | |   terms for the Q2 renewal.     ||
| |   "Pricing question"   | | |                                  ||
| |   28m ago · UNCLAIMED  | | | AI Summary: Renewal inquiry,    ||
| +------------------------+ | |   Q2 contract, needs terms review||
| |   Drift Audio          | | |                                  ||
| |   "Invoice request"    | | | [Claim] [Assign to...] [Escalate]||
| |   45m ago · UNCLAIMED  | | |                                  ||
| +------------------------+ | | INTERNAL NOTE:                   ||
|                            | | Sarah: I handled their last      ||
| MY CONVERSATIONS (2)       | |   renewal, I'll take this one    ||
| +------------------------+ | |                                  ||
| |   Acme Inc             | | | [Add internal note...]           ||
| |   Ana Chen · Claimed   | | +----------------------------------+|
| |   "Thanks for the..."  | |                                     |
| +------------------------+ |                                     |
| |   Summit Media         | |                                     |
| |   David Park · Claimed | |                                     |
| |   "Sounds good, let's" | |                                     |
| +------------------------+ |                                     |
+------------------------------------------------------------------+
```

### Key Pool Features

| Feature | Description |
|---------|-------------|
| **Claim** | Agent clicks "Claim" to take ownership. Conversation moves from queue to "My Conversations" |
| **Presence** | Green dot = available, yellow = busy, gray = offline. Only available agents receive new assignments in auto-assign modes |
| **Internal notes** | Private comments visible only to pool members. Customer never sees these. Like Discord's staff-only channels |
| **Assignment** | Manual claim, or workflow node auto-assigns using round-robin/least-busy/territory strategy |
| **Escalation** | Agent clicks "Escalate" → conversation routes to a different pool (e.g., Sales → Legal) or directly to a manager |
| **SLA tracking** | Timer starts when conversation enters pool. Alerts fire at thresholds. Visible in Admin dashboard |
| **Multi-pool** | An agent can belong to multiple pools. Their inbox shows conversations from all pools |

---

## Communication → CRM: The Progressive Contact Pipeline

### iMessage as Primary Delivery Route

The user's vision: iMessage is the main client-facing channel. Once a lead qualifies, they get a dedicated Smart Line number. Every interaction enriches their vault. The number IS their entry point into the CRM dashboard.

```
PROSPECT FIRST CONTACT
    |
    v
[Message arrives on Smart Line pool number]
    |
    v
WORKFLOW: "New Contact Intake" runs
    |
    +-- Conversational qualification (Discord-style guided questions)
    +-- AI scores the lead from conversation context
    +-- Creates vault with lifecycle_stage = 'new' or 'mql'
    |
    v
QUALIFIED LEAD
    |
    v
[Assigned rep (or pool) works the lead]
    |
    +-- Every message logged in communications table
    +-- Every attachment classified and routed
    +-- Context builds: what they need, deal value, decision makers
    |
    v
DEAL CLOSED → CUSTOMER
    |
    v
[Provision dedicated Smart Line number]
    |
    +-- Customer gets their own iMessage number
    +-- Number maps to their vault in CRM
    +-- All conversations, attachments, voice memos logged
    +-- Rep sees full context before every interaction
    +-- AI summarizes recent activity, flags risks
```

### SendBlue-Style Dashboard: Number → CRM View

SendBlue's model is conversation-centric — you access contacts through their message threads. Airlock's CRM Inbox takes this concept and adds vault context:

```
+------------------------------------------------------------------+
| SMART LINE DASHBOARD                                              |
|------------------------------------------------------------------|
| LINES                     | CONVERSATION         | VAULT CONTEXT |
| +---------------------+   | +------------------+ | +-----------+ |
| | +1 (555) 100-0001   |   | | Jack Chen        | | | ACME INC  | |
| | Sales Pool          |   | | Nova Entertainment| | | Pipeline: | |
| | 3 active convos     |   | | +1 (555) 234-5678| | | Discovery | |
| +---------------------+   | +------------------+ | | Value:    | |
| | +1 (555) 100-0002   |   | | Jack: Need help  | | | $120K     | |
| | Support Pool        |   | | with the contract | | | Health:   | |
| | 1 active convo      |   | | terms for Q2...  | | | 78%       | |
| +---------------------+   | |                  | | +-----------+ |
| | +1 (555) 100-0003   |   | | [AI Summary]     | | | TASKS (3) | |
| | Acme Inc (dedicated) |   | | Renewal inquiry, | | | * Disc    | |
| | Last: 2h ago        |   | | Q2 contract,     | | |   call    | |
| +---------------------+   | | needs terms      | | | * Send    | |
| | +1 (555) 100-0004   |   | | review. Positive | | |   NDA     | |
| | Summit (dedicated)   |   | | sentiment.       | | | * Map org | |
| +---------------------+   | |                  | | +-----------+ |
|                           | | [Attachments]     | | | CONTACTS  | |
| FILTER:                   | | contract-q2.pdf   | | | Jack Chen | |
| [All] [Pool] [Dedicated]  | | logo-v3.png       | | | BD Lead   | |
| [Unread] [Urgent]         | |                  | | | Lisa Wong | |
|                           | | [Reply...]        | | | CFO       | |
|                           | +------------------+ | +-----------+ |
+------------------------------------------------------------------+
```

### Progressive Contact Enrichment (Intercom Pattern)

Every interaction enriches the vault. Following Intercom's Visitor → Lead → User lifecycle, but adapted for B2B:

| Stage | Trigger | Data Collected | Vault State |
|-------|---------|---------------|-------------|
| **Unknown** | First message received | Phone number, timestamp, channel | No vault yet (queue item) |
| **Identified** | Conversational Ask responses | Name, company, role, needs | vault created, lifecycle_stage = 'new' |
| **Qualified** | AI scoring workflow completes | Lead score, qualification tier, interest area | lifecycle_stage = 'mql' or 'sql' |
| **Prospect** | Rep accepts and begins working | Deal value estimate, decision makers, timeline | lifecycle_stage = 'opportunity' |
| **Customer** | Deal closed-won | Full account profile, contract details, success metrics | lifecycle_stage = 'customer', dedicated number provisioned |

### Communication Logging Schema

Every message, call, and file exchange becomes a row in the communications table:

```sql
CREATE TABLE communications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id),
    vault_id UUID REFERENCES vaults(id),
    contact_id UUID REFERENCES vault_contacts(id),

    -- Channel info
    channel TEXT NOT NULL,         -- imessage | sms | email | web_chat | phone_call
    direction TEXT NOT NULL,       -- inbound | outbound
    from_address TEXT NOT NULL,    -- phone number, email, etc.
    to_address TEXT NOT NULL,

    -- Content
    body TEXT,                     -- message text or call notes
    subject TEXT,                  -- email subject, null for messages
    ai_summary TEXT,               -- AI-generated summary
    ai_intent TEXT,                -- AI-classified intent
    ai_sentiment TEXT,             -- positive | neutral | negative

    -- Attachments (stored as JSONB array)
    attachments JSONB DEFAULT '[]',
    -- [{ id, filename, mime_type, size_bytes, classification, storage_url }]

    -- Threading
    thread_id UUID,               -- groups related messages
    reply_to_id UUID REFERENCES communications(id),

    -- Metadata
    smart_line_number TEXT,        -- which Smart Line handled this
    pool_id UUID REFERENCES pools(id),
    assigned_to UUID REFERENCES users(id),

    -- Status
    status TEXT NOT NULL DEFAULT 'received',
    -- received | read | responded | archived

    -- Workflow tracking
    workflow_run_id UUID REFERENCES workflow_runs(id),

    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    read_at TIMESTAMPTZ,
    responded_at TIMESTAMPTZ
);

CREATE INDEX idx_comms_vault ON communications(vault_id, created_at DESC);
CREATE INDEX idx_comms_contact ON communications(contact_id, created_at DESC);
CREATE INDEX idx_comms_thread ON communications(thread_id, created_at);
CREATE INDEX idx_comms_smart_line ON communications(smart_line_number, created_at DESC);
```

### What This Enables: The Full Context Loop

When a rep gets a notification that a customer texted their Smart Line:

1. **Rep opens CRM Inbox** → sees the message in the conversation thread
2. **Vault context panel** shows: deal stage, health score, recent tasks, contacts, last interaction summary
3. **AI summary** of the message: "Customer requesting Q2 renewal terms. Positive sentiment. Has attached a PDF (classified as: contract)."
4. **Attachment auto-routed**: The PDF is already in the vault's documents, with extraction pipeline running
5. **Communication history**: Full thread of all prior interactions across all channels
6. **Suggested actions**: "Create renewal task", "Schedule call", "Forward to legal"
7. **Rep responds** → response logged, AI updates context, next workflow step fires

Every message makes the vault smarter. Every attachment gets classified and routed. Every call gets logged. The CRM builds itself from conversations.

---

## iMessage Attachment Classification Pipeline

When a file arrives via iMessage, the workflow engine classifies and routes it:

### Detection Pipeline

```
Attachment arrives via iMessage bridge webhook
    |
    v
1. EXTRACT METADATA
   - mime_type from attachment table (e.g., "application/pdf")
   - uti from attachment table (e.g., "com.adobe.pdf")
   - filename from transfer_name
   - total_bytes for size classification
   - is_sticker flag
    |
    v
2. CLASSIFY BY MIME TYPE (fast, deterministic)
   - image/* → photo/screenshot
   - video/* → video clip
   - audio/x-caf OR audio/mp4 → voice memo
   - audio/* → audio file
   - application/pdf → document (needs further analysis)
   - text/vcard → contact card
   - text/* → text document
   - other → unknown file
    |
    v
3. DEEP CLASSIFICATION (AI-assisted, for ambiguous types)
   For PDFs:
   - Extract first 2 pages of text
   - AI classify: contract | invoice | report | proposal | agreement | other
   - Confidence score determines routing

   For images:
   - AI classify: screenshot | photo | graphic_design | document_scan
   - If document_scan → OCR → treat as document
    |
    v
4. ROUTE based on classification
   - Contract PDF → Create vault, start extraction pipeline
   - Invoice → Attach to vault, create payment task
   - Contact card → Parse vCard, create/update vault_contact
   - Voice memo → Transcribe, log transcript, create follow-up task
   - Photo/image → Attach to conversation, log to communications
   - Design graphic → Attach to vault's documents, notify relevant team
   - Unknown → Attach to conversation, create review task for rep
```

### Supported MIME Types Reference

| Content Type | MIME Type | UTI | Airlock Classification |
|-------------|-----------|-----|----------------------|
| JPEG image | `image/jpeg` | `public.jpeg` | photo |
| PNG image | `image/png` | `public.png` | photo/screenshot |
| GIF | `image/gif` | `com.compuserve.gif` | image |
| HEIC image | `image/heic` | `public.heic` | photo |
| PDF | `application/pdf` | `com.adobe.pdf` | document (sub-classify) |
| MP4 video | `video/mp4` | `public.mpeg-4` | video |
| MOV video | `video/quicktime` | `com.apple.quicktime-movie` | video |
| Voice memo | `audio/x-caf` | `com.apple.coreaudio-format` | voice_memo |
| M4A audio | `audio/mp4` | `public.mpeg-4-audio` | audio |
| vCard | `text/vcard` | `public.vcard` | contact_card |
| Plain text | `text/plain` | `public.plain-text` | text |

---

## Workflow Storage Schema

```sql
CREATE TABLE workflows (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id),
    name TEXT NOT NULL,
    description TEXT,
    category TEXT NOT NULL DEFAULT 'general',
    -- lead_qualification | communications | deal_automation |
    -- notification | onboarding | contract | custom

    -- The workflow definition (React Flow JSON)
    definition JSONB NOT NULL DEFAULT '{}',
    -- {
    --   nodes: [{ id, type, position, data: { config... } }],
    --   edges: [{ id, source, target, sourceHandle, targetHandle }]
    -- }

    -- Versioning
    version INT NOT NULL DEFAULT 1,
    status TEXT NOT NULL DEFAULT 'draft',  -- draft | active | inactive | archived
    published_at TIMESTAMPTZ,

    -- Trigger config (extracted from trigger node for indexing)
    trigger_type TEXT NOT NULL,             -- inbound_message | form_submitted | stage_change | etc.
    trigger_config JSONB NOT NULL DEFAULT '{}',

    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE workflow_versions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workflow_id UUID NOT NULL REFERENCES workflows(id),
    version INT NOT NULL,
    definition JSONB NOT NULL,
    committed_by UUID REFERENCES users(id),
    committed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    commit_message TEXT,
    UNIQUE(workflow_id, version)
);

CREATE TABLE workflow_runs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workflow_id UUID NOT NULL REFERENCES workflows(id),
    workflow_version INT NOT NULL,

    -- Context
    trigger_data JSONB NOT NULL DEFAULT '{}',
    contact_id UUID REFERENCES vault_contacts(id),
    vault_id UUID REFERENCES vaults(id),

    -- Execution
    status TEXT NOT NULL DEFAULT 'running',
    -- running | waiting | completed | failed | cancelled
    current_node_id TEXT,                   -- which node is currently executing
    variables JSONB NOT NULL DEFAULT '{}', -- mutable workflow state

    -- Execution log
    node_log JSONB NOT NULL DEFAULT '[]',
    -- [{ node_id, started_at, completed_at, input, output, status }]

    started_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at TIMESTAMPTZ,
    error TEXT
);

CREATE INDEX idx_workflow_runs_status ON workflow_runs(workflow_id, status);
CREATE INDEX idx_workflow_runs_contact ON workflow_runs(contact_id);
```

---

## Workflow Builder in the Admin Module

### Where It Lives

The workflow builder is a view within the Admin module (or accessible from any module that has workflow triggers):

```
Admin Module Sub-Panel:
+-------------------------------+
| ADMIN                         |
| > SYSTEM                      |
|   o Dashboard                 |
|   o Feature Control Plane     |
|   o Audit Log                 |
|                               |
| > WORKFLOWS                   |  ← NEW SECTION
|   o All Workflows        (12) |
|   o Lead Qualification    (3) |
|   o Communications        (4) |
|   o Deal Automation       (3) |
|   o Custom                (2) |
|                               |
| > PEOPLE                      |
|   o Members                   |
|   o Roles & Permissions       |
|   o Pools                     |  ← NEW
|                               |
| > SETTINGS                    |
|   o Workspace                 |
|   o Integrations              |
|   o Smart Lines               |  ← NEW (manage iMessage bridges)
+-------------------------------+
```

### Workflow List View

```
+------------------------------------------------------------------+
| WORKFLOWS                                        [+ New Workflow] |
|------------------------------------------------------------------|
| [All] [Lead Qual] [Comms] [Deal Auto] [Custom]                   |
|------------------------------------------------------------------|
| NAME                    | TRIGGER        | STATUS  | LAST RUN    |
|-------------------------|----------------|---------|-------------|
| Lead Qualification      | Inbound Msg    | Active  | 12m ago     |
| Smart Line Router       | Inbound Msg    | Active  | 3m ago      |
| Deal Stage Tasks        | Stage Change   | Active  | 1h ago      |
| Attachment Handler      | Attachment     | Active  | 45m ago     |
| Customer Onboarding     | Stage = Won    | Active  | 2d ago      |
| Stale Lead Cleanup      | Schedule       | Draft   | Never       |
| Notification Digest     | Any Event      | Active  | 5m ago      |
+------------------------------------------------------------------+
```

---

## Cross-Module Workflow Access

While workflows live in Admin for creation/editing, other modules surface them contextually:

| Module | Where Workflows Appear | What User Sees |
|--------|----------------------|---------------|
| **CRM** | Pipeline column headers | "Stage automation: 3 tasks auto-created" info tooltip |
| **CRM** | Inbox | "Routed by: Smart Line Router workflow" badge |
| **CRM** | Lead detail | "Qualification workflow: Step 3 of 5" progress indicator |
| **Contracts** | Generator | "Generation workflow: Clause validation active" |
| **Tasks** | Task detail | "Created by: Deal Stage Tasks workflow" source badge |
| **Home** | Activity feed | "Workflow 'Lead Qualification' processed 5 contacts today" |

---

## How This Connects to Existing Specs

| Existing Spec | Connection to Workflow Engine |
|--------------|------------------------------|
| **Task Lifecycle** (task-lifecycle-deep-dive.md) | Workflows replace hardcoded task_rules table. The "Deal Stage Change → Task Auto-Generation" section is now a workflow template |
| **CRM Comms Brainstorm** (crm-comms-brainstorm.md) | Smart Line Router is a workflow. AI triage is an AI Classify node. Heymarket pool is a Route to Pool action |
| **Patch Workflow** (PatchWorkflow/overview.md) | Patch approval chain is a specialized workflow. Could eventually be modeled as: Trigger(patch.submitted) → Assign(verifier) → Gate(approval) → Assign(admin) → Gate(approval) → Apply |
| **Notifications** (Notifications/overview.md) | Notification delivery is a workflow. Novu is the action provider. Batching, delays, channel selection are workflow nodes |
| **Universal Task System** (TaskSystem/overview.md) | Create Task is a workflow action node. Task automation rules become workflow templates |
| **CRM Views** (CRM/crm-views-design-spec.md) | Qualify view shows contacts mid-workflow. Pipeline shows deal cards with active workflow indicators |

---

## React Flow Integration Details

### Package Setup

```bash
npm install @xyflow/react zustand elkjs
```

### Custom Node Registration

```typescript
// workflow-builder/nodes/index.ts
import { type NodeTypes } from '@xyflow/react';
import { TriggerNode } from './TriggerNode';
import { BranchNode } from './BranchNode';
import { AIClassifyNode } from './AIClassifyNode';
import { SendMessageNode } from './SendMessageNode';
import { CreateTaskNode } from './CreateTaskNode';
import { AssignNode } from './AssignNode';
import { ConversationalAskNode } from './ConversationalAskNode';
import { DelayNode } from './DelayNode';
import { FetchNode } from './FetchNode';
import { RouteToPoolNode } from './RouteToPoolNode';

export const nodeTypes: NodeTypes = {
  trigger: TriggerNode,
  branch: BranchNode,
  ai_classify: AIClassifyNode,
  ai_generate: AIGenerateNode,
  send_message: SendMessageNode,
  create_task: CreateTaskNode,
  assign: AssignNode,
  conversational_ask: ConversationalAskNode,
  delay: DelayNode,
  fetch: FetchNode,
  route_to_pool: RouteToPoolNode,
  update_record: UpdateRecordNode,
  notify: NotifyNode,
  set_sla: SetSLANode,
  log_event: LogEventNode,
  start_workflow: StartWorkflowNode,
  transform: TransformNode,
  batch: BatchNode,
  throttle: ThrottleNode,
  loop: LoopNode,
  experiment: ExperimentNode,
};
```

### Workflow Definition JSON Example

```json
{
  "nodes": [
    {
      "id": "trigger-1",
      "type": "trigger",
      "position": { "x": 250, "y": 0 },
      "data": {
        "trigger_type": "inbound_message",
        "config": {
          "channels": ["imessage", "sms"],
          "filter": {}
        }
      }
    },
    {
      "id": "classify-1",
      "type": "ai_classify",
      "position": { "x": 250, "y": 120 },
      "data": {
        "prompt": "Classify intent: {{trigger_data.body}}",
        "model": "claude-haiku-4-5",
        "categories": ["sales", "support", "legal", "general"],
        "output_key": "intent"
      }
    },
    {
      "id": "branch-1",
      "type": "branch",
      "position": { "x": 250, "y": 240 },
      "data": {
        "paths": [
          {
            "name": "Sales",
            "conditions": [
              { "field": "variables.intent", "op": "equals", "value": "sales" }
            ]
          },
          {
            "name": "Support",
            "conditions": [
              { "field": "variables.intent", "op": "equals", "value": "support" }
            ]
          }
        ],
        "default_path": "General"
      }
    },
    {
      "id": "assign-1",
      "type": "assign",
      "position": { "x": 100, "y": 380 },
      "data": {
        "strategy": "round_robin",
        "pool_id": "sales-pool-uuid"
      }
    },
    {
      "id": "route-1",
      "type": "route_to_pool",
      "position": { "x": 400, "y": 380 },
      "data": {
        "pool_id": "support-pool-uuid",
        "priority": "normal"
      }
    }
  ],
  "edges": [
    { "id": "e1", "source": "trigger-1", "target": "classify-1" },
    { "id": "e2", "source": "classify-1", "target": "branch-1" },
    { "id": "e3", "source": "branch-1", "target": "assign-1", "sourceHandle": "path-0" },
    { "id": "e4", "source": "branch-1", "target": "route-1", "sourceHandle": "path-1" }
  ]
}
```

---

## Demo Priority

For the demo shell HTML prototype:

1. **Workflow List View** — Table of workflows with status badges, trigger types, last run times
2. **Workflow Canvas** — React Flow-style canvas with a few nodes connected (static HTML mockup)
3. **Lead Qualification template** — Pre-built workflow showing the conversational funnel
4. **Smart Line Router template** — Showing iMessage routing logic
5. **Pool Management** — Admin view for creating/managing agent pools

---

## Open Questions

### 1. Workflow builder access control
Who can create/edit workflows? Options:
- **Admin only** — safest, workflows are system configuration
- **Admin + Builder role** — Builders can create, Admins approve before activation
- **Any role with workflow permission** — most flexible, could be chaotic

**Recommendation:** Admin creates workflows. Builders can view and suggest edits (like patches). Gatekeepers approve workflow changes before they go live. This follows our existing governance pattern.

### 2. Workflow limits
How many active workflows per workspace? Knock has unlimited. HubSpot limits by plan tier. For POC: unlimited. For production: TBD based on BullMQ load.

### 3. Conversational Ask across channels
If a workflow starts via iMessage, can a Conversational Ask node send via email instead? Or must it stay on the same channel?

**Recommendation:** Stay on the originating channel by default. Allow explicit channel override in node config for cases like "Follow up via email if no response in 24h."

### 4. Workflow conflicts
What happens when two workflows trigger on the same event? Options:
- **All execute** — parallel execution, last-write-wins for record updates
- **Priority order** — workflows have priority, highest wins
- **First match** — first workflow whose conditions match executes, others skip

**Recommendation:** All execute by default (like Knock). Add priority ordering later if conflicts become a real problem.

### 5. Contract generator as workflow
Should the contract generator's template/clause selection be modeled as a workflow? The template → clause expansion → placeholder fill → preview pipeline maps naturally to workflow nodes. This would let admins customize generation pipelines per contract type.

**Recommendation:** Yes, eventually. For POC, keep the generator as dedicated code. Later, extract the pipeline into a workflow template category.

---

## Research-Backed Decisions (from Perplexity Deep Research)

> Full research saved in `perplexity-research.md`. These are the locked decisions based on findings.

### Workflow Execution: BullMQ + Postgres Hybrid

| Requirement | Implementation |
|---|---|
| Delayed step (wait 24h) | `queue.add('step', data, { delay: 86400000 })` |
| Human-in-the-loop pause (Conversational Ask) | Mark run as `waiting` in Postgres, no Redis job active. On user response: webhook loads run from Postgres, enqueues next step to BullMQ |
| Crash recovery | Bootstrap service queries Postgres for `running` status runs on startup, re-enqueues to BullMQ |
| Exactly-once execution | Unique `jobId` per workflow step + `removeOnComplete` |
| Versioning | Running workflows complete under original definition; new runs use updated version (Temporal patching pattern, simplified) |

**Why not Temporal?** ToolJet migrated FROM Temporal TO BullMQ because it simplified architecture and leveraged existing Redis. We already have BullMQ + Redis in our stack. The orchestration layer (state tracking, resume logic) lives in our Postgres `workflow_runs` table.

### React Flow: Confirmed Architecture

| Decision | Choice | Why |
|---|---|---|
| Layout engine | **ELKjs** (not dagre) | Official template uses it, better hierarchical layout |
| State management | **Zustand** with "No-Store-Actions" pattern | React Flow uses Zustand internally; separate stores for flow, config, and run state |
| Edge routing | **Step edges** (not bezier) | Cleaner for workflow diagrams, BotGhost uses similar straight-line connections |
| Stores | `flowStore` (nodes/edges), `configStore` (selected node), `runStore` (execution) | Prevents re-renders, clean separation |

### iMessage Bridge: Hard Limits

| Metric | Limit | Implication |
|---|---|---|
| Safe outbound/day/Apple ID | **100 messages** | POC: 1 Mac mini handles ~100 outbound/day |
| Recommended unique contacts/day | **50** | Don't blast — conversational use only |
| Spam reports to get flagged | **2-3** | CRITICAL: Make users message us first |
| Scaling to 1,000/day | **10+ Mac minis** (Mac farm) | Each Mac = 1 Apple ID = 1 number = ~100/day |
| chat.db polling | **Every 2 seconds** (read-only) | Don't use file watchers, use polling |
| Concurrency | **5 simultaneous sends** max | Messages.app crashes above this |

**Critical mitigation:** Make the user message us first. This eliminates the "Report Junk" link entirely. Our conversational intake funnel (web widget, QR code, "Text us" CTA) is designed to have the prospect initiate contact.

### AI Lead Scoring: Phased Approach

| Phase | When | Method | Min Data |
|---|---|---|---|
| **POC** | Day 1 | Rule-based weighted signals | 0 (manual weights) |
| **Phase 2** | 100+ labeled leads | Logistic regression | ~100 leads with outcomes |
| **Phase 3** | 500+ labeled leads | Gradient boosting (XGBoost) | 500+ leads |

**Conversation-based scoring features** (what our AI Classify node extracts):
- Response latency (how fast they reply)
- Message length + sentiment (longer, positive = higher)
- Question frequency (asking about pricing/timelines = high intent)
- Engagement recency (days since last message)
- Conversation depth (number of exchanges)
- Firmographic enrichment (company size, industry from domain)

### Pool Assignment: Balanced (Not Round-Robin)

| Decision | Choice | Why |
|---|---|---|
| Default assignment | **Balanced** (fewest active conversations) | Intercom's model — prevents overloading busy agents |
| Offline handling | **Queue** conversations, distribute on agent login | Front's pattern — never lose a message |
| Collision detection | **WebSocket presence** + composing indicators | "Agent X is viewing" + "Agent X is typing" |
| Reply protection | **Optimistic locking** on submit | Check if another reply was submitted since agent opened conversation |
| Presence | **Redis pub/sub** for real-time status | Green/yellow/gray dots, no polling |

### Communication Pipeline: Thin Adapters -> Single Queue

```
iMessage Bridge Webhook ──> iMessage Adapter ──┐
                                                |
Twilio SMS Webhook ────────> SMS Adapter ───────┤──> BullMQ "inbound-messages" queue
                                                |         |
SendGrid Inbound Parse ───> Email Adapter ──────┤         v
                                                |    Workflow Engine
Web Chat WebSocket ────────> Chat Adapter ──────┘    (classify, route, respond)
```

- Dedup: `(externalId, channel)` composite unique index
- Threading: phone pair for iMessage/SMS, References header for email, session ID for web chat
- New conversation: 24-hour inactivity window

### Contract Generation: Ironclad Pattern

**Adopt Ironclad's Global Clause Library model:**
- **Clause Types** = category labels (Indemnification, Limitation of Liability, etc.)
- **Clauses** = actual text under each type, centrally managed
- Workflows reference clause IDs with conditions (Branch node: "if value > $100K, include enhanced indemnity")
- Edit a clause once -> propagates to all workflow configurations

This maps directly to our workflow builder: Template Selector node -> Conversational Ask nodes (collect details) -> Branch nodes (conditional clauses) -> Transform node (merge data) -> Route to Pool (legal review).

---

## Deep Research Prompts for Perplexity

> **STATUS: COMPLETED** — All 7 prompts researched. Results saved in `perplexity-research.md`. Key decisions integrated above.

These were the original research questions:

### 1. Workflow Engine State Machines

> "How do enterprise workflow engines like Temporal.io, Apache Airflow, and Prefect handle long-running workflow state? Specifically: how do they persist execution state across restarts, handle parallel branches that need to merge, implement exactly-once execution guarantees, and manage workflow versioning when a running workflow's definition changes? Compare their approaches and recommend the simplest model for a BullMQ-based workflow engine that needs to support delayed steps (wait 24 hours) and human-in-the-loop pauses (wait for user response)."

**Why we need this:** Our Conversational Ask node suspends execution and waits for a response. That could be minutes or days. We need to persist workflow state and resume correctly.

### 2. React Flow Workflow Builder Architecture

> "Show me the complete architecture of a production workflow/automation builder built with React Flow (@xyflow/react v12). Include: custom node component patterns (how to make nodes with configurable forms inside them), edge routing algorithms (step edges vs bezier), sidebar drag-and-drop to canvas implementation, node configuration panel (right sidebar editor), workflow validation (checking for disconnected nodes, missing configs), and auto-layout with ELKjs. Reference real open-source implementations if possible."

**Why we need this:** We need the exact implementation pattern for the visual builder.

### 3. iMessage Bridge Reliability at Scale

> "For a Mac mini running as an iMessage bridge server (using AppleScript + chat.db SQLite monitoring): What are the failure modes? How do you handle: iMessage rate limits (how many messages per hour before Apple throttles or blocks?), chat.db file locking under high read volume, AppleScript execution timeouts, macOS auto-update interrupting the bridge, multiple Mac minis for redundancy (how to load-balance across bridges?), and monitoring/alerting when the bridge goes down? Are there any production deployments of iMessage bridges handling 1000+ messages/day?"

**Why we need this:** We're betting on iMessage as primary channel. Need to understand the ceiling.

### 4. AI Lead Scoring Models

> "How do modern AI lead scoring systems work? Compare: HubSpot's predictive lead scoring, Salesforce Einstein Lead Scoring, MadKudu, Clearbit Reveal, and 6sense. What data inputs do they use (behavioral, firmographic, technographic, intent)? What ML models work best for B2B lead scoring (logistic regression vs gradient boosting vs neural nets)? What's the minimum training data needed? How do you handle cold-start when you have <100 historical leads? Provide a practical schema for scoring leads from conversation data (iMessage/email threads)."

**Why we need this:** Our AI Classify node needs to score leads from conversation content. Need the right model.

### 5. Shared Inbox / Pool Architecture

> "Compare the technical architecture of shared inbox systems: Intercom, Front, HelpScout, Missive, and Heymarket. How do they handle: conversation assignment (round-robin vs skill-based vs load-balanced), agent presence/availability, collision detection (two agents replying simultaneously), conversation transfer between pools, SLA tracking and escalation, and analytics (response time, resolution time, CSAT). What's the optimal database schema for a multi-pool shared inbox with real-time presence?"

**Why we need this:** Our pool model needs production-grade assignment and collision detection.

### 6. Webhook-Driven Communication Pipelines

> "Design a webhook-driven communication pipeline that receives messages from multiple channels (iMessage bridge, Twilio SMS, email via SendGrid/Mailgun, web chat) and normalizes them into a single format. Include: webhook payload normalization across channels, deduplication (same message arriving twice), threading (grouping related messages into conversations), attachment handling (downloading, classifying, virus scanning), and rate limiting. How do platforms like Nango, Merge, or Unified.to handle multi-channel webhook normalization?"

**Why we need this:** We need one inbound pipeline that handles iMessage, SMS, email, and web chat identically.

### 7. Contract Generation Workflow

> "How do CLM (Contract Lifecycle Management) platforms like Ironclad, DocuSign CLM, and Juro implement contract generation workflows? Specifically: how do they handle template selection → clause library → conditional clause insertion → placeholder resolution → approval routing → signature collection? What does their workflow definition look like internally? How do they handle version control of templates vs generated documents? Can these generation pipelines be modeled as visual workflows (like Salesforce Flows)?"

**Why we need this:** Our contract generator could eventually be a workflow template. Need to understand the CLM pattern.

---

## Sources

### Workflow Builder UX
- [Knock Workflow Builder 3.0](https://knock.app/changelog/2024-10-17) — Rebuilt canvas with drag-and-drop, pan/zoom, conditions debugger
- [Knock Workflows Docs](https://docs.knock.app/designing-workflows/overview) — 8 function types, execution model, versioning
- [Knock Branch Function](https://docs.knock.app/designing-workflows/branch-function) — Up to 10 paths, 5 nesting levels
- [Knock Fetch Function](https://docs.knock.app/designing-workflows/fetch-function) — HTTP enrichment with Liquid templating
- [HubSpot Workflow Actions](https://knowledge.hubspot.com/workflows/choose-your-workflow-actions) — 250 branches, AI actions
- [Salesforce Flow Builder](https://www.salesforceben.com/introduction-salesforce-flow/) — Record-Triggered Flows, Decision elements
- [ActiveCampaign Automation](https://help.activecampaign.com/hc/en-us/articles/222921988) — Goals as checkpoints, multi-trigger

### React Flow
- [React Flow (xyflow)](https://reactflow.dev/) — 35K+ stars, 3.4M weekly downloads, MIT license
- [React Flow Workflow Editor Template](https://reactflow.dev/ui/templates/workflow-editor) — Zustand + ELKjs + shadcn/ui starter
- [Synergy Codes Workflow Builder](https://www.workflowbuilder.io) — Higher-level SDK on React Flow

### Communication Patterns
- [SendBlue API Docs](https://docs.sendblue.com/) — iMessage webhook payloads, attachment handling
- [SendBlue Contacts API](https://docs.sendblue.com/api/resources/contacts/) — E.164 mapping, custom variables
- [Intercom Visitor Lifecycle](https://www.intercom.com/help/en/articles/310) — Progressive identity: Visitor → Lead → User
- [Front Channels API](https://dev.frontapp.com/docs/welcome) — Cross-channel threading, contact resolution
- [HubSpot Conversations](https://knowledge.hubspot.com/inbox/get-started-with-conversations) — Unknown sender handling, unified inbox

### iMessage Technical
- [chat.db Schema Analysis](https://dev.to/arctype/analyzing-imessage-with-sql-f42) — SQLite tables, attachment storage
- [iMessage Attachments](https://linuxsleuthing.blogspot.com/2015/01/getting-attached-apple-messaging.html) — MIME types, UTIs, file handling
- [Apple UTI Reference](https://developer.apple.com/documentation/uniformtypeidentifiers) — Type identifiers for classification

### Discord Bot Builders (Visual Builder UX Reference)
- [BotGhost Command & Event Builder](https://botghost.com/docs/custom-commands-and-events/command-and-event-builder) — Tree-based canvas, yellow root block, blue actions, purple options, condition branching, 6-section sidebar
- [BotGhost Actions](https://botghost.com/docs/custom-commands-and-events/actions) — 10 action categories: Message, Variables, API, Loop, Voice, Role, Channel, Thread, Server, Other
- [BotGhost Events](https://botghost.com/docs/custom-commands-and-events/events) — 19 event trigger categories covering all Discord events
- [BotGhost Variables](https://botghost.com/docs/custom-commands-and-events/variables) — Object vars with dot notation, global/server/channel/user scoping, collection functions
- [BotGhost Advanced Options](https://botghost.com/docs/custom-commands-and-events/command-and-event-builder/advanced-options) — Block colors, keyboard shortcuts, multi-select, template saving, 100-step undo
- [Discord Bot Studio](https://discordbotstudio.org/) — Flowchart editor, command/response nodes, event nodes, code export
- [Discord Bot Studio Docs](https://docs.discordbotstudio.org/setting-up-dbs/getting-started-with-flow-based-bot-creating-with-dbs) — Flowchart connection mechanics, node types, visual programming model
- **[Bot Studio Code Analysis](./bot-studio-code-analysis.md)** — Full analysis of exported Bot Studio source code. Confirms JSON-workflow-with-sequential-executor pattern works. Key takeaways: dual-file system (canvas vs execution), `callNextAction()` chain pattern, `%%var%%` interpolation, `trueActions`/`falseActions` branching, in-memory delays (replace with BullMQ), file-based persistence (replace with Postgres).

### CRM Patterns
- [CRM Workflow Automation](https://databar.ai/blog/article/crm-workflows-the-complete-guide-to-automation-that-actually-works) — Trigger → Action patterns
- [Lead Management Workflows](https://huble.com/blog/the-5-workflows-for-successful-lead-management-in-hubspot) — Qualification patterns
- [AI Lead Scoring](https://readylogic.co/ai-lead-scoring-automate-lead-qualification-routing/) — Score-based routing
