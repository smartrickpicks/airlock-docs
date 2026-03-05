# Record Inspector -- Entity Resolution

## Design Decision

**Inline disambiguation in Signal panel.**

Rejected alternatives:
- Modal dialog -- blocks the user from referencing document content while resolving.
- CRM cross-link panel -- too heavy for the common case; pulls the user out of the contract review flow.

Inline disambiguation keeps the resolution workflow inside the triptych without interrupting document review. The Signal panel (left side) is the natural home because entity resolution is an upstream decision that affects downstream field scoring.

---

## Placement

| Property | Value |
|---|---|
| Panel | Signal panel (left side of triptych) |
| Card type | Entity Resolution card |
| Visibility | Always present when a contract channel is open; content varies by resolution state |
| Multiple entities | Each detected entity gets its own resolution card, stacked vertically |

---

## Card Content

Every entity resolution card shows:

| Element | Description |
|---|---|
| Legal entity name | Primary entity extracted from the contract |
| Counterparties | Other parties identified in the contract |
| Match confidence | Numeric score (0.0 - 1.0) for the top candidate |
| Resolution state | One of: auto-resolved, pending confirmation, ambiguous, no match, new customer |

---

## Resolution States

### Exact Match (Confidence = 1.0)

| Aspect | Behavior |
|---|---|
| UI state | Auto-resolved, no user action required |
| Card appearance | Green border, checkmark icon, matched account name displayed |
| Event feed | Green "Entity resolved" event emitted |
| Evidence | Shows which evidence types confirmed the match (e.g., name_exact, alias) |

### High Confidence (> 0.80)

| Aspect | Behavior |
|---|---|
| UI state | Auto-resolved with confirmation prompt |
| Card appearance | Green border with amber "Confirm?" badge |
| User action | Single-click "Confirm" button or "Override" link |
| Confirm | Locks resolution, emits green event, badge removed |
| Override | Opens candidate list (same as Ambiguous state) |

### Ambiguous (0.40 - 0.80)

| Aspect | Behavior |
|---|---|
| UI state | Inline disambiguation -- user must choose |
| Card appearance | Amber border, candidate list visible |
| Candidate list | Top 5 candidates ordered by confidence score |
| Each candidate row | Account name, confidence score, evidence chips |
| User action | Click a candidate to confirm match |
| "None of these" | Button at bottom of candidate list (see below) |

### Low Confidence (< 0.40)

| Aspect | Behavior |
|---|---|
| UI state | No match found |
| Card appearance | Red border, "No match found" label |
| Search box | Inline search field to find an existing account |
| "Create in CRM" | Button to create a new account (cross-module action) |

### New Customer Detected

| Aspect | Behavior |
|---|---|
| UI state | New entity not found in any existing account |
| Card appearance | Blue border, "New customer detected" label |
| Primary action | "Create in CRM" button |
| Secondary action | Search box to link to an existing account if the detection is wrong |

---

## Evidence Chips

Evidence chips appear on candidate rows (in Ambiguous state) and on resolved cards (showing what confirmed the match). Each chip is a small labeled tag.

| Chip | Score / Behavior | Description |
|---|---|---|
| `name_exact` | 1.0 | Entity name matches account name exactly |
| `name_fuzzy` | 0.6 -- 0.99 | Fuzzy string match on entity name; score shown on chip |
| `address_verified` | Binary (present/absent) | Address confirmed against account records |
| `address_partial` | Binary (present/absent) | Partial address match (e.g., city + state but not street) |
| `account_context` | Binary (present/absent) | Contextual signals from account history support the match |
| `service_context_penalty` | Negative modifier | Service-related terms detected that reduce confidence (see edge case below) |

Chips use color coding:
- Green chip: strong positive signal (name_exact, address_verified)
- Amber chip: partial signal (name_fuzzy, address_partial, account_context)
- Red chip: negative signal (service_context_penalty)

---

## "None of These" Flow

When the user clicks "None of these" on the candidate list:

1. Candidate list collapses.
2. Card transitions to a manual resolution state.
3. A search box appears for finding an existing account.
4. If the user selects an account, a new alias entry is created in the `account_aliases` table.
5. If no account fits, "Create in CRM" button is available.

The alias created through this flow ensures that future contracts from the same entity resolve instantly (score 1.0).

---

## Edge Cases

### Service Context Penalty

| Aspect | Detail |
|---|---|
| Trigger | Service-related terms detected in entity context (e.g., "service provider", "subcontractor") |
| Evidence chip | `service_context_penalty` -- displays "Service context detected: [term]" |
| Impact | Reduces match confidence score |
| User override | User can dismiss the penalty by clicking "Override" on the chip; override is logged |

### Alias Exists

| Aspect | Detail |
|---|---|
| Trigger | Entity name matches an entry in `account_aliases` |
| Resolution | Instant (score 1.0), same as Exact Match |
| Evidence | Card shows `alias` chip with the original alias string |
| No user action required | Auto-resolved with green event |

### Multiple Entities in One Contract

| Aspect | Detail |
|---|---|
| Trigger | Contract contains more than one legal entity (e.g., multi-party agreement) |
| UI | Each entity gets its own resolution card in the Signal panel |
| Order | Cards ordered by extraction position in the document |
| Independence | Each card resolves independently; confirming one does not affect others |

---

## Events Emitted

| Event | Trigger | Consumed By |
|---|---|---|
| `entity.resolved` | User confirms a match or auto-resolution completes | Record Inspector (updates field cards in Entity Resolution section) |
| `entity.override` | User overrides an auto-resolved match | Record Inspector, audit log |
| `entity.alias_created` | User creates a manual alias via "None of these" | Account service, `account_aliases` table |
| `entity.new_customer` | User clicks "Create in CRM" | CRM module (cross-module action) |

---

## Interaction Summary

| User Action | Result |
|---|---|
| Open contract channel | Entity resolution cards appear in Signal panel |
| Click "Confirm" on high-confidence match | Resolution locked, green event emitted |
| Click candidate in ambiguous list | Candidate selected as match, event emitted |
| Click "None of these" | Manual resolution mode with search box |
| Select account from search | Alias created, resolution locked |
| Click "Create in CRM" | Cross-module action to create new account |
| Click "Override" on service_context_penalty chip | Penalty dismissed, confidence recalculated |
| View resolved card | Evidence chips show what confirmed the match |

---

## State Transitions

```
[Document Opened]
       |
       v
[Entity Extracted] --> confidence = 1.0 --> [Auto-Resolved]
       |
       +--> confidence > 0.80 --> [Auto-Resolved + Confirm Prompt]
       |                                  |
       |                          Confirm / Override
       |
       +--> confidence 0.40-0.80 --> [Ambiguous: Show Candidates]
       |                                  |
       |                      Select / "None of These"
       |
       +--> confidence < 0.40 --> [No Match: Search or Create]
       |
       +--> no existing account --> [New Customer Detected]
```

All terminal states emit an `entity.resolved` or `entity.new_customer` event that propagates to Record Inspector field cards.
