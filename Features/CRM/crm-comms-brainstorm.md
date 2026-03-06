# CRM + Smart Communications

> **Status:** SPECCED
>
> **Context:** Expanding the CRM module beyond its current "vault lens" spec to include live deal flow, communications routing, and a progressive user journey. This document synthesizes existing Airlock specs, industry CRM patterns, and the iMessage/phone-first communications vision.

> **Stats that drive this thinking:**
> - 73% of B2B buyers prefer email as outreach channel
> - 83% prefer digital overall, but phone closes faster once a relationship exists
> - 57% of C-level prefer phone for strategic discussions
> - SMS: 45% response rate vs 6% for phone/email; 98% open rate; 90-second avg response
> - 41.2% of salespeople say phone is their most productive tool
> - Phone calls drive ~69% of business inquiries

---

## Problem: What Triggers a Deal?

The current spec handles **post-ingestion** beautifully: contracts upload, entity resolution fires, vaults populate the hierarchy, CRM is a lens. But we haven't addressed:

1. **How does a NEW deal enter the system?** (Not a batch ingest of existing contracts)
2. **How does an existing customer re-engage?** (They have a question, want to renew, send a new deal)
3. **How do multiple contacts within an org get mapped?** (Jack from Nova calls about one thing, Sarah from Nova calls about another)

### The Insight

Once you have a relationship with a customer, **the phone/text is the fastest path to action**. Email is for cold outreach and documentation. But real deals accelerate through conversations. The question is: how do you capture those conversations as CRM intelligence?

---

## The Smart Line: Dedicated Number as CRM Gateway

### Concept

Every parent vault gets a **dedicated line** — a phone number that becomes the single point of contact for that customer relationship. This number routes all inbound calls and texts through an AI triage layer before reaching a human.

**Day 1 (POC):** Your own iMessage number, connected via a Mac running an iMessage bridge server. Zero cost, real blue bubbles.

**Later (Scale):** Either provisioned via Twilio/Vonage API (SMS/MMS) or additional Mac minis with separate Apple IDs (iMessage at scale). Numbers generated on demand when a parent vault is created, included in the welcome packet / portal invite.

### How It Works

```
Customer texts dedicated line
    |
    v
AI Triage (Otto / Smart Router)
    |
    +-- Identifies caller (phone # lookup against vault contacts)
    |
    +-- Reads intent from message content
    |     "Hey, want to discuss the renewal terms" -> Legal / Deal Review
    |     "Can you send me the Q1 report?" -> Documents / Auto-response
    |     "Who handles billing?" -> Route to Finance contact
    |     "We want to expand our deal" -> Sales rep / New opportunity
    |
    +-- Routes to appropriate person:
    |     a) AI auto-response (FAQ, doc retrieval, status check)
    |     b) Route to assigned rep (relationship owner)
    |     c) Route to department (legal, finance, support)
    |     d) Human handoff ("Let me connect you with Sarah from legal...")
    |
    +-- Logs everything:
          - Message content -> vault event feed (Signal panel)
          - Caller identity -> CRM contact enrichment
          - Intent classification -> task creation if needed
          - Routing decision -> audit trail
```

### Why iMessage — The Direct API Approach

Apple's official "Messages for Business" channel requires registering through an approved Messaging Service Provider (MSP) like Zendesk or Sinch — a middleman we don't need. Instead, the open-source ecosystem has solved direct iMessage access via macOS bridge servers.

#### The Architecture: Mac as iMessage Server

A Mac mini (or any always-on Mac) runs a bridge server that exposes iMessage through a REST API. Your application talks to the bridge; the bridge talks to Messages.app.

```
Airlock Backend (FastAPI)
    |
    +--[REST API calls]--> iMessage Bridge Server (Mac mini)
    |                           |
    |                           +-- Reads ~/Library/Messages/chat.db (SQLite)
    |                           +-- Sends via AppleScript (no private APIs)
    |                           +-- Streams new messages via filesystem monitor
    |
    +--[Webhooks from bridge]--> Inbound message processing
```

#### Open-Source Options (Ranked)

| Project | Language | How It Works | Capabilities | Risk Level |
|---------|----------|-------------|-------------|------------|
| **imsg** (steipete) | Swift | AppleScript send + SQLite read | Send/receive, attachments, group chats, reactions (read-only), reply threading, E.164 normalization | Low (no private APIs) |
| **iMessage-API** (danikhan632) | Python/Flask | SQLite read + AppleScript send | Send/receive, filter by sent/received, contact integration from iCloud vCards | Low |
| **imessage-rs** | Rust | Dual path: AppleScript (safe) or Private API injection | 66+ API routes, webhooks, BlueBubbles-compatible REST API | Medium (private API path requires SIP disabled) |
| **textingblue** | iPhone shortcut | Routes through iPhone, not Mac | Send/receive, webhooks, real blue bubbles from your actual phone | Low (uses Shortcuts API) |

#### Why This Beats Apple's Official Channel

- **No MSP middleman** = no Zendesk/Sinch dependency, no per-message fees from an intermediary
- **No Apple registration** = no approval process, no business verification delay
- **Real iMessage** = blue bubbles, read receipts, typing indicators, end-to-end encryption
- **Attachments** = PDFs, images, files up to 100MB natively
- **Group threads** = add/remove people as deal progresses
- **Your own number** = customer texts a real phone number they already know, not a short code
- **Zero customer friction** = no app install, no portal login, just text

#### Risk: Apple's June 2026 Warning

Apple has indicated it may close private API injection loopholes in future macOS updates (flagged for June 2026). However:
- The **AppleScript path** (used by imsg and iMessage-API) doesn't use private APIs — it's the same mechanism Shortcuts uses
- The **SQLite read** path reads a standard database file — unlikely to break
- Worst case: we fall back to SMS via Twilio, which is already supported in the architecture
- The safe approach: use **only AppleScript + SQLite** (no SIP-disabled private API injection)

#### iMessage Rate Limits (Perplexity Research — March 2026)

Apple doesn't publish exact rules, but operational data is consistent across multiple sources (SendBlue, TextingBlue, myCRMSIM, Jared):

