# Patch Workflow

## Summary

Patches are data correction proposals for contracts. A patch proposes a specific change to one or more contract fields, carries structured evidence (when/then/because clauses), and flows through a multi-step approval chain before being applied. The workflow enforces role-based authorization, self-approval prevention, and optimistic locking.

**DECISION:** Patch authoring uses a dedicated Action Focus state -- a full editor experience in the triptych, not inline editing from field cards.

## OrcestrateOS Source

| Source File | Lines | Notes |
|-------------|-------|-------|
| `server/routes/patches.py` | 496 | All patch CRUD, transitions, evidence, history |

## Patch States

The patch lifecycle consists of 12 states:

| State | Description |
|-------|-------------|
| **Draft** | Author is composing the patch; not yet visible to reviewers |
| **Submitted** | Author has submitted for review; enters the approval chain |
| **Needs_Clarification** | Reviewer has requested additional information from the author |
| **Verifier_Responded** | Author has responded to a clarification request |
| **Verifier_Approved** | A verifier (non-author) has approved the patch |
| **Admin_Hold** | An admin has paused the patch (SLA timer pauses) |
| **Admin_Approved** | An admin (non-author) has given final approval |
| **Applied** | The patch has been applied to the contract data |
| **Rejected** | The patch has been rejected at any approval step |
| **Cancelled** | The author has withdrawn the patch |
| **Sent_to_Otto** | The patch has been forwarded to Otto for AI-assisted review |
| **Otto_Returned** | Otto has returned the patch (may need re-review) |

## State Transitions

20 valid transitions with role-based authorization:

| From | To | Authorized Roles | Notes |
|------|----|-----------------|-------|
| Draft | Submitted | Author | Initial submission |
| Draft | Cancelled | Author | Author withdraws before submitting |
| Submitted | Needs_Clarification | Verifier | Reviewer requests more info |
| Submitted | Verifier_Approved | Verifier | First approval gate |
| Submitted | Rejected | Verifier | Reviewer rejects outright |
| Needs_Clarification | Verifier_Responded | Author | Author provides clarification |
| Needs_Clarification | Cancelled | Author | Author withdraws |
| Verifier_Responded | Verifier_Approved | Verifier | Verifier accepts after clarification |
| Verifier_Responded | Needs_Clarification | Verifier | Still needs more info |
| Verifier_Responded | Rejected | Verifier | Rejected after clarification |
| Verifier_Approved | Admin_Approved | Admin | Final approval |
| Verifier_Approved | Admin_Hold | Admin | Admin pauses for review |
| Verifier_Approved | Rejected | Admin | Admin rejects |
| Admin_Hold | Admin_Approved | Admin | Admin releases hold and approves |
| Admin_Hold | Rejected | Admin | Admin rejects from hold |
| Admin_Approved | Applied | System | System applies the patch |
| Admin_Approved | Sent_to_Otto | Admin | Forwarded to Otto for AI review |
| Sent_to_Otto | Otto_Returned | System/Otto | Otto returns patch |
| Otto_Returned | Admin_Approved | Admin | Admin re-approves after Otto return |
| Otto_Returned | Rejected | Admin | Admin rejects after Otto return |

## Self-Approval Prevention

- **Verifier_Approved** transition is blocked if the acting verifier is the patch author
- **Admin_Approved** transition is blocked if the acting admin is the patch author
- The "Approve" button is hidden (not disabled) when the current user is the author -- see [approval-chain.md](./approval-chain.md) for UI details

## Evidence Pack

Every patch carries a structured evidence payload:

| Field | Type | Description |
|-------|------|-------------|
| `when_clause` | JSON condition | The condition under which the patch applies (visual builder output) |
| `then_clause` | JSON action | The specific data change to make (visual builder output) |
| `because_clause` | Free text | Human-readable reasoning for why the patch is needed |
| File attachments | Files | Supporting documents, screenshots, before/after evidence |

## History Tracking

Each patch maintains an immutable transition history stored as a JSONB array:

```
[
  { "from": "Draft", "to": "Submitted", "actor_id": "user_123", "timestamp": "2025-..." },
  { "from": "Submitted", "to": "Verifier_Approved", "actor_id": "user_456", "timestamp": "2025-..." },
  ...
]
```

Every state transition appends a new entry. History is append-only and cannot be edited.

## Optimistic Locking

Patches use a `version` integer field for optimistic concurrency control. Every write operation must include the current version number; if the version has changed since the client last read the patch, the write is rejected with a conflict error. This prevents two reviewers from simultaneously approving or modifying the same patch.

## Related Specs

- [Action Focus](./action-focus.md) -- Triptych layout for patch authoring
- [Approval Chain](./approval-chain.md) -- Vertical timeline UI for approval steps
- [SLA Timer](./sla-timer.md) -- Countdown timer for reviewer deadlines
