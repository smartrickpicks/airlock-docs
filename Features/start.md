# Airlock Features Index

> Master index of all feature specs. Each feature has its own folder with demo HTML + design spec markdown files.

| Feature | Folder | Demo HTML | Status | Key Decisions |
|---------|--------|-----------|--------|--------------|
| **Shell / Master Layout** | `/Shell/` | (none yet) | SPECCED | Module bar + channel sidebar + Signal\|Orchestrate\|Control triptych |
| **Record Inspector** | `/Record Inspector/` | `dossier-demo.html` | SPECCED | Cards with drawers, grouped by section, tier 1 expanded by default |
| **Triage Dashboard** | `/Triage/` | `triage-demo.html` | SPECCED | Triage is home (drill-down to contracts), aggregates notifications |
| **Contract Generator** | `/ContractGenerator/` | `contract-generator-demo.html` | SPECCED | Two-pane builder+preview, 24 contract types (5 verticals), unified v2 clause library (188 clauses) |
| **Admin / Settings** | `/Admin/` | `admin-demo.html` | SPECCED | System overlay: Workspace Admin (9 sections) + Personal Settings (5 sections) |
| **Review Queue** | `/OrgOverview/` | `org-overview-demo.html` | SPECCED | Gatekeeper's cross-vault view: entity cards, handoff signals, activity feed |
| **Patch Workflow** | `/PatchWorkflow/` | (none yet) | SPECCED | Action Focus editor, 12 states, 20 transitions, self-approval block |
| **AI Agent** | `/AIAgent/` | (none yet) | SPECCED | Both: feed events in Signal + dedicated tab in Control panel |
| **Batch Processing** | `/BatchProcessing/` | (none yet) | SPECCED | Async pipeline, semaphore(5), auto-system-pass gates, vault hierarchy integration |
| **Notifications** | `/Notifications/` | (none yet) | SPECCED | Two-level: channel + module, triage as aggregator, in-app only POC |
| **Feature Control Plane** | `/FeatureControlPlane/` | (none yet) | SPECCED | Toggle, calibrate, audit, failure flag for every engine |
| **Document Suite** | `/DocumentSuite/` | (none yet) | SPECCED | TipTap editor + PDF.js viewer, full template system, custom blocks |
| **Vault Hierarchy** | `/VaultHierarchy/` | (none yet) | SPECCED | Parent > Division > Counterparty > Item tree; CRM = vault lens |
| **Universal Task System** | `/TaskSystem/` | (none yet) | SPECCED | Master task table, per-module triage views, Home widgets |
| **CRM Module** | `/CRM/` | (none yet) | SPECCED | Lens over vault hierarchy; react-admin + Atomic CRM patterns |
| **Tasks Module** | `/Tasks/` | (none yet) | SPECCED | Universal work queue; Kanban, table, agenda views |
| **Calendar Module** | `/Calendar/` | (none yet) | SPECCED | Date-centric view from vault dates + task due dates; Schedule-X |
| **Roles & Permissions** | `/Roles/` | (none yet) | SPECCED | Builder/Gatekeeper/Owner + Discord permission model |
| **Tech Stack** | `/TechStack/` | (none yet) | SPECCED | Locked library choices from Perplexity research |
| **Real-Time Events** | `/RealTime/` | (none yet) | SPECCED | WebSocket + Redis Pub/Sub, topic subscriptions, auto-reconnect |
| **Cross-Module Event Bus** | `/EventBus/` | (none yet) | SPECCED | BullMQ queues, event routing, handler pattern, Trigger.dev workflows |
| **Search / Cmd+K** | `/Search/` | (none yet) | SPECCED | Universal search, command palette, slash commands, scoped search |
| **Onboarding** | `/Onboarding/` | (none yet) | SPECCED | Workspace setup wizard, user onboarding, progressive complexity |
| **Workflow Engine** | `/WorkflowEngine/` | (none yet) | BRAINSTORM | Visual drag-and-drop automation builder (React Flow), powers lead qualification, Smart Line routing, deal stage tasks, attachment classification, conversational intake funnels |
| **CRM Communications** | `/CRM/` | (none yet) | BRAINSTORM | Smart Line (iMessage bridge), pool model, AI triage, progressive contact enrichment |
| **Web-to-iMessage Handoff** | `/CRM/` | (none yet) | BRAINSTORM | Smart Widget embeds on website, bridges visitors to iMessage via Smart Line, page-aware context, mobile sms: URI for prospect-initiated contact |
| **Search Indexing** | `/Search/` | (none yet) | BRAINSTORM | MeiliSearch engine for cross-module instant search with badges, typo tolerance, and PostgreSQL sync pipeline |
| **Multi-Format Documents** | `/DocumentSuite/` | (none yet) | BRAINSTORM | Beyond PDF: DOCX, XLSX, PPTX, TXT, MD, EPUB, RTF, images with per-format converters and viewers |
| **Template Compiler** | `/Admin/` | (none yet) | BRAINSTORM | Admin tool: S3/Drive corpus + local LLM auto-generates extraction configs (anchors, synonyms, field mappings) |
| **Google Meet Integration** | `/CRM/` | (none yet) | BRAINSTORM | Google Meet SDK embedded in CRM — call transcription, AI meeting summaries, auto-generated action items |
| **Custom Roles** | `/Roles/` | (none yet) | BRAINSTORM | Discord-style custom role creation with granular permission catalog, drag-to-reorder hierarchy, multi-role per user |

