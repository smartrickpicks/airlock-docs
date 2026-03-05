# Workflow Engine: Perplexity Deep Research Results

> **Date:** 2026-03-04
> **Source:** Perplexity Deep Research on 7 prompts from overview.md
> **Status:** Raw findings — key decisions extracted into overview.md

---

## 1. Workflow Engine State Machines

### Temporal's Approach
Temporal uses **event-sourced durable execution** — the complete state of your workflow (local variables, progress) is automatically persisted at each `await` point. When a worker crashes, Temporal replays the event history on a new worker, resuming exactly where it left off. For long waits, you can call `Workflow.sleep('30 days')` and Temporal will reliably wake the workflow even if processes restart. For human-in-the-loop pauses, Temporal uses **Signals** + `condition()`: the workflow suspends without consuming resources and re-evaluates a predicate each time a signal arrives.

```
// Temporal HITL pattern
const confirmed = await condition(() => humanApproved, '7 days');
if (!confirmed) { /* timed out */ }
```

Temporal also handles workflow versioning via **patching** — allowing running workflows to complete under their original code path while new workflows use updated logic.

### BullMQ as a Simpler Alternative
BullMQ stores jobs in Redis with three states: **wait**, **delayed**, and **active**. For delayed steps, you pass `{ delay: ms }` when adding a job. You can inspect, reschedule (`job.changeDelay()`), or cancel delayed jobs programmatically. ToolJet migrated from Temporal to BullMQ specifically because it "simplified architecture" and leveraged existing Redis, with PostgreSQL for schedule persistence and a bootstrap service for crash recovery.

### Recommended BullMQ Pattern for Our Needs

| Requirement | BullMQ Pattern |
|---|---|
| Delayed step (wait 24h) | `queue.add('step', data, { delay: 86400000 })` |
| Human-in-the-loop pause | Create a "waiting" job; store `workflowRunId` in DB; on user response, add the next-step job to the queue |
| Crash recovery | Persist workflow state (current step, context) to Postgres; on startup, re-enqueue pending jobs from DB |
| Exactly-once execution | Use unique `jobId` per workflow step + `removeOnComplete` |

**Key insight: BullMQ handles individual job scheduling well, but you must build the workflow orchestration layer yourself** — a state table in Postgres that tracks which step each workflow run is on, and a "resume" function that enqueues the next step's job.

### DECISION: Use BullMQ + Postgres hybrid
- BullMQ for job scheduling (delays, retries, rate limiting)
- Postgres `workflow_runs` table for persistent state (current_node_id, variables, status)
- On Conversational Ask: mark run as `waiting`, store context in Postgres, no Redis job active
- On user response: webhook handler loads run from Postgres, enqueues next step to BullMQ
- On crash recovery: bootstrap service queries Postgres for `running` status runs, re-enqueues

---

## 2. React Flow Workflow Builder Architecture

### Core Architecture Stack
The official React Flow **Workflow Editor template** uses: Next.js + React Flow + **ELKjs** (auto-layout) + **Zustand** (state management) + **shadcn/ui** (component library). It includes drag-and-drop sidebar, dark mode, and a workflow runner that executes nodes sequentially.

### Custom Node Pattern
Custom nodes are standard React components passed to `nodeTypes`. You can embed forms, charts, or any interactive elements inside them. Multiple handles (connection points) are specified with unique `id`s:

```jsx
const nodeTypes = { actionNode: ActionNode, conditionNode: ConditionNode };
<ReactFlow nodes={nodes} edges={edges} nodeTypes={nodeTypes} />
```

### State Management with Zustand
React Flow internally uses Zustand, making it the natural choice. The recommended "No-Store-Actions" pattern imports actions directly rather than extracting them through hooks — reducing re-renders and improving type safety. For complex builders, separate **feature stores** (e.g., `nodeConfigStore`, `workflowRunStore`) from the global flow store.

### Key Implementation Details

| Feature | Approach |
|---|---|
| Auto-layout | ELKjs with `useAutoLayout` hook; dagre and d3-hierarchy are alternatives |
| Drag-and-drop sidebar | `onDragStart` sets node type -> `onDrop` reads position from event + `addNodes()` |
| Node config panel | Selected node state drives a right-sidebar `<Panel>` that calls `setNodes()` on change |
| Workflow validation | Traverse graph for disconnected nodes, check `node.data` for required fields |
| Edge routing | Step edges for workflow diagrams; bezier for freeform |

### Reference Implementations
- **React Flow Workflow Editor Template** (Pro): Complete starter with ELKjs + Zustand + runner
- **Synergy Codes Workflow Builder**: Higher-level SDK built on React Flow
- **Knock Workflow Builder 3.0**: Production notification workflow builder with drag-and-drop, conditions debugger