| Metric | Hard Limit | Source |
|---|---|---|
| Safe outbound/day per Apple ID | **100 messages** | TextingBlue, myCRMSIM |
| Recommended unique contacts/day | **50** | myCRMSIM |
| Observed block trigger | ~200/hr sustained for 3 hours | GitHub: ZekeSnider/Jared |
| Spam reports to get flagged | **2-3 reports** | TextingBlue |
| Blocking behavior | **Silent** — no error, no webhook, no warning | Multiple sources |
| Block duration | Can be **permanent** after repeated offenses | Apple Community |

**Critical mitigations:**
- **Make users message us first** — this eliminates the "Report Junk" link entirely. Our conversational intake (web widget, QR code, "Text us at..." CTA) is designed for prospect-initiated contact.
- **Vary message content** — no identical blasts, randomize delays between sends
- **Max 5 simultaneous sends** — Messages.app crashes above this concurrency
- **Poll chat.db every 2 seconds** in read-only mode — don't use file watchers (WAL lag causes false positives)
- **Monitor delivery** — detect when messages don't appear in chat.db after send (silent failure detection)
- **Disable macOS auto-update** on bridge Macs — auto-updates restart the machine and kill the bridge

**Scaling math:** To handle 1,000 outbound messages/day, you need ~10 Mac minis, each with its own Apple ID and number. SendBlue runs "Mac farms" at this scale. Load-balancing happens at the API routing layer, not the Mac level.

#### POC Path: Test With Your Own Number First

Before provisioning anything, the demo uses a real iMessage account:

```
Phase 0 (NOW):
  Your Mac (or Mac mini) running imsg or iMessage-API
      |
  Your iMessage number = the dedicated line
      |
  Airlock connects via localhost REST API
      |
  Inbound messages -> webhook -> Airlock CRM Inbox
  Outbound messages -> API call -> imsg -> iMessage
```

This validates the entire flow — AI triage, contact matching, routing, logging — without any third-party costs or provisioning. When ready to scale:

```
Phase 0: Your number (iMessage bridge, zero cost)
    |
Phase 1: Generated numbers via Twilio/Vonage (SMS/MMS, provisioned per parent vault)
    |
Phase 2: Multiple Mac minis with separate Apple IDs (iMessage at scale)
    |
Phase 3: Hybrid — iMessage for Apple users, SMS for Android, auto-detect
```

**Number generation (Phase 1+):** Airlock provisions a dedicated phone number per Parent Vault via Twilio API on demand. The number appears in the vault's settings, gets included in welcome packets, and routes through the same AI triage layer. Cost: ~$1-2/mo per number + $0.0079/SMS. This runs in parallel with the iMessage bridge — SMS for Android users and fallback, iMessage for Apple users via the bridge.

### The Power of the Org Tree Auto-Build

When Jack from Nova Entertainment calls the dedicated line:
1. System looks up the phone number -> no match
2. AI asks: "Hi, I'm the Airlock assistant for Nova Entertainment. Who am I speaking with?"
3. Jack identifies himself -> system creates a **Contact** under the Nova Entertainment vault
4. Jack says he's in Business Development -> system tags his role
5. Next time Jack calls, system knows him and routes directly

Over time, as different people from Nova call/text:
- The CRM builds an **org chart** automatically from interactions
- Each person's role and communication patterns inform routing
- The vault hierarchy grows organically: Nova (Parent) -> Jack (Contact, Biz Dev), Sarah (Contact, Legal), Mike (Contact, Finance)

---

## CRM Sub-Panel: Revised Structure

Previous version used chamber names (Discover/Build/Review/Ship) as section headers. After comparing with Salesforce's object model and standard B2B playbooks, the sub-panel should tell a **business story** instead. Chambers remain as the automation layer underneath.

### Key Insight: Separate Lead Lifecycle from Deal Pipeline

Salesforce and modern RevOps separate two distinct motions:
- **Lead lifecycle** = from first touch to conversion (Lead → qualified → Account+Contact+Opportunity)
- **Opportunity pipeline** = from opportunity creation to closed-won/lost

Our previous `Pipeline` view mixed both. The revised structure separates them.

### The Airlock Advantage Over Salesforce Here

Salesforce's separation creates a hard "conversion" boundary — a Lead literally becomes a different database object (Contact + Account + Opportunity). History fragments. Fields don't map cleanly. Teams write custom Apex just to handle conversion.

In Airlock, there's **no conversion event**. A vault starts in Discover and its `lifecycle_stage` advances. The "lead" and the "opportunity" are the same vault row the whole time. The CRM sub-panel sections are just filtered views of the same hierarchy at different stages. Zero data migration, zero broken history.

### Revised Sub-Panel

```
+-------------------------------+
| CRM                           |
| > Quick Stats                 |
|   42 Accounts                 |
|   8 New Leads                 |
|   12 Open Opportunities       |
|   3 At Risk                   |
|-------------------------------|
| > DISCOVER (Lead Lifecycle)   |
|   o Inbox               (5)  |  All inbound: texts, emails, calls, forms
|   o New Leads            (8)  |  Raw leads: New / Open / Unworked
|   o Qualify              (3)  |  MQL → SAL → SQL funnel view
|                               |
| > DEALS (Opportunity Pipeline)|
|   o Pipeline                  |  Kanban: Prospecting > Discovery > Proposal > Negotiation > Close
|   o Deal Review               |  Deals in final approval / legal / gatekeeper
|   o Won Deals                 |  Closed-won celebration + metrics
|                               |
| > ACCOUNTS                    |
|   o Accounts                  |  Company list (Parent vaults), health, segments
|   o Contacts                  |  People directory, org tree, roles
|   o Activity Feed             |  Cross-account activity stream
|                               |
| > CUSTOMERS (Post-Sale)       |
|   o Onboarding                |  Implementation, activation, Smart Line setup
|   o Health Monitor            |  Usage, tickets, payment status, risk flags
|   o Renewals                  |  Timeline, forecast, upcoming dates
|   o Expansion                 |  Upsell / cross-sell opportunities
+-------------------------------+
```

### Why This Structure

**DISCOVER** = the lead funnel. A first-time user starts here: "This is where new things come in."

1. **Inbox** = unified communications hub. Texts, emails, missed calls, form submissions. The entry point for everything.

2. **New Leads** = raw, unqualified entities from any source. Entity resolution below confidence threshold. Needs human review.

