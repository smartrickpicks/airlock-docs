# Component Library -- Extraction Checklist

> **Status:** SPECCED
> **Source:** Airlock app audit (M1-M13 components), Shell overview.md, theme.md design token system

## Summary

Airlock's five modules (Contracts, CRM, Tasks, Calendar, Documents) collectively contain 33 organism components, 14 atoms, and 14 molecules -- but zero shared abstractions. Every module has its own button styles, table patterns, filter bars, empty states, and modal implementations. This spec defines the M14 extraction plan: identify duplicate patterns, prioritize extraction order, establish component API conventions, and lay out the migration strategy to move from per-module duplication to a shared `/ui/` component layer.

---

## 1. Duplicate Pattern Inventory

Components that exist across multiple modules but are not shared:

| Pattern | Found In | Instances | Priority |
|---------|----------|-----------|----------|
| **Table with sort/filter** | AccountsTable, LeadsTable, DocumentsTable, TasksTable, MembersTable | 5 | P0 |
| **Filter bar / quick filters** | FilterBar (contracts), FilterPills (CRM), inline filters (tasks, docs) | 4 | P0 |
| **Modal / dialog** | CreateVaultModal, admin settings modals, contract generator confirmation | 2+ | P0 |
| **Button variants** | Button atom exists but modules use raw `<button>` with inline styles or ad-hoc Tailwind | 5+ | P0 |
| **Empty state** | EmptyState atom exists but is not used consistently; modules render their own | 3+ | P1 |
| **Card layout** | EntityCard, DealCard, TaskCard, vault buttons in triage dashboard | 4 | P1 |
| **Side panel / drawer** | OttoDrawer, NotificationCenter, DocumentPreview | 3 | P1 |
| **Toast / snackbar** | Toast atom exists; notification store has toast queue; CRM has inline toasts | 1 | P1 |
| **Dropdown / select** | Inline selects in contract generator, admin settings, CRM pipeline stage picker | 3+ | P1 |
| **Badge / status indicator** | Badge, StatusDot, GateDot, ConfidenceBadge, PatchStateBadge | 5 | P2 |
| **Progress bar** | ProgressBar atom, health score bar in triage dashboard | 2 | P2 |
| **Search input** | SearchInput atom in SubPanel, CommandPalette input | 2 | P2 |

---

## 2. Extraction Priority

### P0 -- Extract First (used everywhere, highest duplication cost)

1. **`DataTable`** -- Unified sortable, filterable, paginated table component. Replaces AccountsTable, LeadsTable, DocumentsTable, TasksTable, and MembersTable. Supports column definitions, row selection, bulk actions, sort state, and server-side pagination. Every module with a list view depends on this.

2. **`FilterBar`** -- Composable filter pill system with saved presets. Replaces FilterBar (contracts), FilterPills (CRM), and inline filter implementations in tasks and documents. Supports text search, select filters, date range filters, and a "save as preset" action.

3. **`Modal`** -- Shared modal dialog with size variants (sm/md/lg/full), close-on-escape, focus trap, scroll lock, and overlay click dismiss. Replaces CreateVaultModal and ad-hoc modals in admin and contract generator.

4. **`Button`** -- Enforce all button usage through the existing Button atom. Currently modules bypass it with raw `<button>` elements and inline Tailwind classes. The atom needs variant expansion (primary, secondary, ghost, danger, icon-only) and consistent size options (sm/md/lg).

### P1 -- Extract Second (high value, moderate duplication)

5. **`Card`** -- Flexible card component with composable sub-components (Header, Body, Footer, StatusIndicator). Replaces EntityCard, DealCard, TaskCard, and vault triage cards. Supports left-border color coding per the theme spec.

6. **`Drawer`** -- Right-slide panel for contextual content. Replaces OttoDrawer, NotificationCenter, and DocumentPreview. Two modes: push (shrinks main content, 320px) and overlay (slides over content, 340px), matching the Tool Dock behavior defined in the Shell overview.

7. **`EmptyState`** -- Consistent empty state with Lucide icon, heading, description, and optional CTA button. Replaces ad-hoc empty renders scattered across module list views.

8. **`Toast`** -- Toast container with auto-dismiss (configurable duration), vertical stacking, action buttons, and severity variants (info, success, warning, error). Wraps the existing notification store toast queue.

### P2 -- Extract Third (nice to have, lower duplication)

9. **`Dropdown`** -- Select and combobox with search, single-select and multi-select variants. Replaces inline `<select>` elements in contract generator, admin, and CRM.

10. **`StatusIndicator`** -- Unify Badge, StatusDot, GateDot, ConfidenceBadge, and PatchStateBadge into a composable system. Supports dot, pill, and badge shapes with gate color semantics (`--gate-red`, `--gate-yellow`, `--gate-purple`, `--gate-green`).

