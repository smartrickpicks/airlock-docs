# Search & Command Palette — Design Spec

> **Status:** SPECCED — Cmd+K command palette for universal search and quick actions.

> **Key Insight:** The command palette is the power user's primary navigation tool. It searches across vaults, tasks, events, and actions — all from a single input. It's always available via Cmd+K, regardless of which module or view is active.

---

## Access

| Trigger | Action |
|---------|--------|
| `Cmd+K` / `Ctrl+K` | Open command palette |
| Click search icon in module bar | Open command palette |
| `/` in any input (when not focused) | Open command palette |
| `Escape` | Close command palette |

---

## Layout

Centered modal overlay with search input and categorized results:

```
+----------------------------------------------------------+
|  [Search icon]  Search vaults, tasks, actions...    [ESC] |
|----------------------------------------------------------|
|                                                           |
|  RECENT                                                   |
|  > Henderson MSA              Contracts · Review · 85%    |
|  > Acme Corp                  CRM · Build                 |
|  > Fix auth timeout           Tasks · In Progress         |
|                                                           |
|  VAULTS                                                   |
|  > Henderson MSA              Contracts · Review · 85%    |
|  > Sony Distribution 2024    Contracts · Build · 62%      |
|  > Warner Amendment           Contracts · Discover · 41%  |
|                                                           |
|  TASKS                                                    |
|  > Enrich Acme Corp account  CRM · Open · Due 2d         |
|  > Review Henderson patch    Contracts · In Review        |
|                                                           |
|  ACTIONS                                                  |
|  > /upload-contract           Upload a new contract       |
|  > /new-task                  Create a new task           |
|  > /goto contracts            Switch to Contracts module  |
|                                                           |
+----------------------------------------------------------+
```

---

## Search Categories

### 1. Vaults (Primary)

Search across all vault names, entity names, and vault metadata.

| Field searched | Weight | Notes |
|---------------|--------|-------|
| Vault name | Highest | Primary display name |
| Entity/counterparty name | High | Parent and counterparty vault names |
| Vault type | Medium | Contract type, deal type, etc. |
| Module | Medium | "contracts", "crm", "tasks" |
| Chamber | Low | "discover", "build", "review", "ship" |

**Result display:** Vault name + module badge + chamber + health % (if applicable)

**Click action:** Navigate to vault triptych: `/{module}/{vault-slug}`

### 2. Tasks

Search across task titles and descriptions from the Universal Task System.

| Field searched | Weight |
|---------------|--------|
| Task title | Highest |
| Task description | Medium |
| Assigned vault name | Medium |
| Module type | Low |

**Result display:** Task title + module badge + status badge + due date

**Click action:** Navigate to task's source vault, or open Tasks module focused on that task.

### 3. Events

Search recent events across all vaults (last 7 days).

| Field searched | Weight |
|---------------|--------|
| Event payload text | Medium |
| Event type | Low |
| Vault name | Medium |

**Result display:** Event headline + vault name + timestamp

**Click action:** Navigate to vault's Signal panel, scrolled to event.

### 4. Actions (Commands)

Slash commands and quick actions. Always available, no search needed (filtered by typing).

| Command | Action |
|---------|--------|
| `/upload-contract` | Open contract upload dialog |
| `/new-task` | Open task creation form |
| `/new-patch {vault}` | Open patch editor for a vault |
| `/goto {module}` | Switch to module |
| `/goto {module}/{view}` | Switch to specific view |
| `/settings` | Open Personal Settings overlay |
| `/admin` | Open Workspace Admin overlay (admin+ only) |
| `/theme dark` / `/theme light` | Switch theme |
| `/export {vault}` | Export vault data |

### 5. People

Search workspace members.

| Field searched | Weight |
|---------------|--------|
| Display name | Highest |
| Email | High |
| Role | Low |

**Result display:** Name + role badge + status (online/offline)

**Click action:** Filter current view by that person, or show their profile.

---

## Search Behavior

### Instant Search (Client-Side)

