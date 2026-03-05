# Airlock

Airlock is a Discord-like platform for enterprise data operations. It replaces OrcestrateOS's monolithic frontend with a modular, channel-based architecture.

## What This Repo Is

This is the **design documentation repo** for Airlock. No production code yet -- only:
- Feature spec markdown files (one folder per feature)
- HTML demo prototypes (visual mockups)
- Design decisions captured during brainstorming

All of these docs will be loaded into **NotebookLM** as the knowledge base for implementation. Write specs that are comprehensive enough for an engineer (or Claude Code) to build from without asking questions.

## Repo Structure

```
/Airlock/
  claude.md                    <- THIS FILE: project context for all agents
  /Features/
    start.md                   <- Master index of all features + status
    /Shell/                    <- Master layout, naming, chambers, triptych
    /Record Inspector/         <- Field cards, entity resolution, doc viewer
    /Triage/                   <- Contracts triage dashboard
    /ContractGenerator/        <- Template builder, unified v2 clause library (188 clauses, 24 types)
    /Admin/                    <- Workspace admin + personal settings
    /OrgOverview/              <- Gatekeeper's cross-vault review queue
    /PatchWorkflow/            <- Approval chain, SLA timer, action focus
    /AIAgent/                  <- Otto AI agent (PydanticAI + LiteLLM runtime)
    /BatchProcessing/          <- Async pipeline, lifecycle state machine
    /Notifications/            <- Two-level notification system (Novu)
    /FeatureControlPlane/      <- Toggle, calibrate, audit, failure flag
    /DocumentSuite/            <- TipTap editor + PDF.js viewer
    /VaultHierarchy/           <- Parent > Division > Counterparty > Item tree
    /TaskSystem/               <- Universal task table, per-module triage
    /CRM/                      <- Vault hierarchy as CRM, react-admin patterns
    /Tasks/                    <- Kanban/table/agenda views
    /Calendar/                 <- Schedule-X, computed from vault dates
    /Roles/                    <- Builder/Gatekeeper/Owner + Discord permissions
    /TechStack/                <- Locked library choices
    /RealTime/                 <- WebSocket + Redis Pub/Sub
    /EventBus/                 <- BullMQ + Trigger.dev
    /Search/                   <- Cmd+K palette, universal search
    /Onboarding/               <- Workspace + user onboarding wizard
    /WorkflowEngine/           <- Visual automation builder (React Flow)
    /Messenger/                <- Internal messaging (prototype only)
    /Security/                 <- Cipher model, threat model, KMS, LLM safety, compliance
  /demo-shell/                 <- Master demo hub (serves all demo HTMLs)
```

## Architecture

- **Modules** = Discord "servers" (Contracts, CRM, Tasks, Calendar, Documents, Admin)
- **Vaults** = Workflow instances within modules (each contract, customer, task)
- **Events** = Immutable actions in vault feeds (extractions, patches, approvals)
- **Triptych** = Signal (left) | Orchestrate (center) | Control (right)
- **Chambers** = Lifecycle stages: Discover > Build > Review > Ship

## Canonical Vocabulary (LOCKED)

See `/Features/Shell/naming-and-hierarchy.md` for the full spec. These terms are non-negotiable:

| Concept | Canonical Term | DO NOT USE |
|---------|---------------|------------|
| Workflow instance | **Vault** | channel, workstream, room, item, record |
| Top-level domain | **Module** | server, workspace, app |
| Lifecycle stage | **Chamber** | phase, step, stage, category |
| Chamber checkpoint | **Gate** | checkpoint, milestone, barrier |
| Screen within chamber | **View** | page, panel, station |
| Three-panel workspace | **Triptych** | layout, split, panels |
| Left panel | **Signal** | feed, events, sidebar |
| Center panel | **Orchestrate** | main, workspace, content |
| Right panel | **Control** | context, sidebar, info |
| Vault list in sub-panel | **Active Vaults** | recent channels |

## Tech Stack (LOCKED)

- Frontend: React + Next.js 14 (App Router, TypeScript), Zustand, Tailwind CSS
- Backend: FastAPI (Python), PostgreSQL, WebSockets (Starlette)
- Infrastructure: Docker Compose, Redis 7
- AI Runtime: PydanticAI + LiteLLM proxy (see `/Features/AIAgent/`)
- CRM: react-admin + Atomic CRM patterns
- Tasks: @hello-pangea/dnd (Kanban)
- Calendar: Schedule-X
- Notifications: Novu self-hosted
- Event Bus: BullMQ + Trigger.dev
- Editor: TipTap
- PDF: PDF.js

