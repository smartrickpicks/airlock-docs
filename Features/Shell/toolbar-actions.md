# Toolbar Actions -- Design Spec

> **Status:** SPECCED
> **Source:** Shell overview (toolbar layout zones), Search overview (existing slash commands), user requirements for contextual quick actions

## Summary

The toolbar sits at the top of the Main Content area and currently displays only a view title, breadcrumb trail, Activity button, and right-side dock icons. This spec adds three new capabilities: (1) a universal Create button that opens a context-sensitive menu based on the active module, (2) contextual quick action buttons that change with the current module and view, and (3) expanded slash commands in the Cmd+K palette for meeting scheduling, contact creation, event creation, and vault creation. Together these turn the toolbar from a passive label into the primary action surface for power users.

---

## Toolbar Layout (Updated)

### Current Layout

```
[Module Icon] [Breadcrumb: Module > Chamber > View] ................................ [Activity] [Dock Icons]
```

### New Layout

```
[Module Icon] [Breadcrumb: Module > Chamber > View] [+ Create ▼] [Quick Actions...] .... [Activity] [Cmd+K Search] [Dock Icons]
```

### Zone Breakdown

| Zone | Position | Width | Contents |
|------|----------|-------|----------|
| Module Identity | Far left | 32px icon + auto breadcrumb | Module icon + breadcrumb trail (Module > Chamber > View) |
| Create Button | After breadcrumb | 96px fixed | Universal `+ Create` button with dropdown chevron |
| Quick Actions | After Create | auto (max 4 buttons + overflow) | Context-sensitive icon buttons, changes per module/view |
| Spacer | Center | flex-1 | Empty space, collapses as viewport narrows |
| Activity | Before dock | 32px icon | Opens Signal panel (existing) |
| Search Trigger | Before dock | 32px icon | Magnifying glass, opens Cmd+K palette |
| Dock Icons | Far right | 6 icons at 32px each | Tasks, CRM, Calendar, Docs, Chat, Otto AI (existing) |

### Toolbar Dimensions

