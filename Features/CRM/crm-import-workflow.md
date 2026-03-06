# CRM Import Workflow -- Design Spec

> **Status:** SPECCED
> **Source:** Gap analysis, implementation requirements

## Summary
Defines how users import existing accounts, contacts, and deals into Airlock CRM from external sources (CSV, other CRMs, manual entry).

### 1. Import Sources

| Source | Method | Supported Data |
|--------|--------|----------------|
| CSV/Excel | File upload | Accounts, Contacts, Deals |
| Salesforce | API sync (future) | Full CRM data |
| HubSpot | API sync (future) | Full CRM data |
| Google Contacts | API sync (future) | Contacts only |
| Manual entry | UI forms | Any entity |
| Contract ingestion | Automatic (existing) | Accounts + contacts via entity resolution |

### 2. CSV Import Flow (MVP)

Step-by-step user flow:
1. **Navigate**: Admin > Import Data (or CRM > Accounts > Import button)
2. **Upload**: Drag/drop or browse for CSV/XLSX file (max 10MB, max 10K rows for MVP)
3. **Preview**: Show first 5 rows, detect column headers
4. **Map fields**: Two-column mapper:
   - Left: CSV column headers (auto-detected)
   - Right: Airlock field (dropdown with all entity fields)
   - Auto-match by name similarity (fuzzy match)
   - "Skip this column" option
   - Required fields highlighted (name for accounts, name+account for contacts)
5. **Configure options**:
   - Entity type: Account | Contact | Deal
   - Duplicate handling: Skip duplicates | Update existing | Create new always
   - Duplicate matching: By name | By email | By phone | By custom field
   - Default values: Set defaults for unmapped required fields (e.g., segment=SMB, source=Import)
6. **Validate**:
   - Show validation results: X valid, Y warnings, Z errors
   - Errors: missing required fields, invalid formats, too-long values
   - Warnings: possible duplicates found, unusual values
   - User can fix inline or download error report
7. **Import**:
   - Background job (BullMQ) processes rows
   - Progress bar with row count
   - Partial success supported (good rows import, bad rows reported)
8. **Results**:
   - Summary: X imported, Y updated, Z skipped, W errors
   - Download error report CSV
   - Link to view imported records

### 3. Deduplication Strategy

| Entity | Match Fields | Match Algorithm |
|--------|-------------|-----------------|
| Account | name | Fuzzy match (Levenshtein distance < 3) + exact domain match |
| Contact | email OR (name + account) | Exact email match, fuzzy name + account match |
| Deal | name + account | Exact match |

When duplicate found:
- **Skip**: Keep existing, log as skipped
- **Update**: Merge fields (imported values overwrite blanks, don't overwrite existing non-blank unless forced)
- **Create new**: Always create, append "(2)" to name if exact match

### 4. Import Validation Rules

| Field | Validation |
|-------|-----------|
| Email | RFC 5322 format |
| Phone | E.164 format (auto-normalize with country code) |
| Value/Amount | Numeric, strip currency symbols, commas |
| Date | ISO 8601, MM/DD/YYYY, DD/MM/YYYY (configurable) |
| Stage | Must match one of 5 pipeline stages (case-insensitive) |
| Segment | Must match Enterprise/Mid-Market/SMB |
| URL | Valid URL format |

### 5. Security Considerations

- Imported data passes through encryption middleware before storage
- PII fields (email, phone, address) encrypted at rest
- Import audit trail: who imported, when, what file, how many records
- Rate limit: max 3 imports per hour per user
- File validation: reject files with macros, scripts, or executable content
- No imported data sent to LLM without redaction

### 6. Deal Creation (Non-Import)

Document all ways a deal can be created:

**Method 1: Manual creation (Pipeline > + New Deal)**
- Modal with fields: Deal name, Account (search/create), Value, Stage (default: Prospecting), Owner, Close date (estimated), Source
- Required: Name, Account, Value
- Side effects: Item vault created, stage tasks auto-created, activity event logged

**Method 2: Convert from Lead (New Leads > Convert to Deal)**
- Pre-fills: Account (from lead's company), contacts (from lead), source (from lead source)
- User adds: Deal name, Value, Close date
- Side effects: Lead archived, deal created, lead history linked to deal

**Method 3: Convert from Qualify (Qualify > Converted column)**
- Same as Method 2 but triggered by drag to Converted column
- Confirmation modal: "Create deal for [Lead Name]?"

**Method 4: From Smart Line (automated)**
- Workflow Engine scores inbound → if score >=80 and intent="deal_discussion" → auto-creates deal in Prospecting
- Owner assigned by territory/round-robin rule
- Needs human confirmation (task: "Review auto-created deal")

**Method 5: From Contract Ingestion (automatic)**
- Entity resolution identifies new customer → Account + Counterparty created
- Contract vault (Item level) auto-created at Discover chamber
- Not a "deal" until manually flagged as such (or workflow triggers)

### 7. Empty States

Define what each screen shows when empty:

| Screen | Empty State Message | Primary Action |
|--------|-------------------|----------------|
| Inbox | "No unread messages. Your Smart Line inbox is clear." | "Set up Smart Line" or "View all messages" |
| New Leads | "No new leads yet. Leads arrive from web forms, Smart Line, or manual entry." | "Import Leads" or "Create Lead" |
| Qualify | "No leads in qualification. Move leads here from New Leads when ready to evaluate." | "View New Leads" |
| Pipeline | "No active deals. Create your first deal or convert a qualified lead." | "+ New Deal" or "Import Deals" |
| Deal Review | "No deals pending review. All clear!" | (no action needed) |
| Won Deals | "No won deals yet. Celebrate here when deals close!" | "View Pipeline" |
| Accounts | "No accounts yet. Import your customer list or create accounts manually." | "Import Accounts" or "+ New Account" |
| Contacts | "No contacts yet. Contacts are created automatically from Smart Line interactions or import." | "Import Contacts" |
| Activity Feed | "No activity yet. Events will appear here as you interact with accounts and contacts." | "View Inbox" |
| Onboarding | "No active onboarding workflows. Onboarding starts automatically when deals are won." | "View Pipeline" |
| Health Monitor | "No customers to monitor yet. Health tracking begins when deals are won." | "View Pipeline" |
| Renewals | "No upcoming renewals. Renewal dates are set when contracts close." | "View Won Deals" |
| Expansion | "No expansion opportunities yet. Create one from an existing account." | "+ New Opportunity" |

---

## Related Specs

- [overview.md](./overview.md) -- CRM module overview
- [crm-data-model.md](./crm-data-model.md) -- Entity schemas for imported data
- [crm-automations.md](./crm-automations.md) -- Scoring and workflows triggered after import
- [/Features/Security/overview.md](/Features/Security/overview.md) -- Encryption and PII handling