3. **Qualify** = the MQL/SAL/SQL funnel. Three qualification stages tracked in the data model:
   - **MQL** (Marketing Qualified) — meets automated criteria (engagement score, firmographic fit)
   - **SAL** (Sales Accepted) — rep/SDR agrees to work this lead
   - **SQL** (Sales Qualified) — rep confirms real buying intent (BANT/CHAMP confirmed)
   - On SQL, vault advances from Discover → Build. The "conversion" in Salesforce terms — but here it's just a chamber state change on the same record.

**DEALS** = the opportunity pipeline. Once qualified, leads become active deals.

4. **Pipeline** = Kanban board with 6 columns, each with clear entry/exit criteria:
   - **Prospecting** — first outreach attempted
   - **Discovery** — initial meeting held, needs identified
   - **Proposal** — terms/pricing shared with buyer
   - **Negotiation** — back-and-forth on specifics
   - **Close** — verbal yes, paperwork in progress
   - **Won/Lost** — terminal states
   Each column is gated: you only move from Discovery → Proposal when specific buyer actions happen (e.g., budget confirmed, decision-maker identified). Not just a vibe — discrete checks.

5. **Deal Review** = gatekeeper view. Deals in the Review chamber needing sign-off. Approval chain, SLA timers.

6. **Won Deals** = celebration + handoff. Win metrics, deal post-mortem, trigger onboarding sequence.

**ACCOUNTS** = the relationship layer. Who our customers are, who works there, what's happening.

7. **Accounts** = Parent + Division vaults viewed as a CRM list. Aggregate health scores, segments, enrichment status.

8. **Contacts** = Counterparty vaults + vault_contacts. Org tree built from communications. Roles, interaction frequency, last touch.

9. **Activity Feed** = cross-account chronological stream. Filterable by event type, account, person. Shows the "living" CRM.

**CUSTOMERS** = post-sale motion. This is where the Smart Line concierge lives.

10. **Onboarding** = new customer setup: provision Smart Line number, send welcome packet, assign team members, activation checklist.

11. **Health Monitor** = at-risk accounts. Declining health scores, missed SLAs, stale deals, support ticket spikes. AI-driven alerts.

12. **Renewals** = calendar-driven view of upcoming renewal dates (computed from contract extraction). Forecast revenue impact.

13. **Expansion** = upsell/cross-sell opportunities. Separate from new business pipeline — tracked as their own opportunity type. This is where enterprise SaaS growth happens.

### Data Model: Qualification Stages

The qualification funnel maps to a `lifecycle_stage` field on the vault:

```sql
ALTER TABLE vaults ADD COLUMN lifecycle_stage TEXT NOT NULL DEFAULT 'new';
-- Values: new | mql | sal | sql | opportunity | customer | churned

ALTER TABLE vaults ADD COLUMN qualification_score INT DEFAULT 0;
-- Composite score from engagement, firmographic fit, intent signals

ALTER TABLE vaults ADD COLUMN qualified_at TIMESTAMPTZ;
-- When the vault crossed from SQL → Opportunity (the "conversion" moment)
```

This is ONE field on the same vault row — not a separate Lead object that gets "converted" into an Opportunity object. The CRM sub-panel sections are just `WHERE lifecycle_stage IN (...)` filters.

### Chambers as Automation Layer (Not UI Labels)

The sub-panel sections (Discover/Deals/Accounts/Customers) are the **user-facing story**. Underneath, chambers still drive cross-module automation:

| Sub-Panel Section | Chamber | What Triggers |
|-------------------|---------|---------------|
| Discover: Inbox, New Leads | Discover | Entity resolution, lead scoring, AI classification |
| Discover: Qualify (SQL) | Discover → Build | Auto-create opportunity tasks, assign rep |
| Deals: Pipeline stages | Build | Contract creation tasks, proposal generation |
| Deals: Deal Review | Review | Gatekeeper notifications, SLA timers, legal review tasks |
| Deals: Won | Ship | Onboarding sequence, Smart Line provisioning, BullMQ events |
| Customers: all | (post-Ship) | Health scoring, renewal alerts, expansion triggers |

---

## Pipeline Stages: Mapped to Chambers

The existing spec maps pipeline to vault chambers. Here's the expanded mapping with deal flow stages:

| Pipeline Stage | Chamber | Gate | What Happens | CRM Task Created |
|---------------|---------|------|-------------|-----------------|
| **New Lead** | Discover | gate_ingest | Contact/entity detected from any source | "Qualify new lead" |
| **Qualified** | Discover | gate_triage | Human confirms: worth pursuing | "Schedule first contact" |
| **Contacted** | Build | gate_extract | First conversation logged (call/text/email) | "Send proposal" |
| **Proposal** | Build | gate_preflight | Terms shared, discussion ongoing | "Follow up on proposal" |
| **Negotiation** | Build | gate_enrich | Back-and-forth on specifics | "Prepare final terms" |
| **Legal Review** | Review | gate_gatekeeper | Contract drafted, legal reviewing | "Complete legal review" |
| **Verbal Close** | Review | gate_owner | Deal agreed, paperwork pending | "Generate contract" |
| **Closed-Won** | Ship | gate_export | Signed, done | "Onboard customer" |
| **Closed-Lost** | (archived) | — | Didn't happen | "Log loss reason" |

---

## Deal Entry Points (Triggers)

How deals enter the pipeline:

| Trigger | Source | CRM Action | Task Created |
|---------|--------|-----------|-------------|
| **Inbound text** | Customer texts dedicated line | AI identifies intent, creates lead or routes to rep | "Respond to inbound" |
| **Inbound call** | Customer calls dedicated line | Call logged, transcription processed, intent classified | "Follow up on call" |
| **Email forward** | Rep forwards email to system | Email parsed, entities extracted, deal created | "Qualify email lead" |
| **Web form** | Website contact/demo request | Form data ingested via webhook | "Qualify web lead" |
| **Contract upload** | Batch or single PDF upload | Entity resolution → vault creation → CRM population | "Review ingested entity" |
| **API webhook** | External system (Stripe, DocuSign, etc.) | Event triggers vault update or creation | "Process webhook event" |
| **Manual creation** | Rep creates deal from CRM | New counterparty vault created with deal metadata | — |
| **Referral** | Existing customer introduces new contact | New lead created with referral source tag | "Qualify referral" |

### POC Priority

For the demo, focus on:
1. **Manual creation** (immediate, no integration needed)
2. **Contract upload** (already built)
3. **Webhook** (simple API endpoint)
4. **Inbound text** (the showcase feature)