| Property | Value |
|----------|-------|
| Height | 48px |
| Background | `--airlock-surface` (#0F1219) |
| Bottom border | 1px solid `--airlock-border` (#1E2330) |
| Padding | 0 16px |
| Z-index | Above scrolling content, below modals |
| Position | Sticky top within Main Content area |

### Responsive Behavior

| Viewport Width | Behavior |
|----------------|----------|
| >= 1280px | Full toolbar: breadcrumb + Create + up to 4 quick actions + Activity + Search + dock |
| 1024-1279px | Breadcrumb truncates to Module > View (drops Chamber). Quick actions collapse to 2 visible + overflow. |
| < 1024px | Not supported in v1. Desktop required message (per Shell spec). |

When quick actions exceed available space, excess buttons collapse into a "More" overflow menu (horizontal ellipsis icon, `MoreHorizontal` from Lucide).

---

## Create Button ("+" Universal Create)

### Appearance

| Property | Value |
|----------|-------|
| Label | `+ Create` |
| Icon | `Plus` (Lucide), 14px, left of text |
| Typography | Fira Sans, 13px, 600 (semibold) |
| Background | `--airlock-cyan` (#00D1FF) at 12% opacity |
| Text color | `--airlock-cyan` (#00D1FF) |
| Border | 1px solid `--airlock-cyan` at 20% opacity |
| Border radius | 6px |
| Padding | 6px 12px |
| Hover | Background brightens to 20% opacity, border to 40% |
| Active | Background at 30% opacity, slight inset shadow |
| Chevron | `ChevronDown` (Lucide), 12px, right of text, rotates 180 degrees when menu open |

### Trigger

| Method | Action |
|--------|--------|
| Click the `+ Create` button | Opens context-sensitive dropdown menu |
| Keyboard: `Cmd+N` | Opens the same dropdown menu |
| Cmd+K then type "new" or "create" | Shows matching create commands in palette |

### Context-Sensitive Menu

The Create menu renders different options depending on which module is currently active. The menu always has two sections: module-specific items (top) and universal items (bottom), separated by a divider.

#### Contracts Module

| Item | Icon | Shortcut | Action |
|------|------|----------|--------|
| New Vault | `Plus` | -- | Opens vault creation form (contract intake) |
| Upload Contract | `Upload` | -- | Opens drag-and-drop upload zone |
| New Patch | `GitBranch` | -- | Opens Action Focus state (patch editor) |
| Schedule Review Meeting | `Video` | `Cmd+Shift+M` | Opens Meeting Scheduler modal |
| New Task | `CheckSquare` | `Cmd+Shift+T` | Opens Quick Task Creation form, linked to current vault if in vault view |

#### CRM Module

| Item | Icon | Shortcut | Action |
|------|------|----------|--------|
| New Contact | `UserPlus` | `Cmd+Shift+C` | Opens Contact Creation form |
| New Account | `Building2` | -- | Opens Account (parent vault) creation form |
| New Deal | `Handshake` | -- | Opens Deal creation form (item vault at level 4) |
| Schedule Meeting | `Video` | `Cmd+Shift+M` | Opens Meeting Scheduler modal |
| New Communication | `MessageSquare` | -- | Opens Messenger compose for selected contact |
| New Task | `CheckSquare` | `Cmd+Shift+T` | Opens Quick Task Creation form |

#### Tasks Module

| Item | Icon | Shortcut | Action |
|------|------|----------|--------|
| New Task | `CheckSquare` | `Cmd+Shift+T` | Opens Quick Task Creation form |
| New Task Board | `Columns` | -- | Opens board creation (name + default columns) |
| Schedule Meeting | `Video` | `Cmd+Shift+M` | Opens Meeting Scheduler modal |

#### Calendar Module

| Item | Icon | Shortcut | Action |
|------|------|----------|--------|
| New Event | `CalendarPlus` | `Cmd+Shift+E` | Opens Event Creation form |
| Schedule Meeting | `Video` | `Cmd+Shift+M` | Opens Meeting Scheduler modal (with Google Meet toggle) |
| New Reminder | `Bell` | -- | Opens inline reminder creator (date + message) |

#### Documents Module

| Item | Icon | Shortcut | Action |
|------|------|----------|--------|
| New Document | `FilePlus` | -- | Opens blank TipTap editor in Orchestrate panel |
| Upload Document | `Upload` | -- | Opens file upload zone (multi-format: PDF, DOCX, XLSX, etc.) |
| New Template | `FileCode` | -- | Opens template builder (TipTap with variable placeholders) |

#### Universal Section (Always Shown at Bottom)

Below a `1px solid --airlock-border` divider:

| Item | Icon | Shortcut | Action |
|------|------|----------|--------|
| New Task | `CheckSquare` | `Cmd+Shift+T` | Quick Task Creation (context-free) |
| Schedule Meeting | `Video` | `Cmd+Shift+M` | Meeting Scheduler modal |
| Quick Note | `StickyNote` | -- | Inline timestamped note (title + body, saved to current vault or personal notes) |

**Deduplication rule:** If a module-specific section already includes "New Task" or "Schedule Meeting", the universal section omits the duplicate. The universal section only fills in items the module section does not provide.

### Recent Creates

The menu shows the user's last 3 create actions at the very top, above the module-specific section, in a "Recent" group with muted styling:

```
+--------------------------------------------+
| RECENT                                      |
|   New Task "Review Henderson terms"    2m   |
|   New Contact "Sarah Blake"           15m   |
|   Upload Contract "sony-dist-v2.pdf"  1h    |
|--------------------------------------------|
| CONTRACTS                                   |
|   + New Vault                               |
|   + Upload Contract                         |
|   + New Patch                               |
|   + Schedule Review Meeting      Cmd+Shift+M|
|   + New Task                     Cmd+Shift+T|
|--------------------------------------------|
|   + Quick Note                              |
+--------------------------------------------+
```

Recent creates are stored in client-side state (localStorage), scoped per user per workspace. Maximum 10 entries retained; 3 displayed.

### Menu Design

| Property | Value |
|----------|-------|
| Width | 320px |
| Max height | 480px (scrollable if content exceeds) |
| Background | `--airlock-card` (#151923) |
| Border | 1px solid `--airlock-border` (#1E2330) |
| Border radius | 8px |
| Shadow | `0 8px 32px rgba(0, 0, 0, 0.5)` |
| Position | Anchored below the Create button, left-aligned |
| Animation | Fade in + slide down, 150ms ease |
| Dismiss | Click outside, Escape, or selecting an item |

### Menu Item Design

| Property | Value |
|----------|-------|
| Height | 36px |
| Padding | 8px 12px |
| Icon | 16px, `--airlock-muted` (#64748B) |
| Label | Fira Sans, 13px, 400, `--airlock-text` (#E2E8F0) |
| Shortcut hint | Fira Code, 11px, `--airlock-muted`, right-aligned |
| Hover | Background `--airlock-surface` (#0F1219), icon color shifts to `--airlock-cyan` |
| Group header | Fira Sans, 11px, 500, uppercase, letter-spacing 0.5px, `--airlock-muted`, 8px top padding |
| Divider | 1px solid `--airlock-border`, 4px vertical margin |

---

## Quick Action Buttons

### Definition

Quick action buttons are small icon-only buttons rendered in the toolbar between the Create button and the spacer. They provide one-click access to frequently used actions for the current context. They are fully contextual -- they change based on:

1. **Active module** (Contracts vs CRM vs Tasks vs Calendar vs Documents)
2. **Active view within the module** (Triage vs Pipeline vs Vault detail)
3. **Active vault** (if viewing a specific vault, actions scope to that vault)

### Quick Action Button Design

| Property | Value |
|----------|-------|
| Size | 32px x 32px |
| Icon size | 16px |
| Icon color | `--airlock-muted` (#64748B) |
| Background | Transparent |
| Border | None |
| Border radius | 6px |
| Hover | Background `rgba(255, 255, 255, 0.06)`, icon color `--airlock-text` (#E2E8F0) |
| Active/Pressed | Background `rgba(255, 255, 255, 0.1)` |
| Disabled | Icon at 30% opacity, cursor `not-allowed` |
| Tooltip | Appears on hover after 300ms delay, Fira Sans 12px, dark background pill |
| Spacing | 4px gap between buttons |
| Separator | 1px vertical divider (`--airlock-border`) between quick actions group and adjacent toolbar elements, 8px horizontal margin |

### Quick Actions by Module and View

#### Contracts Module

**Triage View (dashboard):**

| Button | Icon | Tooltip | Action |
|--------|------|---------|--------|
| Filter | `Filter` | Filter contracts | Toggle filter panel (status, chamber, health range, assignee) |
| Sort | `ArrowUpDown` | Sort order | Dropdown: health ascending/descending, date, name, SLA urgency |
| Columns | `Columns` | Configure columns | Popover with checkbox list of visible table columns |
| Export | `Download` | Export current view | Export filtered triage data as CSV or PDF |

**Vault View (triptych active, viewing a specific contract vault):**

| Button | Icon | Tooltip | Action |
|--------|------|---------|--------|
| New Patch | `GitBranch` | Create patch | Triggers Action Focus state (view state 3) for this vault |
| Schedule Meeting | `Video` | Schedule meeting | Opens Meeting Scheduler with vault pre-linked |
| New Task | `CheckSquare` | Create task | Opens Quick Task Creation linked to this vault |
| Share | `Share2` | Share vault link | Copies deep link to clipboard (`/{module}/{vault-slug}`) with toast confirmation |

#### CRM Module

**Pipeline View:**

| Button | Icon | Tooltip | Action |
|--------|------|---------|--------|
| Filter | `Filter` | Filter pipeline | Toggle filter by stage, owner, value range, health |
| Add Deal | `PlusCircle` | Add deal | Opens Deal creation form |
| Schedule Meeting | `Video` | Schedule meeting | Opens Meeting Scheduler (pre-selects contact if one is focused) |
| Start Call | `Phone` | Start Google Meet | Opens Google Meet in new tab (requires `ENABLE_GOOGLE_MEET_INTEGRATION`) |

**Contact View (vault triptych for a level-3 counterparty vault):**

| Button | Icon | Tooltip | Action |
|--------|------|---------|--------|
| Edit | `Pencil` | Toggle edit mode | Toggles field editing on the contact record in Orchestrate panel |
| Schedule Meeting | `Video` | Schedule meeting | Opens Meeting Scheduler pre-populated with this contact |
| Send Message | `MessageSquare` | Send message | Opens Messenger dock panel with thread for this contact |
| Start Call | `Phone` | Start Google Meet | Opens Google Meet with contact invite |

#### Tasks Module

**Board View (Kanban) or Table View:**

| Button | Icon | Tooltip | Action |
|--------|------|---------|--------|
| Filter | `Filter` | Filter tasks | Toggle filter by status, assignee, priority, module source |
| New Task | `PlusCircle` | New task | Opens inline task creation at top of board/table |
| My Tasks | `User` | My assigned tasks | Toggles filter to only show tasks assigned to current user |
| Sort | `ArrowUpDown` | Sort tasks | Dropdown: priority, due date, status, created date |

#### Calendar Module

**Month / Week / Agenda View:**

| Button | Icon | Tooltip | Action |
|--------|------|---------|--------|
| New Event | `CalendarPlus` | New event | Opens Event Creation form at the currently selected date |
| Schedule Meeting | `Video` | Schedule meeting | Opens Meeting Scheduler with participant picker |
| Today | `Calendar` | Jump to today | Scrolls/navigates calendar to current date |
| View Toggle | `LayoutGrid` | Switch view | Cycles: Month > Week > Agenda > Month (or dropdown with all three) |

#### Documents Module

**Document List View:**

| Button | Icon | Tooltip | Action |
|--------|------|---------|--------|
| New Doc | `FilePlus` | New document | Opens blank TipTap editor in Orchestrate panel |
| Upload | `Upload` | Upload files | Opens multi-file upload zone |
| Filter | `Filter` | Filter documents | Toggle filter by type (PDF, DOCX, XLSX, etc.), vault, date |

**Document Viewer (viewing a specific document in Orchestrate):**

| Button | Icon | Tooltip | Action |
|--------|------|---------|--------|
| Download | `Download` | Download original | Downloads the original file |
| Annotations | `Highlighter` | Toggle annotations | Show/hide extraction annotations overlay |
| Full Screen | `Maximize2` | Full screen | Expands Orchestrate to Artifact Focus state (view state 2) |

### Quick Action Bar Rendering Rules

1. **Maximum 4 visible buttons** per context. If more than 4 actions are registered for a context, the 5th+ actions appear in an overflow menu triggered by a `MoreHorizontal` icon button.
2. **Priority ordering** -- each module's action registry assigns a priority (1-10) to each action. The top 4 by priority are rendered; the rest overflow.
3. **Disabled state** -- if an action is unavailable (e.g., "Start Call" when Google Meet integration is disabled), the button renders at 30% opacity with `cursor: not-allowed` and its tooltip appends "(unavailable)".
4. **Empty state** -- if no quick actions are registered for the current context (unlikely but possible), the quick action zone collapses and the spacer expands.

### Quick Action Overflow Menu

| Property | Value |
|----------|-------|
| Trigger | Click `MoreHorizontal` (three dots) icon button |
| Width | 240px |
| Background | `--airlock-card` (#151923) |
| Items | Same design as Create menu items (36px height, icon + label + optional shortcut) |
| Dismiss | Click outside, Escape, or selecting an item |

---

## Cmd+K Palette Enhancements

### New Slash Commands

These commands extend the existing set defined in the Search overview (`/upload-contract`, `/new-task`, `/new-patch`, `/goto`, `/settings`, `/admin`, `/theme`, `/export`).

| Command | Description | Opens | Required Permission |
|---------|-------------|-------|---------------------|
| `/schedule-meeting` | Schedule a new meeting | Meeting Scheduler modal | `manage_tasks` |
| `/new-event` | Create a calendar event | Event Creation form | `manage_tasks` |
| `/new-contact` | Create a CRM contact | Contact Creation form | `create_vaults` (CRM) |
| `/new-deal` | Create a pipeline deal | Deal Creation form | `create_vaults` (CRM) |
| `/new-vault` | Create vault in current module | Vault Creation form (module-specific) | `create_vaults` |
| `/new-account` | Create a CRM account | Account Creation form (parent vault) | `create_vaults` (CRM) |
| `/start-call` | Start instant Google Meet | Opens Meet in new tab | `manage_tasks` + `ENABLE_GOOGLE_MEET_INTEGRATION` |
| `/new-document` | Create a blank document | Opens TipTap editor in Orchestrate panel | `create_vaults` (Documents) |
| `/quick-note` | Create a timestamped note | Inline note composer | All roles (no special permission) |

### Command Parameter Support

Slash commands accept inline parameters for power-user workflows. Parameters are parsed from the text following the command name.

| Command + Parameters | Parsed Result |
|---------------------|---------------|
| `/schedule-meeting with @billy Tomorrow at 3pm` | Participants: [billy], Date: tomorrow, Time: 15:00 |
| `/new-task "Review contract" --priority=high --due=friday` | Title: "Review contract", Priority: high, Due: next Friday |
| `/new-contact Jane Doe jane@acme.com` | Name: Jane Doe, Email: jane@acme.com |
| `/start-call @dave @mike` | Opens Meet, invites: [dave, mike] |
| `/new-event "Team standup" --daily --time=9:00` | Title: "Team standup", Recurrence: daily, Time: 09:00 |
| `/new-vault "Henderson MSA" --type=distribution` | Name: "Henderson MSA", Type: Distribution |

**Parsing rules:**
- `@mentions` resolve against workspace member display names (fuzzy match)
- Quoted strings are treated as the primary value (title, name)
- `--key=value` flags map to form fields
- Natural language dates ("tomorrow", "next friday", "March 15") are parsed client-side via a date parsing library (e.g., chrono-node or similar)
- Unrecognized parameters are ignored (non-breaking)

### Contextual Command Suggestions

When the Cmd+K palette opens (before the user types anything), it surfaces contextual suggestions based on the current state:

| Context Signal | Suggested Commands |
|---------------|--------------------|
| **Active module is CRM** | `/new-contact`, `/new-deal`, `/schedule-meeting` surface first |
| **Active module is Contracts** | `/upload-contract`, `/new-patch`, `/new-vault` surface first |
| **Active module is Tasks** | `/new-task` surfaces first |
| **Active module is Calendar** | `/new-event`, `/schedule-meeting` surface first |
| **Active module is Documents** | `/new-document` surfaces first |
| **Viewing a specific vault** | `/new-patch {current vault}`, `/new-task` (linked to vault), `/schedule-meeting` (linked to vault) |
| **Recently used commands** | Last 5 unique commands appear in "Recent Actions" group at top |
| **Time of day: 8-10 AM** | `/schedule-meeting` gets a boost (morning meeting scheduling pattern) |
| **Time of day: 4-6 PM** | `/quick-note` gets a boost (end-of-day notes pattern) |

Time-of-day suggestions are a minor relevance boost, not a forced ordering. They influence sort position by +1 rank, not a complete reorder.

---

## Modal Forms

All create forms render as centered modal overlays. They share common design properties.

### Shared Modal Properties

| Property | Value |
|----------|-------|
| Width | 480px |
| Max height | 80vh (scrollable if content exceeds) |
| Background | `--airlock-card` (#151923) |
| Border | 1px solid `--airlock-border` (#1E2330) |
| Border radius | 12px |
| Shadow | `0 16px 64px rgba(0, 0, 0, 0.6)` |
| Backdrop | `rgba(0, 0, 0, 0.6)` with `backdrop-filter: blur(4px)` |
| Position | Centered horizontally and vertically |
| Animation | Backdrop fades in (150ms), modal scales from 0.95 to 1.0 + fades in (200ms ease-out) |
| Dismiss | Click backdrop, Escape key, or Cancel button |
| Header | Fira Sans 15px semibold, `--airlock-text`, 20px padding bottom |
| Footer | Right-aligned buttons, 16px padding top, separated by `1px solid --airlock-border` top border |

### Shared Form Field Properties

| Property | Value |
|----------|-------|
| Label | Fira Sans 12px 500, `--airlock-muted`, 4px margin below |
| Input | Fira Sans 13px 400, `--airlock-text`, `--airlock-surface` background, 1px `--airlock-border` border, 8px 12px padding, 6px border radius |
| Input focus | Border color `--airlock-cyan`, `0 0 0 2px rgba(0, 209, 255, 0.15)` ring |
| Error | Fira Sans 11px 400, `--gate-red` (#EF4444), 4px margin above, input border turns `--gate-red` |
| Field spacing | 16px vertical gap between fields |
| Required marker | Red asterisk after label text |

### Meeting Scheduler Modal

**Header:** "Schedule Meeting"

| Field | Type | Details |
|-------|------|---------|
| Title * | Text input | Placeholder: "Meeting title" |
| Date * | Date picker | Default: today. Calendar dropdown. |
| Start Time * | Time picker | Default: next 30-min increment. 15-minute intervals. |
| End Time * | Time picker | Default: start + 30 minutes. |
| Participants * | Multi-select with search | Search workspace members + CRM contacts (level 3 vaults). Renders as pills. |
| Location | Toggle + text | Toggle: "Google Meet" (auto-generates link when enabled). Or free text for physical location. Requires `ENABLE_GOOGLE_MEET_INTEGRATION` for Meet toggle. |
| Vault Link | Auto-populated read-only | If launched from a vault view, shows vault name as a linked pill. Editable (clear or change). |
| Description | Textarea | Placeholder: "Agenda or notes". 4 rows default. Auto-grows to 8. |
| Prep Brief | Toggle | "Generate prep brief before meeting" -- when enabled, Otto generates a summary of the linked vault 15 minutes before meeting time. Requires `chat_with_otto` permission. |
| Recurrence | Select | Options: None (default), Daily, Weekly, Bi-weekly, Monthly |
| **Buttons** | | **Schedule** (primary, `--airlock-cyan` background) and **Cancel** (ghost) |

### Event Creation Form

**Header:** "New Event"

| Field | Type | Details |
|-------|------|---------|
| Title * | Text input | Placeholder: "Event title" |
| All Day | Toggle | Default: off. When on, hides time pickers. |
| Start Date/Time * | Date + time picker | Default: today, current time rounded to next 15 min. |
| End Date/Time * | Date + time picker | Default: start + 1 hour. |
| Module Tag | Select | Which module to color-tag this event. Options: Contracts, CRM, Tasks, Documents, None. Uses module accent colors. |
| Vault Link | Autocomplete search | Optional. Search vaults to link this event to a specific vault. |
| Description | Textarea | 4 rows, auto-grows. |
| Reminder | Select | Options: None, 15 minutes before, 1 hour before, 1 day before. Default: 15 minutes. |
| **Buttons** | | **Create Event** (primary) and **Cancel** (ghost) |

### Contact Creation Form

**Header:** "New Contact"

| Field | Type | Details |
|-------|------|---------|
| First Name * | Text input | |
| Last Name * | Text input | |
| Email | Text input | Validates email format on blur. |
| Phone | Text input | Accepts any format, normalized on save. |
| Company / Organization | Autocomplete | Searches existing parent vaults (level 1). "Create new" option at bottom if no match. |
| Role / Title | Text input | Placeholder: "e.g., VP of Sales" |
| Parent Vault | Autocomplete | Searches parent or division vaults (level 1-2). Auto-populated if Company is selected. |
| Tags | Multi-select with create | Type to search existing tags or create new. Rendered as colored pills. |
| Notes | Textarea | 3 rows, auto-grows. |
| **Buttons** | | **Create Contact** (primary) and **Cancel** (ghost) |

**Implementation note:** Creating a contact creates a level-3 counterparty vault in the vault hierarchy. The CRM module is a lens over the vault tree -- no separate CRM tables.

### Quick Task Creation

**Header:** "New Task"

| Field | Type | Details |
|-------|------|---------|
| Title * | Text input | Single line. Placeholder: "What needs to be done?" |
| Assignee | Single-select with search | Search workspace members. Default: current user. |
| Priority | Button group | Low / Medium / High / Critical. Default: Medium. Each option has a colored indicator (gate colors: green / amber / yellow / red). |
| Due Date | Date picker | Optional. Calendar dropdown. |
| Module Link | Read-only pill | Auto-populated from current module context. Editable (clear or change). |
| Vault Link | Read-only pill | Auto-populated if launched from a vault view. Editable (clear or change). |
| Description | Textarea (collapsible) | Hidden by default behind "Add description" link. 3 rows when expanded. |
| **Buttons** | | **Create Task** (primary) and **Cancel** (ghost) |

**Quick creation shortcut:** Pressing `Enter` in the Title field (without Shift) submits the form with defaults. `Shift+Enter` opens the Description field.

### Vault Creation Form

**Header:** "New Vault"

| Field | Type | Details |
|-------|------|---------|
| Vault Name * | Text input | Placeholder varies by module context (e.g., "Contract name" in Contracts, "Account name" in CRM). |
| Module | Select | Pre-selected to current module. Options: Contracts, CRM, Tasks, Documents. |
| Vault Type | Select | Module-specific options. Contracts: 24 contract types. CRM: Account / Contact / Deal. |
| Parent Vault | Autocomplete | Optional. Search existing vaults to nest under. |
| Chamber | Read-only | Auto-set to "Discover" for new vaults. |
| Description | Textarea | Optional, 3 rows. |
| **Buttons** | | **Create Vault** (primary) and **Cancel** (ghost) |

---

## Keyboard Shortcuts (New)

These shortcuts are added to the Shell-level keyboard shortcut table (documented in Shell overview).

| Shortcut | Action | Scope |
|----------|--------|-------|
| `Cmd+N` | Open Create menu dropdown | Global (any module) |
| `Cmd+Shift+M` | Open Meeting Scheduler modal | Global |
| `Cmd+Shift+E` | Open Event Creation form | Global |
| `Cmd+Shift+T` | Open Quick Task Creation form | Global |
| `Cmd+Shift+C` | Open Contact Creation form | CRM module only (no-op elsewhere) |
| `Cmd+Shift+D` | Open Deal Creation form | CRM module only |
| `Cmd+Shift+U` | Open file upload zone | Contracts and Documents modules |

**Conflict avoidance:** These shortcuts avoid conflicts with existing Shell shortcuts (`Cmd+1-4` for triptych states, `Cmd+K` for search, `Cmd+/` for Otto, `Cmd+B` for sub-panel toggle). All new shortcuts use `Cmd+Shift+letter` to occupy a separate modifier namespace.

**Shortcut discoverability:** Shortcuts are displayed in the Create menu (right-aligned, muted monospace text) and in the Cmd+K palette next to matching commands.

---

## Action Registry Pattern

### Architecture

Quick actions and Create menu items are managed through a module action registry. Each module registers its available actions during initialization, and the toolbar reads from the registry based on the current context.

```
ActionRegistry
  |
  +-- contracts.actions.ts .... Contracts module action definitions
  +-- crm.actions.ts .......... CRM module action definitions
  +-- tasks.actions.ts ........ Tasks module action definitions
  +-- calendar.actions.ts ..... Calendar module action definitions
  +-- documents.actions.ts .... Documents module action definitions
  +-- universal.actions.ts .... Universal actions (available everywhere)
```

### Action Definition Schema

```typescript
interface ToolbarAction {
  id: string;                          // Unique identifier: "contracts.new-patch"
  module: string;                      // Source module: "contracts", "crm", "universal"
  label: string;                       // Display label: "New Patch"
  icon: string;                        // Lucide icon name: "GitBranch"
  shortcut?: string;                   // Keyboard shortcut: "Cmd+Shift+T"
  slashCommand?: string;              // Cmd+K command: "/new-patch"
  type: "create" | "quick" | "both";   // Where it appears
  contexts: ActionContext[];           // When this action is visible
  permission?: string;                 // Required permission: "create_patches"
  featureFlag?: string;               // Required feature flag: "ENABLE_MEETING_SCHEDULER"
  priority: number;                    // Sort order in quick action bar (1 = highest)
  handler: () => void;                 // Click handler
}

interface ActionContext {
  module?: string;                     // Active module filter
  view?: string;                       // Active view filter: "triage", "pipeline", "vault"
  vaultLevel?: number;                 // Active vault level filter (1-4)
  chamber?: string;                    // Active chamber filter
}
```

### Registration Example

```typescript
// contracts.actions.ts
export const contractsActions: ToolbarAction[] = [
  {
    id: "contracts.new-vault",
    module: "contracts",
    label: "New Vault",
    icon: "Plus",
    type: "create",
    contexts: [{ module: "contracts" }],
    permission: "create_vaults",
    priority: 1,
    handler: () => openModal("vault-creation", { module: "contracts" }),
  },
  {
    id: "contracts.new-patch",
    module: "contracts",
    label: "New Patch",
    icon: "GitBranch",
    type: "both",  // appears in both Create menu AND quick actions
    contexts: [{ module: "contracts", view: "vault" }],
    permission: "create_patches",
    priority: 1,
    handler: () => triggerViewState("action-focus", { type: "patch" }),
  },
  // ...
];
```

### Permission Gating

The toolbar respects Airlock's role-based permission model at every level:

| Check | Behavior |
|-------|----------|
| User lacks `create_vaults` permission | "New Vault" is hidden from Create menu and quick actions |
| User lacks `create_patches` permission | "New Patch" is hidden |
| User lacks `manage_tasks` permission | "New Task" is hidden |
| Feature flag `ENABLE_GOOGLE_MEET_INTEGRATION` is off | "Start Call" and Google Meet toggle in Meeting Scheduler are hidden |
| Feature flag `ENABLE_MEETING_SCHEDULER` is off | "Schedule Meeting" is hidden from all surfaces |

Actions are **hidden, not disabled**, following the Cmd+K precedent from the Search spec: "Actions are hidden, not disabled. If a Builder types `/admin`, nothing appears."

**Exception:** If an action is visible but temporarily unavailable (e.g., "Export" while data is loading), it renders in the disabled state (30% opacity) rather than being hidden.

---

## Event Bus Integration

All create actions emit events to the cross-module event bus (BullMQ) for downstream processing.

| Action | Event Emitted | Consumers |
|--------|--------------|-----------|
| Create Vault | `vault.created` | CRM (if vault level 1-3), Notifications, Search indexing |
| Create Task | `task.created` | Tasks module, Notifications, Search indexing |
| Create Contact | `vault.created` (level 3) | CRM enrichment pipeline, Search indexing |
| Create Deal | `vault.created` (level 4) | CRM pipeline, Notifications, Search indexing |
| Schedule Meeting | `meeting.scheduled` | Calendar, Notifications, Otto prep brief (if enabled) |
| Create Event | `event.created` | Calendar, Notifications |
| Upload Document | `document.uploaded` | Extraction pipeline, Search indexing |
| Start Call | `call.started` | CRM activity log, Notifications |

### Optimistic UI

All create operations use optimistic updates:

1. User submits form.
2. Item appears immediately in the UI with a temporary ID and subtle loading indicator (pulsing `--airlock-cyan` left border on the new item).
3. API request fires asynchronously.
4. On success: temporary ID replaced with server ID, loading indicator removed, toast confirmation appears.
5. On failure: item shows error state (red left border), toast with retry option, item removed after 10 seconds if not retried.

---

## Feature Control Plane Flags

| Flag | Default | Description |
|------|---------|-------------|
| `toolbar.quick_actions.enabled` | `true` | Show/hide quick action buttons in toolbar |
| `toolbar.create_menu.enabled` | `true` | Show/hide the `+ Create` button and dropdown |
| `toolbar.meeting_scheduler.enabled` | `false` | Enable meeting scheduling (depends on `integrations.google_meet.enabled`) |
| `toolbar.quick_task_create.enabled` | `true` | Enable quick task creation from toolbar |
| `toolbar.recent_creates.enabled` | `true` | Show recent creates section in Create menu |
| `toolbar.contextual_suggestions.enabled` | `true` | Enable time-of-day and module-aware command suggestions in Cmd+K |
| `integrations.google_meet.enabled` | `false` | Master toggle for Google Meet integration (affects Start Call and Meet toggle in scheduler) |

---

## Accessibility

| Requirement | Implementation |
|-------------|----------------|
| Keyboard navigation | All toolbar elements are focusable via Tab. Arrow keys navigate within the Create dropdown and overflow menus. |
| Focus trap | When a modal form is open, focus is trapped within the modal. Tab cycles through form fields. Shift+Tab reverses. |
| Screen reader labels | Create button: `aria-label="Create new item"`, `aria-haspopup="menu"`, `aria-expanded`. Quick actions: `aria-label` matches tooltip text. |
| ARIA roles | Create dropdown: `role="menu"`. Menu items: `role="menuitem"`. Quick action buttons: `role="button"`. |
| Color contrast | All text meets WCAG AA minimum contrast (4.5:1 for normal text, 3:1 for large text) against `--airlock-surface`. |
| Reduced motion | Users with `prefers-reduced-motion` see instant state changes instead of animated transitions. |
| Focus visible | All interactive elements show a visible focus ring (`2px solid --airlock-cyan`, `2px offset`) when focused via keyboard. |

---

## Related Specs

- **[Shell Overview](./overview.md)** -- Toolbar position within the three-zone layout, existing toolbar elements, right-side dock tool definitions
- **[View States](./view-states.md)** -- Action Focus state (state 3) triggered by "New Patch" and other creation actions
- **[Search / Cmd+K](../Search/overview.md)** -- Existing slash commands, search palette layout, role-based action filtering, MeiliSearch integration
- **[Roles & Permissions](../Roles/overview.md)** -- Permission catalog (`create_vaults`, `create_patches`, `manage_tasks`, etc.), role hierarchy, ad-hoc overrides
- **[CRM Overview](../CRM/overview.md)** -- Contact and deal creation (vault hierarchy levels 1-4), CRM as lens over vault tree
- **[Tasks Overview](../Tasks/overview.md)** -- Task creation, Kanban/table views, universal task system
- **[Calendar Overview](../Calendar/overview.md)** -- Event creation, Schedule-X integration, computed calendar from vault dates
- **[Documents Overview](../DocumentSuite/overview.md)** -- TipTap editor, document upload, multi-format support
- **[Feature Control Plane](../FeatureControlPlane/overview.md)** -- Feature flag schema, toggle/calibrate/audit/failure-flag pattern
- **[Theme](./theme.md)** -- Design tokens (`--airlock-*`), typography (Fira Sans / Fira Code), icon set (Lucide), effects
- **[Messenger](../Messenger/overview.md)** -- Chat dock tool, "Send Message" action integration