## Security Architecture (LOCKED Principles)

See `/Features/Security/overview.md` for the full 70-question inventory and documentation plan.

### Security Vocabulary

| Term | Definition |
|------|------------|
| **Trust boundary** | The point where encryption/decryption occurs -- only the application layer is trusted |
| **Ciphertext at rest** | All sensitive data in PostgreSQL, Redis, and file storage is encrypted |
| **DEK** | Data Encryption Key -- encrypts actual data |
| **KEK** | Key Encryption Key -- encrypts the DEK (envelope encryption) |
| **OGC chunk** | Ontology-Guided Corpus chunk -- the atomic unit of verified knowledge |
| **Crypto-shredding** | Destroying a key to render all data it encrypted permanently unrecoverable |
| **RLS** | Row-Level Security -- PostgreSQL-enforced tenant isolation via `workspace_id` |
| **HKDF** | HMAC-based Key Derivation Function -- derives child keys from a master |

### Security Principles (Non-Negotiable)

1. **Ciphered at rest, encrypted in transit** -- no plaintext on any storage medium
2. **Middleware-only trust** -- storage backends (PostgreSQL, Redis, S3) are untrusted
3. **No autonomous AI actions** -- Otto is read-only or draft-only; humans approve all changes
4. **Self-approval prevention is unforgeable** -- no API path can bypass SoD
5. **Append-only audit** -- security events cannot be modified or deleted
6. **Workspace isolation** -- no data crosses workspace boundaries without explicit federation
7. **Key custody transparency** -- admins always know who holds their encryption keys
8. **Rehydratable from cold** -- full rebuild from encrypted backups + key material
9. **PII never reaches LLM providers unredacted** -- redaction before the network call
10. **Fail closed** -- when security controls fail, access is denied

### Agent Safety Rules for Security Specs

- Never include real PII in spec examples -- use synthetic data only (e.g., "Acme Corp", "Jane Builder")
- Always specify encrypt/decrypt points when describing data flows
- Use canonical vocabulary -- never "channel", "server", "workspace" (see table above)
- Cross-reference `/Features/Security/overview.md` when making security-relevant decisions

## Source (OrcestrateOS)

Porting from: `/Users/zacharyholwerda/Desktop/OrcestrateOS-sync/`

Key source directories for reference:
- `server/extraction/` -- 7 extractors + dispatcher (1,660 lines)
- `server/preflight_engine.py` -- Quality gates (3,895 lines)
- `server/generation/` -- Contract generator engine (453 lines) + fake data, variation engine, PDF renderer
- `rules/generation/clause_library_v2.json` -- Unified clause library (188 clauses, 388KB)
- `rules/generation/templates/` -- 14 per-type template definitions
- `rules/` -- 30+ JSON config files (schemas, anchors, synonyms)
- `server/routes/` -- API routes (patches, triage, auth, otto chat)
- `server/resolvers/` -- Entity resolution chain

## Design Docs

All feature specs live in `/Features/` -- see `/Features/start.md` for the master index with status.

---

## Multi-Agent Collaboration Guide

Multiple Claude agents may work on this repo simultaneously. Follow these rules to avoid conflicts.

### How to Add Research or Findings to a Feature

1. **Find the right folder.** Every feature has its own directory under `/Features/`. Check `/Features/start.md` for the index.

2. **Read the overview.md FIRST.** Every feature folder has an `overview.md` that captures the locked design decisions. Understand what's already decided before adding anything.

3. **Create a new sub-spec file** for your research. DO NOT modify `overview.md` unless you're updating a decision that was explicitly marked as open. Name your file in `kebab-case.md`:
   - Good: `clause-catalog.md`, `template-expansion-rules.md`, `risk-assessment-framework.md`
   - Bad: `notes.md`, `research.md`, `findings.md` (too vague)

4. **Use this file template** for new spec files:

