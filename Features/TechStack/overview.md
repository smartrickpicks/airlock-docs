# Tech Stack Decisions

> **Status:** LOCKED — All major library choices finalized. This doc captures what was chosen, why, and how each library maps to Airlock features.

> **Research Source:** Comprehensive Perplexity research comparing open-source libraries across CRM, Kanban/Tasks, Calendar, Notifications, and Event Bus domains. Each choice optimized for: MIT/Apache-2.0 license, active maintenance (2024-2025+), self-hosted capability, React compatibility, and Airlock's specific requirements.

---

## Core Platform Stack

| Layer | Choice | Version | License | Notes |
|-------|--------|---------|---------|-------|
| **Frontend Framework** | Next.js 14 (App Router) | 14.x | MIT | TypeScript, server components, file-based routing |
| **UI Styling** | Tailwind CSS | 3.x | MIT | Utility-first, custom Airlock theme tokens |
| **State Management** | Zustand | 4.x | MIT | Lightweight, no boilerplate, supports middleware |
| **Backend API** | FastAPI | 0.115+ | MIT | Python, async-first, OpenAPI auto-docs |
| **Database** | PostgreSQL | 16 | PostgreSQL | JSONB for metadata, append-only events |
| **Real-Time** | WebSockets (Starlette) | native | BSD | Built into FastAPI/Starlette |
| **Containerization** | Docker Compose | — | Apache-2.0 | Local dev: Postgres + Redis + API + Web |

---

## Feature-Specific Libraries

### CRM: Atomic CRM + react-admin