### DECISION: Follow the Pro template pattern
- ELKjs for auto-layout (not dagre)
- Zustand with "No-Store-Actions" pattern
- Step edges (not bezier) for workflow diagrams
- Separate stores: flowStore (nodes/edges), configStore (selected node config), runStore (execution state)

---

## 3. iMessage Bridge Reliability at Scale

### Rate Limits and Thresholds

| Metric | Threshold |
|---|---|
| Safe daily ceiling | **100 outbound/day** per Apple ID |
| Recommended for businesses | **50 unique contacts/day** per device |
| Observed block trigger | ~200/hr for 3 hours |
| Spam report tolerance | As few as **2-3 reports** can flag an account |

### Failure Modes
- **Apple ID flagged/blocked**: Messages silently stop delivering — no error, no webhook, no warning. Can be permanent after repeated offenses.
- **chat.db WAL lag**: The main `chat.db` can lag seconds to minutes behind `chat.db-wal`. Direct file monitoring gives false positives; **polling in read-only mode every 2 seconds** is the proven approach.
- **AppleScript instability**: Weak concurrency, character escaping issues, sandbox restrictions for file attachments (must copy to `~/Pictures` first).
- **macOS auto-update**: Can restart the Mac and kill the bridge process; requires monitoring/alerting.
- **Memory leaks**: Long-running watchers accumulate seen-message IDs; `imessage-kit` caps at 10,000 entries and purges hourly.

### Scaling with Multiple Mac Minis
Services like Sendblue run "Mac farms" — each Mac logged into a dedicated Apple ID with a dedicated number. To send 1,000 messages/day, you need 10+ Apple IDs. Load-balancing is handled at the API routing layer, not at the Mac level. Each Mac is an independent, isolated unit.

### Mitigation Strategies
- Make the **user message you first** — eliminates "Report Junk" link entirely
- Vary message content (no identical blasts), randomize delays between sends
- Default concurrency limit of 5 simultaneous sends to prevent Messages.app crashes
- Build monitoring that detects delivery failures (messages not appearing in `chat.db` after send)

### DECISION: Conservative approach for POC
- POC: 1 Mac mini, 1 Apple ID, user's own number (< 50 messages/day)
- Scale: Add Mac minis as needed, 1 Apple ID per Mac, 50 contacts/day per Mac
- Critical: Make users message us FIRST (eliminates "Report Junk")
- Monitor: Poll chat.db every 2 seconds, detect silent delivery failures
- Disable macOS auto-update on bridge Macs

---

## 4. AI Lead Scoring Models

### Algorithm Comparison

| Algorithm | Accuracy | Interpretability | Best For | Min Data |
|---|---|---|---|---|
| Logistic Regression | Moderate | High | Small datasets, cold start | ~100 leads |
| Random Forest | High | Moderate | Mixed data types, general scoring | ~200+ leads |
| Gradient Boosting (XGBoost) | Highest | Low | Large datasets, nuanced scoring | 500+ leads |
| Neural Networks | Variable | Very Low | Massive datasets only | Thousands |

### Minimum Training Data
Most systems need **100-200 historical leads** with known outcomes (both wins and losses) to identify initial patterns. You need at least **6-12 months of historical data** covering both conversions and non-conversions. For gradient boosting to outperform logistic regression, you generally need 500+ examples.

### Cold-Start Strategy (< 100 Leads)
1. **Start rule-based**: Define scoring tiers manually based on firmographic signals (company size, industry, role title) and behavioral signals (response time, engagement frequency)
2. **Collect data intentionally**: Track outcomes on every lead interaction for 3-6 months
3. **Transition to logistic regression** at ~100 labeled examples
4. **Graduate to GBM** when you have 500+ examples

### Scoring from Conversation Data (Our Use Case)
Extract these features from iMessage/email threads:
- **Response latency**: How fast they reply (strong intent signal)
- **Message length and sentiment**: Longer, positive messages = higher score
- **Question frequency**: Asking about pricing/timelines = high intent
- **Engagement recency**: Days since last message
- **Conversation depth**: Number of back-and-forth exchanges
- **Firmographic enrichment**: Company size, industry from email domain or CRM data

### DECISION: Rule-based first, ML later
- POC: Rule-based scoring with weighted signals (response latency, conversation depth, firmographic data)
- Phase 2: Logistic regression when we have 100+ labeled leads
- Phase 3: Gradient boosting (XGBoost) when we have 500+ labeled leads
- AI Classify node uses Claude for intent/sentiment analysis, not for scoring (scoring is a separate model)

---

## 5. Shared Inbox / Pool Architecture

### Assignment Models

| Platform | Methods | Key Differentiator |
|---|---|---|
| **Intercom** | Manual, Round-Robin, Balanced | Balanced = fewest active conversations |
| **Front** | Round-Robin, Load-Balancing | Load-balancing queues when all agents offline |
| **HelpScout** | Manual, Auto-assign | Simpler model, good for small teams |
| **Missive** | Rule-based routing | Deep email integration |