11. **`ProgressBar`** -- Linear and circular variants with optional label. Supports gate-colored thresholds (green > 50%, amber 25-50%, red < 25%) matching the SLA countdown color logic in theme.md.

---

## 3. Component API Design Guidelines

### Props over configuration objects

```tsx
// Correct
<Button variant="primary" size="md" disabled>Save</Button>

// Avoid
<Button config={{ variant: "primary", size: "md", disabled: true }}>Save</Button>
```

### Composition over monolithic props

```tsx
// Correct -- composable sub-components
<Card>
  <Card.Header title="Henderson MSA" badge={<StatusIndicator gate="green" />} />
  <Card.Body>{content}</Card.Body>
  <Card.Footer actions={<Button variant="ghost">View Vault</Button>} />
</Card>

// Avoid -- deeply nested prop objects
<Card title="Henderson MSA" badge="green" bodyContent={content} footerActions={[...]} />
```

### Variant management with `cva`

Use `class-variance-authority` for Tailwind variant management. Every component defines its variants in a `cva` call co-located with the component file:

```tsx
const buttonVariants = cva("inline-flex items-center justify-center font-semibold transition-colors", {
  variants: {
    variant: {
      primary: "bg-[var(--airlock-cyan)] text-black hover:opacity-90",
      secondary: "bg-[var(--airlock-card)] text-[var(--airlock-text)] border border-[var(--airlock-border)]",
      ghost: "bg-transparent text-[var(--airlock-muted)] hover:text-[var(--airlock-text)]",
      danger: "bg-[var(--gate-red)] text-white hover:opacity-90",
    },
    size: {
      sm: "h-7 px-3 text-xs rounded",
      md: "h-9 px-4 text-sm rounded-lg",
      lg: "h-11 px-6 text-base rounded-lg",
    },
  },
  defaultVariants: { variant: "primary", size: "md" },
});
```

### Accessibility requirements

- Every interactive component receives appropriate `aria-*` attributes
- Keyboard navigation: Tab order, Enter/Space activation, Escape to dismiss
- Focus management: Focus trap in modals, focus return on close
- Screen reader support: `aria-label`, `aria-describedby`, `role` attributes
- Color is never the sole indicator -- always pair with text or icon

### Forwarded refs

All components forward refs via `React.forwardRef` for Playwright test selectors and imperative handles. Components also accept a `data-testid` prop for test targeting.

---

## 4. Design Token Alignment

All shared components consume tokens from the Airlock theme system defined in `theme.md`. No hardcoded colors or spacing values.

### Surface tokens

| Token | Value | Usage |
|-------|-------|-------|
| `--airlock-bg` | #0B0E14 | Page background, modal overlay base |
| `--airlock-surface` | #0F1219 | Panel backgrounds, sidebar, drawer |
| `--airlock-card` | #151923 | Card backgrounds, table row hover, dropdown menu |
| `--airlock-border` | #1E2330 | All borders, dividers, table grid lines |

### Text tokens

| Token | Value | Usage |
|-------|-------|-------|
| `--airlock-text` | #E2E8F0 | Primary text, headings, button labels |
| `--airlock-muted` | #64748B | Secondary text, placeholders, timestamps |

### Accent and gate tokens

| Token | Value | Usage |
|-------|-------|-------|
| `--airlock-cyan` | #00D1FF | Primary accent, active states, links, focus rings |
| `--gate-red` | #EF4444 | Blocked, critical, danger buttons |
| `--gate-yellow` | #EAB308 | Warning, needs attention |
| `--gate-purple` | #A855F7 | In review, pending |
| `--gate-green` | #22C55E | Approved, clear, success |
| `--gate-amber` | #F59E0B | SLA urgent, amber alerts |

### Spacing and shape

| Property | Value | Usage |
|----------|-------|-------|
| Spacing grid | 4px (Tailwind default) | All padding, margin, gap values |
| Card border radius | `rounded-lg` (8px) | Cards, modals, dropdowns |
| Module icon radius | `rounded-xl` (12px) | Module bar icons |
| Focus ring | `ring-2 ring-[var(--airlock-cyan)] ring-offset-2 ring-offset-[var(--airlock-bg)]` | All focusable elements |
| Transition fast | 150ms ease | Hover states, button presses |
| Transition normal | 300ms ease-in-out | Panel transitions, content swaps |

---

## 5. File Structure