| Property | Value |
|----------|-------|
| **Library** | [react-admin](https://github.com/marmelab/react-admin) + [Atomic CRM](https://github.com/marmelab/atomic-crm) |
| **License** | MIT |
| **Why chosen** | Atomic CRM is a complete CRM reference implementation built on react-admin. Provides data grid, relationship management, pipeline views, and contact management out of the box. react-admin handles CRUD, filtering, pagination, and data provider abstraction. |
| **Airlock mapping** | CRM module views. The vault hierarchy IS the CRM — react-admin provides the list/detail/edit patterns for browsing parent vaults, divisions, counterparties. Atomic CRM's pipeline view maps to the Counterparty vault pipeline (Discover > Build > Review > Ship). |
| **Customization needed** | Replace Atomic CRM's flat contact model with vault hierarchy tree. Wire data provider to Airlock's vault API. Apply Airlock OLED theme. |
| **Key features used** | DataGrid for vault lists, ReferenceField for parent-child links, FilterList for chamber filtering, EditGuesser for metadata forms, Datagrid bulk actions for triage operations. |

**Rejected alternatives:**
- Twenty CRM (AGPL-3.0 — license incompatible)
- Huly (EPL-2.0 — restrictive)
- Building from scratch (unnecessary given react-admin's maturity)

---

### Tasks / Kanban: @hello-pangea/dnd

| Property | Value |
|----------|-------|
| **Library** | [@hello-pangea/dnd](https://github.com/hello-pangea/dnd) |
| **License** | Apache-2.0 |
| **Why chosen** | Community-maintained fork of react-beautiful-dnd (Atlassian dropped it). Stable, well-documented, keyboard accessible, works with virtualization. Used in production by many companies. |
| **Airlock mapping** | Tasks module Kanban board (columns = chambers: Discover, Build, Review, Ship). Triage dashboard lane cards. Any drag-and-drop reordering in the UI. |
| **Customization needed** | Custom drag handles with vault gate colors. Drop zone validation (e.g., can't drag from Ship back to Discover without admin override). Integration with vault state machine. |
| **Key features used** | DragDropContext, Droppable columns, Draggable cards, keyboard navigation, multi-drag (shift+click). |

**Alternative considered:**
- pragmatic-drag-and-drop (Apache-2.0, also from Atlassian ecosystem) — viable backup if @hello-pangea/dnd becomes unmaintained. More low-level, requires more custom code but is framework-agnostic.

---

### Calendar: Schedule-X

| Property | Value |
|----------|-------|
| **Library** | [Schedule-X](https://github.com/schedule-x/schedule-x) |
| **License** | MIT (core) |
| **Why chosen** | Modern, Material Design-inspired calendar with month/week/day/agenda views. React adapter available. Lightweight core with plugin system. Active development (2024-2025). |
| **Airlock mapping** | Calendar module. Shows dates extracted from contracts (effective dates, termination dates, renewal deadlines), task due dates, SLA deadlines. Clicking a calendar event navigates to the source vault. |
| **Customization needed** | Dark theme (Airlock OLED palette). Gate-colored event indicators. Click handler to navigate to source vault. Custom event renderer showing vault name + chamber badge. |
| **Key features used** | Month view (default landing), week view for SLA-heavy periods, agenda view for daily work planning. Drag-to-reschedule for task due dates. |

**Note:** Premium plugins exist but the MIT core covers all Airlock needs (month/week/day views, event CRUD, drag-resize).

**Rejected alternatives:**
- FullCalendar (mixed license — premium features require commercial license)
- react-big-calendar (MIT but aging, less active maintenance)

---

### Notifications: Novu

| Property | Value |
|----------|-------|
| **Library** | [Novu](https://github.com/novuhq/novu) |
| **License** | MIT |
| **Why chosen** | Full notification infrastructure: in-app, email, push, SMS, chat channels. Self-hosted option. React components for notification center. Workflow engine for complex notification rules. Supports digest/batching. |
| **Airlock mapping** | Two-level notification system. Channel-level: events in Signal panel. Module-level: badges on module bar + triage aggregation. Future: email digests for SLA warnings, push for urgent gate approvals. |
| **Customization needed** | Custom notification center UI (replace Novu's default bell with Airlock's triage-integrated design). Notification triggers mapped to vault events. Self-hosted deployment in Docker Compose. |
| **Key features used** | In-app notification center, notification preferences per user, digest mode (batch similar notifications), workflow templates for different event types (SLA, patch approval, extraction complete). |

**Architecture:**
- POC: In-app only (Novu React components embedded in module bar badges + Signal panel)
- V2: Email notifications for urgent/SLA events via Novu's email channel
- V3: Push notifications for mobile (Novu's push channel)

**Rejected alternatives:**
- Knock (proprietary/cloud-only)
- Custom build (notification infrastructure is surprisingly complex — digests, preferences, delivery guarantees)

---

### Event Bus / Background Jobs: BullMQ

| Property | Value |
|----------|-------|
| **Library** | [BullMQ](https://github.com/taskforcesh/bullmq) |
| **License** | MIT |
| **Why chosen** | Redis-backed job queue with Python SDK (bullmq). Supports delayed jobs, retries, rate limiting, job dependencies, and event listeners. Battle-tested at scale. Python SDK means it works with FastAPI backend. |
| **Airlock mapping** | Cross-module event bus. When a contract vault emits `gate.advanced(ship)`, BullMQ dispatches jobs to: CRM handler (create/update account), Tasks handler (create onboarding tasks), Calendar handler (create renewal reminders), Notifications handler (alert stakeholders). Also powers batch processing pipeline. |
| **Customization needed** | Job type definitions per vault event type. Dead letter queue for failed cross-module propagations. Admin UI integration (BullMQ has a board UI for monitoring). |
| **Key features used** | Named queues per module, job priorities (urgent > action > info maps to BullMQ priority levels), delayed jobs for SLA reminders, repeatable jobs for health score recalculation. |

**Companion: Trigger.dev**

| Property | Value |
|----------|-------|
| **Library** | [Trigger.dev](https://github.com/triggerdotdev/trigger.dev) |
| **License** | Apache-2.0 |
| **Why chosen** | Orchestration layer on top of BullMQ for complex multi-step workflows. Provides durable execution (survives server restarts), visual workflow editor, and TypeScript-first API. Self-hostable. |
| **Airlock mapping** | Complex workflows like: batch ingest pipeline (download > extract > preflight > auto-pass > notify), approval chains (submit > verify > admin > promote), and cross-module flywheel (contract shipped > create CRM entry > create tasks > schedule calendar events). |
| **When to use** | BullMQ for simple fire-and-forget jobs. Trigger.dev for multi-step orchestrations that need retry, branching, and visibility. |

---

### Rich Text Editor: TipTap

| Property | Value |
|----------|-------|
| **Library** | [TipTap](https://github.com/ueberdosis/tiptap) |
| **License** | MIT (core) |
| **Why chosen** | ProseMirror-based, headless (full styling control), extensible via plugins. Supports slash commands, collaborative editing, custom node types. Used in Notion-like editors. |
| **Airlock mapping** | Home workspace editor (Notion-style customizable blocks), patch authoring "because" clause, contract generator clause editing, document annotations. |
| **Key extensions** | StarterKit (basic formatting), Placeholder, SlashCommand (custom), TaskList, Table, CodeBlock, Collaboration (Y.js for real-time). |

---

### PDF Viewing: PDF.js

| Property | Value |
|----------|-------|
| **Library** | [PDF.js](https://github.com/nicbarker/pdf.js) (Mozilla) |
| **License** | Apache-2.0 |
| **Why chosen** | Industry standard for browser PDF rendering. Already proven in OrcestrateOS's document viewer. Supports text extraction, page navigation, zoom, search. |
| **Airlock mapping** | Document viewer in Orchestrate panel. Annotation overlay for extraction results (green/amber/red highlights on extracted field locations). |

---

## Dependency Graph

```
Next.js 14
  +-- Zustand (state)
  +-- Tailwind CSS (styling)
  +-- TipTap (rich text)
  +-- react-admin (CRM module views)
  +-- @hello-pangea/dnd (Kanban/tasks)
  +-- Schedule-X (Calendar module)
  +-- Novu React (notification center)
  +-- PDF.js (document viewer)

FastAPI
  +-- BullMQ Python SDK (event bus)
  +-- Trigger.dev (workflow orchestration)
  +-- psycopg2 (PostgreSQL)
  +-- python-jose (JWT auth)
  +-- httpx (external API calls)

Infrastructure
  +-- PostgreSQL 16 (primary data)
  +-- Redis 7 (BullMQ queues, caching, session)
  +-- Docker Compose (local dev)
  +-- Novu self-hosted (notifications)
```

---

## What We're NOT Using (And Why)

| Category | Rejected | Why |
|----------|----------|-----|
| CSS Framework | shadcn/ui, Chakra, MUI | Tailwind + custom components gives full control over Airlock's unique aesthetic |
| State Management | Redux, Jotai, Recoil | Zustand is simpler, sufficient for our needs, no boilerplate |
| ORM | Prisma, Drizzle, SQLAlchemy | Direct SQL via psycopg2 — we need full control over queries and migrations |
| Auth | NextAuth, Clerk, Auth0 | Custom JWT + Google OAuth — ported from OrcestrateOS, full control |
| CRM | Twenty (AGPL), Huly (EPL) | License restrictions incompatible with Airlock |
| Calendar | FullCalendar (mixed license) | Premium features require commercial license |
| Drag-and-Drop | dnd-kit | @hello-pangea/dnd has better out-of-box Kanban patterns |
| Notification | Knock (cloud-only) | Need self-hosted for data sovereignty |

---

## Version Pinning Strategy

- **Lock major versions** in package.json / requirements.txt
- **Renovate bot** for automated dependency update PRs
- **No bleeding edge** — wait for .1 or .2 patch releases before adopting new majors
- **License audit** on every dependency update (enforce MIT/Apache-2.0/BSD)

---

## Related Specs

- **[Shell Overview](../Shell/overview.md)** — Where these libraries render in the UI
- **[Vault Hierarchy](../VaultHierarchy/overview.md)** — Data model that react-admin browses
- **[Notifications](../Notifications/overview.md)** — Novu integration architecture
- **[Batch Processing](../BatchProcessing/overview.md)** — BullMQ pipeline architecture