---

## Communications Layer Architecture

### Phase 0: Your Number (iMessage Bridge — NOW)

```
Your Mac (mini or laptop) running imsg / iMessage-API
    |
    +-- Messages.app signed into your iMessage account
    |
    +-- Bridge server exposes REST API (localhost or LAN)
    |     GET  /messages          -> fetch conversation history
    |     POST /send              -> send iMessage
    |     GET  /recent_contacts   -> list active conversations
    |     WS   /stream            -> real-time new message events
    |
    +-- Airlock Backend (FastAPI) connects to bridge:
    |
    |   Inbound flow:
    |     Bridge detects new message (filesystem monitor on chat.db)
    |         -> webhook to Airlock /api/comms/inbound
    |         -> phone # lookup against vault_contacts
    |         -> AI intent classification (Otto)
    |         -> route to rep or auto-respond
    |         -> log to communications table
    |         -> notify via Novu
    |
    |   Outbound flow:
    |     Rep types reply in Airlock CRM Inbox
    |         -> POST to bridge /send
    |         -> bridge sends via AppleScript -> Messages.app
    |         -> message appears as real iMessage (blue bubble)
    |         -> log to communications table
```

### Phase 1: Webhooks + Manual Ingest

```
External Event (email forward, web form, API webhook)
    |
    +--[POST /api/comms/inbound]--> Normalize to communications record
    |                               channel = 'email' | 'webhook' | 'form'
    |
    +--[Entity resolution]--> Match to vault contact or create new lead
    |
    +--[BullMQ job]--> Create task in universal task table
    |                  module_type = 'crm', task_type = 'inbound'
    |
    +--[Route]--> CRM Inbox view
    |
    +--[Novu]--> Notify assigned rep
```

### Phase 2: Generated Numbers + SMS Fallback

```
Parent Vault created -> admin clicks "Provision Smart Line"
    |
    +--[Twilio API]--> Number provisioned, stored in vault_phone_numbers
    |                  E.164 format, $1-2/mo
    |
    +--[Twilio webhook config]--> Inbound SMS/MMS -> POST /api/comms/inbound
    |
    +--[Same routing pipeline as Phase 0]:
    |     Phone # lookup -> AI classify -> route -> log -> notify
    |
    +--[Channel detection]:
    |     Apple device? -> route through iMessage bridge (blue bubbles)
    |     Android/other? -> route through Twilio SMS (green bubbles)
    |     Email? -> route through email pipeline
    |
    +--[Knock-style fallback chain]:
          Try iMessage -> if undelivered after 60s -> fall back to SMS -> log channel used
```

### Phase 3: Smart Router + Human Handoff

