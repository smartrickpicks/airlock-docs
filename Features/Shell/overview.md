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

## Related Specs

- **[Naming & Hierarchy](./naming-and-hierarchy.md)** -- FOUNDATIONAL: Canonical vocabulary, route tree, navigation hierarchy
- **[Universal Chambers](./universal-chambers.md)** -- FOUNDATIONAL: 4-chamber lifecycle as UI navigation structure
- [Module Bar](./module-bar.md) -- Far-left icon strip
- [Triptych](./triptych.md) -- Three-panel workspace (Signal / Orchestrate / Control)
- [View States](./view-states.md) -- Triptych states: Overview / Inspect / Edit / Approve
- [Theme](./theme.md) -- Color tokens, typography, effects
- [Roles](../Roles/overview.md) -- Multi-tier permission model (role determines chamber view)
