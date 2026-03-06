# Shell / Master Layout

## Summary

Airlock uses a Discord-inspired three-zone master layout that persists across the entire application. Users never navigate away from this shell; all content loads inline within the main content area.

## Layout Zones

| Zone | Width | Position | Purpose |
|------|-------|----------|---------|
| Module Bar | 56-72px fixed | Far left | Module switching (like Discord server icons) |
| Sub-Panel | 240px fixed | Left of content | Chamber-organized navigation within the active module |
| Main Content | Remaining width (flex) | Main area | Dashboard views OR vault triptych (Signal / Orchestrate / Control) |

## Structural Diagram

```
+--------+-----------+------------------------------------------+
| Module | Sub-      |           Main Content                   |
|  Bar   | Panel     |  Signal | Orchestrate | Control          |
| 56-72px| 240px     |  ~280px |   flex-1    | ~300px           |
|        |           |         |             |                  |
|  [A]   | Chambers  | (only when viewing a vault)              |
|  [C]   | Views     |                                          |
|  [R]   | Vaults    | (dashboard views use full width)         |
|  [T]   |           |                                          |
+--------+-----------+---------+-------------+------------------+
```

## Core Concepts

> **Canonical vocabulary:** See [Naming & Hierarchy](./naming-and-hierarchy.md) for the definitive term list.

### Modules

Modules are the top-level organizational units, equivalent to Discord "servers." Each module represents a major domain within the application.

| Module | Icon | Description |
|--------|------|-------------|
| Contracts | FileText | Contract lifecycle management |
| CRM | Users | Customer and entity management |
| Tasks | CheckSquare | Task tracking and assignment |
| Calendar | Calendar | Scheduling and deadline views |
| Documents | FolderOpen | Document library and templates |

**Admin is NOT a module** — it's a system overlay accessed from the user avatar. See [Naming & Hierarchy](./naming-and-hierarchy.md#admin--settings).

### Vaults

A vault is Airlock's fundamental unit of work — the interactive, living record of how something moves from Discover to Ship. It wraps the source material + annotation layer + metadata + audit trail + collaboration context into one secure container.

When a vault is selected, the main content area renders the **triptych** (Signal | Orchestrate | Control). Each person sees a role-appropriate view based on the vault's current gate.

See [Universal Chambers](./universal-chambers.md) for the full architecture.

### Module Sub-Panel

Each module's sub-panel is organized by **chambers** (Discover > Build > Review > Ship), with views inside each chamber section. A mini-dashboard at the top shows vault counts per chamber. Active Vaults are listed at the bottom.

### Navigation Flow

1. User clicks a module icon in the Module Bar (or the Airlock icon for Home)
2. The Sub-Panel shows that module's chamber-organized views + Active Vaults
3. Module loads the **last-visited view** in the main content area
4. User clicks a chamber view (e.g., "Triage Board") for a dashboard, or a vault for the triptych
5. The main content loads context-aware content based on the selection, the user's role, and the vault's current gate

## Principles

- **Everything inline.** No page navigation. No route changes that leave the shell. Content loads within the main area.
- **Context always visible.** The Module Bar and Sub-Panel remain visible in all states (though triptych panels may resize).
- **Keyboard-first power users.** Cmd+1/2/3/4 for triptych states. Arrow keys for vault navigation. Escape to return to Overview state.
- **Automatic view adaptation.** The triptych adjusts its panel layout based on context (see view-states.md), not manual toggles.
- **Last-visited persistence.** Switching back to a module returns you to whatever you were last looking at.

## Right-Side Tool Dock

The shell includes a right-side tool dock -- a vertical icon bar mirroring the Module Bar but providing **contextual lenses on other modules' data**, scoped to the current context.

```
LEFT = WHERE you are (Module Bar -- Contracts, CRM, Tasks)
RIGHT = WHAT you need (Tool Dock -- other modules' data, scoped to current context)
```

| Dock Tool | Icon | Panel Behavior | Content |
|-----------|------|---------------|---------|
| Tasks | CheckSquare | Push panel (320px, shrinks content) | To-do list scoped to active module/vault |
| CRM | Users | Push panel (320px) | Related entities for current context |
| Calendar | Calendar | Push panel (320px) | Upcoming dates for current context |
| Documents | File | Push panel (320px) | Attached documents for current context |
| Chat | MessageSquare | Slide-over drawer (340px, overlays content) | Messenger conversations |
| Otto AI | Bot | Slide-over drawer (340px, overlays content) | AI assistant chat |

Every dock panel has a **scope toggle** (`[Module Name] | All`) that switches between module-scoped and global views.

See [Messenger](../Messenger/overview.md) for Chat dock details.

---

## Home

Home is the workspace landing page -- a Notion-style customizable page with module triage widgets and status blocks. It's NOT a module; it's the default view when no module is active (click the Airlock icon in the Module Bar).

### Default Home Blocks

| Block | Content |
|-------|---------|
| **Inbox Feed** | Combined activity feed: urgent notifications, mentions, assigned items |
| **Module Triage Widgets** | One table widget per module showing open issues (backed by universal task system) |
| **Module Status Cards** | Chamber distribution chart + health metrics per module |