```markdown
# [Title] -- [Design Spec | Research | Catalog]

> **Status:** SPECCED | BRAINSTORM | RESEARCH
> **Source:** [Where this research came from -- OrcestrateOS files, external research, etc.]

## Summary

[2-3 sentences: what this document adds to the feature]

## [Content sections...]

---

## Related Specs

- [Link to overview.md](./overview.md) -- Parent feature spec
- [Link to other related specs]
```

5. **Cross-reference** your new file from the parent `overview.md` by adding a row to its "Related Specs" section (or equivalent). If there's no such section, add one at the bottom.

6. **Update `/Features/start.md`** ONLY if you're adding a brand new feature folder. Don't update it for sub-spec files within an existing feature.

### File Naming Rules

| Type | Pattern | Example |
|------|---------|---------|
| Feature overview | `overview.md` | `ContractGenerator/overview.md` |
| Sub-feature spec | `kebab-case.md` | `ContractGenerator/clause-catalog.md` |
| Demo prototype | `feature-name-demo.html` | `ContractGenerator/contract-generator-demo.html` |
| Research notes | `topic-research.md` | `ContractGenerator/clause-risk-research.md` |

### What NOT to Do

- **Don't create new top-level folders** without checking `start.md` first -- the feature might already have a home
- **Don't rename existing files** -- other specs cross-reference them
- **Don't use old vocabulary** (channel, workstream, phase, stage) -- see Canonical Vocabulary above
- **Don't write production code** -- this repo is design docs only
- **Don't modify the demo shell** (`demo-shell/index.html`) without coordinating -- it has complex JS state
- **Don't duplicate specs** -- if a topic is already covered in another feature folder, cross-reference it instead

---

## Clause Library Agent Instructions

If you're the agent working on expanding the clause library, here's exactly where your work goes.

**IMPORTANT:** The clause system uses the **v2 unified architecture**. There are NO generation pairs. Clauses carry both generation AND extraction config in a single record. See `v2-migration-brief.md` for full context on the v1 → v2 shift.

### Your Target Directory

```
/Features/ContractGenerator/
  overview.md                    <- READ THIS FIRST (v2 clause schema + pipeline defined here)
  v2-migration-brief.md          <- v1 → v2 migration analysis (reference)
  contract-generator-demo.html   <- Visual demo of the builder
  clause-catalog.md              <- YOUR FILE: expanded clause catalog
  template-definitions.md        <- YOUR FILE: per-type template structures (if needed)
```

### What Already Exists (in overview.md)

The v2 unified clause schema (15+ fields per clause):

| Field | Type | Description |
|---|---|---|
| `clause_id` | string | Format: `CATEGORY-SECTION-TYPE-VERSION` (e.g., `GEN-RECITALS-DIST-V1`) |
| `clause_type` | string | Functional category (101 types, e.g., `PARTIES_IDENTIFICATION`, `ROYALTY_RATE`) |
| `version` | integer | Clause version (1, 2, etc.) |
| `contract_types` | list | Which of the 24 types this applies to |
| `section` | string | Template section (e.g., `recitals`, `scope`, `compensation`) |
| `risk_level` | enum | `standard`, `elevated`, `critical`, `high` |
| `body` | text | Legal prose with `{{CHECK_CODE}}` variable markers |
| `variables` | list | Variable definitions with embedded extraction config |
| `variables[].field` | string | Check code (e.g., `OPP_EFFECTIVE_DATE`) |
| `variables[].canonical_key` | string | SF-compatible field key (e.g., `Effective_Date__c`) |
| `variables[].extraction` | object | Extraction config: extractor_type, anchor_patterns, negative_anchors, preferred_zones, proximity_chars, confidence_floor |
| `triggers` | list | Conditions for auto-selection (AND logic): equals, in, not_equals, exists, pattern, default |
| `generation_config` | object | `{priority: 10, fallback: "[TO_BE_DEFINED]"}` |
| `extraction_config` | object | `{tier: 1, preflight_weight: 1.0, section_anchors: {...}}` |
| `cuad_labels` | list | CUAD taxonomy tags (79 labels available) |
| `approved_by` | string | Legal reviewer who approved |
| `last_reviewed` | timestamp | Last legal review date |

**24 contract types** across 5 verticals: Music Core (4), Music Ancillary (7), Film (5), TV (4), Cross-Entertainment (4).

**2 section types:** `boilerplate` (fixed text) and `conditional` (populated by trigger-matched clauses).

**188 clauses** currently in `clause_library_v2.json`.

