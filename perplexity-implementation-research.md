# Airlock Implementation Research Prompt

> **Purpose:** Paste everything below the horizontal rule into Perplexity Deep Research mode.
> **Created:** 2026-03-04
> **Supersedes:** `research-prompt.md` (tech stack selection — already completed)

---

You are evaluating and improving an agentic development workflow for a complex enterprise SaaS platform called **Airlock**.

Airlock is a Discord-like platform for enterprise data operations. It replaces a monolithic Python/FastAPI beta (OrcestrateOS) with a modular, channel-based architecture where AI agents and human operators collaborate through governed workflows.

Please analyze this architecture and provide a comprehensive research report covering all 8 domains below. Cite specific GitHub repos, blog posts, documentation, and 2025-2026 patterns — not legacy approaches.

---

## Platform Context

### What Airlock Is

Airlock manages enterprise data workflows (contract lifecycle management, entity resolution, quality gates, document processing) through a Discord-inspired interface. Work items move through governed lifecycle stages with role-based oversight.

### Canonical Vocabulary (Non-Negotiable)

These terms are locked. All research and recommendations must use them:

| Concept | Term | Never Say |
|---------|------|-----------|
| Top-level domain | **Module** | server, workspace, app |
| Workflow instance | **Vault** | channel, workstream, room, item, record |
| Lifecycle stage | **Chamber** | phase, step, stage, category |
| Chamber checkpoint | **Gate** | checkpoint, milestone, barrier |
| Screen within chamber | **View** | page, panel, station |
| Three-panel workspace | **Triptych** | layout, split, panels |
| Left panel | **Signal** | feed, events, sidebar |
| Center panel | **Orchestrate** | main, workspace, content |
| Right panel | **Control** | context, sidebar, info |
| Vault list in sidebar | **Active Vaults** | recent channels |

### Architecture

- **Modules** = Discord "servers" — top-level functional domains (Contracts, CRM, Tasks, Calendar, Documents)
- **Vaults** = Workflow instances within modules — each contract, customer, task, or document
- **Chambers** = 4 universal lifecycle stages every vault passes through:
  - **Discover** (Red) — Intake, triage, assignment
  - **Build** (Yellow) — Extraction, standardization, enrichment
  - **Review** (Purple) — Quality review, approval chain
  - **Ship** (Green) — Export, sync, publish
- **Gates** = Checkpoints within chambers requiring human action (e.g., `gate_triage`, `gate_preflight`, `gate_gatekeeper`)
- **Triptych** = Three-panel workspace for viewing vaults:
  - Signal (left, ~280px) — Event feed, flags, AI suggestions
  - Orchestrate (center, flex) — Main work area (record inspector, document viewer, editor)
  - Control (right, ~300px) — Health score, SLA timer, approval chain, audit trail, AI agent
- **Triptych States** = Overview, Inspect, Edit, Approve — auto-switching based on user actions
- **Admin** = System overlay (not a module) — Workspace Admin + Personal Settings

### Roles

Three primary roles with Discord-style permissions:
- **Builder** — Creates and standardizes vault content (primary in Discover + Build chambers)
- **Gatekeeper** — Reviews quality and approves (primary in Review chamber)
- **Owner** — Final sign-off and release authority (primary in Ship chamber)

### Tech Stack (Locked — Do Not Suggest Alternatives)

**Frontend:**
- Next.js 14 (App Router, TypeScript)
- Zustand (state management)
- Tailwind CSS (styling, custom OLED theme tokens)
- react-admin + Atomic CRM (CRM module views)
- @hello-pangea/dnd (Kanban/drag-and-drop)
- Schedule-X (calendar views)
- TipTap (rich text editor)
- PDF.js (document viewer)
- React Flow (visual workflow builder)

**Backend:**
- FastAPI (Python, async-first)
- PostgreSQL 16 (JSONB metadata, append-only events)
- WebSockets (Starlette, built into FastAPI)
- Redis 7 (BullMQ queues, caching, sessions)
- BullMQ + Trigger.dev (event bus, background jobs, workflow orchestration)
- Novu self-hosted (notification infrastructure)
- PydanticAI + LiteLLM proxy (AI agent runtime — "Otto")

