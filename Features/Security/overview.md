# Security & Architecture -- Documentation Plan

> **Status:** SPECCED
> **Source:** Exploration of all 26 feature specs + OrcestrateOS codebase analysis (server/auth.py, migrations/001_core_tables.sql, file_storage.py, routes/)

## Summary

This document is the anchor for Airlock's security architecture. It contains: (1) a restated vision with detected ambiguities, (2) a persona-by-scope documentation matrix, (3) a 70-question inventory that must be answered before agents write code, (4) a versioned documentation table of contents from MVP to enterprise, and (5) the minimum set of agent-governance artifacts required for safe, deterministic development.

No production code. Design docs only.

---

## 1. Vision Restatement

### Core Thesis

Airlock is a **middleware-only data-operations platform** where the application layer never trusts the persistence layer. The architecture rests on five pillars:

1. **Ciphered at rest, encrypted in transit.** Every byte stored in PostgreSQL, Redis, or file storage is ciphertext. The application reconstructs meaning on-the-fly. If an attacker steals the disk, copies the database, or a cloud provider's employee browses the storage bucket, they get noise.

2. **Stateless, rehydratable infrastructure.** All services are stateless containers. All durable state lives in encrypted PostgreSQL + encrypted file storage + encrypted Redis snapshots. The entire system can be abandoned and rebuilt from cold backups plus key material. Ransomware becomes a nuisance, not a catastrophe.

3. **Ontology-Guided Corpus (OGC) as the one true knowledge layer.** Documents are not stored as opaque blobs with LLM-generated summaries. They are decomposed into ontology-guided, human-verified chunks -- each with a stable identity, a trust flag, and extraction provenance. The chunk is the atomic unit of knowledge.

4. **LLM as deterministic recall gateway.** The LLM does not generate creative fiction from contract data. It maps structured queries against OGC chunks and formats the results. Temperature is low. Output schemas are strict. The model retrieves and formats; it does not invent.

5. **Crawl-first, cost-minimal development.** MVP runs on a single laptop with Docker Compose, zero mandatory SaaS subscriptions, and self-hosted components. Enterprise features (HSM, multi-AZ, compliance) layer on top without architectural rewrites.

### What Is Already Decided

These decisions are locked across existing feature specs and cannot be reopened here:

| Domain | Decision | Source Spec |
|--------|----------|-------------|
| Authentication | JWT + Google OAuth (custom implementation ported from OrcestrateOS `auth.py`) | Roles, Admin |
| Authorization | Three-layer role model: Org (Member/Lead/Director/Executive), Module (Builder/Gatekeeper/Owner/Designer/Viewer), Agentic (16 functional roles) | Roles |
| Permissions | Discord-style computed permissions: additive with deny-always-wins, ceiling enforced by org role | Roles |
| Self-Approval | Hardcoded prevention -- no API path, no admin override can bypass SoD | Roles, PatchWorkflow |
| Approval Escalation | Risk-based: Low (1 GK), Medium (1 GK + 1 Owner), High (2 GK + 1 Owner), Critical (all + cooling period) | PatchWorkflow |
| Audit Trail | Full action audit trail: flag toggles, role changes, calibration updates, patch applications, lifecycle transitions | FeatureControlPlane, Admin |
| Tenant Boundary | `workspace_id` as isolation key (application-layer enforcement) | Admin, FeatureControlPlane |
| Feature Flags | 27 flags, database-backed, per-workspace targeting, 0-100% rollout, circuit breaker | FeatureControlPlane |
| AI Runtime | PydanticAI + LiteLLM proxy, 8 typed tools (all read-only or draft-only), no autonomous actions | AIAgent |
| AI Safety | Max 5 tool calls/message, RBAC-filtered tool availability, circuit breaker (5 errors/60s -> 300s cooldown) | AIAgent |
| Model Routing | Claude Sonnet 4 -> GPT-4o fallback -> Ollama local dev | AIAgent |
| Enrichment | 9 sources fired in parallel (2s timeout), graceful degradation | AIAgent |
| Session Persistence | `otto_sessions` + `otto_messages` tables, indefinite retention, per-message cost tracking | AIAgent |
| Real-Time | Starlette WebSocket + Redis Pub/Sub, JWT auth at connection time, heartbeat cleanup | RealTime |
| Event Bus | BullMQ (Redis-backed) + Trigger.dev for multi-step workflows | EventBus |
| Tech Stack | React/Next.js 14, FastAPI, PostgreSQL 16, Redis 7, Docker Compose -- all MIT/Apache-2.0, self-hostable | TechStack |

---

## 2. Ambiguities and Assumptions

The following items are genuinely open and must be resolved through the sub-spec documents. Each is tagged with whether it blocks MVP.