For small workspaces (< 500 vaults), search runs entirely client-side against a pre-loaded index:

- Index loaded on app boot
- Updated via WebSocket events (new vault, vault renamed, etc.)
- Fuzzy matching with Fuse.js or similar
- Results appear as user types (no debounce needed)

### Server Search (Large Workspaces)

For larger workspaces, search queries the API with debounce:

```
GET /api/search?q=henderson&workspace_id=ws_123&limit=10
```

Response groups results by category:

```json
{
  "vaults": [{ "id": "...", "name": "Henderson MSA", "module": "contracts", "chamber": "review", "health": 85 }],
  "tasks": [{ "id": "...", "title": "Review Henderson patch", "module": "contracts", "status": "in_review" }],
  "events": [],
  "actions": [{ "command": "/goto contracts/henderson-msa", "label": "Go to Henderson MSA" }]
}
```

### Threshold

| Workspace size | Strategy |
|---------------|----------|
| < 500 vaults | Client-side index (Fuse.js) |
| 500+ vaults | Server-side search with debounce (250ms) |

---

## Keyboard Navigation

| Key | Action |
|-----|--------|
| `Arrow Up/Down` | Navigate results |
| `Enter` | Execute selected result |
| `Tab` | Cycle between categories |
| `Escape` | Close palette (or clear input if text present) |
| `Cmd+K` (when open) | Focus search input |

---

## Recent Items

The palette opens with a "Recent" section showing the last 5-10 items the user navigated to. Stored in client-side state (localStorage), scoped per user.

| Tracked | Not tracked |
|---------|-------------|
| Vault visits | Search queries |
| View navigation | Filter changes |
| Task opens | Hover actions |

---

## Scoped Search

When inside a module, the palette can be scoped:

| Context | Default scope | Behavior |
|---------|--------------|----------|
| Home | All modules | Global search |
| Contracts module | Contracts vaults + tasks | Results from contracts first, others below |
| CRM module | CRM vaults + tasks | CRM results first |
| Any module | Scoped to active module | Toggle: "Search all modules" link at bottom |

---

## Role-Based Filtering

| Role | Actions visible | Vaults visible |
|------|----------------|----------------|
| Builder | Create task, create patch, upload | Only assigned vaults |
| Gatekeeper | All Builder + approve, review | Assigned + team vaults |
| Owner | All Gatekeeper + admin commands | All vaults |
| Architect | All | All |

Actions are hidden, not disabled. If a Builder types `/admin`, nothing appears.

---

## Implementation Notes

### API Endpoint

```
GET /api/search
  ?q=string           (search query, min 2 chars)
  &workspace_id=uuid
  &module=string      (optional scope)
  &categories=string  (comma-separated: vaults,tasks,events,actions)
  &limit=number       (default 10 per category)
```

### Search Index (PostgreSQL)

For server-side search, use PostgreSQL full-text search:

```sql
-- Add tsvector column to vaults table
ALTER TABLE vaults ADD COLUMN search_vector tsvector;
CREATE INDEX idx_vaults_search ON vaults USING gin(search_vector);

-- Update trigger
CREATE TRIGGER trg_vaults_search_update
BEFORE INSERT OR UPDATE ON vaults
FOR EACH ROW EXECUTE FUNCTION
  tsvector_update_trigger(search_vector, 'pg_catalog.english', name, vault_type);
```

For fuzzy matching: `pg_trgm` extension with `similarity()` function.

---

## Search Indexing Engine — Brainstorm

> **Status:** BRAINSTORM — Evaluating a dedicated search index engine to replace client-side Fuse.js and server-side pg_trgm for cross-platform search.

### Why Upgrade Beyond pg_trgm

The current search architecture relies on Fuse.js for small workspaces and PostgreSQL full-text search (`tsvector` + `pg_trgm`) for larger ones. This works for single-table queries, but Airlock's search surface spans **6+ modules** — vaults, tasks, events, contacts, documents, and communications. The requirements that push us beyond `pg_trgm`:

- **Cross-module unified search** — A single query like "Thomas" must return results from vaults, CRM contacts, tasks, emails, documents, and workspace members simultaneously. Doing this across multiple PostgreSQL tables with JOINs and UNIONs is slow and complex.
- **Badge-based result types** — Every result must carry a module badge (Contracts, CRM, Tasks, etc.) so users can distinguish "Thomas" the person from "Thomas" in a contract from "Thomas" in an email thread.
- **Typo-tolerant fuzzy matching** — `pg_trgm` supports trigram similarity, but it is not designed for typo-tolerant instant search with ranked results across heterogeneous document types.
- **Instant results (<50ms)** — A dedicated search engine keeps a denormalized, pre-computed index in memory. PostgreSQL full-text search requires disk I/O and query planning overhead that grows with data volume.
- **Faceted filtering** — Module scoping, role-based visibility, and chamber filtering are natural facets in a search engine but require complex WHERE clauses in SQL.

### MeiliSearch vs Typesense vs ElasticSearch

| Criteria | MeiliSearch | Typesense | ElasticSearch |
|----------|-------------|-----------|---------------|
| **Language** | Rust | C++ | Java (JVM) |
| **Binary size** | ~70 MB | ~30 MB | ~500 MB+ (JVM) |
| **Typo tolerance** | Built-in, excellent | Built-in, excellent | Plugin-based, config-heavy |
| **Instant search** | Yes, <50ms out of the box | Yes, <50ms out of the box | Possible but requires tuning |
| **License** | MIT | GPL-3 (cloud: commercial) | SSPL (not OSI-approved) |
| **Docker footprint** | Lightweight (~128 MB RAM) | Lightweight (~128 MB RAM) | Heavy (~1 GB+ RAM minimum) |
| **Best for** | <1M documents, instant UX | <1M documents, instant UX | Enterprise scale, 10M+ docs |
| **Faceted search** | Native | Native | Native |
| **Multi-index search** | Federated search across indices | Multi-search API | Cross-index queries |
| **SDKs** | Python, JS, Go, Rust, etc. | Python, JS, Go, Ruby, etc. | Official clients for all langs |
| **Self-hosted complexity** | Single binary, minimal config | Single binary, minimal config | Cluster setup, JVM tuning |

**Recommendation: MeiliSearch**

MeiliSearch is the best fit for Airlock at current scale (<1M documents). It is MIT-licensed, ships as a single Rust binary (~70 MB), requires minimal RAM (~128 MB), has built-in typo tolerance and instant search, and provides federated multi-index search for querying vaults, tasks, contacts, and documents in a single request. It aligns with Airlock's existing stack (FastAPI + BullMQ + Redis) without introducing JVM overhead or SSPL licensing concerns.

### Indexed Content Types

Each module writes to its own MeiliSearch index. Federated search queries all indices in a single request and merges results with module badges.

| Index name | Source table(s) | Fields indexed | Badge label |
|------------|----------------|----------------|-------------|
| `vaults` | `vaults`, `vault_metadata` | name, entity_name, vault_type, module, chamber, metadata (JSON) | Contracts / CRM / etc. (by module) |
| `tasks` | `tasks` | title, description, assigned_vault_name, assignee_name, status, due_date | Tasks |
| `contacts` | `vaults` (type=counterparty/person) | name, email, phone, role, company (parent vault name) | CRM |
| `documents` | `documents`, `document_text` | filename, extracted_text, annotations, vault_name | Documents |
| `communications` | `messages`, `emails` | message_body, sender_name, sender_email, intent_classification, vault_name | Communications |
| `events` | `events` | event_payload_text, event_type, vault_name, timestamp | Events |
| `members` | `workspace_members` | display_name, email, role, status | People |

### Badge System in Results

Every search result carries a **module badge** that visually tags where the result lives. This is the core UX differentiator for cross-platform search: when a user types "Thomas", they see:

| Result | Badge | Detail |
|--------|-------|--------|
| Thomas Burke | `CRM` | Contact, VP Sales at Acme Corp |
| Thomas in Henderson MSA | `Contracts` | Vault, Review chamber, 85% health |
| Thomas mentioned in patch notes | `Documents` | Document, henderson-msa-patch-v3.pdf |
| Email from Thomas about renewal | `Communications` | Email, intent: renewal inquiry |
| Thomas assigned to fix auth | `Tasks` | Task, In Progress, due 2d |
| Thomas Burke (workspace member) | `People` | Builder role, online |

Badge colors map to module theme colors defined in the shell theme spec. Each badge is clickable and navigates to the source record.

### Indexing Pipeline

PostgreSQL remains the source of truth. MeiliSearch is a read-only search replica kept in sync via an event-driven pipeline.

```
PostgreSQL (source of truth)
    |
    +-- pg_notify on INSERT/UPDATE
    |
    +-- BullMQ indexing job
    |
    +-- MeiliSearch (search index)
    |
FastAPI /api/search --> MeiliSearch --> formatted results with badges
```

**Sync mechanism:**

1. **PostgreSQL triggers** fire `pg_notify` on INSERT, UPDATE, or DELETE to any indexed table (vaults, tasks, documents, etc.).
2. **FastAPI listener** receives `pg_notify` events and enqueues a BullMQ job with the table name, record ID, and operation type.
3. **BullMQ indexing worker** fetches the full record from PostgreSQL, transforms it into the MeiliSearch document schema, and pushes it to the appropriate MeiliSearch index via the MeiliSearch Python SDK.
4. **Deletes** are handled by removing the document from MeiliSearch by its primary key.
5. **Bulk reindex** job available for initial setup or recovery: iterates all rows in a table and pushes them to MeiliSearch in batches of 1,000.
6. **Real-time awareness** — The existing WebSocket event bus (used for vault updates, task changes, etc.) can also trigger client-side search index invalidation so the command palette re-fetches when data changes.

### Search API Change

The existing `/api/search` endpoint gains an internal routing layer:

```
GET /api/search
  ?q=string           (search query, min 1 char for MeiliSearch)
  &workspace_id=uuid
  &module=string      (optional scope — filters to one MeiliSearch index)
  &categories=string  (comma-separated index names to query)
  &limit=number       (default 5 per index, max 20)
```

**Behavior:**

- If the workspace has MeiliSearch indexing enabled, the endpoint queries MeiliSearch using federated multi-index search and returns badged results grouped by module.
- If MeiliSearch is unavailable or not configured, the endpoint falls back to the existing PostgreSQL `tsvector` + `pg_trgm` search (graceful degradation).
- Response format remains the same as the current spec (grouped by category), with the addition of a `badge` field on each result:

```json
{
  "vaults": [{ "id": "...", "name": "Thomas Henderson MSA", "module": "contracts", "chamber": "review", "health": 85, "badge": "Contracts" }],
  "contacts": [{ "id": "...", "name": "Thomas Burke", "email": "tburke@acme.com", "role": "VP Sales", "badge": "CRM" }],
  "tasks": [{ "id": "...", "title": "Thomas review pending", "module": "contracts", "status": "in_review", "badge": "Tasks" }],
  "communications": [{ "id": "...", "snippet": "Thomas mentioned renewal terms...", "sender": "tburke@acme.com", "badge": "Communications" }],
  "members": [{ "id": "...", "name": "Thomas Burke", "role": "builder", "status": "online", "badge": "People" }],
  "actions": [{ "command": "/goto crm/thomas-burke", "label": "Go to Thomas Burke", "badge": "Action" }]
}
```

---

## Related Specs

- **[Shell / Naming](../Shell/naming-and-hierarchy.md)** — Cmd+K is a shell-level feature, not module-specific.
- **[Universal Task System](../TaskSystem/overview.md)** — Tasks are a searchable category.
- **[Roles & Permissions](../Roles/overview.md)** — Search results filtered by role.