**Infrastructure:**
- Docker Compose (local dev: Next.js + FastAPI + PostgreSQL + Redis + Novu + LiteLLM)
- Custom JWT + Google OAuth (ported from OrcestrateOS)

### Current State of the Project

**Design Documentation Repo (Airlock):**
- 57 markdown spec files across 25 feature folders
- 18 HTML demo prototypes (interactive visual mockups)
- 1 master demo hub serving all demos
- 4 research PDFs + 17 architecture mockups
- 1 component catalog (1,187 lines documenting shared UI patterns)
- 16,545 total lines of specification
- 22 features fully SPECCED, 3 in BRAINSTORM
- Zero production code — no `package.json`, no `.github/`, no CI/CD
- All specs structured for ingestion by Google NotebookLM

**Source Reference Repo (OrcestrateOS — Incomplete Beta):**
- Python FastAPI backend: 10,408 lines across 38 source files
- 7 field extractors + dispatcher (1,715 lines) — text, date, boolean, pattern, picklist, split
- Preflight quality engine (3,908 lines) — validation rules, risk scoring, section guidance
- Contract generation engine (3,024 lines) — clause selection, interpolation, variation, PDF rendering
- Unified v2 clause library: 172 clauses with embedded extraction + generation config (410KB JSON)
- 14 per-contract-type template definitions across 5 verticals (Music Core, Music Ancillary, Film, TV, Cross-Entertainment)
- 24 contract types supported
- Entity resolution chain (847 lines) — account index, context scorer, Salesforce resolver
- 44 API route files (16,365 lines)
- 18 PostgreSQL tables, 25 migrations
- 50+ test files
- No real frontend — only HTML viewer prototypes
- This is reference code for porting logic, NOT the source of truth. Airlock specs supersede all OrcestrateOS patterns.

### Development Model

- **Claude Code** is the primary coding agent (not Cursor, not Copilot)
- Claude Code uses a **custom skill system** for structured workflows:
  - `brainstorming` — Explore intent, propose approaches, design before coding
  - `writing-plans` — Create detailed implementation plans from specs
  - `subagent-driven-development` — Dispatch fresh subagent per task with two-stage review (spec compliance, then code quality)
  - `test-driven-development` — TDD for each implementation task
  - `systematic-debugging` — Structured debugging workflow
  - `finishing-a-development-branch` — Merge/PR/cleanup decision workflow
  - `verification-before-completion` — Evidence-based completion claims
  - `requesting-code-review` / `receiving-code-review` — Structured code review
- **Multi-agent collaboration rules** prevent conflicts (documented in `claude.md`):
  - Canonical vocabulary enforced across all agents
  - Feature folder ownership (one agent per folder)
  - New sub-spec files for research (never modify `overview.md` without explicit permission)
  - Cross-referencing required between specs
- **NotebookLM** ingests all specs to create a knowledge base for generating documentation, training materials, and marketing content

---

## Research Domain 1: Agentic Coding Architecture

**What we have:** Claude Code with custom skills (brainstorming, writing-plans, subagent-driven-development, TDD, debugging, code review, branch finishing). Multi-agent collaboration rules in `claude.md`. The subagent-driven-development skill dispatches a fresh subagent per task with two-stage review (spec compliance first, then code quality).

**What we need to know:**

1. What are the best practices for multi-agent parallel development in 2025-2026? How do teams using Claude Code, Cursor, or similar AI coding tools structure agent work to prevent merge conflicts and context pollution?

2. How do large engineering teams (Vercel, Supabase, Linear, Stripe) structure their AI-assisted development workflows? What patterns exist for agent specialization (frontend agent vs. backend agent vs. testing agent)?

3. What orchestration patterns exist for coordinating multiple AI coding agents on the same codebase? How do you handle shared state, database migrations, and API contract changes when agents work in parallel?

