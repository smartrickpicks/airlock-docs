# Channel Sidebar

## Summary

The Channel Sidebar is the 240px panel between the Module Bar and the Triptych. Its content changes entirely when the user switches modules. It provides navigation to individual records, views, and tools within the active module. The Contracts module is the most complex and serves as the reference implementation.

## Dimensions

| Property | Value |
|----------|-------|
| Width | 240px fixed |
| Height | 100vh |
| Position | Right of Module Bar |
| Background | `--airlock-surface` (#0F1219) |
| Border | 1px right border, `--airlock-border` (#1E2330) |

## Anatomy (Top to Bottom)

1. **Module Header** -- Module name + action buttons
2. **Search Bar** -- Filter channels
3. **Pinned Channels** -- Always-visible priority items
4. **Channel Groups** -- Collapsible groups of channel items
5. **Grouping Controls** -- Dropdown to change grouping mode

---

## Module Header

| Property | Value |
|----------|-------|
| Height | 48px |
| Padding | 0 16px |
| Font | Fira Sans, 15px semibold, `--airlock-text` |
| Background | `--airlock-surface` (#0F1219) |
| Border | 1px bottom border, `--airlock-border` |

Content: Module name on the left. Action icon(s) on the right (e.g., Plus icon to create new channel item).

---

## Search Bar

Inline filter input below the module header.

| Property | Value |
|----------|-------|
| Margin | 8px 12px |
| Height | 32px |
| Background | `--airlock-bg` (#0B0E14) |
| Border | 1px solid `--airlock-border`, focus: `--airlock-cyan` |
| Border radius | 6px |
| Font | Fira Sans, 13px, placeholder in `--airlock-muted` |
| Icon | Lucide Search, 14px, left-aligned inside input |

Filter capabilities:
- Name / title text match
- Entity / account name
- Contract type or category
- Status (gate color)
- Date range (via advanced filter popover)

---

## Pinned Channels

Pinned channels appear above the scrollable channel list. They are persistent, always visible, and cannot be reordered by grouping changes.

### Contracts Module Pinned Channels

| Channel | Icon | Badge |
|---------|------|-------|
| Triage Dashboard | Lucide AlertTriangle | Red badge: open issue count |
| Generator | Lucide PlusCircle | None |

### Pinned Channel Item Style

| Property | Value |
|----------|-------|
| Height | 36px |
| Padding | 0 12px |
| Font | Fira Sans, 13px semibold, `--airlock-text` |
| Background (hover) | `--airlock-card` (#151923) |
| Background (active) | `--airlock-card` with left 2px border in `--airlock-cyan` |
| Icon | 16px, `--airlock-muted`, active: `--airlock-cyan` |

Separator: 1px line in `--airlock-border` below pinned section, 8px margin below.

---

## Channel Groups

Channels are organized into collapsible groups. The grouping mode determines how channels are categorized.

### Group Header

| Property | Value |
|----------|-------|
| Height | 28px |
| Padding | 0 12px |
| Font | Fira Sans, 11px uppercase, letter-spacing 0.5px, `--airlock-muted` |
| Chevron | Lucide ChevronDown (12px), rotates -90deg when collapsed |
| Cursor | Pointer |

Click to collapse/expand. Collapsed state hides all channel items in the group.

### Channel Item Anatomy (Contracts Module)

Each channel item represents a single contract. This is the most detailed channel layout.

```
[Gate Dot] Contract Name [Health %]
           Entity Name - Contract Type [Unread Badge]
```

| Element | Details |
|---------|---------|
| Gate Dot | 8px circle, filled with gate color (red/yellow/purple/green). Left-aligned. |
| Contract Name | Fira Sans, 13px, `--airlock-text`. Truncate with ellipsis at container edge. |
| Health % | Fira Code, 11px. Color: green (80-100%), yellow (50-79%), red (0-49%). Right-aligned on line 1. |
| Entity Name | Fira Sans, 12px, `--airlock-muted`. Second line. |
| Contract Type | Fira Sans, 12px, `--airlock-muted`. After entity, dash-separated. |
| Unread Badge | 16px red circle, white text 10px. Right-aligned on line 2. Only visible when count > 0. |

### Channel Item Dimensions

| Property | Value |
|----------|-------|
| Height | 44px (two-line layout) |
| Padding | 6px 12px |
| Background (default) | Transparent |
| Background (hover) | `--airlock-card` (#151923) |
| Background (active) | `--airlock-card` with 2px left border in `--airlock-cyan` |
| Border radius | 4px (inset from sidebar edges) |
| Margin | 1px 8px |

### Gate Dot Colors

| Gate State | Color | Token |
|------------|-------|-------|
| Blocked / Critical | Red | #EF4444 |
| Warning / Needs Attention | Yellow | #EAB308 |
| In Review / Pending | Purple | #A855F7 |
| Clear / Approved | Green | #22C55E |

---

## Grouping Modes

**Decision:** Default grouping is **By Entity/Account.**

A dropdown control at the top of the channel list (or in the module header overflow menu) lets users switch grouping.

| Grouping Mode | Group Headers | Description |
|---------------|---------------|-------------|
| By Entity (default) | Entity/account name | All contracts for a given customer grouped together |
| By Batch | Batch ID or batch name | Contracts grouped by processing batch |
| By Status | Gate color label (Blocked, Warning, Review, Clear) | Contracts grouped by current gate state |
| By Lifecycle State | Lifecycle stage (Draft, Active, Expiring, Terminated) | Contracts grouped by lifecycle position |

Switching grouping mode re-sorts and re-groups the channel list with a 200ms crossfade transition. Active channel remains highlighted and scrolled into view.

---

## Right-Click Context Menu

Right-clicking a channel item opens a context menu.

| Action | Description |
|--------|-------------|
| Open in New Tab | Opens the channel's triptych content in a browser tab (deep link) |
| Pin to Top | Moves the channel to the pinned section |
| Mark as Read | Clears unread badge for this channel |
| Copy Link | Copies a shareable deep link to clipboard |
| View in Triage | Opens the Triage Dashboard filtered to this record |
| Archive | Moves channel to archived state (with confirmation) |

### Context Menu Style

| Property | Value |
|----------|-------|
| Background | `--airlock-card` (#151923) |
| Border | 1px solid `--airlock-border` |
| Border radius | 6px |
| Shadow | 0 8px 24px rgba(0,0,0,0.5) |
| Item height | 32px |
| Item font | Fira Sans, 13px, `--airlock-text` |
| Item hover | Background `--airlock-surface` |
| Separator | 1px line in `--airlock-border` between action groups |

---

## Other Module Sidebar Layouts

While Contracts is the reference implementation, other modules use simplified channel items.

| Module | Channel Item Layout |
|--------|-------------------|
| CRM | [Avatar] Entity Name / Role - Status |
| Tasks | [Priority dot] Task Title / Assignee - Due Date |
| Calendar | (No traditional channel list; shows date navigation + event list) |
| Documents | [FileType icon] Document Name / Folder Path - Modified Date |
| Admin | [Section icon] Settings Category / Description |

---

## Scroll Behavior

- Channel list scrolls independently of the Module Header, Search, and Pinned sections (which remain fixed)
- Scroll area: from below pinned section to bottom of sidebar
- Scrollbar: Thin (6px), `--airlock-border` track, `--airlock-muted` thumb, only visible on hover/scroll
- Overscroll: Elastic bounce (native OS behavior)
