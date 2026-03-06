# Evidence Pack

## Summary

The evidence pack is the structured evidence payload that accompanies every patch through the approval chain. It provides reviewers with the context needed to make informed approval or rejection decisions. Every patch -- from Draft through Applied -- carries its evidence pack as an immutable audit record.

The evidence pack consists of five components: a **when clause** (conditional logic for when the patch applies), a **then clause** (the specific data mutation), a **because clause** (business reasoning), **file attachments** (supporting documents), and a **before/after snapshot** (auto-captured field values at patch creation time).

**DECISION:** Evidence is structured, not free-form. The when/then clauses use visual builders that produce validated JSON. This ensures machine-readable evidence that can be audited, diffed, and replayed.

## Evidence Pack Structure

### When Clause (JSON Condition)

The when clause defines the condition under which the patch applies. It is authored via a visual builder and stored as validated JSON.

**Supported Operators:**

| Operator | Description | Applicable Types |
|----------|-------------|------------------|
| `equals` | Exact match | string, number, boolean, date |
| `notEquals` | Negated exact match | string, number, boolean, date |
| `contains` | Substring or array membership | string, array |
| `notContains` | Negated substring or array membership | string, array |
| `greaterThan` | Numeric or date comparison | number, date |
| `lessThan` | Numeric or date comparison | number, date |
| `greaterThanOrEqual` | Inclusive numeric or date comparison | number, date |
| `lessThanOrEqual` | Inclusive numeric or date comparison | number, date |
| `matches` | Regex pattern match | string |
| `isEmpty` | Field is null, empty string, or empty array | all |
| `isNotEmpty` | Field has a value | all |

**Logical Grouping:**

Conditions can be combined using `AND`, `OR`, and `NOT` logical operators. Groups can be nested up to 3 levels deep.

**Example JSON Output:**

```json
{
  "logic": "AND",
  "conditions": [
    {
      "field": "territory",
      "operator": "equals",
      "value": "North America"
    },
    {
      "logic": "OR",
      "conditions": [
        {
          "field": "royalty_rate",
          "operator": "greaterThan",
          "value": 0.15
        },
        {
          "field": "advance_amount",
          "operator": "greaterThan",
          "value": 50000
        }
      ]
    }
  ]
}
```

---

### Then Clause (JSON Action)

The then clause defines the specific data mutation the patch will perform. It is authored via a visual builder and stored as validated JSON.

**Supported Operations:**

| Operation | Description | Applicable Types | Example |
|-----------|-------------|------------------|---------|
| `set` | Replace the current value | all | Set `territory` to `"Worldwide"` |
| `append` | Add an item to an array field | array | Append `"Digital"` to `distribution_channels` |
| `remove` | Remove an item from an array field | array | Remove `"Physical"` from `distribution_channels` |
| `increment` | Add a numeric delta (positive or negative) | number | Increment `royalty_rate` by `0.02` |
| `clear` | Set the field value to null | all | Clear `expiration_date` |

**Example JSON Output:**

```json
{
  "actions": [
    {
      "field": "territory",
      "operation": "set",
      "value": "Worldwide"
    },
    {
      "field": "royalty_rate",
      "operation": "increment",
      "value": 0.02
    },
    {
      "field": "distribution_channels",
      "operation": "append",
      "value": "Digital"
    }
  ]
}
```

---

### Because Clause (Free Text)

The because clause captures the human-readable business reasoning for the patch. It is the author's explanation of why the change is necessary.

**Guidelines for Authors:**

| Guideline | Description |
|-----------|-------------|
| **What changed** | Describe the factual change being proposed |
| **Why it matters** | Explain the business impact or contractual significance |
| **What evidence supports it** | Reference attached files, conversations, or external sources |

**Constraints:**

| Constraint | Value |
|-----------|-------|
| Minimum length | 10 characters |
| Maximum length | 2,000 characters |
| Format | Plain text (no Markdown, no HTML) |
| Required | Yes -- patch cannot be submitted without a because clause |

---

### File Attachments

File attachments provide supporting documentation for the patch -- screenshots, scanned amendments, correspondence, spreadsheets, or any other evidence.

**Supported Formats:**

| Format | MIME Type | Notes |
|--------|-----------|-------|
| PDF | `application/pdf` | Contracts, amendments, correspondence |
| PNG | `image/png` | Screenshots, diagrams |
| JPG/JPEG | `image/jpeg` | Photos, scanned documents |
| CSV | `text/csv` | Financial data, rate tables |
| XLSX | `application/vnd.openxmlformats-officedocument.spreadsheetml.sheet` | Spreadsheets |
| TXT | `text/plain` | Plain text notes, logs |
| DOCX | `application/vnd.openxmlformats-officedocument.wordprocessingml.document` | Word documents |