```
/apps/web/src/components/
  /atoms/              <-- Existing: Badge, Button, GateDot, Icon, ProgressBar,
  |                        SearchInput, StatusDot, Toast (keep, refine APIs)
  |
  /molecules/          <-- Existing: ModuleIcon, VaultItem, FilterBar, EventCard,
  |                        ConfidenceBadge, PatchStateBadge (keep, refine APIs)
  |
  /organisms/          <-- Module-specific compositions (keep here, consume /ui/)
  |
  /templates/          <-- ShellLayout, TriptychLayout (keep here)
  |
  /ui/                 <-- NEW: Shared compound components extracted from modules
      DataTable.tsx        Column defs, sort, filter, pagination, row selection
      FilterBar.tsx        Composable filter pills, presets, search integration
      Modal.tsx            Size variants, focus trap, close-on-escape, overlay
      Drawer.tsx           Push and overlay modes, right-slide, close button
      Card.tsx             Header/Body/Footer composition, left-border color
      EmptyState.tsx       Icon + heading + description + CTA
      Toast.tsx            Container, auto-dismiss, stacking, severity variants
      Dropdown.tsx         Select/combobox, search, single/multi-select
      StatusIndicator.tsx  Dot/pill/badge shapes, gate color semantics
      ProgressBar.tsx      Linear + circular, labeled, threshold colors
      index.ts             Barrel export for all /ui/ components
```

The `/ui/` directory sits alongside existing atomic design layers. Organisms in module folders import from `/ui/` instead of reimplementing. Over time, some atoms and molecules may migrate into `/ui/` if their API stabilizes.

---

## 6. Migration Strategy

Each component follows a four-step extraction cycle:

### Step 1 -- Build the shared component

Create the new component in `/ui/` with a clean API. Write it from scratch informed by the existing implementations, not by copying one module's version. Include TypeScript types, `cva` variants, ref forwarding, and accessibility attributes.

### Step 2 -- Prove with one module

Update ONE module to consume the new shared component. Choose the module with the most complex usage (usually Contracts or CRM) so the API is validated against real requirements. Fix any API gaps discovered during integration.

### Step 3 -- Sweep remaining modules

Update all other modules to use the shared component. This is a mechanical find-and-replace pass. Each module switch is a separate commit for clean rollback.

### Step 4 -- Delete old implementations

Remove the module-specific implementations that the shared component replaces. Update imports across the codebase. Verify no dead code remains.

### Recommended extraction order

| Order | Component | Prove With | Rationale |
|-------|-----------|------------|-----------|
| 1 | Button | Contracts | Most instances of raw `<button>` bypass |
| 2 | Modal | Contracts (CreateVaultModal) | Foundation for all dialogs |
| 3 | DataTable | CRM (AccountsTable) | Highest complexity, most modules depend on it |
| 4 | FilterBar | Contracts (FilterBar) | Closely coupled with DataTable |
| 5 | Card | CRM (EntityCard, DealCard) | Used in dashboards and triage views |
| 6 | Drawer | Contracts (OttoDrawer) | Tool Dock panels depend on this |
| 7 | EmptyState | Tasks | Simple extraction, high visual consistency win |
| 8 | Toast | CRM | Wraps existing notification store |
| 9 | Dropdown | Admin | Settings pages exercise most select variants |
| 10 | StatusIndicator | Contracts | Unifies five different badge/dot components |
| 11 | ProgressBar | Contracts (triage) | Final cleanup pass |

---

## 7. Testing Requirements

### Unit tests (per component)

- Renders with default props without crashing
- Renders all variant combinations (variant x size matrix)
- Fires callback props (onClick, onClose, onFilter, onSort)
- Accessibility: passes `axe-core` automated checks
- Keyboard navigation: Tab, Enter, Space, Escape behave correctly
- Ref forwarding: `ref.current` points to the expected DOM element

### Visual regression (Playwright)

- Screenshot each component in every variant/size combination
- Screenshot in both light context (future) and dark context (current OLED theme)
- Capture hover, focus, active, and disabled states
- Compare against baseline screenshots on every PR

### Storybook (future milestone)

- One story per component with all variants displayed
- Interactive controls for props (Storybook Controls addon)
- Serves as the living component catalog for design review
- Deployed to internal URL for non-engineer stakeholders to browse

---

## Related Specs

- [Shell Overview](./overview.md) -- Master layout, three-zone architecture, Tool Dock
- [Theme](./theme.md) -- OLED dark palette, typography, gate colors, transitions
- [Toolbar Actions](./toolbar-actions.md) -- Contextual actions that consume Button, Modal, Dropdown
- [Admin](../Admin/overview.md) -- Settings UI exercises many shared components
- [Tasks](../Tasks/overview.md) -- Kanban and table views consume DataTable, Card, FilterBar
- [CRM](../CRM/overview.md) -- Pipeline, accounts, leads all use tables, cards, filters
- [Search](../Search/overview.md) -- CommandPalette shares patterns with Dropdown and SearchInput
- [Record Inspector](../Record%20Inspector/overview.md) -- Field cards consume Card and StatusIndicator