4. What is the optimal context window strategy for AI coding agents working on a large codebase (57 specs, 25 features, eventually 50K+ lines of code)? How do you prevent context pollution while maintaining awareness of cross-cutting concerns?

5. How should CLAUDE.md (project-level agent instructions) be structured for maximum agent effectiveness? What patterns exist for project-level AI configuration files?

---

## Research Domain 2: Agent Task Lifecycle

**What we have:** A skill chain: brainstorm > write-plans > subagent-driven-development (dispatch implementer, spec reviewer, code quality reviewer per task) > finishing-a-development-branch. But we want to validate and improve this against industry best practices.

**What we need to know:**

1. What does the official, end-to-end lifecycle of an AI coding agent completing a task look like? Research how leading agentic development teams define the complete workflow from receiving a task to shipping code. Include every step: understanding requirements, exploring codebase, designing approach, implementing, testing, reviewing, committing, and merging.

2. How do AI-first development teams handle the handoff between design/planning and implementation? Specifically: when specs exist as markdown documents, what is the best process for an agent to consume the spec, verify understanding, and translate it into code?

3. What quality gates should exist in an AI agent's task lifecycle? Research patterns for: self-review before submission, spec compliance validation, code quality review, test coverage requirements, and human-in-the-loop checkpoints.

4. How should an AI agent handle blocked or ambiguous tasks? What escalation patterns exist? When should the agent ask clarifying questions vs. make a judgment call vs. flag for human review?

5. What post-completion automation should an agent perform? Research patterns for: auto-generating documentation from implemented code, updating task tracking, creating NotebookLM-compatible summaries, and triggering downstream workflows (e.g., updating related specs, notifying stakeholders).

---

## Research Domain 3: GitHub Workflow and Repo Commissioning

**What we have:** A design-documentation repo with no production code, no `.github/` directory, no CI/CD. An incomplete beta repo (OrcestrateOS) with a simple CI workflow (pytest + TruffleHog). We need to commission the first implementation repo.

**What we need to know:**

1. **Monorepo vs. polyrepo** for our specific stack: Next.js 14 (App Router, TypeScript) + FastAPI (Python) + shared configuration. What are the trade-offs? Compare: Turborepo, Nx, pnpm workspaces, or simple npm workspaces for managing a TypeScript frontend alongside a Python backend. How do companies with similar stacks (TypeScript + Python) structure their repos?

2. What should go in the **initial commit** when commissioning a new repo from existing design docs? Should the 57 markdown specs migrate into the implementation repo (e.g., `/docs/specs/`) or stay in a separate design repo? How do you preserve design history while starting fresh with production code?

3. **Branch strategy for AI-driven development:** Should we use trunk-based development with short-lived feature branches, or a more structured branching model? How do AI-authored PRs fit into the review workflow? What naming conventions work best when agents create branches?

4. **CI/CD pipeline design** for a Next.js + FastAPI + Docker monorepo: What GitHub Actions workflows should exist from day one? Include: TypeScript type-checking, ESLint, Prettier, Python linting (Ruff/Black), pytest, database migrations, preview deployments, and Docker build verification.

5. **Pre-commit hooks and code quality automation:** What toolchain works best for enforcing quality across TypeScript and Python in the same repo? Include: Husky, lint-staged, commitlint (conventional commits), and any monorepo-specific considerations.

6. **PR review workflow for AI-authored code:** How should PRs created by AI agents be reviewed differently from human-authored PRs? What automated checks should run? Should there be a mandatory human reviewer, or can AI-reviewed code be auto-merged under certain conditions?

---

## Research Domain 4: Repo Structure for Next.js 14 + FastAPI

**What we have:** 25 feature specs organized as `/Features/<FeatureName>/overview.md`. We need a production directory structure that mirrors this modular architecture.

**What we need to know:**

1. What is the optimal **monorepo directory layout** for Next.js 14 (App Router) + FastAPI + shared types + documentation? Provide a concrete directory tree. Consider: how Next.js App Router routes map to Airlock's Module > Chamber > View hierarchy (e.g., `/contracts/triage-board` = Contracts module > Discover chamber > Triage Board view).