**Limits:**

| Constraint | Value |
|-----------|-------|
| Max file size (per file) | 10 MB |
| Max total size (per patch) | 50 MB |
| Max attachment count | 10 files |

**Storage:**

- Files are uploaded to object storage (S3-compatible)
- Access is controlled via presigned URLs with configurable expiry (default: 1 hour)
- File metadata (name, size, MIME type, upload timestamp, uploader ID) stored in the patch record
- Files are immutable once uploaded -- no in-place replacement, only new uploads

---

### Before/After Snapshot

The before/after snapshot is an auto-captured record of the field's current value at the moment the patch enters Draft, alongside the proposed new value from the then clause.

| Property | Description |
|----------|-------------|
| **Capture trigger** | Automatically captured when the author selects a target field in the patch editor |
| **Before value** | The field's value at the moment of Draft creation, serialized as JSON |
| **After value** | The proposed value computed by applying the then clause to the before value |
| **Immutability** | The before value is frozen at capture time and cannot be modified, even if the underlying field changes before the patch is applied |
| **Staleness indicator** | If the field's live value diverges from the captured before value before the patch is applied, a "stale snapshot" warning appears |
| **Post-application update** | When the patch reaches Applied state, the after value is updated with the actual resulting value (which should match the proposed value unless a conflict occurred) |

---

## Visual Builder UX

The when clause and then clause are authored via visual builders -- structured form interfaces that produce validated JSON. The builders are designed to be accessible to non-technical users while supporting power users with a raw JSON fallback.

### Field Picker

- Searchable dropdown populated from the `master_schema_map` (152 fields)
- Fields are organized by category (e.g., Financial, Territory, Rights, Parties, Dates)
- Each field entry shows: field name, data type icon, and a brief description
- Recently used fields appear at the top of the list
- Keyboard navigation: type to filter, arrow keys to navigate, Enter to select

### When Clause Builder

```
+-------------------------------------------------------------------+
| WHEN                                                    [+ Group] |
|-------------------------------------------------------------------|
| [AND v]                                                           |
|   +-------------------------------------------------------------+ |
|   | [territory     v] [equals        v] [North America      ]   | |
|   |                                                       [x]   | |
|   +-------------------------------------------------------------+ |
|   +-------------------------------------------------------------+ |
|   | [OR v]                                                       | |
|   |   [royalty_rate v] [greaterThan   v] [0.15               ]   | |
|   |   [advance_amt  v] [greaterThan   v] [50000              ]   | |
|   |                                                       [x]   | |
|   +-------------------------------------------------------------+ |
|                                                                   |
| [+ Add Condition]                              [Raw JSON toggle]  |
+-------------------------------------------------------------------+
```

**Operator dropdown** changes based on the selected field's data type:

| Field Type | Available Operators |
|------------|-------------------|
| string | equals, notEquals, contains, notContains, matches, isEmpty, isNotEmpty |
| number | equals, notEquals, greaterThan, lessThan, greaterThanOrEqual, lessThanOrEqual, isEmpty, isNotEmpty |
| boolean | equals, notEquals |
| date | equals, notEquals, greaterThan, lessThan, greaterThanOrEqual, lessThanOrEqual, isEmpty, isNotEmpty |
| array | contains, notContains, isEmpty, isNotEmpty |

**Nesting rules:**

- Maximum 3 levels of group nesting
- Each group has a logic selector (AND / OR)
- NOT is applied per-condition via a toggle, not as a group wrapper
- Attempting to add a 4th nesting level shows a validation warning

### Then Clause Builder

```
+-------------------------------------------------------------------+
| THEN                                                              |
|-------------------------------------------------------------------|
|   +-------------------------------------------------------------+ |
|   | [territory     v] [set           v] [Worldwide           ]   | |
|   |                                                       [x]   | |
|   +-------------------------------------------------------------+ |
|   +-------------------------------------------------------------+ |
|   | [royalty_rate   v] [increment     v] [0.02               ]   | |
|   |                                                       [x]   | |
|   +-------------------------------------------------------------+ |
|                                                                   |
| [+ Add Action]                                 [Raw JSON toggle]  |
+-------------------------------------------------------------------+
```

**Operation dropdown** changes based on the selected field's data type:

| Field Type | Available Operations |
|------------|---------------------|
| string | set, clear |
| number | set, increment, clear |
| boolean | set, clear |
| date | set, clear |
| array | set, append, remove, clear |

### Raw JSON Toggle

- Located at the bottom-right of each builder
- Toggles between visual builder and a syntax-highlighted JSON editor
- Switching to raw JSON preserves the current builder state as JSON
- Switching back to visual builder parses the JSON and populates the builder
- If the JSON is malformed or uses unsupported constructs, the visual builder shows a parse error and stays in raw JSON mode