### What To Write

Create **`clause-catalog.md`** with your expanded clause research. Structure it as:

```markdown
# Clause Catalog -- Expanded Library

> **Status:** RESEARCH
> **Source:** [your research sources]

## Summary

Expanded clause library for the Contract Generator. Organized by contract type
and section. Each clause follows the v2 unified schema defined in overview.md.

## Clauses by Vertical

### Music Core

#### Distribution

| clause_id | clause_type | section | risk_level | variables | triggers |
|---|---|---|---|---|---|
| GEN-SCOPE-DIST-V1 | TERRITORY_SCOPE | scope | standard | [OPP_TERRITORY] | [OPP_CONTRACT_TYPE equals Distribution] |

**Body:**
> The Label shall have the exclusive right to distribute... in {{OPP_TERRITORY}}...

**Variables:**
- `OPP_TERRITORY` (canonical: `Territory__c`, extractor: text, anchors: ["territory", "throughout"], confidence: 0.7)

[...repeat for each clause]

### Film
[...same pattern]

## Risk Assessment Notes

[Research on risk classification, edge cases, legal considerations]

## Extraction Coverage Gaps

[Fields that need better anchor patterns, low-confidence extractions, missing clause coverage]

---

## Related Specs

- [overview.md](./overview.md) -- v2 clause schema + generation pipeline
- [v2-migration-brief.md](./v2-migration-brief.md) -- v1 → v2 migration context
```

### Key Constraints

- **Use the v2 schema exactly** -- every clause needs `clause_type`, `version`, `generation_config`, and `extraction_config`
- **Use `{{CHECK_CODE}}` syntax** for variables in clause bodies (e.g., `{{OPP_TERRITORY}}`, `{{OPP_EFFECTIVE_DATE}}`, `{{ACCT_LEGAL_ENTITY}}`)
- **NO generation pairs** -- trigger logic is embedded in each clause's `triggers[]` array
- **Every clause needs a `risk_level`** -- use `standard` (default), `elevated`, `critical`, or `high`
- **Include extraction config per variable** -- each variable should have `extractor_type`, `anchor_patterns`, `negative_anchors`, `preferred_zones`, `proximity_chars`, `confidence_floor`
- **Clause ID format:** `CATEGORY-SECTION-TYPE-VERSION` -- categories: `GEN`, `RISK`, `CUAD`, `MUSIC`, `FILM`, `TV`, `BOILER`
- **Map clauses to the 24 contract types** -- a clause can apply to multiple types
- **Triggers use AND logic** -- all conditions must match. Available: `equals`, `in`, `not_equals`, `exists`, `not_exists`, `pattern`, `contains`, `default`

---

## Skills & Plugins (For Build Phase)

### Active Now (Design Phase)
- `ui-ux-pro-max` -- UI/UX design intelligence, style recommendations
- `feature-dev` -- Feature development workflow (requirements, design, implementation, testing)
- `frontend-design` -- UI/UX design patterns and accessibility standards
- `claude-md-management` -- Auto-maintain this CLAUDE.md file
- `explanatory-output-style` -- Explain "why" behind decisions in spec docs

### When Coding Starts
- `typescript-lsp` -- TypeScript language server (type checking, go-to-definition, diagnostics)
- `playwright` -- Browser automation for testing HTML prototypes and React app
- `security-guidance` -- Passive vulnerability scanning (secrets, auth bypass, injection)
- `code-review` -- Structured code review with quality scoring
- `pr-review-toolkit` -- PR workflow: review comments, suggestions, issue detection
- `commit-commands` -- Conventional commits for clean git history
- `code-simplifier` -- Complexity detection, suggest simplifications
- `context7` -- Fetch up-to-date docs for Next.js 14, FastAPI, Zustand, Tailwind
- `claude-code-setup` -- Project scaffolding for monorepo initialization

## Rules

- No production code in this repo (design docs only, for now)
- Every design decision must be documented in the relevant feature's markdown files
- Each feature folder mirrors how the codebase will be structured
- HTML demos are visual prototypes -- they don't need to be production-quality
- Use canonical vocabulary from `/Features/Shell/naming-and-hierarchy.md` everywhere
- Cross-reference related specs -- no orphaned documents
- Write for NotebookLM ingestion -- specs should be self-contained and comprehensive