Home blocks are user-customizable (drag to reorder, hide/show) with workspace-level defaults.

---

## How the Shell Specs Fit Together

This folder contains 8 companion specs that define every aspect of the shell. Here's how they connect:

```
overview.md (this file)
    |
    +-- naming-and-hierarchy.md .... Canonical vocabulary (MUST READ FIRST)
    |       Defines: Vault, Module, Chamber, Gate, View, Triptych
    |       Route pattern: /contracts/henderson-msa/record-inspector
    |
    +-- universal-chambers.md ...... 4-stage lifecycle (MUST READ SECOND)
    |       Discover > Build > Review > Ship
    |       Each chamber maps to roles: Builder, Gatekeeper, Owner
    |       Chamber determines what views/actions are available
    |
    +-- module-bar.md .............. Left icon strip
    |       72px fixed width, Airlock home icon at top
    |       5 module icons + divider + user avatar + settings gear
    |       Notification badges per module
    |
    +-- channel-sidebar.md ......... Sub-panel organization
    |       240px fixed width, chamber-organized navigation
    |       Mini-dashboard, chamber groups, Active Vaults list
    |       Module-specific sidebar content
    |
    +-- triptych.md ................ Three-panel workspace
    |       Signal (280px) | Orchestrate (flex-1) | Control (300px)
    |       Resizable via drag handles, min widths enforced
    |       Panel collapse to 80px icon-only mode
    |
    +-- view-states.md ............. Triptych modes
    |       Overview (Cmd+1), Inspect (Cmd+2), Edit (Cmd+3), Approve (Cmd+4)
    |       Auto-switching based on user actions
    |       Escape returns to Overview
    |
    +-- theme.md ................... Design tokens
    |       OLED dark palette, Fira Sans + Fira Code typography
    |       Chamber colors, accent colors, spacing scale
    |       CSS custom properties: --surface-*, --chamber-*, --accent-*
    |
    +-- toolbar-actions.md ........ Contextual toolbar actions
            Universal Create button, context-sensitive quick actions
            Expanded Cmd+K slash commands, modal creation forms
            Action registry pattern, permission gating
```

### Reading Order for New Developers

1. **This file** (overview.md) -- understand the 3-zone layout and principles
2. **naming-and-hierarchy.md** -- learn the canonical vocabulary
3. **universal-chambers.md** -- understand the 4-chamber lifecycle
4. **triptych.md** + **view-states.md** -- understand the workspace panels and modes
5. **module-bar.md** + **channel-sidebar.md** -- understand navigation
6. **theme.md** -- understand design tokens
7. **toolbar-actions.md** -- understand contextual toolbar actions and Create menu

---

## Responsive Behavior (v1)

Airlock is designed for desktop viewports (1280px+). v1 does not include responsive/mobile layouts.

| Viewport Width | Behavior |
|----------------|----------|
| >= 1280px | Full layout: Module Bar + Sub-Panel + Main Content + optional Dock Panel |
| 1024-1279px | Sub-Panel collapses to icon-only mode (56px). Expandable on hover. |
| < 1024px | Not supported in v1. Show a "Desktop required" message. |

---

## Keyboard Shortcuts (Shell-Level)

| Shortcut | Action |
|----------|--------|
| `Cmd+1` | Switch to Overview triptych state |
| `Cmd+2` | Switch to Inspect (Artifact Focus) state |
| `Cmd+3` | Switch to Edit (Action Focus) state |
| `Cmd+4` | Switch to Approve (Gate Lock) state |
| `Escape` | Return to Overview state / close dock panel |
| `Cmd+K` | Open universal search / command palette |
| `Cmd+/` | Toggle Otto AI dock panel |
| `Cmd+B` | Toggle Sub-Panel visibility |
| `Arrow Up/Down` | Navigate vault list in Sub-Panel |
| `Enter` | Open selected vault |

---

## Related Specs

- **[Naming & Hierarchy](./naming-and-hierarchy.md)** -- FOUNDATIONAL: Canonical vocabulary, route tree, navigation hierarchy
- **[Universal Chambers](./universal-chambers.md)** -- FOUNDATIONAL: 4-chamber lifecycle as UI navigation structure
- [Module Bar](./module-bar.md) -- Far-left icon strip
- [Channel Sidebar](./channel-sidebar.md) -- Sub-panel organization within active module
- [Triptych](./triptych.md) -- Three-panel workspace (Signal / Orchestrate / Control)
- [View States](./view-states.md) -- Triptych states: Overview / Inspect / Edit / Approve
- [Theme](./theme.md) -- Color tokens, typography, effects
- [Roles](../Roles/overview.md) -- Multi-tier permission model (role determines chamber view)
- [Messenger](../Messenger/overview.md) -- Chat dock tool spec
- [Toolbar Actions](./toolbar-actions.md) -- Contextual quick actions, universal Create button, expanded slash commands
- [Search](../Search/overview.md) -- Cmd+K command palette