### Collision Detection
Every major shared inbox implements collision detection:
1. **Presence events**: When an agent opens a conversation, broadcast a WebSocket event to all other agents viewing that conversation
2. **Composing indicator**: When an agent starts typing a reply, broadcast "Agent X is typing..."
3. **Optimistic locking**: On reply submission, check if another reply was submitted since the agent opened the conversation; if so, show a conflict dialog

### Agent Presence and Availability
Intercom's round-robin skips agents with status "Away" and continues the rotation. Front's load-balancing **queues** conversations when all agents are offline and distributes them as agents come online.

### Database Schema Recommendations
- **conversations**: `id, channel, status (open/snoozed/closed), pool_id, assigned_agent_id, created_at, last_message_at`
- **pools**: `id, name, assignment_method (round_robin/balanced/manual), sla_response_seconds`
- **pool_agents**: `pool_id, agent_id, is_active, last_assigned_at, open_conversation_count`
- **conversation_events**: `id, conversation_id, type (assigned/replied/transferred/escalated), agent_id, timestamp`
- **presence**: Redis pub/sub for real-time agent status + conversation viewing state

### DECISION: Balanced assignment + collision detection from day one
- Default assignment: "Balanced" (fewest active conversations) — not round-robin
- Collision detection via WebSocket: show "Agent X is viewing" + "Agent X is typing"
- Optimistic locking on reply: prevent duplicate responses
- Queue conversations when all agents offline (Front pattern)
- Redis pub/sub for real-time presence

---

## 6. Webhook-Driven Communication Pipeline

### Normalized Message Schema

```typescript
interface InboundMessage {
  id: string;               // Our internal ID
  externalId: string;       // Channel-specific message ID
  channel: 'imessage' | 'sms' | 'email' | 'webchat';
  direction: 'inbound' | 'outbound';
  contactIdentifier: string; // E.164 phone or email
  conversationId: string;    // Threaded conversation ID
  body: string;
  attachments: Attachment[];
  meta: Record<string, any>; // Channel-specific data
  receivedAt: Date;
}
```

### Deduplication
Use `externalId` + `channel` as a composite unique key. Nango handles this by reconciling each webhook to a specific integration + connection, then forwarding with the raw payload wrapped in a normalized envelope.

### How Nango/Merge Handle This
- **Nango**: Receives webhooks on per-integration URLs, verifies HMAC signatures, and either forwards the raw payload to your app or triggers a sync that writes to Nango's normalized data layer
- **Merge**: Provides a fully pre-built unified schema for common entities (contacts, messages) across integrations
- **For our use case**: Build our own thin adapter layer per channel (iMessage webhook -> normalize, Twilio webhook -> normalize, SendGrid inbound parse -> normalize) feeding into a single BullMQ queue

### Threading Strategy
Thread by `(contactIdentifier, channel)` for 1:1 conversations. For email, use `In-Reply-To` / `References` headers. For SMS/iMessage, thread by phone number pair. Assign a `conversationId` on first contact and reuse it within a configurable window (e.g., 24 hours of inactivity = new conversation).

### DECISION: Thin adapter layer per channel -> single BullMQ queue
- Each channel gets a webhook endpoint + adapter that produces `InboundMessage`
- All adapters feed into a single `inbound-messages` BullMQ queue
- Dedup via `(externalId, channel)` composite unique index on communications table
- Threading: phone pair for iMessage/SMS, References header for email, session ID for web chat
- 24-hour inactivity window = new conversation thread

---

## 7. Contract Generation Workflow

### CLM Platform Patterns

**Ironclad's model**:
1. **Global Clause Library**: Two components — **Types** (category labels like "Indemnification") and **Clauses** (actual text under each type)
2. **Conditional clause insertion**: In Workflow Designer, highlight text -> Add Clause -> set conditions (e.g., "show this indemnity clause only if contract value > $100K")
3. **Changes propagate globally**: Edit a global clause once and all workflow configurations using it update automatically

**DocuSign CLM's model**:
1. Open "New Agreement" -> 2. Select template + recipient -> 3. Fill conditional form -> CLM merges data into template.

**Juro's model**:
Smartfields (data entry points) + Q&A workflow + conditional logic (clauses auto-added/removed based on field values) + mass generation for bulk contracts.

### Modeling as Visual Workflows

| CLM Step | Our Workflow Node |
|---|---|
| Template selection | **Template Selector** node (dropdown) |
| Q&A / form fill | **Conversational Ask** node (collect party details) |
| Conditional clause insertion | **Branch** node (if contract_value > X -> include clause Y) |
| Placeholder resolution | **Transform** node (merge data into template) |
| Approval routing | **Route to Pool** node (legal review pool) |
| Signature collection | **External Action** node (trigger e-sign API) |