## Status Legend

- **SPECCED** -- Full design spec written with decisions captured
- **INVENTORIED** -- OrcestrateOS code analyzed, Airlock mapping identified, decisions pending
- **BRAINSTORMING** -- In active design discussion
- **BUILDING** -- HTML prototype in progress
- **APPROVED** -- Spec + prototype approved, ready for implementation

## Architecture Decisions (Cross-Cutting)

1. **Module hierarchy:** Modules = Discord servers (far-left icon bar). Vaults = work units within modules.
2. **Default grouping:** Contracts grouped by entity/account (not by batch)
3. **View state switching:** Automatic based on context (not manual toggles)
4. **AI integration:** Both feed events (Signal panel) + dedicated tab (Control panel)
5. **Triage as home:** Analysts start in Triage Dashboard, drill into vaults
6. **Patch authoring:** Dedicated Action Focus state (full editor, not inline)
7. **Entity disambiguation:** Inline in Signal panel (not modal, not CRM cross-link)
8. **Notifications:** In-app only for POC. Triage Dashboard aggregates module-level notifications.
9. **Feature Control Plane:** Every ported feature gets toggle, calibration, audit, failure flagging.
10. **Theme:** Option B tokenized -- extract shared theme.base.css from demos, evolve to Airlock OLED palette.
11. **Vault hierarchy IS the CRM:** No separate CRM tables. Vault tree (Parent > Division > Counterparty > Item) stores all relationship data.
12. **Canonical vocabulary:** Vault, Module, Chamber, Gate, View, Triptych, Signal/Orchestrate/Control, Active Vaults.
13. **Tech stack locked:** Atomic CRM + react-admin, @hello-pangea/dnd, Schedule-X, Novu, BullMQ.
14. **Universal task system:** One master task table. Module triage = filtered SQL view. Home = summary widgets.
15. **CRM is vault hierarchy:** No separate CRM tables. react-admin browses vault tree levels 1-3.
16. **Calendar is computed:** No stored calendar events. Renders dates from vault extraction + task due dates.
17. **One workflow engine:** All automations (lead qual, routing, deal tasks, notifications, onboarding) use the same visual builder. React Flow canvas, stored as JSON, executed by BullMQ.
18. **Conversational intake over static forms:** Smart Line and web widget use Conversational Ask workflow nodes to qualify and route dynamically — like Discord onboarding flows.
19. **Pool model for shared inboxes:** Heymarket-style shared queues with claim, presence, internal notes, assignment strategies (manual/round-robin/least-busy/territory).
20. **Prospect-initiated contact:** Web widget on mobile uses `sms:` URI to open Messages app with pre-filled message — prospect sends first text, eliminating iMessage rate limit risk. Desktop path collects phone and sends one bridge text with explicit user consent.
21. **Search indexing:** MeiliSearch for cross-module instant search. PostgreSQL remains source of truth; MeiliSearch syncs via BullMQ on INSERT/UPDATE triggers. Every result has a module badge.
22. **Multi-format documents:** Not just PDFs. DOCX, XLSX, PPTX, TXT, MD, EPUB all convert to normalized text for extraction. Per-format viewers in Orchestrate panel. Per-format feature flags.
23. **Template compiler:** Admin panel where you feed a corpus of sample documents (S3/Drive/upload) + local LLM auto-generates extraction configs. Onboard new customers in minutes, not hours.
24. **Google Meet built-in:** Google Meet SDK in CRM vault detail pages. Click to call, AI transcribes, action items auto-populate in task board.
25. **Custom roles (Discord-style):** Admins create unlimited custom roles with granular permission checkboxes. Drag-to-reorder hierarchy. Multiple roles per user (additive). System base roles (Builder/Gatekeeper/Owner/Designer/Viewer) remain as defaults.
26. **Calibration with context:** Every calibration slider shows plain-English impact description: "What happens if I raise this? What happens if I lower it?" Live preview of affected document counts where possible.

## Master Plan File

Full architecture spec lives in the OrcestrateOS worktree plan file (will be migrated to this repo).