2. How should **react-admin integrate into a Next.js App Router project**? react-admin expects its own routing — what patterns exist for embedding it as a sub-application within specific Next.js routes (the CRM module)?

3. How should **shared type definitions** work between TypeScript (frontend) and Python (backend)? Options: OpenAPI codegen, shared JSON schemas, manual sync. What works best for a team where AI agents write most of the code?

4. What **Docker Compose structure** works best for local development with: Next.js dev server, FastAPI with hot reload, PostgreSQL 16, Redis 7, Novu self-hosted, and LiteLLM proxy? How do you manage environment variables across 6+ services?

5. Where should the **57 existing markdown specs** live in the implementation repo? Options: `/docs/specs/` mirroring the feature structure, inline with source code (`/apps/web/src/features/contracts/SPEC.md`), or kept in a separate repo. What pattern keeps specs in sync with implementation?

---

## Research Domain 5: Custom Claude Code Skills

**What we have:** A skill system with brainstorming, planning, TDD, debugging, code review, and branch management skills. We need additional skills specific to Airlock's workflow.

**What we need to know:**

1. What **custom Claude Code skills** would provide the biggest productivity gains for this project? Evaluate these candidates and suggest others:
   - **Spec validator agent** — Reads a feature spec and validates that the implementation matches all requirements
   - **UI parity checker** — Compares HTML demo prototype to React implementation for visual/functional parity
   - **Seed data builder** — Generates deterministic test fixtures from spec definitions
   - **Acceptance test generator** — Reads spec acceptance criteria and generates Playwright/pytest tests
   - **NotebookLM prompt generator** — Produces structured summaries from implemented features for NotebookLM ingestion
   - **Design token enforcer** — Validates that components use Airlock theme tokens (not raw colors/sizes)
   - **Canonical vocabulary linter** — Flags any use of banned terms (channel, workstream, phase, stage, etc.)
   - **PR auditor** — Validates PRs against the linked spec before merge
   - **Migration validator** — Ensures database migrations are reversible and match schema specs

2. How should skills be structured for maximum reusability? What is the optimal format for a Claude Code skill file (prompt template, checklist, integration hooks)?

3. What patterns exist for **skill composition** — chaining multiple skills into a workflow (e.g., implement > validate > test > review > document)?

---

## Research Domain 6: Seed Data Strategy

**What we have:** An 800-line `fake_data.py` generator in OrcestrateOS that creates test data for 24 contract types. Real test entities from the beta (parent companies, their contracts, lifecycle stages) that need to be anonymized with dummy information before use in Airlock.

**What we need to know:**

1. How do **enterprise SaaS platforms** (Salesforce, HubSpot, Notion, Linear) structure their seed/demo data for development and testing? What makes seed data effective vs. just filling tables with random strings?

2. What is the best approach for **scenario-based seed bundles**? We need test data that exercises:
   - All 4 chambers (Discover > Build > Review > Ship) with vaults at different stages
   - All 3 roles (Builder, Gatekeeper, Owner) with appropriate permissions
   - The full vault hierarchy (Parent > Division > Counterparty > Item)
   - Edge cases: corrupted documents, encoding issues, duplicate entities, ambiguous company names
   - Entity resolution scenarios: subsidiary trees, abbreviations, merged/acquired companies
   - Approval pipelines: pending, approved, rejected, escalated patches

3. How should seed data be **anonymized** from real production scenarios? We have real test entities from our beta that contain realistic workflows but potentially sensitive information. What patterns exist for creating faithful anonymized clones that preserve relationship structures, timing patterns, and edge cases?

4. What tooling exists for **deterministic random generation** of enterprise data? We need reproducible test datasets (same seed = same data) with realistic timestamps, company names, contract terms, and financial figures.

5. How should seed data integrate with the **development workflow**? Should it be: SQL migration files, JSON fixtures, programmatic generators (like our existing `fake_data.py`), or a combination? How do you keep seed data in sync as the schema evolves?

