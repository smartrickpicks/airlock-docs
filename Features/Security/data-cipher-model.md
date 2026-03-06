# Data Cipher Model -- Design Spec

> **Status:** SPECCED
> **Source:** Security overview.md (Q8-Q18, A1-A3, A12), OrcestrateOS codebase analysis
> **Resolves:** A1 (cipher scope), A2 (key custody), A12 (file storage encryption)

## Summary

This document defines the complete encryption-at-rest architecture for Airlock. It specifies which data is encrypted, how it is encrypted, who holds the keys, and how the system migrates from OrcestrateOS's plaintext storage to Airlock's cipher-first model. Every decision includes rationale and trade-offs considered. This is the authoritative reference for any agent writing encryption code.

No production code. Design docs only.

---

## 1. Cipher Scope (Resolves A1)

### Decision: Column-level encryption for sensitive fields, NOT full Transparent Data Encryption (TDE).

**Rationale:** TDE (PostgreSQL's `pg_tde` or filesystem-level LUKS/dm-crypt) encrypts the entire database at the block level. It protects against one threat: physical disk theft. It does NOT protect against:

- SQL injection (attacker queries through the application -- sees plaintext because TDE decrypts transparently)
- Database admin access (DBA connects via `psql` and reads all data)
- Backup theft without TDE keys (but backups made with `pg_dump` produce plaintext SQL)
- Application-layer vulnerabilities (SSRF, IDOR -- the application already has the TDE key loaded)

Column-level encryption addresses all four of these threats. The application encrypts before writing and decrypts after reading. PostgreSQL never sees plaintext for sensitive columns. A stolen `pg_dump` backup contains only ciphertext for encrypted columns.

**Trade-off acknowledged:** Column-level encryption prevents PostgreSQL from querying, sorting, or indexing encrypted columns using native SQL operators. This requires blind indexes (HMAC-based hashes stored in separate columns) for any encrypted field that must be searchable, and application-layer filtering for range queries or sorting. This trade-off is acceptable because:

1. Most sensitive fields (file URLs, message content, before/after values) are never used in WHERE clauses
2. Fields that require lookup (email, phone) use blind indexes for exact-match dedup
3. All search functionality already routes through the Search module (application-layer), not raw SQL

### Encryption Classification: All OrcestrateOS Tables

The following classification covers every table in OrcestrateOS's schema (18 core tables from migration 001, plus tables added in subsequent migrations). Each column is classified using four sensitivity tiers:

| Tier | Label | Definition |
|------|-------|------------|
| T1 | **Public** | Safe to expose in logs, error messages, and API responses without restriction |
| T2 | **Internal** | Business metadata that should not leak externally but is not PII or confidential |
| T3 | **Confidential** | Business-sensitive data (financial terms, extraction results, legal content) |
| T4 | **Restricted/PII** | Personally identifiable information or data whose exposure causes direct harm |

---

### Table 1: `workspaces`

| Column | Classification | Encrypted? | Blind Index? | Rationale |
|--------|---------------|------------|-------------|-----------|
| id | T1 Public | NO | N/A | UUID primary key, used in all foreign keys and RLS |
| name | T2 Internal | NO | N/A | Workspace name needed for display, search, and admin UI. Not PII by itself |
| mode | T1 Public | NO | N/A | Enum value (`sandbox`/`production`), used in query filters |
| created_at | T1 Public | NO | N/A | Timestamp, used in ordering and audit |
| updated_at | T1 Public | NO | N/A | Timestamp |
| deleted_at | T1 Public | NO | N/A | Soft-delete timestamp |
| version | T1 Public | NO | N/A | Optimistic lock counter |
| metadata (JSONB) | T2 Internal | NO | N/A | Workspace config (feature flags, settings). No PII by design -- enforce at application layer |

**Encryption count:** 0 columns. Workspace-level metadata is structural, not sensitive.

---

### Table 2: `users`

| Column | Classification | Encrypted? | Blind Index? | Rationale |
|--------|---------------|------------|-------------|-----------|
| id | T1 Public | NO | N/A | UUID primary key |
| email | T4 PII | YES | YES (HMAC) | Email is PII. Blind index enables login lookup and dedup |
| display_name | T4 PII | YES | NO | Full name is PII. Not searched by SQL -- displayed in UI after decryption |
| avatar_url | T2 Internal | NO | N/A | URL to avatar image. Not PII -- typically a gravatar hash or OAuth profile URL |
| google_sub | T4 PII | YES | YES (HMAC) | Google OIDC subject identifier. Links user to Google identity. Blind index for OAuth login matching |
| status | T1 Public | NO | N/A | Enum (`active`/`inactive`), used in query filters |
| created_at | T1 Public | NO | N/A | Timestamp |
| updated_at | T1 Public | NO | N/A | Timestamp |

**Encryption count:** 3 columns (email, display_name, google_sub). 2 blind indexes (email, google_sub).

---

### Table 3: `user_workspace_roles`

| Column | Classification | Encrypted? | Blind Index? | Rationale |
|--------|---------------|------------|-------------|-----------|
| user_id | T1 Public | NO | N/A | Foreign key to users |
| workspace_id | T1 Public | NO | N/A | Foreign key to workspaces |
| role | T1 Public | NO | N/A | Enum, used in permission computation. Must remain queryable |
| granted_at | T1 Public | NO | N/A | Timestamp |
| granted_by | T1 Public | NO | N/A | Foreign key to users |

**Encryption count:** 0 columns. Role assignments are structural metadata.

---

### Table 4: `batches`

| Column | Classification | Encrypted? | Blind Index? | Rationale |
|--------|---------------|------------|-------------|-----------|
| id | T1 Public | NO | N/A | UUID primary key |
| workspace_id | T1 Public | NO | N/A | Foreign key, RLS filter |
| name | T2 Internal | YES | NO | Batch names may contain client names or project identifiers (e.g., "Sony Q4 2024 Upload") |
| source | T1 Public | NO | N/A | Enum (`upload`/`merge`/`import`) |
| batch_fingerprint | T2 Internal | NO | N/A | Content hash for dedup -- not sensitive, derived from file hashes |
| status | T1 Public | NO | N/A | Enum |
| record_count | T1 Public | NO | N/A | Numeric aggregate |
| created_at | T1 Public | NO | N/A | Timestamp |
| updated_at | T1 Public | NO | N/A | Timestamp |
| deleted_at | T1 Public | NO | N/A | Soft-delete timestamp |
| version | T1 Public | NO | N/A | Optimistic lock |
| metadata (JSONB) | T2 Internal | NO | N/A | Batch config, no PII by design |
| upload_session_id | T1 Public | NO | N/A | Foreign key (added migration 015) |

**Encryption count:** 1 column (name).

---

### Table 5: `accounts`

| Column | Classification | Encrypted? | Blind Index? | Rationale |
|--------|---------------|------------|-------------|-----------|
| id | T1 Public | NO | N/A | UUID primary key |
| batch_id | T1 Public | NO | N/A | Foreign key |
| workspace_id | T1 Public | NO | N/A | RLS filter |
| account_name | T3 Confidential | YES | YES (HMAC) | Counterparty legal entity names are business-confidential. Blind index needed for entity resolution dedup |
| billing_country | T2 Internal | NO | N/A | Country code, used in filtering. Low sensitivity |
| billing_city | T3 Confidential | YES | NO | City + account name could identify specific business relationships |
| account_fingerprint | T2 Internal | NO | N/A | Content hash for dedup, derived from name normalization |
| created_at | T1 Public | NO | N/A | Timestamp |
| updated_at | T1 Public | NO | N/A | Timestamp |
| deleted_at | T1 Public | NO | N/A | Soft-delete |
| version | T1 Public | NO | N/A | Optimistic lock |
| metadata (JSONB) | T3 Confidential | YES | NO | May contain entity resolution results with legal entity details, addresses, subsidiary info |

**Encryption count:** 3 columns (account_name, billing_city, metadata). 1 blind index (account_name).

---

### Table 6: `contracts`

| Column | Classification | Encrypted? | Blind Index? | Rationale |
|--------|---------------|------------|-------------|-----------|
| id | T1 Public | NO | N/A | UUID primary key |
| batch_id | T1 Public | NO | N/A | Foreign key |
| account_id | T1 Public | NO | N/A | Foreign key |
| workspace_id | T1 Public | NO | N/A | RLS filter |
| contract_fingerprint | T2 Internal | NO | N/A | Content hash for dedup |
| contract_id_source | T1 Public | NO | N/A | Enum |
| file_url | T4 Restricted | YES | NO | Document storage location reveals business relationships and file structure |
| file_name | T3 Confidential | YES | YES (HMAC) | Filenames contain client names, deal names, dates (e.g., "Sony_Distribution_2024.pdf") |
| status | T1 Public | NO | N/A | Enum, used in query filters |
| health_score | T1 Public | NO | N/A | Numeric score |
| created_at | T1 Public | NO | N/A | Timestamp |
| updated_at | T1 Public | NO | N/A | Timestamp |
| deleted_at | T1 Public | NO | N/A | Soft-delete |
| version | T1 Public | NO | N/A | Optimistic lock |
| metadata (JSONB) | T3 Confidential | YES | NO | Contains extraction results: legal entity names, financial terms, territory definitions, royalty rates |
| upload_session_id | T1 Public | NO | N/A | Foreign key (migration 015) |
| autospark_run_id | T1 Public | NO | N/A | External job ID (migration 019) |
| autospark_result_json (JSONB) | T3 Confidential | YES | NO | AutoSpark extraction results containing contract terms (migration 019) |
| autospark_submitted_at | T1 Public | NO | N/A | Timestamp |
| autospark_completed_at | T1 Public | NO | N/A | Timestamp |
| canonical_copy_id | T1 Public | NO | N/A | Foreign key (migration 020) |
| canonical_copy_state | T1 Public | NO | N/A | Enum |
| canonical_copy_updated_at | T1 Public | NO | N/A | Timestamp |

**Encryption count:** 4 columns (file_url, file_name, metadata, autospark_result_json). 1 blind index (file_name).

---

### Table 7: `documents`

| Column | Classification | Encrypted? | Blind Index? | Rationale |
|--------|---------------|------------|-------------|-----------|
| id | T1 Public | NO | N/A | UUID primary key |
| contract_id | T1 Public | NO | N/A | Foreign key |
| batch_id | T1 Public | NO | N/A | Foreign key |
| workspace_id | T1 Public | NO | N/A | RLS filter |
| document_fingerprint | T2 Internal | NO | N/A | Content hash for dedup |
| file_url | T4 Restricted | YES | NO | Document storage location |
| file_name | T3 Confidential | YES | YES (HMAC) | Filenames contain client and deal information |
| section_name | T2 Internal | NO | N/A | Section label (e.g., "recitals", "scope") -- structural, not sensitive |
| created_at | T1 Public | NO | N/A | Timestamp |
| updated_at | T1 Public | NO | N/A | Timestamp |
| deleted_at | T1 Public | NO | N/A | Soft-delete |
| version | T1 Public | NO | N/A | Optimistic lock |
| metadata (JSONB) | T3 Confidential | YES | NO | Document-level metadata may contain extraction fragments |

**Encryption count:** 3 columns (file_url, file_name, metadata). 1 blind index (file_name).

---

### Table 8: `patches`

| Column | Classification | Encrypted? | Blind Index? | Rationale |
|--------|---------------|------------|-------------|-----------|
| id | T1 Public | NO | N/A | UUID primary key |
| workspace_id | T1 Public | NO | N/A | RLS filter |
| batch_id | T1 Public | NO | N/A | Foreign key |
| record_id | T1 Public | NO | N/A | Foreign key reference |
| field_key | T2 Internal | NO | N/A | Field identifier (e.g., "OPP_TERRITORY"), structural |
| author_id | T1 Public | NO | N/A | Foreign key |
| status | T1 Public | NO | N/A | Enum, used in workflow queries |
| intent | T2 Internal | NO | N/A | Short intent label, no PII |
| when_clause (JSONB) | T3 Confidential | YES | NO | Condition logic may reference field values containing deal terms |
| then_clause (JSONB) | T3 Confidential | YES | NO | Action definition may contain proposed field values |
| because_clause | T3 Confidential | YES | NO | Free-text justification may reference client names or deal specifics |
| evidence_pack_id | T1 Public | NO | N/A | Foreign key |
| submitted_at | T1 Public | NO | N/A | Timestamp |
| resolved_at | T1 Public | NO | N/A | Timestamp |
| file_name | T3 Confidential | YES | NO | Source document filename |
| file_url | T4 Restricted | YES | NO | Document location |
| before_value | T3 Confidential | YES | NO | May contain financial terms, royalty rates, territory names |
| after_value | T3 Confidential | YES | NO | May contain financial terms, royalty rates, territory names |
| history (JSONB) | T3 Confidential | YES | NO | Full patch history includes all before/after snapshots |
| created_at | T1 Public | NO | N/A | Timestamp |
| updated_at | T1 Public | NO | N/A | Timestamp |
| deleted_at | T1 Public | NO | N/A | Soft-delete |
| version | T1 Public | NO | N/A | Optimistic lock |
| metadata (JSONB) | T3 Confidential | YES | NO | May contain extraction context |

**Encryption count:** 9 columns (when_clause, then_clause, because_clause, file_name, file_url, before_value, after_value, history, metadata). 0 blind indexes -- patches are queried by status/author/workspace, never by encrypted content.

---

### Table 9: `evidence_packs`

| Column | Classification | Encrypted? | Blind Index? | Rationale |
|--------|---------------|------------|-------------|-----------|
| id | T1 Public | NO | N/A | UUID primary key |
| patch_id | T1 Public | NO | N/A | Foreign key |
| workspace_id | T1 Public | NO | N/A | RLS filter |
| author_id | T1 Public | NO | N/A | Foreign key |
| blocks (JSONB) | T3 Confidential | YES | NO | Evidence blocks contain document excerpts, selection text, and quoted contract clauses |
| status | T1 Public | NO | N/A | Enum |
| created_at | T1 Public | NO | N/A | Timestamp |
| updated_at | T1 Public | NO | N/A | Timestamp |
| deleted_at | T1 Public | NO | N/A | Soft-delete |
| version | T1 Public | NO | N/A | Optimistic lock |
| metadata (JSONB) | T2 Internal | NO | N/A | Pack metadata (block count, types), not content |

**Encryption count:** 1 column (blocks).

---

### Table 10: `annotations`

| Column | Classification | Encrypted? | Blind Index? | Rationale |
|--------|---------------|------------|-------------|-----------|
| id | T1 Public | NO | N/A | UUID primary key |
| workspace_id | T1 Public | NO | N/A | RLS filter |
| author_id | T1 Public | NO | N/A | Foreign key |
| target_type | T1 Public | NO | N/A | Enum |
| target_id | T1 Public | NO | N/A | Foreign key reference |
| content | T3 Confidential | YES | NO | Free-text annotation may reference deal terms, counterparty names, financial specifics |
| annotation_type | T1 Public | NO | N/A | Enum |
| created_at | T1 Public | NO | N/A | Timestamp |
| updated_at | T1 Public | NO | N/A | Timestamp |
| deleted_at | T1 Public | NO | N/A | Soft-delete |
| version | T1 Public | NO | N/A | Optimistic lock |
| metadata (JSONB) | T2 Internal | NO | N/A | Annotation metadata (formatting, UI state), not content |

**Encryption count:** 1 column (content).

---

### Table 11: `annotation_links`

| Column | Classification | Encrypted? | Blind Index? | Rationale |
|--------|---------------|------------|-------------|-----------|
| id | T1 Public | NO | N/A | UUID primary key |
| annotation_id | T1 Public | NO | N/A | Foreign key |
| linked_type | T1 Public | NO | N/A | Enum |
| linked_id | T1 Public | NO | N/A | Foreign key reference |
| created_at | T1 Public | NO | N/A | Timestamp |

**Encryption count:** 0 columns. Pure junction table with only structural references.

---

### Table 12: `rfis` (Requests for Information)

| Column | Classification | Encrypted? | Blind Index? | Rationale |
|--------|---------------|------------|-------------|-----------|
| id | T1 Public | NO | N/A | UUID primary key |
| workspace_id | T1 Public | NO | N/A | RLS filter |
| patch_id | T1 Public | NO | N/A | Foreign key |
| author_id | T1 Public | NO | N/A | Foreign key |
| target_record_id | T1 Public | NO | N/A | Foreign key reference |
| target_field_key | T2 Internal | NO | N/A | Field identifier, structural |
| question | T3 Confidential | YES | NO | Free-text question about contract terms, may reference specific values |
| response | T3 Confidential | YES | NO | Free-text response with contract term clarifications |
| responder_id | T1 Public | NO | N/A | Foreign key |
| status | T1 Public | NO | N/A | Enum |
| custody_status | T1 Public | NO | N/A | Enum (migration 005) |
| created_at | T1 Public | NO | N/A | Timestamp |
| updated_at | T1 Public | NO | N/A | Timestamp |
| deleted_at | T1 Public | NO | N/A | Soft-delete |
| version | T1 Public | NO | N/A | Optimistic lock |
| metadata (JSONB) | T2 Internal | NO | N/A | RFI workflow metadata |

**Encryption count:** 2 columns (question, response).

---

### Table 13: `triage_items`

| Column | Classification | Encrypted? | Blind Index? | Rationale |
|--------|---------------|------------|-------------|-----------|
| id | T1 Public | NO | N/A | UUID primary key |
| workspace_id | T1 Public | NO | N/A | RLS filter |
| batch_id | T1 Public | NO | N/A | Foreign key |
| record_id | T1 Public | NO | N/A | Foreign key reference |
| field_key | T2 Internal | NO | N/A | Field identifier, structural |
| issue_type | T2 Internal | NO | N/A | Issue classification label |
| severity | T1 Public | NO | N/A | Enum |
| source | T1 Public | NO | N/A | Enum |
| status | T1 Public | NO | N/A | Enum |
| resolved_by | T1 Public | NO | N/A | Foreign key |
| resolved_at | T1 Public | NO | N/A | Timestamp |
| created_at | T1 Public | NO | N/A | Timestamp |
| updated_at | T1 Public | NO | N/A | Timestamp |
| deleted_at | T1 Public | NO | N/A | Soft-delete |
| version | T1 Public | NO | N/A | Optimistic lock |
| metadata (JSONB) | T3 Confidential | YES | NO | May contain issue details with extracted field values or contract terms |

**Encryption count:** 1 column (metadata).

---

### Table 14: `signals`

| Column | Classification | Encrypted? | Blind Index? | Rationale |
|--------|---------------|------------|-------------|-----------|
| id | T1 Public | NO | N/A | UUID primary key |
| workspace_id | T1 Public | NO | N/A | RLS filter |
| batch_id | T1 Public | NO | N/A | Foreign key |
| record_id | T1 Public | NO | N/A | Foreign key reference |
| field_key | T2 Internal | NO | N/A | Field identifier |
| signal_type | T2 Internal | NO | N/A | Signal classification |
| severity | T1 Public | NO | N/A | Enum |
| rule_id | T1 Public | NO | N/A | Reference to QA rule |
| message | T3 Confidential | YES | NO | Signal messages may quote extracted values (e.g., "Territory 'North America' conflicts with...") |
| created_at | T1 Public | NO | N/A | Timestamp |
| metadata (JSONB) | T3 Confidential | YES | NO | May contain field values that triggered the signal |

**Encryption count:** 2 columns (message, metadata).

---

### Table 15: `selection_captures`

| Column | Classification | Encrypted? | Blind Index? | Rationale |
|--------|---------------|------------|-------------|-----------|
| id | T1 Public | NO | N/A | UUID primary key |
| workspace_id | T1 Public | NO | N/A | RLS filter |
| author_id | T1 Public | NO | N/A | Foreign key |
| document_id | T1 Public | NO | N/A | Foreign key |
| field_id | T1 Public | NO | N/A | Foreign key reference |
| rfi_id | T1 Public | NO | N/A | Foreign key |
| page_number | T1 Public | NO | N/A | Numeric |
| coordinates (JSONB) | T1 Public | NO | N/A | Bounding box coordinates, not sensitive |
| selected_text | T3 Confidential | YES | NO | Verbatim text selected from contract document -- may contain any contract content |
| purpose | T1 Public | NO | N/A | Enum |
| created_at | T1 Public | NO | N/A | Timestamp |
| metadata (JSONB) | T2 Internal | NO | N/A | UI state (highlight color, selection tool), not content |

**Encryption count:** 1 column (selected_text).

---

### Table 16: `audit_events`

| Column | Classification | Encrypted? | Blind Index? | Rationale |
|--------|---------------|------------|-------------|-----------|
| id | T1 Public | NO | N/A | UUID primary key |
| workspace_id | T1 Public | NO | N/A | RLS filter |
| event_type | T2 Internal | NO | N/A | Event classification, used in query filters |
| actor_id | T1 Public | NO | N/A | Foreign key |
| actor_role | T1 Public | NO | N/A | Role at time of action |
| timestamp_iso | T1 Public | NO | N/A | Timestamp, used in ordering |
| dataset_id | T1 Public | NO | N/A | Foreign key reference |
| batch_id | T1 Public | NO | N/A | Foreign key |
| record_id | T1 Public | NO | N/A | Foreign key reference |
| field_key | T2 Internal | NO | N/A | Field identifier |
| patch_id | T1 Public | NO | N/A | Foreign key |
| before_value | T3 Confidential | YES | NO | Audit trail of field value changes -- may contain financial terms |
| after_value | T3 Confidential | YES | NO | Audit trail of field value changes -- may contain financial terms |
| metadata (JSONB) | T2 Internal | NO | N/A | Event metadata (IP address, user agent) -- see note below |

**Design decision on audit_events.metadata:** This JSONB column stores event metadata such as IP addresses and user agent strings. While IP addresses could be considered PII under GDPR, encrypting this column would prevent security investigations (searching for suspicious IPs, correlating requests). Decision: keep plaintext for MVP. Enterprise phase adds a configurable PII-scrub policy that hashes IP addresses after a retention window (90 days default).

**Encryption count:** 2 columns (before_value, after_value).

**Note:** The append-only trigger on `audit_events` still functions correctly with encrypted columns. The trigger prevents UPDATE and DELETE; it does not inspect column values.

---

### Table 17: `api_keys`

| Column | Classification | Encrypted? | Blind Index? | Rationale |
|--------|---------------|------------|-------------|-----------|
| key_id | T1 Public | NO | N/A | UUID primary key |
| workspace_id | T1 Public | NO | N/A | RLS filter |
| key_hash | T2 Internal | NO | N/A | Already hashed (SHA-256). This IS the blind index -- the raw key is never stored |
| key_prefix | T2 Internal | NO | N/A | First 8 chars of key for display (e.g., "ak_xJ7m..."). Not sufficient to reconstruct key |
| scopes (JSONB) | T1 Public | NO | N/A | Permission scopes, structural |
| created_by | T1 Public | NO | N/A | Foreign key |
| created_at | T1 Public | NO | N/A | Timestamp |
| expires_at | T1 Public | NO | N/A | Timestamp |
| last_used_at | T1 Public | NO | N/A | Timestamp |
| revoked_at | T1 Public | NO | N/A | Timestamp |

**Encryption count:** 0 columns. The raw API key is never stored; only its SHA-256 hash is persisted.

---

### Table 18: `idempotency_keys`

| Column | Classification | Encrypted? | Blind Index? | Rationale |
|--------|---------------|------------|-------------|-----------|
| key_hash | T1 Public | NO | N/A | SHA-256 of idempotency key, primary key |
| workspace_id | T1 Public | NO | N/A | RLS filter |
| endpoint | T1 Public | NO | N/A | API route path |
| response (JSONB) | T3 Confidential | YES | NO | Cached API response may contain any data returned by the endpoint |
| created_at | T1 Public | NO | N/A | Timestamp |
| expires_at | T1 Public | NO | N/A | TTL for cleanup |

**Encryption count:** 1 column (response).

---

### Post-Core Tables (Migrations 003-024)

#### `drive_connections` (migration 004)

| Column | Classification | Encrypted? | Blind Index? | Rationale |
|--------|---------------|------------|-------------|-----------|
| id | T1 Public | NO | N/A | UUID |
| workspace_id | T1 Public | NO | N/A | RLS filter |
| connected_by | T1 Public | NO | N/A | Foreign key |
| drive_email | T4 PII | YES | YES (HMAC) | Google account email |
| access_token | T4 Restricted | YES | NO | OAuth access token -- compromise grants Drive API access |
| refresh_token | T4 Restricted | YES | NO | OAuth refresh token -- long-lived credential, highest sensitivity |
| token_expiry | T1 Public | NO | N/A | Timestamp |
| scopes | T1 Public | NO | N/A | OAuth scope list, structural |
| status | T1 Public | NO | N/A | Enum |
| connected_at | T1 Public | NO | N/A | Timestamp |
| updated_at | T1 Public | NO | N/A | Timestamp |
| metadata (JSONB) | T2 Internal | NO | N/A | Connection metadata |

**Encryption count:** 3 columns (drive_email, access_token, refresh_token). 1 blind index (drive_email).

---

#### `drive_import_provenance` (migration 004)

| Column | Classification | Encrypted? | Blind Index? | Rationale |
|--------|---------------|------------|-------------|-----------|
| id | T1 Public | NO | N/A | UUID |
| workspace_id | T1 Public | NO | N/A | RLS filter |
| source_file_id | T2 Internal | NO | N/A | Google Drive file ID -- opaque identifier, not sensitive by itself |
| source_file_name | T3 Confidential | YES | NO | File names contain client/deal info |
| source_mime_type | T1 Public | NO | N/A | MIME type |
| source_size_bytes | T1 Public | NO | N/A | Numeric |
| drive_id | T2 Internal | NO | N/A | Drive identifier |
| drive_modified_time | T1 Public | NO | N/A | Timestamp |
| drive_md5 | T2 Internal | NO | N/A | Content hash, not sensitive |
| version_number | T1 Public | NO | N/A | Numeric |
| supersedes_id | T1 Public | NO | N/A | Foreign key |
| imported_by | T1 Public | NO | N/A | Foreign key |
| imported_at | T1 Public | NO | N/A | Timestamp |
| batch_id | T1 Public | NO | N/A | Foreign key |
| metadata (JSONB) | T2 Internal | NO | N/A | Import metadata |

**Encryption count:** 1 column (source_file_name).

---

#### `workbook_sessions` (migration 004)

| Column | Classification | Encrypted? | Blind Index? | Rationale |
|--------|---------------|------------|-------------|-----------|
| id | T1 Public | NO | N/A | UUID |
| user_id | T1 Public | NO | N/A | Foreign key |
| workspace_id | T1 Public | NO | N/A | RLS filter |
| environment | T1 Public | NO | N/A | Enum |
| source_type | T1 Public | NO | N/A | Enum |
| source_ref | T2 Internal | NO | N/A | Reference identifier |
| session_data (JSONB) | T3 Confidential | YES | NO | Active session state may contain field values being edited, unsaved patches |
| status | T1 Public | NO | N/A | Enum |
| created_at | T1 Public | NO | N/A | Timestamp |
| updated_at | T1 Public | NO | N/A | Timestamp |
| last_accessed_at | T1 Public | NO | N/A | Timestamp |
| metadata (JSONB) | T2 Internal | NO | N/A | Session UI state |

**Encryption count:** 1 column (session_data).

---

#### `anchors` (migration 005)

| Column | Classification | Encrypted? | Blind Index? | Rationale |
|--------|---------------|------------|-------------|-----------|
| id | T1 Public | NO | N/A | UUID |
| document_id | T1 Public | NO | N/A | Foreign key |
| workspace_id | T1 Public | NO | N/A | RLS filter |
| anchor_fingerprint | T2 Internal | NO | N/A | Content hash for dedup |
| node_id | T1 Public | NO | N/A | DOM node reference |
| char_start | T1 Public | NO | N/A | Numeric offset |
| char_end | T1 Public | NO | N/A | Numeric offset |
| selected_text | T3 Confidential | YES | NO | Verbatim text from contract document |
| field_id | T1 Public | NO | N/A | Foreign key reference |
| field_key | T2 Internal | NO | N/A | Field identifier |
| page_number | T1 Public | NO | N/A | Numeric |
| created_by | T1 Public | NO | N/A | Foreign key |
| created_at | T1 Public | NO | N/A | Timestamp |
| updated_at | T1 Public | NO | N/A | Timestamp |
| deleted_at | T1 Public | NO | N/A | Soft-delete |
| version | T1 Public | NO | N/A | Optimistic lock |
| metadata (JSONB) | T2 Internal | NO | N/A | Anchor UI metadata |

**Encryption count:** 1 column (selected_text).

---

#### `corrections` (migration 005)

| Column | Classification | Encrypted? | Blind Index? | Rationale |
|--------|---------------|------------|-------------|-----------|
| id | T1 Public | NO | N/A | UUID |
| document_id | T1 Public | NO | N/A | Foreign key |
| workspace_id | T1 Public | NO | N/A | RLS filter |
| anchor_id | T1 Public | NO | N/A | Foreign key |
| rfi_id | T1 Public | NO | N/A | Foreign key |
| field_id | T1 Public | NO | N/A | Foreign key reference |
| field_key | T2 Internal | NO | N/A | Field identifier |
| original_value | T3 Confidential | YES | NO | Original extracted value (financial terms, names, dates) |
| corrected_value | T3 Confidential | YES | NO | Corrected value |
| correction_type | T1 Public | NO | N/A | Enum |
| status | T1 Public | NO | N/A | Enum |
| decided_by | T1 Public | NO | N/A | Foreign key |
| decided_at | T1 Public | NO | N/A | Timestamp |
| created_by | T1 Public | NO | N/A | Foreign key |
| created_at | T1 Public | NO | N/A | Timestamp |
| updated_at | T1 Public | NO | N/A | Timestamp |
| deleted_at | T1 Public | NO | N/A | Soft-delete |
| version | T1 Public | NO | N/A | Optimistic lock |
| metadata (JSONB) | T2 Internal | NO | N/A | Correction workflow metadata |

**Encryption count:** 2 columns (original_value, corrected_value).

---

#### `reader_node_cache` (migration 005)

| Column | Classification | Encrypted? | Blind Index? | Rationale |
|--------|---------------|------------|-------------|-----------|
| id | T1 Public | NO | N/A | UUID |
| document_id | T1 Public | NO | N/A | Foreign key |
| source_pdf_hash | T2 Internal | NO | N/A | Content hash |
| ocr_version | T1 Public | NO | N/A | Version identifier |
| quality_flag | T1 Public | NO | N/A | Enum |
| nodes (JSONB) | T3 Confidential | YES | NO | Full OCR text extraction output -- contains all document content in structured form |
| page_count | T1 Public | NO | N/A | Numeric |
| created_at | T1 Public | NO | N/A | Timestamp |
| updated_at | T1 Public | NO | N/A | Timestamp |
| metadata (JSONB) | T2 Internal | NO | N/A | OCR processing metadata |

**Encryption count:** 1 column (nodes). This is a high-impact column: it contains the full OCR-extracted text of contract documents.

---

#### `ocr_escalations` (migration 005)

| Column | Classification | Encrypted? | Blind Index? | Rationale |
|--------|---------------|------------|-------------|-----------|
| All columns | T1-T2 | NO | N/A | Contains only structural metadata (IDs, statuses, timestamps). No document content. |

**Encryption count:** 0 columns.

---

#### `upload_sessions` (migration 015)

| Column | Classification | Encrypted? | Blind Index? | Rationale |
|--------|---------------|------------|-------------|-----------|
| All columns | T1-T2 | NO | N/A | Contains only structural metadata (IDs, statuses, counts, timestamps). No document content or PII. |

**Encryption count:** 0 columns.

---

#### `conflict_resolutions` (migration 019)

| Column | Classification | Encrypted? | Blind Index? | Rationale |
|--------|---------------|------------|-------------|-----------|
| id | T1 Public | NO | N/A | UUID |
| workspace_id | T1 Public | NO | N/A | RLS filter |
| contract_id | T1 Public | NO | N/A | Foreign key |
| autospark_run_id | T1 Public | NO | N/A | Job ID |
| field_key | T2 Internal | NO | N/A | Field identifier |
| sf_api_name | T2 Internal | NO | N/A | Salesforce field API name |
| conflict_type | T1 Public | NO | N/A | Enum |
| agent_value | T3 Confidential | YES | NO | AI-extracted value (financial terms, names) |
| analyst_value | T3 Confidential | YES | NO | Human-entered value |
| resolved_value | T3 Confidential | YES | NO | Final resolved value |
| resolved_by | T1 Public | NO | N/A | Resolver role |
| reason | T3 Confidential | YES | NO | Free-text explanation may reference deal specifics |
| evidence_quote | T3 Confidential | YES | NO | Verbatim quote from contract |
| evidence_document_url | T4 Restricted | YES | NO | Document location |
| created_at | T1 Public | NO | N/A | Timestamp |
| updated_at | T1 Public | NO | N/A | Timestamp |
| deleted_at | T1 Public | NO | N/A | Soft-delete |
| version | T1 Public | NO | N/A | Optimistic lock |
| metadata (JSONB) | T2 Internal | NO | N/A | Resolution workflow metadata |

**Encryption count:** 6 columns (agent_value, analyst_value, resolved_value, reason, evidence_quote, evidence_document_url).

---

#### `canonical_copies` (migration 020)

| Column | Classification | Encrypted? | Blind Index? | Rationale |
|--------|---------------|------------|-------------|-----------|
| id | T1 Public | NO | N/A | UUID |
| workspace_id | T1 Public | NO | N/A | RLS filter |
| contract_id | T1 Public | NO | N/A | Foreign key |
| status | T1 Public | NO | N/A | Enum |
| source_storage_key | T3 Confidential | YES | NO | Storage path reveals file structure |
| source_hash_sha256 | T2 Internal | NO | N/A | Content hash |
| canonical_storage_key | T3 Confidential | YES | NO | Storage path for OCR output |
| canonical_hash_sha256 | T2 Internal | NO | N/A | Content hash |
| manifest_json (JSONB) | T2 Internal | NO | N/A | OCR job manifest (page counts, processing flags) |
| ocr_confidence_json (JSONB) | T2 Internal | NO | N/A | Per-page confidence scores, not content |
| error_message | T2 Internal | NO | N/A | Processing error text, no PII |
| actor_id | T1 Public | NO | N/A | Foreign key |
| created_at | T1 Public | NO | N/A | Timestamp |
| updated_at | T1 Public | NO | N/A | Timestamp |
| completed_at | T1 Public | NO | N/A | Timestamp |

**Encryption count:** 2 columns (source_storage_key, canonical_storage_key).

---

#### `kiwi_chat_sessions` (migration 021)

| Column | Classification | Encrypted? | Blind Index? | Rationale |
|--------|---------------|------------|-------------|-----------|
| id | T1 Public | NO | N/A | UUID |
| workspace_id | T1 Public | NO | N/A | RLS filter |
| contract_id | T1 Public | NO | N/A | Foreign key |
| user_id | T1 Public | NO | N/A | Foreign key |
| status | T1 Public | NO | N/A | Enum |
| created_at | T1 Public | NO | N/A | Timestamp |
| updated_at | T1 Public | NO | N/A | Timestamp |

**Encryption count:** 0 columns. Session metadata only.

---

#### `kiwi_chat_messages` (migration 021)

| Column | Classification | Encrypted? | Blind Index? | Rationale |
|--------|---------------|------------|-------------|-----------|
| id | T1 Public | NO | N/A | UUID |
| session_id | T1 Public | NO | N/A | Foreign key |
| role | T1 Public | NO | N/A | Enum (`user`/`assistant`/`system`) |
| content | T3 Confidential | YES | NO | User messages may contain PII, questions about deal terms. Assistant messages may echo contract data. |
| metadata (JSONB) | T2 Internal | NO | N/A | Message metadata (token count, model used) |
| created_at | T1 Public | NO | N/A | Timestamp |

**Encryption count:** 1 column (content).

**Design note:** In the Airlock schema, these become `otto_sessions` and `otto_messages`. The same encryption classification applies. The `assistant_response` column in `otto_messages` is also encrypted (unlike OrcestrateOS's plaintext storage) because assistant responses may echo back user-provided PII or extracted contract terms, even after redaction -- the response is a representation of a conversation that contained sensitive context.

---

#### `extraction_metrics` (migration 022)

| Column | Classification | Encrypted? | Blind Index? | Rationale |
|--------|---------------|------------|-------------|-----------|
| All columns | T1-T2 | NO | N/A | Contains only performance metrics (durations, counts, distributions). No document content or PII. Field names in `field_hit_map` are check codes (e.g., "OPP_TERRITORY"), not values. |

**Encryption count:** 0 columns.

---

#### `confirmed_corpus` (migration 023)

| Column | Classification | Encrypted? | Blind Index? | Rationale |
|--------|---------------|------------|-------------|-----------|
| id | T1 Public | NO | N/A | UUID |
| contract_id | T1 Public | NO | N/A | Foreign key |
| workspace_id | T1 Public | NO | N/A | RLS filter |
| contract_type | T2 Internal | NO | N/A | Type classification, used in query filters |
| extracted_data (JSONB) | T3 Confidential | YES | NO | Full confirmed extraction results -- contains all contract field values |
| source_file_url | T4 Restricted | YES | NO | Document location |
| confirmed_by | T1 Public | NO | N/A | Foreign key |
| confirmed_at | T1 Public | NO | N/A | Timestamp |

**Encryption count:** 2 columns (extracted_data, source_file_url).

---

### Encryption Summary

| Category | Tables | Encrypted Columns | Blind Indexes |
|----------|--------|-------------------|---------------|
| Core 18 (migration 001) | 18 | 30 | 6 |
| Drive & Sessions (migration 004) | 3 | 5 | 1 |
| Evidence Inspector (migration 005) | 4 | 4 | 0 |
| Upload Sessions (migration 015) | 1 | 0 | 0 |
| Conflict Resolutions (migration 019) | 1 | 6 | 0 |
| Canonical Copies (migration 020) | 1 | 2 | 0 |
| Kiwi Chat (migration 021) | 2 | 1 | 0 |
| Extraction Metrics (migration 022) | 1 | 0 | 0 |
| Confirmed Corpus (migration 023) | 1 | 2 | 0 |
| **Total** | **32** | **50** | **7** |

### Fields That MUST Remain Plaintext

These field categories are never encrypted, across all tables:

1. **All UUIDs and foreign keys** -- Used in JOINs, WHERE clauses, and RLS enforcement. Encrypting them would break relational integrity.
2. **All timestamps** (`created_at`, `updated_at`, `deleted_at`, etc.) -- Used in ordering, retention policies, and SLA calculations.
3. **Enum columns** (`status`, `role`, `severity`, `source`, `mode`, etc.) -- Used in query filters and workflow transitions.
4. **Numeric scores, counts, and aggregations** (`health_score`, `record_count`, `page_count`, `duration_ms`, etc.) -- Used in dashboards and reporting.
5. **Boolean flags** (`is_active`, etc.) -- Used in query filters.
6. **Content hashes and fingerprints** (`contract_fingerprint`, `batch_fingerprint`, `document_fingerprint`, `source_pdf_hash`, etc.) -- Derived from content, not sensitive themselves. Needed for dedup.
7. **Field identifiers** (`field_key`, `sf_api_name`) -- Structural metadata (e.g., "OPP_TERRITORY"), not the field values themselves.
8. **Version counters** -- Optimistic locking integers.
9. **OAuth scopes** -- Standard scope strings, not credentials.

---

## 2. Algorithm Selection (Answers Q9)

### Decision: AES-256-GCM (Authenticated Encryption with Associated Data)

**Why AES-256-GCM:**

- **Authenticated encryption:** GCM mode provides both confidentiality (AES-CTR) and integrity (GHASH authentication tag). A tampered ciphertext fails decryption rather than producing corrupted plaintext silently.
- **Industry standard:** Used by TLS 1.3, AWS KMS, Google Cloud KMS, and HashiCorp Vault. Well-understood, well-audited, hardware-accelerated (AES-NI) on all modern CPUs.
- **AEAD semantics:** Associated Authenticated Data (AAD) binds ciphertext to its context. We bind each ciphertext to `{table_name}.{column_name}.{row_id}`, preventing an attacker from moving encrypted data between rows or columns (e.g., copying an encrypted `before_value` into an `after_value` column).

**Alternatives considered and rejected:**

| Algorithm | Why Rejected |
|-----------|-------------|
| AES-256-CBC | No built-in authentication. Requires separate HMAC, increasing complexity and risk of encrypt-then-MAC implementation errors. Vulnerable to padding oracle attacks if MAC verification is not constant-time. |
| ChaCha20-Poly1305 | Excellent alternative (used by WireGuard, Signal). Faster than AES on devices without AES-NI. Rejected only because Python `cryptography` library has better AES-GCM documentation and the server runs on AES-NI hardware. Would be acceptable as a future option. |
| XChaCha20-Poly1305 | Extended nonce (192-bit) eliminates nonce reuse risk. Appealing but less widely supported in Python ecosystem. Consider for enterprise phase. |
| RSA/ECIES | Asymmetric encryption. Slower by orders of magnitude. Not appropriate for high-volume column encryption. Used only for key wrapping in envelope encryption patterns. |

### Parameters

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| Key size | 256-bit | Maximum AES key size. 128-bit is sufficient against brute force, but 256-bit provides margin against quantum computing advances (Grover's algorithm halves effective key strength). |
| Nonce | 96-bit, randomly generated per encryption operation | GCM standard nonce size. Random generation via `os.urandom(12)`. NEVER reuse a nonce with the same key -- nonce reuse breaks GCM's authentication guarantee and leaks the authentication key. |
| Authentication tag | 128-bit (16 bytes) | Maximum GCM tag length. Appended to ciphertext. Verified before any decryption output is returned. |
| AAD | `{table_name}.{column_name}.{row_id}` | Binds ciphertext to its storage location. If an attacker copies ciphertext from `patches.before_value` row X to `patches.after_value` row Y, decryption fails because AAD does not match. |

### Storage Format

```
base64(nonce || ciphertext || tag)
```

- **Single-field storage:** The nonce (12 bytes), ciphertext (variable), and tag (16 bytes) are concatenated and base64-encoded into a single TEXT column value.
- **Decryption:** base64-decode, split first 12 bytes (nonce), last 16 bytes (tag), and middle bytes (ciphertext).
- **Column type:** Encrypted columns use PostgreSQL `TEXT` type. Original column types (also TEXT in OrcestrateOS) are preserved semantically at the application layer.

### Nonce Safety Analysis

With 96-bit random nonces and AES-256-GCM, the birthday-bound collision probability reaches 50% at approximately 2^48 encryptions (~281 trillion) under a single key. With workspace-level key derivation (Section 3), each workspace has its own key, and the largest workspace would need to encrypt 281 trillion values to hit the birthday bound. At Airlock's scale (thousands of contracts per workspace, dozens of fields per contract), the actual encryption count per key will be in the low millions -- seven orders of magnitude below the danger threshold. Nonce reuse is not a practical concern.

### Library

Python `cryptography` package, specifically `cryptography.hazmat.primitives.ciphers.aead.AESGCM`:

```python
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
import os, base64

def encrypt(plaintext: str, key: bytes, aad: str) -> str:
    """Encrypt plaintext string. Returns base64-encoded nonce||ciphertext||tag."""
    nonce = os.urandom(12)  # 96-bit random nonce
    aesgcm = AESGCM(key)
    ct = aesgcm.encrypt(nonce, plaintext.encode("utf-8"), aad.encode("utf-8"))
    # ct already includes the 16-byte tag appended by AESGCM
    return base64.b64encode(nonce + ct).decode("ascii")

def decrypt(token: str, key: bytes, aad: str) -> str:
    """Decrypt base64-encoded token. Returns plaintext string."""
    raw = base64.b64decode(token)
    nonce = raw[:12]
    ct_with_tag = raw[12:]
    aesgcm = AESGCM(key)
    plaintext = aesgcm.decrypt(nonce, ct_with_tag, aad.encode("utf-8"))
    return plaintext.decode("utf-8")
```

---

## 3. Key Derivation Hierarchy (Answers Q11, Resolves A2)

### Decision: Three-level HKDF-based key derivation for MVP

```
Master Key (environment variable for MVP, KMS for enterprise)
  |
  +-- Workspace Key = HKDF-SHA256(master_key, salt="airlock-workspace", info=workspace_id)
       |
       +-- Category Key = HKDF-SHA256(workspace_key, salt="airlock-category", info=category_name)
```

### Categories

Each workspace derives five category keys, one per data domain:

| Category | Tables Covered | Rationale for Separation |
|----------|---------------|------------------------|
| `documents` | contracts, documents, canonical_copies, confirmed_corpus, drive_import_provenance, reader_node_cache | Document content and file references. Highest volume of encrypt/decrypt operations. |
| `contacts` | users, accounts, drive_connections | Identity data, PII, OAuth tokens. Separate key enables targeted rotation if a credential leak is suspected. |
| `communications` | annotations, rfis, kiwi_chat_messages, selection_captures, signals | User-generated content: messages, questions, annotations, text selections. |
| `extractions` | patches, evidence_packs, corrections, conflict_resolutions, triage_items | Extraction results and correction workflows. Contains financial terms, deal specifics. |
| `operations` | workbook_sessions, idempotency_keys, audit_events | Operational data: session state, cached responses, audit trail values. |

### Why Three Levels, Not Four

**Four-level (Master -> Workspace -> Vault -> Field)** was considered and rejected for MVP:

- **Pro:** Per-Vault keys enable crypto-shredding of individual contracts.
- **Con:** Airlock's Vault hierarchy is 4 levels deep (Parent -> Division -> Counterparty -> Item). Deriving a key for each of the thousands of Vaults in a workspace adds key management complexity with minimal security benefit at MVP scale.
- **Con:** Category-level keys already isolate data domains. If a communication key is compromised, document encryption remains intact.
- **Enterprise upgrade path:** Add a Vault-level derivation step (`HKDF(category_key, salt="airlock-vault", info=vault_id)`) when per-Vault crypto-shredding is required. This is additive -- existing data re-encrypted during a scheduled rotation window.

### HKDF Parameters

| Parameter | Value |
|-----------|-------|
| Algorithm | HKDF-SHA256 |
| Input key material (IKM) | Parent key (master or workspace) |
| Salt | Fixed per level: `b"airlock-workspace"` or `b"airlock-category"` |
| Info | Context-specific: `workspace_id.encode()` or `category_name.encode()` |
| Output length | 32 bytes (256 bits, matching AES-256 key size) |

### MVP Key Custody (Resolves A2)

**Decision: Master key from environment variable, with documented generation procedure.**

**Generation:**

```bash
openssl rand -base64 32
# Output example: K7xMp2qN5vR8tY3wE6hJ9sL1fA4gB0cD2iU7oP5mX8=
```

This produces 256 bits of cryptographically secure randomness, base64-encoded to 44 characters.

**Storage:**

- Stored in `.env` file at repository root (NOT committed to git -- `.gitignore` must include `.env`)
- Docker Compose passes it to FastAPI container via `environment` directive
- The `.env` file is the ONLY location of the master key on the development machine

**Risks acknowledged:**

| Risk | Mitigation |
|------|-----------|
| `.env` file readable by any process running as the developer's OS user | Acceptable for single-developer MVP. Enterprise migrates to KMS. |
| `docker inspect` exposes environment variables | Acceptable for local development. Never use environment variables in production multi-tenant deployment. |
| No key rotation mechanism | Document manual rotation procedure (generate new key, re-encrypt all data, swap `.env` value, restart containers). Enterprise adds automated rotation via KMS API. |
| Loss of `.env` file = permanent data loss | Backup `.env` to a separate, encrypted location (e.g., 1Password vault, encrypted USB drive). Document this in setup guide. |

### Enterprise Key Custody

**Decision: KMS-managed master key with derived keys cached in worker memory.**

| Component | Implementation |
|-----------|---------------|
| Master key | Created and stored in AWS KMS (or HashiCorp Vault). Never leaves KMS in plaintext -- KMS provides `encrypt` and `decrypt` API operations. |
| Workspace key derivation | FastAPI startup calls KMS to decrypt the master key, derives workspace keys via HKDF, caches derived keys in process memory. |
| Key cache lifetime | Derived keys cached for the lifetime of the FastAPI worker process. Cleared on worker shutdown or restart. |
| Key cache protection | Derived keys stored in Python `bytes` objects. No serialization to disk, logs, or error reports. The `__repr__` of the key holder class returns `"<EncryptionKey [REDACTED]>"`. |
| KMS latency | KMS `decrypt` call: 5-50ms. Called once per workspace at startup (not per request). Acceptable even for cold starts. |
| Rotation | Rotate KEK in KMS. Re-wrap all workspace keys with the new KEK. Data re-encryption is NOT required -- only the master key changes, and workspace keys are re-derived at next startup. |
| Crypto-shredding | Delete a workspace's derived key material and mark the workspace as shredded. All data encrypted under that workspace's key tree becomes permanently unrecoverable. Satisfies GDPR right-to-erasure without touching encrypted rows. |

---

## 4. Encrypt/Decrypt Middleware Architecture (Answers Q10)

### Decision: Repository-layer encryption with SQLAlchemy column-level type decorators

**Alternatives considered:**

| Approach | Pro | Con | Verdict |
|----------|-----|-----|---------|
| **FastAPI middleware** (intercepts all DB I/O) | Broad coverage, single implementation point | Encrypts non-sensitive data unnecessarily. Cannot distinguish between tables/columns. Performance overhead on every query. | Rejected |
| **Per-query manual** (encrypt/decrypt in each route handler) | Most precise control | Developers must remember to encrypt on every write and decrypt on every read. One missed call = plaintext leak. Does not scale. | Rejected |
| **SQLAlchemy TypeDecorator** (column-level) | Schema-enforced. Developer declares a column as `EncryptedText` and encryption is automatic. Cannot forget. | Requires SQLAlchemy ORM (Airlock is already committed to this for the Next.js/FastAPI stack). Slight learning curve for custom types. | **Selected** |

### Implementation Design

```python
from sqlalchemy import Text, TypeDecorator
from airlock.crypto import encrypt, decrypt, get_category_key

class EncryptedText(TypeDecorator):
    """
    Transparently encrypts/decrypts text columns using AES-256-GCM.

    Usage in model:
        class Contract(Base):
            file_url = Column(EncryptedText(category="documents"), nullable=True)
            metadata = Column(EncryptedJSON(category="documents"), nullable=True)
    """
    impl = Text
    cache_ok = True

    def __init__(self, category: str, *args, **kwargs):
        self.category = category
        super().__init__(*args, **kwargs)

    def process_bind_param(self, value, dialect):
        """Called on INSERT/UPDATE. Encrypts plaintext before sending to PostgreSQL."""
        if value is None:
            return None
        # AAD constructed from table+column context (set by model mixin)
        aad = self._get_aad()
        return encrypt(value, get_category_key(self.category), aad)

    def process_result_value(self, value, dialect):
        """Called on SELECT. Decrypts ciphertext after reading from PostgreSQL."""
        if value is None:
            return None
        aad = self._get_aad()
        return decrypt(value, get_category_key(self.category), aad)
```

**For JSONB columns** that must be encrypted, a companion `EncryptedJSON` type serializes to JSON string, encrypts, and stores as TEXT. On read, it decrypts and deserializes back to a Python dict/list.

```python
import json

class EncryptedJSON(TypeDecorator):
    """Encrypts JSONB columns. Stored as encrypted TEXT in PostgreSQL."""
    impl = Text
    cache_ok = True

    def __init__(self, category: str, *args, **kwargs):
        self.category = category
        super().__init__(*args, **kwargs)

    def process_bind_param(self, value, dialect):
        if value is None:
            return None
        json_str = json.dumps(value, separators=(",", ":"), default=str)
        aad = self._get_aad()
        return encrypt(json_str, get_category_key(self.category), aad)

    def process_result_value(self, value, dialect):
        if value is None:
            return None
        aad = self._get_aad()
        json_str = decrypt(value, get_category_key(self.category), aad)
        return json.loads(json_str)
```

**Trade-off:** Encrypted JSONB columns lose all PostgreSQL JSONB query operators (`->`, `->>`, `@>`, `?`). Any filtering on encrypted JSONB must happen at the application layer after decryption. This is acceptable because:

1. The primary JSONB queries in OrcestrateOS are on `metadata` columns, which are typically read in full (not queried by sub-key at the SQL level)
2. The Search module provides application-layer search across all content
3. Specific fields that need lookup have dedicated blind index columns

### Blind Index Implementation

For columns that require exact-match lookup (email dedup, entity resolution, file dedup):

```python
import hmac, hashlib

def compute_blind_index(value: str, key: bytes, context: str) -> str:
    """
    HMAC-SHA256 blind index for encrypted columns.
    Deterministic: same value always produces same index (enabling lookups).
    Context prevents cross-column index attacks.
    """
    msg = f"{context}:{value}".encode("utf-8")
    return hmac.new(key, msg, hashlib.sha256).hexdigest()
```

Blind indexes are stored in separate columns (e.g., `users.email_idx` alongside encrypted `users.email`). The blind index column is indexed by PostgreSQL for fast lookups:

```sql
-- Example: users table with blind index
ALTER TABLE users ADD COLUMN email_idx TEXT;
CREATE UNIQUE INDEX idx_users_email_idx ON users(email_idx);
```

**Blind index limitations:**

- Supports only exact-match queries (WHERE email_idx = computed_hash)
- Does NOT support LIKE, range queries, or partial matching
- Does NOT support sorting by the original plaintext value
- The same plaintext always produces the same hash (by design), so frequency analysis is theoretically possible if the attacker knows the value distribution

### AAD Binding Strategy

The AAD (Additional Authenticated Data) for each encryption operation is constructed as:

```
{table_name}.{column_name}.{row_id}
```

For example: `contracts.file_url.ctr_abc123def456`

This binding prevents three categories of attack:

1. **Column swap:** Copying ciphertext from `before_value` to `after_value` in the same row fails because AAD includes column name.
2. **Row swap:** Copying ciphertext from row A to row B fails because AAD includes row ID.
3. **Table swap:** Copying ciphertext from `patches.before_value` to `audit_events.before_value` fails because AAD includes table name.

**Implementation note:** The row ID must be available at encryption time. For INSERT operations, the row ID is generated before the INSERT (Airlock uses application-generated UUIDs, not database sequences), so the ID is always available.

---

## 5. File Storage Encryption (Answers Q13, Resolves A12)

### Decision: Encrypt-then-store with envelope encryption

OrcestrateOS's `file_storage.py` writes raw bytes directly to the filesystem at `$FILE_STORAGE_ROOT/{workspace_id}/{batch_id}/{hash}_{filename}`. Airlock replaces this with encrypt-then-store.

### Flow

```
Client uploads file to FastAPI endpoint
  |
  v
1. Validate file (size check, type check)
  |
  v
2. Compute content_hash = SHA-256(plaintext_bytes) [for dedup, BEFORE encryption]
  |
  v
3. Generate file DEK = os.urandom(32) [random 256-bit Data Encryption Key for this file]
  |
  v
4. Encrypt file: encrypted_bytes = AES-256-GCM(plaintext_bytes, file_dek, aad=workspace_id/vault_id/content_hash)
  |
  v
5. Encrypt DEK: encrypted_dek = AES-256-GCM(file_dek, workspace_documents_key, aad=document_id)
  |
  v
6. Store encrypted file at: {storage_root}/{workspace_id}/{vault_id}/{content_hash}.enc
  |
  v
7. Store encrypted DEK in document_keys table:
     document_id, encrypted_dek, content_hash, file_size, created_at
  |
  v
8. Return storage_key: "enc://{workspace_id}/{vault_id}/{content_hash}.enc"
```

### Retrieval Flow

```
Request to retrieve file by storage_key
  |
  v
1. Parse storage_key to extract workspace_id, vault_id, content_hash
  |
  v
2. Load encrypted_dek from document_keys table
  |
  v
3. Decrypt DEK: file_dek = AES-256-GCM-decrypt(encrypted_dek, workspace_documents_key, aad=document_id)
  |
  v
4. Read encrypted file bytes from filesystem
  |
  v
5. Decrypt file: plaintext_bytes = AES-256-GCM-decrypt(encrypted_bytes, file_dek, aad=workspace_id/vault_id/content_hash)
  |
  v
6. Return plaintext bytes to caller (streamed, not buffered entirely in memory for large files)
```

### Why Envelope Encryption for Files

Per-file DEKs (Data Encryption Keys) add one level of indirection:

1. **Key rotation without re-encryption:** Rotating the workspace key only requires re-encrypting the DEKs (small, fast), not the files themselves (large, slow). A workspace with 10,000 files and 50GB of content requires re-encrypting only 10,000 x 32-byte DEKs (~320KB) instead of 50GB of file data.

2. **Per-file revocation:** Deleting a file's DEK renders the file permanently unrecoverable without touching the encrypted file blob. Useful for targeted deletion compliance.

3. **Performance:** File encryption uses a per-file key held in memory during the operation. The workspace key is only used to unwrap the DEK, not to encrypt the potentially multi-megabyte file.

### New Table: `document_keys`

```sql
CREATE TABLE IF NOT EXISTS document_keys (
    id TEXT PRIMARY KEY,
    document_id TEXT NOT NULL,
    workspace_id TEXT NOT NULL REFERENCES workspaces(id),
    encrypted_dek TEXT NOT NULL,       -- base64(nonce || encrypted_dek || tag)
    content_hash TEXT NOT NULL,         -- SHA-256 of plaintext, for dedup
    file_size_bytes BIGINT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    rotated_at TIMESTAMPTZ            -- set when DEK is re-wrapped with new workspace key
);

CREATE INDEX idx_document_keys_workspace ON document_keys(workspace_id);
CREATE INDEX idx_document_keys_content_hash ON document_keys(workspace_id, content_hash);
```

### Storage Format Change

| Aspect | OrcestrateOS | Airlock |
|--------|-------------|---------|
| Path pattern | `{root}/{workspace}/{batch}/{hash}_{filename}` | `{root}/{workspace}/{vault}/{content_hash}.enc` |
| Storage key prefix | `local://` | `enc://` (local encrypted) or `s3enc://` (S3 encrypted) |
| Filename in path | Original filename visible | Content hash only -- filename stored encrypted in `contracts.file_name` |
| File content | Plaintext | AES-256-GCM ciphertext |
| Key per file | None | Per-file DEK in `document_keys` table |
| Dedup check | SHA-256 hash prefix in filename | SHA-256 content hash in `document_keys.content_hash` |

---

## 6. Deduplication with Encryption (Answers Q14)

### Decision: Pre-encryption content hash as blind index

**Problem:** AES-256-GCM uses random nonces. Encrypting the same file twice produces different ciphertext. Without a pre-encryption dedup check, every re-upload stores a duplicate encrypted blob.

**Solution:**

1. Compute `content_hash = SHA-256(plaintext_bytes)` BEFORE encryption
2. Store `content_hash` in `document_keys.content_hash` (plaintext -- not sensitive, just a fingerprint)
3. Before encrypting a new upload, check: `SELECT id FROM document_keys WHERE workspace_id = %s AND content_hash = %s`
4. If match found: skip encryption and storage, return existing storage_key
5. If no match: proceed with encrypt-then-store flow

**Security analysis of plaintext content hash:**

- The SHA-256 hash reveals whether two files have identical content (an attacker who knows a specific file's content can verify it exists by computing the hash)
- This is the same information leakage as OrcestrateOS's current `{hash}_{filename}` pattern, but without the filename
- The hash does NOT reveal file content (SHA-256 is preimage-resistant)
- Acceptable trade-off: dedup saves significant storage, and the hash reveals only "this exact file exists in this workspace" -- not its content

**Cross-workspace dedup:** NOT supported. Content hashes are scoped to `workspace_id`. Two workspaces uploading the same file store separate encrypted copies with separate DEKs. This maintains workspace isolation.

---

## 7. Redis Value Encryption (Answers Q15)

### Decision: Encrypt BullMQ job payloads that contain Vault data

Redis is an in-memory store that optionally persists to disk (RDB snapshots, AOF logs). If Redis data is persisted and the disk is stolen, or if Redis is misconfigured with network access, unencrypted Vault data leaks. This breaks the "steal the disk" guarantee.

### What to Encrypt

| Redis Usage | Contains Vault Data? | Encrypt? | Rationale |
|------------|---------------------|----------|-----------|
| BullMQ job `data` payloads | YES -- extraction pipelines pass document content, field values | YES | Job data flows through Redis queues and may persist in failed-job storage |
| Enrichment cache entries | YES -- contain Vault context with counterparty names, deal terms, field values | YES | Cache entries may persist across requests |
| WebSocket Pub/Sub message payloads | YES -- real-time event data includes field values, patch content | YES | Published messages are briefly held in Redis |
| BullMQ job metadata | NO -- job ID, queue name, timestamps, retry count | NO | Structural data, no PII or business content |
| WebSocket subscription keys | NO -- channel names are workspace/vault IDs | NO | Opaque identifiers |
| Rate limit counters | NO -- numeric counters keyed by IP/user | NO | No content |
| Session tokens | NO -- already opaque random strings | NO | Not sensitive themselves (the token is the credential, and it is already random) |
| Feature flag cache | NO -- flag names and boolean values | NO | Public configuration data |

### Implementation

Encryption happens at the application layer, before data enters Redis:

```python
# Producer: encrypt before adding to queue
async def enqueue_extraction(workspace_id: str, job_data: dict):
    key = get_category_key_for_redis("extractions", workspace_id)
    encrypted_payload = encrypt(json.dumps(job_data), key, aad=f"bullmq.extraction.{workspace_id}")
    await queue.add("extraction", {"encrypted": encrypted_payload, "workspace_id": workspace_id})

# Consumer: decrypt after receiving from queue
async def process_extraction(job):
    workspace_id = job.data["workspace_id"]
    key = get_category_key_for_redis("extractions", workspace_id)
    job_data = json.loads(decrypt(job.data["encrypted"], key, aad=f"bullmq.extraction.{workspace_id}"))
    # ... process decrypted data
```

**Performance note:** Redis encryption adds ~0.1ms per encrypt/decrypt operation for typical job payloads (1-10KB). This is negligible compared to the network round-trip to Redis (~0.5-2ms) and the actual job processing time (100ms-30s for extractions).

**Key for Redis encryption:** Uses the same workspace category keys derived in Section 3. No separate key hierarchy for Redis.

---

## 8. JWT and API Key Security (Answers Q16, Q17)

### JWT (Q16)

**Current state in OrcestrateOS:** `jwt_utils.py` implements a custom HS256 JWT signer/verifier. `JWT_SECRET` is read from an environment variable with no rotation mechanism, a 24-hour expiry, and no refresh token support.

**Airlock MVP decisions:**

| Aspect | Decision | Rationale |
|--------|----------|-----------|
| Algorithm | HS256 (HMAC-SHA256) for MVP | Simple, fast, sufficient for single-service architecture. RS256 (RSA) needed only when multiple services must verify independently without sharing the signing key. |
| Secret generation | `openssl rand -base64 32` (256-bit) | Produces 256 bits of entropy. HS256 requires the key to be at least as long as the hash output (256 bits). |
| Secret storage | `.env` file (same as master encryption key) | Single secrets management approach for MVP. |
| Token TTL | 15 minutes (access token) | Reduced from OrcestrateOS's 24 hours. Shorter TTL limits the window of token theft exploitation. |
| Refresh token | New: 7-day refresh token, stored in HttpOnly cookie | Enables silent token renewal without re-authentication. Refresh tokens are opaque random strings stored server-side (in `user_sessions` table), not JWTs. |
| Rotation | Dual-key verification during grace period | See rotation procedure below. |

**Rotation procedure:**

1. Generate new JWT secret: `openssl rand -base64 32`
2. Set `JWT_SECRET_NEW` in environment alongside existing `JWT_SECRET`
3. Signing uses `JWT_SECRET_NEW` (all new tokens signed with new key)
4. Verification tries `JWT_SECRET_NEW` first, falls back to `JWT_SECRET` (tokens signed with old key still validate)
5. Grace period: 2x access token TTL = 30 minutes
6. After grace period: remove `JWT_SECRET`, rename `JWT_SECRET_NEW` to `JWT_SECRET`
7. Any tokens signed with the old key that have not expired are invalidated when the old key is removed

**Enterprise upgrade:** RS256 with asymmetric key pair. Private key held in KMS (signs tokens), public key distributed to all services (verifies tokens). Enables token verification without sharing the signing key.

### API Keys (Q17)

**Current state in OrcestrateOS:** API keys stored as `sha256(key)` in `api_keys.key_hash`. SHA-256 is used for lookup, not for password storage.

**Airlock MVP decisions:**

| Aspect | Decision | Rationale |
|--------|----------|-----------|
| Hashing | SHA-256 (unchanged from OrcestrateOS) | API keys are 256-bit random values, not human-chosen passwords. Brute-forcing a 256-bit random key is computationally infeasible regardless of hash speed. bcrypt/Argon2 add unnecessary latency (100ms+ per verification) for high-frequency API calls. |
| Key generation | `secrets.token_urlsafe(32)` | Produces 43 characters encoding 256 bits of entropy. URL-safe base64 avoids special characters that cause issues in HTTP headers. |
| Key format | `ak_{workspace_prefix}_{random}` | Prefix enables visual identification: `ak_` marks it as an Airlock API key, workspace prefix (4 chars) identifies the owning workspace. Example: `ak_ws01_K7xMp2qN5vR8tY3wE6hJ9sL1fA4gB0cD`. |
| Display | Show full key exactly once at creation time. Show only `key_prefix` (first 8 chars) thereafter. | Standard practice (GitHub, Stripe, AWS). Once the creation dialog closes, the full key is never retrievable. If lost, the user generates a new key. |
| Storage | `sha256(full_key)` in `api_keys.key_hash`. No salting needed (keys are 256-bit random, so rainbow tables are irrelevant). | Fast lookup via hash index on `key_hash`. |
| Scope | Per-workspace, per-permission-set | Each API key belongs to one workspace and carries a JSON array of scopes (e.g., `["read:contracts", "write:patches"]`). Scopes are checked on every API request. |
| Expiration | Optional `expires_at` timestamp. Default: no expiration for MVP. Enterprise: mandatory expiration with configurable maximum (e.g., 90 days). | No-expiration is acceptable for developer API keys in MVP. Enterprise enforces rotation via mandatory expiration. |
| Revocation | Set `revoked_at` timestamp. Revoked keys fail authentication immediately. | Revocation is instant (no cache to clear -- API key verification hits the database). |

---

## 9. Backup Strategy (Answers Q18)

### Decision: Encrypted backups with separate key backup

**The critical rule:** Data backup and key backup MUST be stored in different locations. An attacker who steals the data backup must ALSO compromise the key backup location to access any data. These should be independent systems with independent access controls.

### Database Backup

| Aspect | MVP | Enterprise |
|--------|-----|-----------|
| Tool | `pg_dump --format=custom` | `pg_basebackup` + WAL archiving for point-in-time recovery |
| Frequency | Daily (manual or cron) | Continuous WAL archiving + daily base backup |
| Content | SQL dump contains ciphertext for all encrypted columns. Non-encrypted columns (IDs, timestamps, enums) are plaintext. | Same, plus WAL segments for point-in-time recovery |
| Encryption of backup file | Backup file encrypted with `gpg --symmetric` using a backup-specific passphrase | Backup stored in encrypted S3 bucket (SSE-KMS) with a separate backup KMS key |
| Storage | Local encrypted drive or separate encrypted cloud storage (NOT the same machine running PostgreSQL) | S3 with cross-region replication, versioning enabled, lifecycle policy for retention |
| Retention | 30 days rolling | 90-day rolling daily, 1-year rolling weekly, 7-year rolling monthly (compliance) |

**Why `pg_dump` output is "doubly safe":** The dump contains SQL statements like `INSERT INTO contracts (file_url) VALUES ('base64encodedciphertexthere')`. Even if the backup file's GPG encryption is compromised, the attacker sees only ciphertext for all encrypted columns. They need the master encryption key to derive workspace keys and decrypt column values.

### File Storage Backup

| Aspect | MVP | Enterprise |
|--------|-----|-----------|
| Content | All files at `{storage_root}` are already encrypted (Section 5) | Same |
| Tool | `rsync` to backup location | S3 cross-region replication |
| Additional encryption | None needed -- files are already AES-256-GCM encrypted | Optional: S3 SSE-KMS adds a second encryption layer |
| Storage | Separate encrypted drive | S3 in a different AWS region |

### Key Backup

| Aspect | MVP | Enterprise |
|--------|-----|-----------|
| What to back up | Master key from `.env` file | KMS key ID (the master key itself is held in KMS and backed up by AWS/GCP automatically) |
| Where to store | Encrypted password manager (1Password, Bitwarden) or encrypted USB drive in a physically separate location | KMS provides built-in key backup. Additionally: key material escrowed with a trusted third party via Shamir Secret Sharing (3-of-5 threshold) |
| Access control | Only the workspace owner/founder | Dual-control: requires two authorized personnel to reconstruct the key |
| Verification | Monthly: verify the backed-up key can derive known workspace keys | Quarterly: automated restore test to a staging environment |

### Recovery Procedure

1. Provision fresh infrastructure (Docker Compose up on a new machine)
2. Restore PostgreSQL from `pg_dump` backup: `pg_restore --dbname=airlock backup.dump`
3. Restore encrypted files from backup location to `{storage_root}`
4. Load master key from key backup into `.env` file
5. Start FastAPI -- it derives workspace keys from master key and begins serving decrypted data
6. Verify: run automated health check that decrypts a sample of rows from each encrypted table

**Recovery Time Objective (MVP):** 1 hour for a single-machine deployment (download backup, restore database, restore files, configure secrets, start services).

**Recovery Point Objective (MVP):** Up to 24 hours of data loss (daily backup frequency). Acceptable for MVP -- enterprise phase with WAL archiving achieves RPO of seconds.

### Backup Testing

| Test | Frequency | Procedure |
|------|-----------|-----------|
| Backup integrity | Weekly (automated) | Verify backup file checksums match recorded values |
| Restore to staging | Quarterly | Full restore to a separate environment, verify data accessibility |
| Key accessibility | Monthly | Verify the backed-up master key can decrypt a known test value |
| Cross-team key access | Semi-annually (enterprise) | Verify that the Shamir Secret Sharing threshold can reconstruct the key |

---

## 10. MVP Shortcuts and Enterprise Migration Path

| Area | MVP Approach | Enterprise Upgrade | Migration Complexity |
|------|-------------|-------------------|---------------------|
| Master key storage | `.env` file | AWS KMS / HashiCorp Vault | Low -- change key source in config, no data re-encryption needed |
| Key hierarchy depth | Master -> Workspace -> Category (3 levels) | Master -> Workspace -> Vault -> Field (4 levels) | Medium -- requires re-derivation of keys and re-encryption of affected data in a migration window |
| File encryption | Envelope with per-file DEK, category-level wrapping key | Same pattern, KMS-managed wrapping key | Low -- swap key source, re-wrap DEKs only |
| Redis encryption | Selective (job payloads, cache entries, Pub/Sub payloads) | All values encrypted | Medium -- extend encryption to remaining Redis key types |
| Backup key storage | Encrypted password manager / USB drive | HSM escrow with Shamir Secret Sharing | Low -- operational process change, no code changes |
| Key rotation | Manual procedure, documented in runbook | Automated via KMS API with zero-downtime rotation | Medium -- add rotation job, re-wrap DEKs on schedule |
| TDE | Not used | Optional additional layer (defense in depth) | Low -- PostgreSQL config change, no application code changes |
| Blind indexes | HMAC-SHA256, manual column creation | Searchable encryption (e.g., CipherStash) for richer query patterns | High -- schema changes, new dependencies |
| JWT signing | HS256 with shared secret | RS256 with KMS-held private key | Medium -- change JWT library config, distribute public key |
| Audit of crypto operations | Application-level logging of encrypt/decrypt counts | KMS audit log integration (CloudTrail for AWS) | Low -- enable KMS logging, ingest into monitoring |

### Migration Order (Recommended)

When moving from MVP to enterprise, migrate in this order:

1. **KMS integration** (replaces `.env` master key) -- highest security impact, lowest code change
2. **JWT upgrade to RS256** -- enables microservice architecture
3. **Automated key rotation** -- reduces operational risk
4. **Per-Vault key derivation** -- enables crypto-shredding per contract
5. **Full Redis encryption** -- completes the "steal the disk" guarantee for Redis
6. **TDE as additional layer** -- defense in depth
7. **Searchable encryption** -- only if blind indexes prove insufficient for query patterns

---

## Appendix A: Encryption Category Quick Reference

For developer convenience, this table maps every encrypted column to its encryption category and blind index status.

| Table | Column | Category | Blind Index Column |
|-------|--------|----------|-------------------|
| users | email | contacts | email_idx |
| users | display_name | contacts | -- |
| users | google_sub | contacts | google_sub_idx |
| batches | name | documents | -- |
| accounts | account_name | contacts | account_name_idx |
| accounts | billing_city | contacts | -- |
| accounts | metadata | contacts | -- |
| contracts | file_url | documents | -- |
| contracts | file_name | documents | file_name_idx |
| contracts | metadata | documents | -- |
| contracts | autospark_result_json | documents | -- |
| documents | file_url | documents | -- |
| documents | file_name | documents | file_name_idx |
| documents | metadata | documents | -- |
| patches | when_clause | extractions | -- |
| patches | then_clause | extractions | -- |
| patches | because_clause | extractions | -- |
| patches | file_name | extractions | -- |
| patches | file_url | extractions | -- |
| patches | before_value | extractions | -- |
| patches | after_value | extractions | -- |
| patches | history | extractions | -- |
| patches | metadata | extractions | -- |
| evidence_packs | blocks | extractions | -- |
| annotations | content | communications | -- |
| rfis | question | communications | -- |
| rfis | response | communications | -- |
| triage_items | metadata | extractions | -- |
| signals | message | communications | -- |
| signals | metadata | communications | -- |
| selection_captures | selected_text | communications | -- |
| audit_events | before_value | operations | -- |
| audit_events | after_value | operations | -- |
| idempotency_keys | response | operations | -- |
| drive_connections | drive_email | contacts | drive_email_idx |
| drive_connections | access_token | contacts | -- |
| drive_connections | refresh_token | contacts | -- |
| drive_import_provenance | source_file_name | documents | -- |
| workbook_sessions | session_data | operations | -- |
| anchors | selected_text | communications | -- |
| corrections | original_value | extractions | -- |
| corrections | corrected_value | extractions | -- |
| reader_node_cache | nodes | documents | -- |
| conflict_resolutions | agent_value | extractions | -- |
| conflict_resolutions | analyst_value | extractions | -- |
| conflict_resolutions | resolved_value | extractions | -- |
| conflict_resolutions | reason | extractions | -- |
| conflict_resolutions | evidence_quote | extractions | -- |
| conflict_resolutions | evidence_document_url | extractions | -- |
| canonical_copies | source_storage_key | documents | -- |
| canonical_copies | canonical_storage_key | documents | -- |
| kiwi_chat_messages | content | communications | -- |
| confirmed_corpus | extracted_data | documents | -- |
| confirmed_corpus | source_file_url | documents | -- |

---

## Appendix B: Performance Budget

Encryption adds overhead to every read and write of sensitive columns. This budget ensures the cipher model does not degrade user experience below acceptable thresholds.

| Operation | Baseline (plaintext) | With Encryption | Budget | Acceptable? |
|-----------|---------------------|----------------|--------|-------------|
| Single column encrypt (1KB text) | 0ms | ~0.05ms | <1ms | YES |
| Single column decrypt (1KB text) | 0ms | ~0.05ms | <1ms | YES |
| Full contract record read (4 encrypted columns) | ~2ms (DB query) | ~2.2ms (+0.2ms decrypt) | <5ms | YES |
| Full contract record write (4 encrypted columns) | ~3ms (DB insert) | ~3.2ms (+0.2ms encrypt) | <5ms | YES |
| File upload 10MB (encrypt + store) | ~50ms (write to disk) | ~80ms (+30ms AES-256-GCM with AES-NI) | <200ms | YES |
| File download 10MB (read + decrypt) | ~30ms (read from disk) | ~60ms (+30ms decrypt) | <200ms | YES |
| Blind index computation (HMAC-SHA256) | 0ms | ~0.01ms | <1ms | YES |
| BullMQ job payload encrypt (5KB JSON) | 0ms | ~0.1ms | <1ms | YES |
| Key derivation (HKDF, per workspace, at startup) | 0ms | ~0.05ms per workspace | <1ms | YES |

**Conclusion:** AES-256-GCM with AES-NI hardware acceleration adds negligible overhead (sub-millisecond for column operations, <50ms for 10MB file operations). The cipher model does not require a performance exemption for any operation.

---

## Related Specs

- [overview.md](./overview.md) -- 70-question security inventory (parent spec)
- [auth-and-identity.md](./auth-and-identity.md) -- Authentication flows, JWT lifecycle, OAuth integration (future)
- [kms-and-key-hierarchy.md](./kms-and-key-hierarchy.md) -- Enterprise KMS integration, key rotation automation (future)
- [llm-safety.md](./llm-safety.md) -- PII redaction before LLM calls, prompt injection defense (future)
- [threat-model.md](./threat-model.md) -- STRIDE analysis, adversary personas, attack surface enumeration (future)