| # | Ambiguity | Blocks MVP? | Impact |
|---|-----------|-------------|--------|
| A1 | **Cipher scope:** Does "ciphered data" mean all PostgreSQL columns, only JSONB payloads, only document blobs, or full Transparent Data Encryption (TDE)? OrcestrateOS stores plaintext in all 18 core tables. | Yes | Determines schema migration strategy and performance budget |
| A2 | **Key custody:** Who holds the master key? User-derived (passphrase), workspace-admin-held, or HSM/KMS-managed? This changes the threat model fundamentally. | Yes | Affects whether a stolen backup is useful to an attacker |
| A3 | **OGC chunk definition:** What constitutes a "chunk"? A clause, a field extraction, a page, or a semantic unit? How is chunk identity stable across re-extraction? | Yes | Determines knowledge layer schema |
| A4 | **LLM determinism boundary:** What does "deterministic" mean -- same prompt always produces same output, or structured-output-only mode? Temperature=0 does not guarantee identical completions across providers. | Yes | Affects reproducibility guarantees |
| A5 | **Ransomware scope:** Does "ransomware-resilient" mean encrypted backup + fast restore, immutable infrastructure, or both? How fast must rehydration complete? | No (enterprise) | Determines backup topology and RTO/RPO |
| A6 | **Cost ceiling:** What is the actual monthly budget for MVP? $0 (fully free-tier), ~$20 (one cloud VM), ~$100 (small cluster)? | Yes | Constrains tool and hosting choices |
| A7 | **Compliance targets:** Which frameworks (SOC 2, ISO 27001, HIPAA, GDPR) are in-scope for enterprise phase? Each has distinct control requirements. | No (enterprise) | Shapes audit and control requirements |
| A8 | **Multi-tenant isolation model:** Shared database with `workspace_id` Row-Level Security, separate schemas per tenant, or separate databases? OrcestrateOS uses `workspace_id` with application-layer `WHERE` clauses only. | No (enterprise) | Determines data isolation guarantees for customer security reviews |
| A9 | **Passkey/WebAuthn timeline:** Is passwordless auth an MVP requirement or enterprise-phase? Google OAuth is already decided for MVP. | No | Affects auth stack complexity |
| A10 | **DID scope:** Are Decentralized Identifiers for user identity, document provenance, or both? Which DID method (did:web, did:key, did:ion)? | No (enterprise) | Determines identity federation approach |
| A11 | **Otto PII exposure:** When Otto's enrichment pipeline sends Vault context to Claude/GPT, does that context contain PII (counterparty names, financial terms, legal entities)? If so, what redaction is required? | Yes | Directly affects LLM integration design |
| A12 | **File storage encryption:** OrcestrateOS stores files at `$FILE_STORAGE_ROOT/{workspace_id}/{batch_id}/{hash}_{filename}` in plaintext. Does MVP require at-rest encryption for file blobs? | Yes | Affects file_storage.py port |
| A13 | **JWT secret management:** OrcestrateOS reads `JWT_SECRET` from an environment variable with no rotation mechanism. Is this acceptable for MVP? | Yes | Affects auth infrastructure complexity |
| A14 | **Audit event immutability scope:** OrcestrateOS enforces append-only via trigger on `audit_events`. Does this extend to all event tables (`channel_events`, `otto_messages`) or just audit? | Yes | Determines tamper-evidence boundary |

---

## 3. Personas x Documentation Scopes Matrix

Each cell describes what that persona must understand from that documentation scope.