```
AI (Otto) is handling inbound conversation
    |
    +--[Smart Router rules (Knock-style workflow)]:
    |     Rule 1: Known contact + assigned rep -> route to rep's Airlock inbox
    |     Rule 2: Unknown contact -> AI greeting + capture name/role/intent
    |     Rule 3: Simple question -> AI auto-response (FAQ, doc retrieval, status)
    |     Rule 4: Complex/urgent -> escalate to human with full context
    |     Rule 5: Department request -> route to dept queue
    |     Rule 6: After-hours -> AI acknowledges + creates task for next morning
    |
    +--[Human handoff]:
    |     AI detects: "I need to talk to someone about the contract terms"
    |     Otto: "Let me connect you with Sarah from our legal team"
    |     System: Forward thread to Sarah's Airlock inbox
    |             Sarah sees full conversation history
    |             Sarah's responses go through the same dedicated line
    |             Customer never sees Sarah's personal number
    |
    +--[Heymarket-style pool features]:
    |     Multiple agents can view the same conversation
    |     Presence indicators: "Sarah is viewing this thread"
    |     Private comments: agents discuss internally without customer seeing
    |     Assignment: drag conversation to another agent
    |
    +--[Audit trail]:
          Every routing decision logged with reason
          Handoff logged: Otto -> Sarah (Legal), reason: contract terms
          SLA timer starts on human handoff

---

## Companies to Study

### Heymarket — The Pool Model (Key Reference)

Heymarket's shared inbox is the closest analog to what we're building. Their model:

**One number, many agents:**
- A shared phone number is created for a team (support, sales, etc.)
- Every team member sees every message in that inbox
- Any team member can reply — responses go out from the shared number
- Customers never see individual rep numbers

**Assignment + Collaboration:**
- Messages can be **assigned** to specific team members
- Team members can see when someone else is viewing or typing a reply (presence)
- **Private comments** let agents @ mention teammates within a thread without the customer seeing
- Chats can be tagged, categorized, and marked as completed

**Multi-Inbox Architecture:**
- Admins create one inbox per department, per region, or per employee
- Full control over which teammates access each inbox
- Maps directly to our model: one inbox per Parent Vault, with team members assigned per vault

**Omnichannel:**
- Single shared inbox receives from SMS, MMS, Facebook Messenger, WhatsApp, Apple Business Messages, and email
- All channels funnel into one conversation thread per contact

**What we take from this:**
- The "pool" model: multiple agents behind one number, with assignment and presence
- Private comments for internal collaboration within a customer thread
- Per-department inbox creation maps to per-vault Smart Lines
- Omnichannel convergence into a single thread per contact

### Knock — The Routing Infrastructure (Key Reference)

Knock is notification infrastructure, not a CRM — but their routing model is exactly what Airlock's Smart Line needs on the backend.

**Workflow Orchestration:**
- Define workflows that determine which channels messages route to
- Apply functions before delivery: **batch** (collapse multiple notifications into one), **throttle** (rate limit), **delay** (pause execution)
- Fallback sequences: if push fails, try SMS, then email — platform handles channel-specific formatting

**Channel Groups:**
- Combine multiple providers into a single logical channel (e.g., APNs + FCM = "Push")
- Maps to our model: iMessage + SMS + email = "Smart Line" — one logical channel, multiple underlying transports

**Multi-Tenant:**
- Built for SaaS with tenant isolation
- Environments, version control, CLI-first developer workflow

**What we take from this:**
- The workflow-as-routing-rule pattern: define once, route everywhere
- Batching to prevent notification fatigue (collapse 5 messages into a digest)
- Fallback chains: iMessage -> SMS -> email -> in-app notification
- Channel groups: abstract away the transport layer so the CRM doesn't care if it's iMessage or SMS

### Other Companies

| Company | What They Do | What We Can Learn |
|---------|-------------|------------------|
| **Quo (formerly OpenPhone)** | Shared phone numbers, CRM-integrated calling/texting. $105M raised. Auto-logs calls + SMS to HubSpot/Salesforce. | The "shared number" model. How they handle routing. |
| **Salesmsg** | Two-way SMS/MMS with HubSpot integration. Campaign automation. | Sales-specific texting workflows. Sequence automation. |
| **Intercom** | AI-first customer platform. Fin AI agent handles triage. | How Fin routes conversations. Resolution vs escalation logic. |
| **Knock AI** | AI SDR that engages on Slack/LinkedIn/WhatsApp, detects intent, qualifies, routes. | Intent detection patterns. Qualification logic. |

---

## Data Model Additions

### `vault_phone_numbers` Table

```sql
CREATE TABLE vault_phone_numbers (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vault_id UUID NOT NULL REFERENCES vaults(id),  -- Parent vault (level 1)
    phone_number TEXT NOT NULL UNIQUE,               -- E.164 format
    provider TEXT NOT NULL DEFAULT 'imessage_bridge',   -- imessage_bridge | twilio | vonage
    provider_sid TEXT,                                -- Provider's ID (Twilio SID, bridge instance ID)
    bridge_host TEXT,                                  -- For iMessage bridge: hostname/IP of Mac running bridge
    bridge_port INT,                                   -- For iMessage bridge: port of REST API
    is_personal BOOLEAN DEFAULT false,                 -- true = owner's personal number (Phase 0)
    status TEXT NOT NULL DEFAULT 'active',            -- active | suspended | released
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### `vault_contacts` Table (extends vault hierarchy)

```sql
CREATE TABLE vault_contacts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vault_id UUID NOT NULL REFERENCES vaults(id),     -- Counterparty vault (level 3)
    name TEXT NOT NULL,
    email TEXT,
    phone TEXT,                                         -- E.164 format
    role TEXT,                                          -- "Business Development", "Legal", "Finance"
    title TEXT,                                         -- "VP of Partnerships"
    is_primary BOOLEAN DEFAULT false,
    source TEXT NOT NULL DEFAULT 'manual',              -- manual | entity_resolution | inbound_comms
    first_interaction TIMESTAMPTZ,
    last_interaction TIMESTAMPTZ,
    interaction_count INT DEFAULT 0,
    metadata JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### `communications` Table

```sql
CREATE TABLE communications (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    workspace_id UUID NOT NULL REFERENCES workspaces(id),
    vault_id UUID REFERENCES vaults(id),               -- Parent vault this relates to
    contact_id UUID REFERENCES vault_contacts(id),     -- Who communicated

    -- Channel
    channel TEXT NOT NULL,                              -- sms | imessage | call | email | webhook
    direction TEXT NOT NULL,                            -- inbound | outbound

    -- Content
    from_number TEXT,
    to_number TEXT,
    subject TEXT,                                       -- For email
    body TEXT,                                          -- Message content or call transcript
    media_urls JSONB DEFAULT '[]',                     -- Attachments

    -- AI Processing
    intent_classification TEXT,                         -- sales | support | legal | general | unknown
    intent_confidence FLOAT,
    sentiment TEXT,                                     -- positive | neutral | negative
    summary TEXT,                                       -- AI-generated summary

    -- Routing
    routed_to UUID REFERENCES users(id),              -- Which team member received this
    routing_reason TEXT,                                -- Why they were chosen
    ai_handled BOOLEAN DEFAULT false,                  -- Was this fully handled by AI?

    -- Metadata
    provider_sid TEXT,                                  -- Twilio message SID, etc.
    duration_seconds INT,                              -- For calls

    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_comms_vault ON communications(vault_id, created_at DESC);
CREATE INDEX idx_comms_contact ON communications(contact_id, created_at DESC);
```

---

## Progressive Disclosure: The First-Time User Journey

### Step 1: Empty State (Day 1)

User opens CRM module for the first time. They see:

```
+-------------------------------------------+
|  Welcome to CRM                           |
|                                           |
|  Your CRM is built from your vault        |
|  hierarchy. As you ingest contracts and   |
|  interact with customers, relationships   |
|  build automatically.                     |
|                                           |
|  Start by:                                |
|  [1] Upload contracts (auto-populates)    |
|  [2] Create an account manually           |
|  [3] Connect a communication channel      |
|                                           |
+-------------------------------------------+
```

### Step 2: First Account (Day 1-2)

After uploading contracts or creating an account:
- Sub-panel populates with the account in Active Vaults
- Pipeline shows the first card
- Quick Stats light up: "1 Account, 1 Active Deal"

### Step 3: First Communication (Day 3+)

After setting up a dedicated line or forwarding an email:
- Inbox view gets its first item
- Contact auto-created from caller/sender
- Activity Feed shows the conversation
- The "aha moment": CRM is filling itself from interactions

### Step 4: Pattern Recognition (Week 2+)

- Health Monitor starts showing trends
- Pipeline Kanban has cards moving through stages
- Org tree grows as more contacts interact
- AI starts suggesting: "Jack from Nova hasn't been contacted in 14 days"

---

## What to Demo

For the demo shell, the CRM module should showcase:

1. **Pipeline Kanban** — 5-column board with draggable deal cards, showing the deal flow
2. **Account Detail** — Click an account, see the vault hierarchy: parent > divisions > counterparties > items, with aggregate health
3. **Contact Directory** — Org tree view showing people mapped to roles
4. **Inbox** — Unified feed of inbound communications (mocked texts, emails, calls)
5. **Smart Line Demo** — Mock conversation showing AI triage, intent detection, and routing
6. **Activity Timeline** — Cross-account activity stream with filterable events

### Demo Priority Order

1. Pipeline (everyone expects it, validates the core flow)
2. Account Detail (shows vault hierarchy as CRM)
3. Inbox (showcases the communications vision)
4. Activity Timeline (shows the "living" CRM)
5. Contact Directory (shows the auto-built org tree)
6. Smart Line Demo (the wow factor)

---

## Open Questions — Resolved

### 1. Number provisioning model

**Answer: One number per parent vault, generated on demand.**

Phase 0: Your personal iMessage number covers all vaults (single-tenant POC).
Phase 1+: Each parent vault gets its own number. Provisioned via Twilio when an admin clicks "Provision Smart Line" in vault settings. At $1-2/mo per number, a workspace with 50 parent vaults costs $50-100/mo for numbers alone — trivial at enterprise scale.

For departments within a vault, use **Heymarket's pool model** instead of separate numbers: one number, multiple agents assigned, with routing rules determining who gets each message.

### 2. iMessage vs SMS

**Answer: iMessage first via direct bridge (no Apple Business Chat needed), SMS as fallback.**

The open-source iMessage bridge approach (imsg, iMessage-API) bypasses Apple's MSP requirement entirely. You run a Mac with Messages.app and expose a REST API. No registration, no Apple Business Chat approval, no per-message intermediary fees.

The safe path uses only AppleScript for sending + SQLite for reading — no private APIs, no SIP disabling. If Apple ever restricts this, SMS via Twilio is the fallback, already wired into the same pipeline.

**Channel detection strategy:**
- Check if recipient has iMessage (bridge can detect blue vs green delivery)
- Apple device? -> iMessage via bridge
- Android/unknown? -> SMS via Twilio
- Both fail? -> email fallback

### 3. Call handling

**Answer: Start with call logging + callback (Phase 0-1), add live routing later (Phase 3+).**

Phase 0-1: If someone calls the Smart Line, it goes to voicemail. Voicemail transcription (Twilio or Whisper API) creates a communications record, triggers a task, and notifies the assigned rep. Rep calls back from Airlock.

Phase 3+: Live call routing via Twilio Programmable Voice. AI answers, classifies intent, either handles it or transfers to a human. Complex but proven (Intercom's Fin does this). Not needed for POC.

### 4. AI response boundaries

**Answer: Otto auto-responds to informational queries only. Anything with commitment authority requires a human.**

| Can Auto-Respond | Requires Human |
|-----------------|----------------|
| "What's the status of my contract?" | "I want to change the terms" |
| "Can you send me the Q1 report?" | "We need to discuss pricing" |
| "Who handles billing?" | "I want to cancel" |
| "What are your office hours?" | "Let's set up a meeting" |
| "I need the latest invoice" | "We're interested in expanding" |

The boundary: **read-only questions auto-respond; anything that could create, modify, or delete a relationship requires human involvement.** This maps to our evidence-first UI principle — Otto surfaces evidence and routes to a human who makes the decision.

### 5. Compliance

**Answer: Build compliance into the architecture from day one.**

- **SMS opt-in (TCPA):** When a parent vault is created and a number provisioned, the welcome packet includes opt-in language. First outbound SMS includes required opt-out instructions ("Reply STOP to unsubscribe"). Store consent timestamps in vault_contacts.
- **Call recording consent:** Don't record calls in Phase 0-1. When we add live routing (Phase 3+), play a consent disclosure at the start. Two-party consent states (CA, FL, etc.) require explicit agreement. Store consent in communications record metadata.
- **iMessage:** End-to-end encrypted by Apple. No consent issue for message content — it's a direct conversation. Logging message content server-side (in our communications table) should be disclosed in the customer agreement.
- **GDPR/data retention:** Communications table gets a retention policy. Auto-archive after X days (configurable per workspace). Right-to-erasure: DELETE FROM communications WHERE contact_id = $1.

### 6. Cost model

**Answer: Phase 0 is free. Phase 1+ is negligible at enterprise scale.**

| Component | Phase 0 Cost | Phase 1+ Cost |
|-----------|-------------|--------------|
| iMessage bridge | $0 (your Mac) | $500-800 one-time (Mac mini per high-volume vault) |
| Twilio number | $0 | $1-2/mo per parent vault |
| Twilio SMS | $0 | $0.0079/outbound SMS |
| Twilio voice (Phase 3) | $0 | $0.0085/min inbound, $0.014/min outbound |
| AI intent classification | API costs | ~$0.003 per message (Claude Haiku) |

**Example at scale:** 50 parent vaults, 500 messages/day, 20 calls/day:
- Numbers: $100/mo
- SMS: $119/mo
- Voice: $51/mo
- AI: $45/mo
- **Total: ~$315/mo** — trivial for enterprise SaaS

The iMessage bridge handles Apple-to-Apple messages at zero marginal cost (only the Mac hardware). SMS costs only apply to Android recipients and fallbacks.

---

## Google Meet SDK Integration

> **Added:** 2026-03-04 — Video calling built into CRM with AI-powered meeting intelligence.

### The Vision

Click a button on a vault's detail page → launch a Google Meet call → AI transcribes the call in real-time → action items auto-populate in the task board → meeting summary appears as a vault event.

No switching apps. No manually copying notes. The call happens inside Airlock's context, and the intelligence flows directly into the vault's Signal feed and the universal task system.

### Why Google Meet SDK

Google provides the [Google Meet REST API](https://developers.google.com/meet/api/guides/overview) and [Meet Add-ons SDK](https://developers.google.com/meet/add-ons/guides/overview) for embedding meeting functionality:

| Capability | API | Notes |
|-----------|-----|-------|
| Create meetings programmatically | Meet REST API | Generate meeting link from vault context |
| Embed Meet in web app | Meet Add-ons SDK | Side panel or main stage integration |
| Real-time transcription | Meet REST API (transcripts) | Access transcripts after meeting ends |
| Recording | Meet REST API (recordings) | Auto-record with consent |
| Participant management | Meet REST API | Track who joined from which vault |

### User Flow

```
Vault Detail Page (CRM or Contracts)
    |
    [Start Call] button in Control panel
    |
    v
Google Meet launches (embedded or new tab)
    |
    +-- Meeting linked to vault_id
    |
    +-- Otto AI joins as silent observer (optional)
    |     - Real-time transcription via Google's API
    |     - Intent detection running in background
    |     - Action item extraction from conversation
    |
    +-- Meeting ends
    |
    v
Post-Meeting Processing
    |
    +-- Full transcript saved to vault events
    |     event_type: 'meeting.completed'
    |     payload: { transcript, duration, participants, recording_url }
    |
    +-- AI generates meeting summary
    |     event_type: 'ai.meeting_summary'
    |     payload: { summary, key_decisions, open_questions }
    |
    +-- Action items extracted → tasks created
    |     "Jack will send the revised schedule by Friday"
    |     → Task: "Send revised schedule" assigned to Jack, due Friday
    |     → Module: CRM, linked to vault
    |
    +-- CRM contact enrichment
          Participants mapped to vault_contacts
          Interaction count incremented
          Last interaction timestamp updated
```

### Architecture

```
Airlock Frontend
    |
    +-- Google Meet SDK (embedded iframe or popup)
    |     - OAuth consent for Google Workspace
    |     - Meeting created via Meet REST API
    |     - Recording + transcription enabled
    |
    +-- FastAPI Backend
    |     - POST /api/meetings/create { vault_id, participants }
    |     - Webhook from Google: meeting ended
    |     - GET transcript via Meet REST API
    |     - Process with Otto (summary + action items)
    |     - Create vault events + tasks
    |
    +-- Otto AI Agent
          - Receives full transcript
          - Tool: create_task (from action items)
          - Tool: summarize_meeting
          - Tool: update_contact (enrichment)
```

### Meeting Intelligence Features

| Feature | Description | Source |
|---------|-------------|--------|
| **Auto-transcription** | Full meeting transcript saved to vault | Google Meet API |
| **AI summary** | 3-5 bullet point summary of key decisions | Otto (post-meeting) |
| **Action item extraction** | "X will do Y by Z" → task creation | Otto (post-meeting) |
| **Contact enrichment** | New participants → vault_contacts, role detection | Otto + entity resolution |
| **Sentiment tracking** | Was the customer positive/negative? Trends over time | Otto analysis |
| **Follow-up scheduling** | "Let's meet next week" → Calendar event | Otto + Calendar module |
| **Deal stage advancement** | Meeting outcome suggests pipeline stage change | Otto suggestion → human confirm |

### Data Model

```sql
CREATE TABLE meetings (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vault_id UUID NOT NULL REFERENCES vaults(id),
    workspace_id UUID NOT NULL REFERENCES workspaces(id),
    google_meet_id TEXT,                    -- Google's meeting ID
    meeting_url TEXT,                       -- meet.google.com/xxx-yyy-zzz
    title TEXT,
    scheduled_at TIMESTAMPTZ,
    started_at TIMESTAMPTZ,
    ended_at TIMESTAMPTZ,
    duration_seconds INT,
    recording_url TEXT,                     -- Google Drive link (if recorded)
    transcript TEXT,                        -- Full transcript
    summary TEXT,                           -- AI-generated summary
    action_items JSONB DEFAULT '[]',        -- Extracted action items
    participants JSONB DEFAULT '[]',        -- [{name, email, vault_contact_id}]
    metadata JSONB DEFAULT '{}',
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_meetings_vault ON meetings(vault_id, started_at DESC);
```

### Prerequisites

- Google Workspace account with Meet enabled
- Google Cloud project with Meet API enabled
- OAuth consent screen configured for Airlock
- Connectors panel: "Connect Google Workspace" (existing Google Drive connector extends)

### Feature Control Plane

| Flag | Default | Description |
|------|---------|-------------|
| ENABLE_GOOGLE_MEET | false | Google Meet integration |
| ENABLE_MEETING_TRANSCRIPTION | true | Auto-transcribe meetings |
| ENABLE_MEETING_RECORDING | false | Auto-record (requires consent) |
| ENABLE_ACTION_ITEM_EXTRACTION | true | AI extracts action items from transcript |

### Privacy & Compliance

- Recording requires explicit consent from all participants (enforced by Google Meet's recording disclosure)
- Transcripts stored in Airlock's database, subject to workspace retention policy
- Action item extraction is opt-in per workspace
- Participants are informed that Otto AI is observing (transparency requirement)
- GDPR: right to erasure applies to meeting records

---

## iMessage Bridge Provider Lock — DECIDED: imsg (Swift)

After evaluating all 4 open-source options, **imsg (steipete)** is the locked provider choice:

| Criterion | imsg | iMessage-API | imessage-rs | textingblue |
|-----------|------|-------------|------------|-------------|
| Language | Swift | Python/Flask | Rust | iPhone Shortcuts |
| Private APIs | No | No | Optional (risky) | No |
| Attachment support | Full | Partial | Full | Limited |
| Reply threading | Yes | No | Yes | No |
| E.164 normalization | Yes | Manual | Yes | Manual |
| Active maintenance | Yes (2026) | Stale (2024) | Active | Active |
| Group chats | Yes | No | Yes | No |
| Reactions (read) | Yes | No | Yes | No |

**Why imsg wins:**
1. Swift = native macOS, best Messages.app integration
2. No private APIs = safe from Apple lockdowns
3. Reply threading = enables proper conversation mapping to communications table
4. E.164 normalization = clean phone number matching against vault_contacts
5. Active maintenance = steipete is a known iOS/macOS developer

**Deployment:**
```bash
# On Mac mini bridge server
brew install imsg
imsg serve --port 8765 --webhook-url https://api.airlock.app/webhooks/imessage
```

**Airlock integration:**
- FastAPI route `/webhooks/imessage` receives inbound messages
- Outbound: POST to `http://bridge-mac:8765/api/send`
- Health check: GET `http://bridge-mac:8765/api/health` every 30s
- Feature Control Plane shows bridge status (green/amber/red)

---

## Agent Presence System

Real-time presence tracking for pool assignment and collision prevention.

### Architecture

Uses Redis pub/sub for real-time status broadcasting. No polling.

```
Agent opens CRM Inbox → WebSocket connects → presence published to Redis
    |
    +-- Redis channel: `presence:{workspace_id}`
    +-- Payload: { user_id, status, pool_ids, active_count, last_seen }
    |
Other agents' browsers receive presence updates via their WebSocket
    |
    +-- UI shows: green dot (online), yellow (busy), gray (away/offline)
```

### Presence States

| State | Indicator | Assignment Eligible | Trigger |
|-------|-----------|-------------------|---------|
| **Online** | Green dot | Yes | CRM Inbox tab is active |
| **Busy** | Yellow dot | Reduced priority | active_count >= max_concurrent |
| **Away** | Gray dot | No | Tab inactive > 5 minutes, or manual toggle |
| **Offline** | No dot | No | WebSocket disconnected |
| **Do Not Disturb** | Red dot | No | Manual toggle |

### Collision Prevention

When two agents view the same unclaimed conversation:

1. **Viewing indicator**: "Sarah is viewing" badge appears on conversation card
2. **Composing indicator**: "Sarah is typing..." shows in the conversation thread
3. **Claim lock**: When agent clicks "Claim", optimistic lock check:
   - If conversation is still unclaimed → claim succeeds
   - If another agent claimed in the last 2 seconds → show "Alex already claimed this. View their response?"
4. **Reply lock**: On reply submit, check if another reply was submitted since the agent opened the conversation
   - If yes → show "Alex sent a reply while you were composing. [View their reply] [Send anyway]"

### Redis Schema

```
# Presence hash (per user, per workspace)
HSET presence:{workspace_id}:{user_id} status "online" pools "pool1,pool2" active_count 3 last_seen 1709640000

# Presence channel (broadcasts)
PUBLISH presence:{workspace_id} '{"user_id":"uuid","status":"online","active_count":3}'

# Conversation viewers (TTL: 30s, refreshed while viewing)
SETEX conversation_viewer:{conversation_id}:{user_id} 30 "1"

# Composing indicator (TTL: 5s, refreshed while typing)
SETEX composing:{conversation_id}:{user_id} 5 "1"
```

---

## Feature Control Plane Integration

### Communication Toggles

| Flag | Default | Description |
|------|---------|-------------|
| `comms.imessage_bridge_enabled` | true | iMessage bridge integration |
| `comms.sms_enabled` | false | Twilio SMS fallback |
| `comms.email_enabled` | false | Email inbound/outbound |
| `comms.web_chat_enabled` | false | Web chat widget |
| `comms.google_meet_enabled` | false | Google Meet integration |
| `comms.ai_auto_response_enabled` | true | Otto auto-responds to read-only queries |
| `comms.ai_intent_classification` | true | AI classifies message intent |

### Calibration

| Key | Default | Range | Description |
|-----|---------|-------|-------------|
| `comms.max_outbound_per_day` | 100 | 10-500 | Per-bridge iMessage daily limit |
| `comms.max_simultaneous_sends` | 5 | 1-10 | Concurrent Messages.app sends |
| `comms.chat_db_poll_interval_ms` | 2000 | 500-5000 | chat.db polling frequency |
| `comms.follow_up_delay_hours` | 24 | 1-72 | Wait before auto follow-up |
| `comms.max_follow_ups` | 2 | 0-5 | Max outbound without reply |
| `comms.inactivity_timeout_minutes` | 5 | 1-30 | Presence "away" threshold |
| `comms.conversation_window_hours` | 24 | 1-72 | Inactivity before new thread |

### Health Monitoring

| Metric | Alert Threshold | Description |
|--------|----------------|-------------|
| Bridge connectivity | > 60s disconnected | iMessage bridge health check failed |
| Message delivery rate | < 90% in 1hr window | Too many delivery failures |
| Avg response time (pool) | > SLA duration | Pool agents not responding fast enough |
| Queue depth | > 20 unclaimed | Too many conversations waiting for agents |
| AI classification accuracy | < 80% human-override rate | AI intent classification unreliable |

---

## Related Specs

- **[CRM Module](./overview.md)** — Current CRM spec (vault lens model)
- **[Vault Hierarchy](../VaultHierarchy/overview.md)** — The data model CRM views
- **[Universal Task System](../TaskSystem/overview.md)** — CRM tasks backed by master table
- **[Tasks Module](../Tasks/overview.md)** — Universal work queue
- **[Notifications](../Notifications/overview.md)** — Alert system
- **[Event Bus](../EventBus/overview.md)** — Cross-module event propagation
- **[Meeting Intelligence](../MeetingIntelligence/overview.md)** — Full meeting lifecycle spec (builds on Google Meet SDK section above)
- **[Video Call Integration](../Messenger/video-call-integration.md)** — Initiating calls from Messenger conversations

---

## Sources

### CRM & Pipeline
- [B2B Sales Pipeline Management 2026](https://prospeo.io/s/b2b-sales-pipeline-management) — Pipeline stages and KPIs
- [B2B Sales Pipeline Stages Guide](https://www.default.com/post/b2b-sales-pipeline) — Stage definitions
- [UX/UI Design for B2B Sales Software](https://www.neuronux.com/post/ux-ui-design-for-b2b-sales-software) — Progressive disclosure in CRM

### iMessage API Ecosystem
- [iMessage-API (danikhan632)](https://github.com/danikhan632/iMessage-API) — Python/Flask iMessage bridge server
- [imsg (steipete)](https://github.com/steipete/imsg) — Swift CLI for Messages.app, AppleScript send + SQLite read
- [textingblue/imessage-api](https://github.com/textingblue/imessage-api) — iPhone-based iMessage REST API via Shortcuts
- [The iMessage API Revolution (HackMD)](https://hackmd.io/@ZURhi6cqSnWHVFu2MCiTpg/BJJXqdqd-l) — Ecosystem overview, bridge architecture, risk analysis
- [Apple Messages for Business REST API](https://register.apple.com/resources/messages/msp-rest-api/) — Official Apple MSP API (for reference, not our approach)

### Companies Studied
- [Heymarket Shared Inbox](https://www.heymarket.com/product/shared-inbox/) — Pool model, multi-agent shared number
- [Heymarket Omnichannel Inbox](https://www.heymarket.com/product/omnichannel-shared-inbox/) — SMS + iMessage + WhatsApp convergence
- [Heymarket Team Chat Management](https://help.heymarket.com/hc/en-us/articles/115003762291-Managing-Chats-with-a-Team) — Assignment, presence, private comments
- [Knock Notification Infrastructure](https://knock.app/manuals/notification-infrastructure/introduction-to-notification-infrastructure) — Workflow orchestration, channel routing
- [Knock Workflows](https://docs.knock.app/concepts/workflows) — Batching, delays, fallback chains
- [Knock Channel Groups](https://knock.app/manuals/notification-infrastructure/understanding-notification-channels) — Multi-provider channel abstraction
- [Quo (formerly OpenPhone)](https://www.quo.com/features/crm-with-text-messaging) — CRM with text messaging
- [Knock AI SDR](https://www.knock-ai.com/blog/ai-sdr-tools) — Intent detection, smart lead routing
- [AI-Powered CRM Solutions 2026](https://www.kustomer.com/resources/blog/ai-powered-crm-solutions/) — AI routing patterns
