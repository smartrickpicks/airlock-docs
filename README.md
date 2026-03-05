# Airlock

> Design documentation for Airlock — a Discord-like platform for enterprise data operations.

## What Is This Repo?

This repo contains **design specifications only**. No production code. Everything here feeds into [Google NotebookLM](https://notebooklm.google.com/) as the knowledge base for implementation.

**Contents:**
- 59 markdown spec files across 25 feature folders
- 18 HTML demo prototypes (interactive visual mockups)
- 1 master demo hub serving all demos
- 4 research whitepapers (PDFs)
- 17 architecture diagrams (PNGs/JPGs)

## Quick Start

Serve the demo hub locally:

```bash
python3 -m http.server 5173
```

Then open [http://localhost:5173/demo-shell/](http://localhost:5173/demo-shell/)

## Architecture

Airlock replaces OrcestrateOS's monolithic frontend with a modular, channel-based architecture.

- **Modules** = Discord "servers" (Contracts, CRM, Tasks, Calendar, Documents)
- **Vaults** = Workflow instances within modules
- **Chambers** = Lifecycle stages: Discover > Build > Review > Ship
- **Triptych** = Signal (left) | Orchestrate (center) | Control (right)

See [Features/Shell/overview.md](Features/Shell/overview.md) for the full layout spec.

## Feature Index

See [Features/start.md](Features/start.md) for the master index of all 25 features with status.

## Repo Structure

```
/Airlock/
  CLAUDE.md              # Agent instructions (project constitution)
  README.md              # This file
  Features/
    start.md             # Master feature index
    Shell/               # Master layout, triptych, chambers
    Record Inspector/    # Field cards, entity resolution
    Triage/              # Contracts triage dashboard
    ContractGenerator/   # Template builder, v2 clause library
    Admin/               # Workspace admin + personal settings
    OrgOverview/         # Gatekeeper's cross-vault review queue
    PatchWorkflow/       # Approval chain, SLA timer
    AIAgent/             # Otto AI agent (PydanticAI + LiteLLM)
    ... (25 feature folders total)
  demo-shell/            # Master demo hub
  Pub Specs/             # Architecture diagrams (Git LFS)
  NotebookLM/            # Research whitepapers (Git LFS)
```

## Tech Stack (Locked)

| Layer | Technology |
|-------|-----------|
| Frontend | React, Next.js 14 (App Router), TypeScript, Zustand, Tailwind |
| Backend | FastAPI (Python), PostgreSQL 16, WebSockets |
| Infrastructure | Docker Compose, Redis 7 |
| AI Runtime | PydanticAI, LiteLLM proxy |
| CRM | react-admin + Atomic CRM |
| Tasks | @hello-pangea/dnd (Kanban) |
| Calendar | Schedule-X |
| Notifications | Novu self-hosted |
| Event Bus | BullMQ + Trigger.dev |
| Editor | TipTap |
| PDF | PDF.js |
| Workflow | React Flow |

Full stack decisions in [Features/TechStack/overview.md](Features/TechStack/overview.md).

## Related Repos

- **airlock-docs** (this repo) — Design documentation
- **airlock** (planned) — Implementation monorepo (Turborepo + pnpm)
- **OrcestrateOS** (reference) — Source beta being ported