|  | Conceptual Architecture | Threat Model & Guarantees | API & Schema Contracts | Local Dev / Low-Cost Infra | Governance Rules for Agents | LLM Integration Policies |
|---|---|---|---|---|---|---|
| **Founder / Architect** | - How cipher-first middleware maps to Vault hierarchy (L1-L4). - How the "steal the disk" guarantee holds across PostgreSQL, Redis, and file storage. | - Threat model covers insider, cloud-provider, and supply-chain adversaries. - Ransomware rehydration RTO/RPO targets validated. | - API surface is minimal and auth-gated. - `workspace_id` tenant boundary with RLS path confirmed. | - Docker Compose is the sole MVP dependency. - $0/month achievable for solo-developer mode. | - Otto can never autonomously apply patches or escalate permissions. - Self-approval prevention is unforgeable. | - Model routing (Claude -> GPT -> Ollama) and cost exposure understood. - OGC chunk strategy keeps LLM as recall-only. |
| **Backend Engineer** | - Cipher/encrypt boundaries mapped to FastAPI middleware layers. - Which data crosses trust boundaries at each API endpoint. | - Attack vectors each API endpoint must defend against. - Rate limiting and input validation requirements. | - Encryption middleware with clear encrypt/decrypt points. - Schema changes for cipher columns documented. | - Full stack runs locally with `docker compose up`. - Env vars and secret generation documented. | - RBAC filtering for Otto tool availability implemented at API layer. - Max 5 tool calls/message enforced server-side. | - PII redaction before enrichment context reaches LLM providers. - Prompt injection defense in Otto chat endpoint. |
| **Frontend Engineer** | - What data arrives already-decrypted vs. what requires client-side handling. - Triptych panel security boundaries. | - JWT lifecycle (refresh, expiry, revocation). - CSP and XSS boundaries for the Signal/Orchestrate/Control panels. | - Which API responses contain encrypted payloads. - WebSocket auth flow and reconnection. | - Connect to local API/WebSocket without TLS complexity. - CORS configuration for localhost. | - Agent actions displayed with correct attribution (AI-drafted vs. human-approved). - Tool call audit trail in Control panel Context tab. | - LLM cost per session displayed in Control panel. - Streaming SSE error boundaries handled. |
| **AI/ML Engineer** | - How OGC chunks flow from extraction to retrieval to LLM context. - Enrichment sources mapped to security tiers. | - Prompt injection defense for user messages and document content. - Data exfiltration prevention through tool calls. | - VaultContext redaction schema defined. - Which enrichment fields are PII-sensitive classified. | - Ollama runs locally for air-gapped LLM dev. - LiteLLM proxy configured without external API keys. | - Tool safety rules (read-only, draft-only) implemented. - Circuit breaker at 5 errors/60s enforced. | - Temperature/determinism guarantees per use case. - PII redaction rules for each of 9 enrichment sources. |
| **Security Engineer** | - Full data flow audited from upload to storage to extraction to LLM. - All trust boundaries identified. | - All attack surfaces enumerated (API, WebSocket, LLM, file upload, OAuth, BullMQ). - Penetration testing scope defined. | - `workspace_id` isolation verified across all API paths. - Permission computation code audited. | - Docker Compose secrets handling verified (no credentials in images). - `.env` file handling audited. | - Self-approval prevention verified as unbypassable via API. - Role simulation sandbox security boundaries confirmed. | - LLM provider data handling policies assessed. - PII redaction verified before any data reaches external providers. |
| **DevOps Engineer** | - Deployment topology from single-node MVP to multi-node enterprise. - Service dependency map. | - TLS termination, firewall rules, network segmentation configured. - Intrusion detection set up. | - Database migrations with cipher column additions managed. - Backup encryption configured. | - Docker Compose with health checks, restart policies, resource limits. - Local TLS with mkcert. | - Otto circuit breaker state monitored. - Anomalous tool call patterns alerted. | - LiteLLM proxy container managed. - API key rotation automated. - LLM cost per workspace monitored. |
| **Customer Security Reviewer (CISO)** | - Architecture diagram showing trust boundaries and data flow. - Tenant isolation model explained. | - Threat model document with STRIDE analysis. - Evidence of penetration testing and vulnerability management. | - API authentication and authorization mechanisms reviewed. - Input validation evidence. | - N/A (customers don't run local dev). | - AI cannot take autonomous actions -- evidence provided. - Audit trail for all AI interactions available. | - PII redaction policy documented. - LLM provider data processing agreements (DPAs) confirmed. |

---

## 4. MVP Question Inventory (Local-First, Low-Cost)

Assumptions: single developer laptop, Docker Compose only, free-tier or self-hosted components (PostgreSQL, Redis, Ollama or API with strict caps).

### 4.1 Core Concepts and Vocabulary

| # | Question | Why It Matters |
|---|----------|---------------|
| Q1 | What is the precise definition of a "trust boundary" in Airlock's middleware-only model? Where in the FastAPI request lifecycle does encryption/decryption occur? | Determines whether middleware intercepts all DB I/O or operates per-repository |
| Q2 | Does "middleware-only" mean that PostgreSQL, Redis, and file storage are all treated as untrusted persistence backends? | If yes, every write must encrypt and every read must decrypt -- significant performance implications |
| Q3 | How does the cipher model interact with PostgreSQL's JSONB query capabilities? Encrypted JSONB cannot be indexed or queried by the database engine. | May require blind-index patterns or application-layer filtering for all JSONB queries |
| Q4 | Which Airlock vocabulary terms map to security domain concepts? (Vault = encrypted container? Chamber = authorization scope? Gate = access control checkpoint?) | Ensures the security spec uses canonical terms from `/Features/Shell/naming-and-hierarchy.md` consistently |
| Q5 | Does the `workspace_id` tenant boundary from OrcestrateOS remain the primary isolation mechanism, and if so, does it get promoted to PostgreSQL Row-Level Security (RLS)? | OrcestrateOS uses application-layer `WHERE workspace_id = %s` with no database-level enforcement |
| Q6 | Are `channel_events` (the immutable Vault event log) security-critical? If an attacker can modify events, they can rewrite decision lineage. | Determines whether `channel_events` needs the same append-only database trigger as `audit_events` |
| Q7 | What is the canonical term for the security boundary itself? "Airlock" is the product name -- is it also the security metaphor (data passes through an airlock between trusted and untrusted zones)? | Naming consistency for all security documentation |

### 4.2 Data and Cipher Model (MVP)

| # | Question | Why It Matters |
|---|----------|---------------|
| Q8 | Which columns in OrcestrateOS's 18 core tables contain sensitive data that must be encrypted at rest? Candidates: `contracts.file_url`, `accounts.account_name`, `patches.before_value`/`after_value`, all JSONB `metadata` columns. | Defines encryption scope -- encrypting everything vs. targeted columns has very different performance and complexity profiles |
| Q9 | What cipher algorithm and mode? AES-256-GCM (authenticated encryption with associated data) is the standard choice. Does the team have constraints or preferences? | Algorithm choice affects key size (256-bit), nonce management (96-bit per encrypt), and whether ciphertext includes integrity verification |
| Q10 | Where does encryption/decryption execute -- FastAPI middleware (intercepts all DB I/O), repository layer (per-table), or field-level decorators (per-column)? | Middleware is broadest but may encrypt non-sensitive data unnecessarily. Field-level is precise but requires marking every sensitive column explicitly |
| Q11 | How are encryption keys derived? Options: (a) single master key from env var, (b) per-workspace key derived from master via HKDF, (c) per-Vault key derived from workspace key. | Affects key rotation scope -- rotating one Vault's key should not require re-encrypting all Vaults in the workspace |
| Q12 | Can the MVP use PostgreSQL Transparent Data Encryption (TDE) as a simpler starting point, with application-layer encryption added in enterprise phase? | TDE protects at-rest data from disk theft without application code changes, but does NOT protect from SQL injection, admin access, or backup theft without TDE keys |
| Q13 | How are file blobs encrypted? OrcestrateOS's `file_storage.py` writes raw bytes to `$FILE_STORAGE_ROOT/{workspace_id}/{batch_id}/{hash}_{filename}`. Does Airlock encrypt-then-store or store-then-encrypt? | Encrypt-then-store is standard; requires key access at write time and produces different ciphertext per encryption (even for identical files) |
| Q14 | How does deduplication work with encrypted data? OrcestrateOS uses `sha256(file_bytes)[:16]` for dedup. If files are encrypted with different nonces, identical files produce different ciphertexts. | May need a content hash computed before encryption and stored separately as a blind index |
| Q15 | Does Redis need encrypted values? Redis stores BullMQ job payloads (which may contain Vault data), WebSocket subscription state, and enrichment cache snapshots. | If Vault data leaks through Redis job payloads, the "steal the disk" guarantee breaks |
| Q16 | How is the JWT signing key managed in MVP? OrcestrateOS reads `JWT_SECRET` from an env var with no rotation mechanism. | At minimum: document how to generate a strong secret. Ideally: support key rotation with old-key grace period for in-flight tokens |
| Q17 | How are API keys protected at rest? OrcestrateOS stores `sha256(key)` in `api_keys.key_hash`. Is SHA-256 sufficient or should bcrypt/Argon2 be used? | SHA-256 is fast (vulnerable to brute force on short keys). bcrypt adds cost but is slower for high-frequency API key validation |
| Q18 | What is the backup strategy for encrypted data? If you back up encrypted PostgreSQL and lose the encryption key, the backup is permanently unrecoverable. | Key backup strategy is as critical as data backup strategy |

### 4.3 LLM and Knowledge Layer (MVP)

| # | Question | Why It Matters |
|---|----------|---------------|
| Q19 | What data from the 9 enrichment sources is sent to external LLM providers? Specifically, does `DealFields` (territory, legal_entity, counterparty_type) contain PII? | Determines redaction requirements before any LLM API call |
| Q20 | What is the PII classification for each VaultContext field? Per-field assessment needed: public, internal, confidential, restricted. | Classification drives which fields are redacted, replaced, or passed through |
| Q21 | What is the redaction strategy -- remove PII before sending, replace with indexed placeholders (e.g., `[ENTITY_1]`), or send encrypted tokens? | Placeholder approach preserves semantic structure for better LLM responses; removal may degrade quality; encrypted tokens are opaque to the model |
| Q22 | How is prompt injection defended against in the Otto chat endpoint? Users type free-text into Signal panel; document content is injected into system prompts via enrichment. Both are attack vectors. | OrcestrateOS has zero prompt injection defense -- this is an entirely new requirement for Airlock |
| Q23 | What constitutes an "OGC chunk"? Is it a clause from the contract generator's 188 clauses, a field extraction result, a document page, or a custom semantic unit? | Defines the granularity of the knowledge layer and determines how chunks are stored, indexed, and retrieved |
| Q24 | How are OGC chunks stored and retrieved? Vector database (pgvector), full-text search (PostgreSQL tsvector), or the existing `search_corpus` enrichment source (up to 10 matching lines per Vault)? | The `search_corpus` source already exists but uses simple text matching -- is this sufficient for deterministic recall? |
| Q25 | What makes a chunk "human-verified"? Is it a boolean flag, a verification event in the audit trail, or a Gatekeeper approval via the Gate system? | Determines the trust level of knowledge layer content and whether unverified chunks can reach the LLM |
| Q26 | How does the determinism guarantee work across different LLM providers? Claude, GPT-4o, and Ollama/Llama produce different outputs even at temperature=0. | May require structured output schemas (PydanticAI `result_type`) to constrain variance across providers |
| Q27 | What is the data retention policy for Otto sessions? They currently persist indefinitely. GDPR right-to-deletion may require purging session history. | Affects `otto_sessions`/`otto_messages` schema and whether soft-delete is sufficient |
| Q28 | What context window management strategy prevents token overflow? The 9 enrichment sources plus full conversation history could exceed model context limits. | Need a truncation, summarization, or sliding-window strategy for long-running sessions |
| Q29 | When using the local Ollama model (`otto-local`), does the same PII redaction apply, or is it bypassed since data never leaves the machine? | Local models have fundamentally different trust boundaries than external providers |

### 4.4 Local Dev and Cost Constraints

| # | Question | Why It Matters |
|---|----------|---------------|
| Q30 | Can Docker Compose be the sole orchestration tool for MVP, or is lightweight Kubernetes (k3s) needed for secrets management and health-check restarts? | Docker Compose is simpler but lacks native secrets management and rolling update capability |
| Q31 | What Docker images are needed? Current list: PostgreSQL 16, Redis 7, FastAPI (custom), Next.js 14 (custom), LiteLLM proxy, Novu. Others? | Defines the `docker-compose.yml` service topology and total resource requirements |
| Q32 | How are secrets passed to Docker containers? Options: `.env` file (convenient but insecure), Docker secrets (Swarm-only), or lightweight self-hosted vault (e.g., Infisical). | `.env` files are visible in `docker inspect` and risk being committed to git |
| Q33 | What is the minimum hardware requirement for the full local stack? PostgreSQL + Redis + FastAPI + Next.js + LiteLLM + Novu + Ollama -- Ollama alone needs 8-16GB RAM for Llama 3.1. | Total stack could exceed 32GB RAM, limiting who can run it locally |
| Q34 | Is local TLS (HTTPS) required for MVP dev, or is plain HTTP acceptable for localhost? | OAuth callback flows often require HTTPS. WebSocket security in production requires `wss://`. |
| Q35 | How is the dev database seeded with test data without exposing real contract data? OrcestrateOS has a seed workspace (`ws_SEED0100000000000000000000`) with demo accounts. | Need a synthetic data generator that respects the cipher model and produces realistic Vault hierarchies |
| Q36 | What is the cost of running Ollama locally vs. using Claude/GPT API? Is there a monthly API cost budget for development? | Ollama is free but lower quality. Claude Sonnet at moderate dev use: ~$5-20/month. |
| Q37 | Does the "no mandatory SaaS" requirement mean zero external API calls, or just that external services are optional with local fallbacks for everything? | Affects whether Google OAuth is acceptable (requires Google's servers) vs. needing a fully local auth option |

---

## 5. Enterprise Question Inventory

Assumptions: multi-tenant SaaS, Zero Trust posture, heavy security scrutiny, potential DIDs/passkeys/chain logging, audit and compliance requirements.

### 5.1 Threat Model and Guarantees

| # | Question | Why It Matters |
|---|----------|---------------|
| Q38 | What is the formal threat model? STRIDE analysis covering: Spoofing (identity), Tampering (data integrity), Repudiation (audit), Information Disclosure (confidentiality), Denial of Service (availability), Elevation of Privilege (authorization). | Provides structured adversary analysis required by SOC 2 and ISO 27001 |
| Q39 | Who are the adversaries? (a) External attacker, (b) malicious insider, (c) curious cloud provider employee, (d) nation-state, (e) supply-chain compromise (dependency injection). Which are in-scope? | Scope determines control depth and cost |
| Q40 | What are the RTO (Recovery Time Objective) and RPO (Recovery Point Objective) for ransomware scenarios? | Determines backup frequency, rehydration architecture, and infrastructure cost |
| Q41 | What is the availability SLA target? 99.9% (8.7h downtime/year), 99.95%, or 99.99%? | Affects redundancy architecture (single-node vs. multi-AZ) and operational cost |
| Q42 | What is the blast radius model? If one workspace is compromised, can the attacker reach other workspaces? | Determines whether workspace isolation is logical (RLS) or physical (separate databases/containers) |
| Q43 | What compliance certifications are targeted and in what order? SOC 2 Type I then Type II? ISO 27001? GDPR assessment? | Each has a 6-18 month timeline and specific control requirements |
| Q44 | What is the incident response plan? Who is notified, in what order, with what SLAs? | Required for SOC 2 and ISO 27001 certification |
| Q45 | What is the vulnerability disclosure policy? Bug bounty program, responsible disclosure, or internal-only? | Affects public trust and security researcher engagement |
| Q46 | What network segmentation is required? Should the LLM proxy be on a separate network from the database? Should Redis be internal-only? | Determines firewall rules, Docker network topology, and service mesh requirements |

### 5.2 Key Management and Cipher Architecture

| # | Question | Why It Matters |
|---|----------|---------------|
| Q47 | What KMS is used in production? AWS KMS, GCP Cloud KMS, HashiCorp Vault, or self-hosted (SoftHSM)? | Determines key storage security, access control, and audit capabilities for cryptographic operations |
| Q48 | What is the key hierarchy? Proposed: Root Key (KMS-held) -> Workspace Key (derived) -> Vault Key (derived) -> Field Key (derived). Is this the right granularity? | Deeper hierarchy enables more granular revocation but increases key management complexity |
| Q49 | What is the key rotation strategy? How often are keys rotated, and what happens to data encrypted with old keys? | Re-encryption on rotation is expensive; envelope encryption avoids it by rotating only the Key Encryption Key (KEK) |
| Q50 | How are keys backed up and escrowed? If the KMS is unavailable, can data still be decrypted? | Single point of failure for all data access -- must be addressed before production |
| Q51 | Is envelope encryption the approach? (Data encrypted with Data Encryption Key, DEK encrypted with KEK stored in KMS.) | Standard pattern that decouples data encryption from key management and enables cheap rotation |
| Q52 | How are encryption keys distributed to multiple FastAPI worker processes? Shared memory, environment injection at startup, or per-request KMS API calls? | Per-request KMS calls add latency (~5-50ms each); caching keys in worker memory has key-exposure implications |
| Q53 | What is the crypto-shredding policy? Can a workspace be permanently destroyed by deleting its workspace key, rendering all data unrecoverable? | Provides a GDPR-compliant "right to erasure" without touching every encrypted row |
| Q54 | How is key usage audited? Should every encrypt/decrypt operation be logged, or only key lifecycle events (create, rotate, revoke)? | KMS audit logs are typically separate from application audit logs; both needed for compliance |

### 5.3 Identity, Passkeys, and DIDs

| # | Question | Why It Matters |
|---|----------|---------------|
| Q55 | What is the passkey (WebAuthn) implementation strategy? Replace passwords entirely, or offer as a second factor alongside OAuth? | Passkeys eliminate phishing attacks but require careful enrollment UX and recovery paths |
| Q56 | What authenticator types are supported? Platform authenticators (TouchID, FaceID, Windows Hello), roaming authenticators (YubiKey), or both? | Affects device enrollment flow, cross-device authentication, and hardware requirements |
| Q57 | What is the recovery path if a user loses all their authenticators? Recovery codes, admin-initiated reset, or social recovery? | Must balance security (no easy bypass) with usability (users lose phones) |
| Q58 | What DID method is used? `did:web` (DNS-anchored, simple), `did:key` (self-certifying, no infrastructure), or `did:ion` (Bitcoin-anchored, decentralized)? | `did:web` is simplest but requires DNS control; `did:key` is portable but lacks discoverability; `did:ion` is most decentralized but heaviest |
| Q59 | Are DIDs for user identity (replacing email-based auth), document provenance (proving a contract was processed by Airlock), or organizational identity (workspace verification)? | Scope determines schema complexity and verification infrastructure |
| Q60 | How do DIDs interact with the existing Google OAuth flow? Is OAuth the bootstrap mechanism that creates the DID, or does the DID eventually replace OAuth? | Integration pattern affects auth middleware and migration path |
| Q61 | What Verifiable Credentials does Airlock issue? Examples: "this user is a Gatekeeper in workspace X", "this contract was verified by Gatekeeper Y on date Z". | Determines credential schema design and issuance/verification flows |
| Q62 | How does federated identity work across workspaces? Can a user's identity (and trust reputation) transfer when they move between organizations? | DIDs enable portable identity; OAuth alone does not |

### 5.4 LLM Safety and Determinism

| # | Question | Why It Matters |
|---|----------|---------------|
| Q63 | What is the formal prompt injection defense strategy? Three categories: (a) direct injection (user crafts malicious chat message), (b) indirect injection (malicious content embedded in uploaded documents), (c) tool-use injection (LLM tricked into calling tools with unintended arguments). | PydanticAI's typed tools help with (c); categories (a) and (b) need explicit defense layers |
| Q64 | What is the output validation strategy? Should every LLM response be validated against a Pydantic schema before display to prevent hallucinated tool calls or injected content? | Prevents malformed responses from reaching the UI and provides a defense-in-depth layer |
| Q65 | How is model behavior monitored in production? Log every prompt/response pair? Sample a percentage? Only log anomalies (high token count, tool call failures)? | Full logging is expensive but necessary for compliance; sampling reduces cost but may miss incidents |
| Q66 | What is the LLM provider data processing agreement (DPA) status? Do Anthropic's and OpenAI's DPAs cover Airlock's enterprise customers' contract data? | Enterprise customers will require this documentation before allowing their data to reach third-party models |
| Q67 | How is model drift detected? If a provider silently updates their model, Airlock's extraction/classification quality could degrade. | Need baseline metrics, regression test suites, and alerting on quality score changes |
| Q68 | What is the human-in-the-loop escalation path when the LLM produces low-confidence results? Otto already creates draft-only patches, but is the confidence threshold formalized? | The Feature Control Plane has calibration sliders but no documented confidence-to-escalation mapping |
| Q69 | How is AI attribution handled? When a patch is AI-drafted and human-approved, is the full attribution chain preserved for legal and audit purposes? | `author_type: "ai_assisted"` exists in the schema but may need to be more granular (which model, which enrichment sources, which tool calls) |
| Q70 | What is the AI cost governance model? Can a workspace admin set a monthly cost ceiling? What happens when the ceiling is reached -- degrade to local model, disable Otto, or alert-only? | Cost tracking exists per session/message but there is no enforcement mechanism |

---

## 6. Documentation Table of Contents (MVP to Enterprise)

### Tier 1 -- MVP Crawl (Single Laptop, Docker Compose, $0/month)

```
/Features/Security/
  overview.md                          <- THIS FILE
  data-cipher-model.md
  auth-and-identity.md
  rbac-enforcement.md
  llm-safety-policy.md
  local-dev-playbook.md
  audit-and-immutability.md
```

#### 6.1 `data-cipher-model.md` -- Encryption at Rest

**Purpose:** Define exactly which data is encrypted, with what algorithm, at what application layer, and with what key hierarchy for MVP.

**Questions answered:** Q8, Q9, Q10, Q11, Q12, Q13, Q14, Q15, Q18 | Ambiguities: A1, A2, A3, A12

**Sections:**
1. Cipher scope -- which PostgreSQL columns, JSONB fields, file blobs, and Redis values are encrypted
2. Algorithm selection -- AES-256-GCM rationale, nonce management, authenticated encryption
3. Key derivation hierarchy -- master key -> workspace key -> Vault key (HKDF)
4. Encrypt/decrypt middleware architecture -- where in FastAPI's request lifecycle
5. File storage encryption -- encrypt-then-store flow for document blobs
6. Deduplication with encryption -- blind index for content-hash dedup
7. Redis value encryption -- BullMQ job payloads, enrichment cache
8. MVP shortcuts -- what can use TDE or plaintext and defer to enterprise phase

#### 6.2 `auth-and-identity.md` -- JWT, OAuth, API Keys, Sessions

**Purpose:** Port and harden OrcestrateOS authentication (`auth.py`, `jwt_utils.py`, `role_scope.py`) for Airlock's expanded role model.

**Questions answered:** Q16, Q17 | Ambiguities: A9, A13

**Sections:**
1. JWT lifecycle -- issuance, refresh, expiry, revocation, key rotation with grace period
2. Google OAuth flow -- callback, token exchange, session creation
3. API key management -- generation, hashing (SHA-256 vs. bcrypt), scoping per workspace, rotation
4. Session management -- stateless JWT vs. server-side sessions, cookie security
5. Auth middleware architecture -- porting OrcestrateOS `resolve_auth` to Airlock
6. Sandbox/dev mode security boundaries
7. WebSocket authentication flow -- JWT query parameter, connection validation, heartbeat

#### 6.3 `rbac-enforcement.md` -- Permission Computation, Tenant Isolation

**Purpose:** Formalize the security properties of the Roles spec and define API-layer enforcement guarantees.

**Questions answered:** Q1, Q2, Q3, Q4, Q5 | Ambiguities: A8

**Sections:**
1. Permission computation security proof -- additive model, deny-always-wins, ceiling enforcement
2. Self-approval prevention -- why it is unforgeable (no API path, no admin override)
3. `workspace_id` tenant boundary -- application-layer `WHERE` to PostgreSQL RLS migration path
4. Role simulation sandbox -- security boundaries (read-only, no state mutation)
5. API endpoint permission matrix -- every endpoint mapped to required role(s)
6. Vault member inheritance -- cascading membership from parent to child Vaults with `inherited` flag

#### 6.4 `llm-safety-policy.md` -- PII Redaction, Prompt Injection, Determinism

**Purpose:** Define how data flows to and from LLM providers with security controls at every boundary.

**Questions answered:** Q19, Q20, Q21, Q22, Q23, Q24, Q25, Q26, Q27, Q28, Q29 | Ambiguities: A4, A11

**Sections:**
1. PII classification per VaultContext field -- public/internal/confidential/restricted
2. Redaction strategy -- placeholder replacement (`[ENTITY_1]`) vs. removal vs. encrypted tokens
3. Prompt injection defense -- direct (user messages), indirect (document content), tool-use (argument manipulation)
4. Output validation schema -- PydanticAI `result_type` enforcement before UI display
5. OGC chunk definition -- granularity, identity stability, trust flag
6. Chunk storage and retrieval -- pgvector vs. tsvector vs. `search_corpus` enrichment
7. Determinism across providers -- structured output schemas, temperature constraints
8. Context window management -- truncation, summarization, sliding-window strategy
9. Local model security boundaries -- Ollama trust model, redaction bypass policy
10. Data retention for AI sessions -- GDPR compliance, soft-delete, purge schedule

#### 6.5 `local-dev-playbook.md` -- Docker Compose, Secrets, Seeding

**Purpose:** Step-by-step guide to running Airlock securely on a single laptop with zero SaaS dependencies.

**Questions answered:** Q30, Q31, Q32, Q33, Q34, Q35, Q36, Q37 | Ambiguities: A6

**Sections:**
1. Docker Compose service topology -- service list, network layout, port mapping
2. Secret generation and management -- `.env` handling, secret rotation, git-ignore patterns
3. Local TLS with mkcert -- self-signed certs for OAuth callback and WSS
4. Database seeding -- synthetic data generator, cipher-aware seed script, demo Vault hierarchy
5. Ollama local LLM setup -- model download, LiteLLM config, air-gapped mode
6. Hardware requirements -- minimum RAM/CPU/disk for full stack
7. Cost estimation -- free-tier resource mapping, optional API cost budget

#### 6.6 `audit-and-immutability.md` -- Append-Only Events, Tamper Evidence

**Purpose:** Define which data stores are append-only and how tamper evidence works across the platform.

**Questions answered:** Q6 | Ambiguities: A14

**Sections:**
1. Append-only table catalog -- `audit_events`, `channel_events`, `otto_messages`: scope and enforcement
2. Database trigger enforcement -- PostgreSQL triggers preventing UPDATE/DELETE
3. Hash chain for tamper evidence -- optional chained hash per event for integrity verification
4. Decision lineage -- who proposed, who verified, who promoted (traced through events)
5. Audit event schema for security events -- login, permission change, key access, LLM call

---

### Tier 2 -- Enterprise Walk / Run (Multi-Tenant SaaS, Compliance, Zero Trust)

```
/Features/Security/
  security-constitution.md
  threat-model.md
  kms-and-key-hierarchy.md
  passkeys-and-dids.md
  compliance-and-governance.md
  ransomware-and-rehydration.md
  deployment-and-monitoring.md
```

#### 6.7 `security-constitution.md` -- Immutable Security Principles

**Purpose:** 10 non-negotiable principles that all security decisions must satisfy. This is the "bill of rights" for Airlock's security posture.

**Principles:**
1. **Ciphered at rest, encrypted in transit** -- No plaintext data on any storage medium.
2. **Middleware-only trust** -- The application layer is the sole trusted boundary. Storage backends are untrusted.
3. **No autonomous AI actions** -- All AI operations are read-only or draft-only. Humans approve every change.
4. **Self-approval prevention is unforgeable** -- No API path, no permission override, no admin action can bypass separation of duties.
5. **Append-only audit** -- Security-relevant events cannot be modified or deleted. Ever.
6. **Workspace isolation** -- No data crosses workspace boundaries without explicit federation.
7. **Key custody transparency** -- Users and admins always know who holds the keys to their data.
8. **Rehydratable from cold** -- The entire system can be rebuilt from encrypted backups and key material.
9. **PII never reaches untrusted LLM providers unredacted** -- Redaction happens before the network call.
10. **Fail closed** -- When security controls fail, access is denied, not granted.

#### 6.8 `threat-model.md` -- STRIDE Analysis

**Purpose:** Formal threat model for enterprise security reviews and compliance audits.

**Questions answered:** Q38, Q39, Q40, Q41, Q42, Q43, Q44, Q45, Q46

**Sections:**
1. STRIDE analysis per component -- API, WebSocket, LLM proxy, file upload, OAuth, BullMQ, Redis
2. Adversary catalog -- external attacker, malicious insider, cloud provider, supply-chain
3. Attack surface enumeration -- every ingress point with attack vectors
4. Blast radius analysis -- workspace isolation depth (logical vs. physical)
5. Network segmentation requirements -- service mesh, mTLS, internal-only services
6. Penetration testing scope and cadence

#### 6.9 `kms-and-key-hierarchy.md` -- Full KMS Spec

**Purpose:** Production key management for multi-tenant encrypted data.

**Questions answered:** Q47, Q48, Q49, Q50, Q51, Q52, Q53, Q54

**Sections:**
1. KMS provider selection -- AWS KMS vs. HashiCorp Vault vs. self-hosted (with decision criteria)
2. Key hierarchy -- root -> workspace -> Vault -> field (diagram)
3. Envelope encryption implementation -- DEK/KEK pattern
4. Key rotation without re-encryption -- KEK rotation, DEK re-wrap
5. Key backup and escrow -- disaster recovery for key material
6. Key distribution to FastAPI workers -- caching strategy, memory security
7. Crypto-shredding for workspace deletion -- GDPR erasure via key destruction
8. KMS audit logging -- every key lifecycle event tracked

#### 6.10 `passkeys-and-dids.md` -- WebAuthn, DIDs, Verifiable Credentials

**Purpose:** Passwordless authentication and decentralized identity for enterprise deployment.

**Questions answered:** Q55, Q56, Q57, Q58, Q59, Q60, Q61, Q62 | Ambiguities: A9, A10

**Sections:**
1. WebAuthn registration and authentication flows
2. Authenticator type support -- platform (biometric) and roaming (hardware key)
3. Recovery paths -- recovery codes, admin reset, cross-device sync
4. DID method selection and rationale
5. DID-OAuth bridge -- OAuth as bootstrap, DID as long-term identity
6. Verifiable Credential schemas -- role attestations, document provenance
7. Federated identity across workspaces -- portable trust

#### 6.11 `compliance-and-governance.md` -- SOC 2, ISO 27001, GDPR

**Purpose:** Map Airlock's security controls to compliance framework requirements.

**Questions answered:** Q43, Q44, Q45 | Ambiguities: A7

**Sections:**
1. SOC 2 Type II control mapping -- Trust Service Criteria to Airlock controls
2. ISO 27001 Annex A mapping -- 114 controls to Airlock implementations
3. GDPR data processing requirements -- consent, erasure (crypto-shredding), data portability
4. Data retention and deletion policies -- per-table retention schedule
5. Incident response plan -- detection, triage, containment, recovery, post-mortem
6. Vulnerability disclosure policy -- responsible disclosure program
7. Third-party risk management -- LLM provider assessments, sub-processor register

#### 6.12 `ransomware-and-rehydration.md` -- Backup, Recovery, Rehydration

**Purpose:** Ensure Airlock can recover from total infrastructure compromise within defined RTO/RPO.

**Questions answered:** Q40, Q41 | Ambiguities: A5

**Sections:**
1. Immutable infrastructure pattern -- stateless containers, all state in encrypted persistence
2. Backup topology -- database (WAL archival), files (encrypted blob snapshots), keys (escrowed), config (git-versioned)
3. Rehydration sequence -- step-by-step from encrypted backups to running system
4. RTO/RPO targets -- quantified recovery objectives
5. Testing cadence -- quarterly rehydration drills with documented results
6. Stateless service architecture -- why any container can be replaced without data loss

#### 6.13 `deployment-and-monitoring.md` -- Production Operations

**Purpose:** Production deployment, CI/CD, monitoring, and cost governance for enterprise Airlock.

**Sections:**
1. CI/CD pipeline with security gates -- SAST, DAST, dependency scanning, image signing
2. Deployment topology -- single-node to multi-AZ progression
3. Monitoring stack -- Prometheus/Grafana or alternatives
4. Security alerting rules -- anomalous auth, permission escalation, LLM cost spike
5. Database scaling -- read replicas, connection pooling, partitioning strategy
6. Performance baselines -- latency targets per API endpoint
7. Zero Trust network -- service mesh (mTLS), API gateway, WAF

---

## 7. Agent-Governance Artifacts

The minimum set of concrete artifacts that must exist so that future agents know exactly what to do.

### 7.1 Root CLAUDE.md Security Section

| Property | Value |
|----------|-------|
| **Location** | `/CLAUDE.md` -- append to existing file |
| **Owner** | Founder / Architect |
| **Target audience** | All Claude Code agents working in this repo |
| **When created** | Immediately (before any Security sub-specs are written) |
| **Update frequency** | Every time a security decision is locked |

**Content to add:**
- Security Vocabulary table (cipher, KMS, DEK, KEK, OGC chunk, trust boundary, RLS, HKDF)
- Security Decisions (Locked) section mirroring the Architecture Decisions list
- Security file structure under `/Features/Security/`
- LLM Safety Rules for agents: "Never include real PII in spec examples. Use synthetic data only. Always specify encrypt/decrypt points when describing data flows."
- Cipher model one-liner for quick reference

### 7.2 Security Constitution

| Property | Value |
|----------|-------|
| **Location** | `/Features/Security/security-constitution.md` |
| **Owner** | Founder + Security Engineer |
| **Target audience** | All engineers, auditors, AI agents, customer security reviewers |
| **When created** | Before MVP implementation begins |
| **Update frequency** | Annually, or when threat model fundamentally changes |

10 immutable principles (see Section 6.7 above).

### 7.3 Cipher and Key Hierarchy Spec

| Property | Value |
|----------|-------|
| **Location** | `/Features/Security/data-cipher-model.md` (MVP) + `/Features/Security/kms-and-key-hierarchy.md` (enterprise) |
| **Owner** | Backend Engineer + Security Engineer |
| **Target audience** | Backend engineers implementing encryption, DevOps managing keys |
| **When created** | Before database schema is finalized |
| **Update frequency** | On key rotation schedule change, cipher algorithm update, or new data classification |

Column-level cipher map, key derivation diagram, envelope encryption flow, rotation procedure.

### 7.4 LLM Access Policy

| Property | Value |
|----------|-------|
| **Location** | `/Features/Security/llm-safety-policy.md` |
| **Owner** | AI/ML Engineer + Security Engineer |
| **Target audience** | AI/ML engineers building Otto, security reviewers, compliance auditors |
| **When created** | Before Otto chat endpoint is connected to external LLM providers |
| **Update frequency** | When new enrichment sources are added, new tools are defined, or PII classification changes |

Per-field PII classification, redaction implementation, prompt injection test suite, provider DPA requirements, cost ceiling enforcement rules.

### 7.5 Local Dev Playbook

| Property | Value |
|----------|-------|
| **Location** | `/Features/Security/local-dev-playbook.md` |
| **Owner** | DevOps Engineer |
| **Target audience** | Any engineer setting up Airlock locally |
| **When created** | When Docker Compose configuration is first written |
| **Update frequency** | Every time a new service is added to `docker-compose.yml` |

Step-by-step: clone, generate secrets, `docker compose up`, verify, seed database, connect OAuth, run Ollama. Hardware requirements. Cost table.

### 7.6 Ransomware Response and Rehydration Playbook

| Property | Value |
|----------|-------|
| **Location** | `/Features/Security/ransomware-and-rehydration.md` |
| **Owner** | DevOps Engineer + Founder |
| **Target audience** | Operations team, incident responders, customer security reviewers |
| **When created** | Before production deployment |
| **Update frequency** | Quarterly (after rehydration drills) |

Runbook: detect ransomware -> isolate -> assess blast radius -> retrieve encrypted backups -> retrieve keys from escrow -> rehydrate database -> rehydrate file storage -> rehydrate Redis -> verify integrity -> restore service. Estimated time per step.

---

## 8. Next 3 Concrete Docs to Write

Prioritized for a local, low-cost MVP start:

### Priority 1: `/Features/Security/overview.md` (This File)

**Rationale:** Anchor document. Establishes the security vision, catalogs every decided item (from Roles, AIAgent, Admin, FeatureControlPlane specs), enumerates every open gap, and cross-references all sub-specs. Must exist before any other security document.

**Dependencies:** None. Uses this plan as input. Done.

### Priority 2: `/Features/Security/data-cipher-model.md`

**Rationale:** The "steal the disk" guarantee is the founder's core thesis. This document resolves ambiguities A1-A3 (cipher scope, key custody, OGC chunk definition) and answers Q8-Q18. Without this, no other security spec can specify how data is protected. It directly affects the database schema design.

**Dependencies:** `overview.md` must exist (for cross-referencing).

### Priority 3: `/Features/Security/llm-safety-policy.md`

**Rationale:** Otto is already spec'd with 8 tools, 9 enrichment sources, and streaming SSE. The AI runtime (`/Features/AIAgent/ai-runtime.md`) defines VaultContext with fields like `deal_fields.legal_entity` and `deal_fields.territory` that are PII. Before any LLM integration code is written, the PII classification and redaction strategy must be locked. Prompt injection defense is currently completely absent.

**Dependencies:** `overview.md` and `data-cipher-model.md` should exist first.

---

## Related Specs

- [Shell / Naming and Hierarchy](../Shell/naming-and-hierarchy.md) -- Canonical vocabulary
- [Roles / Overview](../Roles/overview.md) -- RBAC system, permission computation, self-approval prevention
- [AIAgent / Overview](../AIAgent/overview.md) -- Otto tools, enrichment, session persistence
- [AIAgent / AI Runtime](../AIAgent/ai-runtime.md) -- VaultContext, PydanticAI, LiteLLM
- [AIAgent / Session Persistence](../AIAgent/session-persistence.md) -- otto_sessions/otto_messages schema
- [FeatureControlPlane / Overview](../FeatureControlPlane/overview.md) -- Circuit breaker, feature flags
- [Admin / Overview](../Admin/overview.md) -- Workspace admin, connector management
- [RealTime / Overview](../RealTime/overview.md) -- WebSocket JWT auth, heartbeat
- [EventBus / Overview](../EventBus/overview.md) -- BullMQ, event routing
- [PatchWorkflow / Overview](../PatchWorkflow/overview.md) -- Approval chain, SoD enforcement
- [TechStack / Overview](../TechStack/overview.md) -- Locked library choices