### Real-Time Validation

| Check | Timing | Error Display |
|-------|--------|---------------|
| Field exists in schema | On field selection | Inline error below field picker |
| Operator is valid for field type | On operator selection | Operator dropdown filters to valid options |
| Value matches field type | On value input (debounced 300ms) | Inline error below value input |
| JSON syntax (raw mode) | On keystroke (debounced 500ms) | Red underline on invalid lines |
| Nesting depth limit | On group add | Warning toast, group add blocked |
| At least one condition (when) | On submit attempt | Builder border turns red, error message |
| At least one action (then) | On submit attempt | Builder border turns red, error message |

---

## Evidence Review Requirements

Reviewers must engage with every piece of evidence before they can approve a patch. This ensures no approval is rubber-stamped.

### Evidence Engagement Tracking

| Mechanism | Description |
|-----------|-------------|
| **Seen detection** | Each evidence item tracks a "seen" status via Intersection Observer (scroll-into-view) |
| **Minimum dwell time** | Item must be visible in the viewport for at least 2 seconds to register as seen |
| **Unseen indicator** | Unseen items display a blue dot (8px circle, `#3B82F6`) to the left of the item label |
| **Approve button state** | Approve button is disabled until all evidence items are marked as seen |
| **Tooltip on disabled Approve** | "Review all evidence items before approving (X of Y reviewed)" |

### Evidence Display Order

| # | Item | Display |
|---|------|---------|
| 1 | Before/After Snapshot | Always visible at the top -- pinned, not scrollable |
| 2 | When Clause | Rendered as a visual condition tree (read-only builder view) |
| 3 | Then Clause | Rendered as a visual action list (read-only builder view) |
| 4 | Because Clause | Rendered as formatted text block |
| 5 | File Attachments | Rendered as a file list with preview thumbnails |

### File Attachment Preview

| File Type | Preview Behavior |
|-----------|-----------------|
| PDF | Opens in a modal using PDF.js viewer with page navigation, zoom, and text selection |
| PNG / JPG | Opens in a modal image viewer with zoom and pan |
| CSV | Opens in a modal with a formatted table view (read-only) |
| XLSX | Opens in a modal with a formatted spreadsheet view (read-only, first sheet only) |
| TXT | Opens in a modal with monospace text display |
| DOCX | Opens in a modal with rendered document view (read-only) |

### Before/After Snapshot Pinning

The before/after snapshot is always visible in the review interface. It does not scroll with the rest of the evidence pack. It is rendered as a fixed comparison card at the top of the evidence review panel:

```
+-------------------------------------------------------------------+
| BEFORE / AFTER                                     [Stale Warning] |
|-------------------------------------------------------------------|
| Field: territory                                                  |
| Before: North America                                             |
| After:  Worldwide                                                 |
|                                                                   |
| Field: royalty_rate                                               |
| Before: 0.15                                                      |
| After:  0.17                                                      |
+-------------------------------------------------------------------+
```

---

## Evidence Versioning

Evidence evolves through the patch lifecycle. Permissions and mutability change at each state transition.

### Mutability by State

| Patch State | Author Can Modify Original | Author Can Add New | Evidence Frozen |
|-------------|---------------------------|-------------------|----------------|
| **Draft** | Yes -- full edit access | Yes | No |
| **Submitted** | No -- read-only | No | Yes |
| **Needs_Clarification** | No -- original is frozen | Yes -- response evidence only | Partially |
| **Verifier_Responded** | No | No | Yes |
| **Verifier_Approved** | No | No | Yes |
| **Admin_Hold** | No | No | Yes |
| **Admin_Approved** | No | No | Yes |
| **Applied** | No | No | Yes (snapshot updates with actuals) |
| **Rejected** | No | No | Yes |
| **Cancelled** | No | No | Yes |

### Response Evidence (Needs_Clarification)

When a reviewer requests clarification, the author can add new evidence items but cannot modify or delete the original evidence. Response evidence is visually separated from the original:

```
+-------------------------------------------------------------------+
| ORIGINAL EVIDENCE                                     [Submitted] |
|-------------------------------------------------------------------|
| When: territory equals "North America"                            |
| Then: set territory to "Worldwide"                                |
| Because: "Amendment signed expanding territory rights..."         |
| Attachments: amendment-v2.pdf                                     |
+-------------------------------------------------------------------+

+-------------------------------------------------------------------+
| RESPONSE EVIDENCE                          [Clarification Round 1] |
|-------------------------------------------------------------------|
| Added by: @jane.doe on 2025-12-04                                 |
| Because: "Per legal review, the amendment was countersigned on..." |
| Attachments: countersigned-amendment.pdf, legal-memo.pdf          |
+-------------------------------------------------------------------+
```