**Key architectural insight from Ironclad: separating the clause library from the workflow** — clauses are centrally managed, and workflows reference them by ID with conditions. This prevents duplication and makes governance scalable.

### DECISION: Contract generator is eventually a workflow template
- POC: Keep dedicated contract generator code
- Phase 2: Extract as "Contract Generation" workflow template category
- Adopt Ironclad's pattern: Global Clause Library (separate from workflows) + workflows reference clause IDs
- Each contract type gets its own workflow template with conditional clause insertion via Branch nodes

---

## Sources

[1] Temporal: Beyond State Machines - https://temporal.io/blog/temporal-replaces-state-machines-for-distributed-applications
[2] Temporal Patterns: Process Manager with Signals - https://james-carr.org/posts/2026-02-03-temporal-process-manager/
[3] ToolJet: Migrating from Temporal to BullMQ - https://docs.tooljet.com/docs/setup/workflow-temporal-to-bullmq-migration/
[4] BullMQ Delayed Jobs - https://oneuptime.com/blog/post/2026-01-21-bullmq-delayed-jobs/view
[5] BullMQ Architecture - https://docs.bullmq.io/guide/architecture
[6] React Flow Workflow Editor Template - https://reactflow.dev/ui/templates/workflow-editor
[7] React Flow Handles - https://reactflow.dev/learn/customization/handles
[8] React Flow Custom Nodes - https://reactflow.dev/learn/customization/custom-nodes
[9] Zustand State Management in React Flow - https://www.youtube.com/watch?v=41FsulrcrQg
[10] Synergy Codes: State Management in React Flow - https://www.synergycodes.com/blog/state-management-in-react-flow
[11] React Flow: Using a State Management Library - https://reactflow.dev/learn/advanced-use/state-management
[12] React Flow Auto Layout - https://reactflow.dev/examples/layout/auto-layout
[13] React Flow Custom Nodes Example - https://reactflow.dev/examples/nodes/custom-node
[14] Knock: Designing Workflows - https://docs.knock.app/designing-workflows/overview
[15] Texting Blue: Avoid iMessage Blocks - https://texting.blue/blog/avoid-to-imessage-blocks/
[16] myCRMSIM: iMessage Daily Contact Limits - https://support.mycrmsim.com/article/managing-imessage-daily-contact-limits
[17] GitHub: Jared iMessage Limits - https://github.com/ZekeSnider/Jared/issues/65
[18] Apple Community: iMessage Spam Block - https://discussions.apple.com/thread/255921863
[19] Deep Dive into iMessage Bridge - https://fatbobman.com/en/posts/deep-dive-into-imessage/
[20] AI Algorithms for Lead Scoring - https://sales-mind.ai/blog/ai-algorithms-lead-scoring
[21] AI Lead Scoring Revolution - https://www.articsledge.com/post/ai-lead-scoring
[22] Monday.com: AI Lead Scoring Guide - https://monday.com/blog/crm-and-sales/ai-lead-scoring/
[23] Landbase: AI-Powered Lead Scoring - https://www.landbase.com/blog/how-to-enhance-lead-scoring-with-ai-powered-insights
[24] Warmly: AI Lead Scoring - https://www.warmly.ai/p/blog/ai-lead-scoring
[25] Intercom: Organize Team Inboxes - https://www.intercom.com/help/en/articles/197-organize-team-inboxes
[26] Front: Round Robin vs Load Balancing - https://help.front.com/en/articles/1315904
[27] EmailAnalytics: Shared Mailbox Tools - https://emailanalytics.com/shared-mailbox-tools/
[33] Nango: Implement Webhooks - https://nango.dev/docs/implementation-guides/webhooks/implement-webhooks
[34] Nango: Receive Webhooks - https://nango.dev/docs/implementation-guides/platform/webhooks-from-nango
[35] Nango: Best iPaaS for API Unification - https://nango.dev/blog/best-ipaas-for-api-unification
[36] Ironclad: Global Clause Library - https://support.ironcladapp.com/hc/en-us/articles/30659446762647
[37] Ironclad: Use Clauses in Workflow Designer - https://support.ironcladapp.com/hc/en-us/articles/30659495044759
[38] Ironclad: Conditional Clause in Workflow Designer - https://support.ironcladapp.com/hc/en-us/articles/12250550045463
[39] Ironclad: Manage Clauses - https://support.ironcladapp.com/hc/en-us/articles/30659470735127
[40] DocuSign CLM: Create a Contract - https://www.docusign.com/blog/create-contract-clm-steps
[41] Juro: Automated Contract Templates - https://juro.com/learn/automated-templates
[42] Juro: Contract Generation - https://juro.com/learn/contract-generation