---

## Research Domain 7: NotebookLM Knowledge Generation

**What we have:** 57 markdown specs + 4 research PDFs already structured for NotebookLM ingestion. The goal is that every implemented feature automatically produces documentation for any audience.

**What we need to know:**

1. What is the **optimal structure for NotebookLM source documents**? Should each feature be one document or split by sub-spec? What formatting (headers, tables, code blocks, diagrams) produces the best NotebookLM outputs?

2. How should we structure **prompts for NotebookLM** to generate different content types from the same source material? We need to generate:
   - Developer documentation (API reference, architecture guide, code examples)
   - Administrator guides (configuration, permissions, feature control)
   - Business user documentation (workflows, use cases, best practices)
   - Training materials (onboarding guides, role-specific tutorials)
   - Marketing content (feature explanations, competitive advantages, industry use cases)
   - Executive summaries (ROI analysis, capability overviews)
   - Sales materials (demo scripts, feature comparison charts)

3. What is the **NotebookLM source limit** and how should we prioritize 57+ markdown files? Can we create a hierarchical ingestion strategy (core architecture docs first, then module-specific specs)?

4. How do we create a **documentation flywheel** where: specs generate code > code generates API docs > API docs + specs feed NotebookLM > NotebookLM generates user docs + training > training informs new specs? What tooling supports this cycle?

5. What post-implementation automation should exist so that when an agent finishes a feature, a NotebookLM-compatible summary (under 5,000 characters) is automatically generated?

---

## Research Domain 8: Documentation Ecosystem Design

**What we have:** 16,545 lines of design specs. We want to build a documentation ecosystem comparable to Salesforce or Discord's developer documentation.

**What we need to know:**

1. What does a **Salesforce-level documentation ecosystem** look like structurally? Break down the categories: admin guide, developer guide, API reference, user manual, architecture guide, integration guide, release notes, troubleshooting guide. How many documents does a mature enterprise SaaS platform maintain?

2. How does **Discord structure its developer documentation**? What patterns make it effective? How do they handle versioning, search, and cross-referencing between docs?

3. What **documentation-as-code tools** work best for a Next.js + FastAPI stack? Evaluate: Nextra (Next.js docs framework), Docusaurus, Mintlify, Fumadocs, Starlight. Also: TypeDoc (TypeScript API docs), Sphinx (Python API docs), Storybook (component documentation).

4. How do you maintain **consistency between spec docs and auto-generated API docs**? When a FastAPI endpoint changes, how does the developer documentation automatically update?

5. What patterns exist for **modular documentation** that can be assembled for different audiences? E.g., the "Vault" concept needs explanation for: developers (data model, API), admins (permissions, configuration), business users (workflow, lifecycle), and marketing (value proposition). How do you write once and assemble many ways?

6. How should documentation be **organized for scalability**? With 25 features across 5 modules, what information architecture prevents the docs from becoming an unnavigable mess as the platform grows?

---

## Output Format Requested

For each of the 8 research domains, provide:

1. **Recommended approach** — Your top recommendation with rationale
2. **2-3 alternatives** — With trade-offs and when each is appropriate
3. **Specific tools and libraries** — With links to repos/docs, noting license and maintenance status
4. **Concrete examples** — Directory structures, configuration files, workflow diagrams, or code patterns where applicable
5. **Reference implementations** — Link to GitHub repos, blog posts, or documentation from teams that have solved similar problems
6. **Airlock-specific recommendations** — Tailored to the locked tech stack (Next.js 14, FastAPI, PostgreSQL 16, Redis 7, etc.) — not generic advice

For Domain 2 (Agent Task Lifecycle), provide a **complete step-by-step workflow** showing every stage an AI coding agent goes through from receiving a task to shipping code, including decision points, quality gates, and handoff protocols.

Focus on patterns from 2025-2026. Prioritize self-hosted, MIT/Apache-2.0 licensed solutions. Cite your sources.