**Response evidence rules:**

| Rule | Description |
|------|-------------|
| Additive only | Author can add new because text, new attachments. Cannot modify original. |
| Round tracking | Each clarification cycle creates a new response round (Round 1, Round 2, etc.) |
| Max rounds | No hard limit, but after 3 rounds a warning suggests escalation |
| Reviewer visibility | Response evidence is highlighted with a cyan left border to distinguish from original |

### Post-Application Snapshot Update

When a patch reaches the **Applied** state, the before/after snapshot is updated one final time:

| Field | Before | Proposed After | Actual After |
|-------|--------|---------------|-------------|
| territory | North America | Worldwide | Worldwide |
| royalty_rate | 0.15 | 0.17 | 0.17 |

If the actual after value differs from the proposed after value (due to a concurrent change or conflict resolution), the discrepancy is flagged with an amber warning.

---

## API Contract

### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/vaults/{vault_id}/patches` | Create a new patch with its initial evidence pack |
| `GET` | `/api/vaults/{vault_id}/patches/{patch_id}/evidence` | Retrieve the full evidence pack for a patch |
| `PATCH` | `/api/vaults/{vault_id}/patches/{patch_id}/evidence` | Add evidence items (used during Needs_Clarification response) |
| `POST` | `/api/vaults/{vault_id}/patches/{patch_id}/evidence/attachments/presign` | Request a presigned upload URL for a file attachment |
| `DELETE` | `/api/vaults/{vault_id}/patches/{patch_id}/evidence/attachments/{attachment_id}` | Remove an attachment (Draft state only) |

### Create Patch (POST)

**Request Body:**

```json
{
  "title": "Expand territory to Worldwide",
  "target_fields": ["territory", "royalty_rate"],
  "when_clause": {
    "logic": "AND",
    "conditions": [
      { "field": "territory", "operator": "equals", "value": "North America" }
    ]
  },
  "then_clause": {
    "actions": [
      { "field": "territory", "operation": "set", "value": "Worldwide" },
      { "field": "royalty_rate", "operation": "increment", "value": 0.02 }
    ]
  },
  "because_clause": "Amendment signed expanding territory rights to Worldwide effective Q1 2026.",
  "version": 0
}
```

**Response:** `201 Created` with the full patch object including auto-captured before/after snapshot.

### Retrieve Evidence (GET)

**Response Body:**

```json
{
  "patch_id": "patch_abc123",
  "when_clause": { "..." },
  "then_clause": { "..." },
  "because_clause": "Amendment signed expanding territory rights...",
  "snapshot": {
    "captured_at": "2025-12-01T10:30:00Z",
    "fields": [
      { "field": "territory", "before": "North America", "after": "Worldwide" },
      { "field": "royalty_rate", "before": 0.15, "after": 0.17 }
    ],
    "is_stale": false
  },
  "attachments": [
    {
      "id": "att_001",
      "filename": "amendment-v2.pdf",
      "mime_type": "application/pdf",
      "size_bytes": 245760,
      "uploaded_at": "2025-12-01T10:32:00Z",
      "uploaded_by": "user_123",
      "download_url": "https://storage.example.com/...(presigned)..."
    }
  ],
  "response_evidence": [
    {
      "round": 1,
      "added_at": "2025-12-04T14:00:00Z",
      "added_by": "user_123",
      "because_clause": "Per legal review, the amendment was countersigned on...",
      "attachments": [ "..." ]
    }
  ]
}
```

### Presigned Upload Flow

1. Client calls `POST .../evidence/attachments/presign` with `{ "filename": "doc.pdf", "mime_type": "application/pdf", "size_bytes": 245760 }`
2. Server validates file type and size limits, returns `{ "upload_url": "https://...", "attachment_id": "att_001" }`
3. Client uploads file directly to the presigned URL (PUT)
4. Client confirms upload by calling `PATCH .../evidence` with the `attachment_id`
5. Server verifies the file exists in storage and links it to the patch

---

## Related Specs

- [Patch Workflow Overview](./overview.md) -- States, transitions, optimistic locking
- [Approval Chain](./approval-chain.md) -- Vertical timeline UI for approval steps
- [Action Focus](./action-focus.md) -- Triptych layout for patch authoring (editor + preview)
- [Diff View](./diff-view.md) -- Visual comparison of original vs proposed changes
- [SLA Timer](./sla-timer.md) -- Countdown timer for reviewer deadlines
- [Document Suite](/Features/DocumentSuite/overview.md) -- TipTap editor, PDF.js viewer, custom blocks
