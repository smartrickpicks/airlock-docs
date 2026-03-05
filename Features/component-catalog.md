I now have all the information needed to produce a comprehensive inventory. Here is the complete structured report:

---

# OrchestrateOS UI Component Inventory

All color references resolve through `theme.base.css`. Token values are listed at the end for quick reference.

---

## Design Token Reference (theme.base.css)

| Token | Value |
|---|---|
| `--bg-darkest` | `#0d1017` |
| `--bg-page` | `#12161f` |
| `--bg-panel` | `#1a1f2e` |
| `--bg-card` | `#1e2436` |
| `--bg-card-hover` | `#242b3d` |
| `--bg-input` | `#161b28` |
| `--text-primary` | `#e8eaed` |
| `--text-secondary` | `#9aa0a6` |
| `--text-muted` | `#6b7280` |
| `--accent-blue` | `#0091ea` |
| `--accent-blue-dim` | `rgba(0,145,234,0.15)` |
| `--green` | `#4caf50` |
| `--amber` | `#ff9800` |
| `--red` | `#f44336` |
| `--teal` | `#26a69a` |
| `--purple` | `#7b1fa2` |
| `--border` | `#2a3040` |
| `--border-light` | `#333b4f` |
| `--radius-sm` | `4px` |
| `--radius-md` | `6px` |
| `--radius-lg` | `10px` |
| `--radius-xl` | `12px` |
| `--mono` | SF Mono / Cascadia Code / Consolas / Monaco |
| `--sidebar-bg` | `#0F1A2E` |
| `--sidebar-border` | `#233A5B` |

---

## Category 1: Layout

### 1.1 Application Shell / Page Layout

**Component:** App Layout (flex shell)
**Classes:** `.app-layout`
**Demos:** Triage, Record Inspector, Contract Generator
**Visual:** `display:flex`, `height: calc(100vh - 78px)` — accounts for 34px view-switcher + 44px top bar. Full-bleed container that houses sidebar + main content side by side.
**Interaction:** Static layout shell.
**Reusable:** Yes — identical across Triage, Dossier, ContractGen.

**Component:** App Layout (Admin variant)
**Classes:** `.app-layout` (Admin version), `display:flex`, `min-height: calc(100vh - 48px)`, `margin-top: 48px`
**Demos:** Admin
**Visual:** Same flex pattern but accounts for 48px fixed topbar.
**Reusable:** Slight variant; could unify with a `--topbar-h` token.

**Component:** Org Overview App Shell
**Classes:** `.ov-app`
**Demos:** Org Overview
**Visual:** `display:flex`, `height:100vh`, `overflow:hidden` — houses sidebar + `.ov-content-area`.
**Reusable:** Functionally equivalent to `.app-layout` but uses `ov-` namespace.

---

### 1.2 Top Bar

**Component:** Top Bar (Primary / 44px)
**Classes:** `.top-bar`
**Demos:** Triage, Record Inspector, Contract Generator
**Visual:** `height:44px`, `background: var(--bg-panel)`, `border-bottom: 1px solid var(--border)`, `padding: 0 16px`, `gap:10px`. Flex row.
**Interaction:** Static container; child buttons have hover states.
**Reusable:** Yes — identical across 3 demos.

**Component:** App Topbar (Admin / 48px, fixed)
**Classes:** `.app-topbar`
**Demos:** Admin
**Visual:** `position:fixed; top:0; left:0; right:0; height:48px; z-index:1000`, `background: var(--sidebar-bg, #0F1A2E)`, `border-bottom: 1px solid var(--sidebar-border, #233A5B)`. 4px taller and uses the sidebar dark palette rather than `--bg-panel`.
**Interaction:** Fixed; always visible.
**Reusable:** Admin-specific variant.

**Component:** Org Overview Top Bar
**Classes:** `.ov-top-bar`
**Demos:** Org Overview
**Visual:** `height: var(--ov-topbar-h)` (44px), `background: var(--ov-bg-topbar)`, `border-bottom: 1px solid var(--ov-border)`, `padding: 0 20px`.
**Reusable:** Equivalent to `.top-bar`.

---

### 1.3 Sidebar (Primary Navigation, 200px)

**Component:** Sidebar (Left Nav)
**Classes:** `.sidebar`
**Demos:** Triage, Record Inspector, Contract Generator
**Visual:** `width:200px`, `min-width:200px`, `background: var(--bg-page)`, `border-right: 1px solid var(--border)`, `display:flex; flex-direction:column; padding:12px 0; overflow-y:auto`.
**Interaction:** Scroll overflow on tall viewports. Nav items highlight on hover/active.
**Reusable:** Yes — copy-paste identical across 3 demos.

**Component:** Global Nav Sidebar (Admin, collapsible, 240px)
**Classes:** `.nav-sidebar`, `.nav-sidebar.collapsed`
**Demos:** Admin
**Visual:** `width:240px`, collapsed: `width:72px`. `background: var(--sidebar-bg, #0F1A2E)`. Has `transition: width 0.25s ease`. Uses `--sidebar-*` tokens (darker blue palette).
**Interaction:** Collapse button `.sidebar-collapse-btn` toggles width. Labels hidden in collapsed state. Icon wraps get `border-radius:12px; padding:10px` pill treatment when collapsed.
**Reusable:** Admin-exclusive (different nav depth).

**Component:** Admin Secondary Sidebar (180px)
**Classes:** `.ad-sidebar`
**Demos:** Admin
**Visual:** `width:180px`, `background: var(--bg-page)`, `border-right: 1px solid var(--border)`, `overflow-y:auto`. Narrower than main sidebar.
**Interaction:** Tab switching — `.sb-nav-item.active` drives `.ad-tab-content` visibility.
**Reusable:** Admin-specific secondary panel.

**Component:** Org Overview Sidebar
**Classes:** `.ov-sidebar`
**Demos:** Org Overview
**Visual:** `width: var(--ov-sidebar-w)` (200px), `background: var(--ov-bg-sidebar)`, `border-right: 1px solid var(--ov-border)`, `padding:16px 12px; overflow-y:auto`.
**Reusable:** Equivalent to `.sidebar`; uses ov- namespace.

---

### 1.4 Main Content Areas

**Component:** Main Content (scrollable body)
**Classes:** `.main-content`
**Demos:** Triage (`flex:1; overflow-y:auto; padding:20px 24px`), Admin (`flex:1; overflow-y:auto; padding:16px 20px`), Org Overview (`.ov-main`, `flex:1; overflow-y:auto; padding:20px 24px`)
**Visual:** Fills remaining flex space. Custom scrollbar: `width:5px; background:var(--border); border-radius:3px`.
**Reusable:** Yes — same pattern across all demos.

**Component:** Contract Generator Split Layout (Builder + Preview)
**Classes:** `.content-body`, `.builder-panel`, `.preview-panel`
**Demos:** Contract Generator only
**Visual:** `.content-body` is `display:flex; overflow:hidden`. `.builder-panel` is `width:420px; min-width:420px; border-right: 1px solid var(--border)`. `.preview-panel` is `flex:1; background:var(--bg-darkest)`.
**Interaction:** Builder panel is a scrollable form; preview panel renders live document.
**Reusable:** ContractGen-specific split layout.

**Component:** Record Inspector Split (Left + Right Panel)
**Classes:** `.left-panel`, `.right-panel`
**Demos:** Record Inspector
**Visual:** `.left-panel`: `width:340px; min-width:340px; background:var(--bg-panel); border-right:1px solid var(--border); overflow-y:auto`. `.right-panel`: `flex:1; background:var(--bg-panel); overflow-y:auto`. Custom scrollbar `width:4px/5px`.
**Reusable:** Dossier-specific.

---

### 1.5 View Switcher (Demo-only Chrome)

**Component:** View Switcher Bar
**Classes:** `.view-switcher`, `.view-btn`, `.view-btn.active`
**Demos:** Triage, Record Inspector, Contract Generator
**Visual:** `display:flex; align-items:center; gap:6px; padding:5px 12px; background:#0d1017 (or var(--bg-darkest)); border-bottom:1px solid var(--border); z-index:100`. `.view-btn`: `padding:4px 14px; border-radius:4px; border:1px solid var(--border); font-size:12px`. Active: `background:var(--accent-blue); color:#fff`.
**Interaction:** Click switches demo state. Hover: `background:var(--bg-card)`.
**Reusable:** Demo-chrome only; not a production component.

---

### 1.6 Action Bar (Bottom Bar)

**Component:** Action Bar (Contract Generator footer)
**Classes:** `.action-bar`
**Demos:** Contract Generator
**Visual:** `display:flex; align-items:center; height:44px; background:var(--bg-panel); border-top:1px solid var(--border); padding:0 16px; gap:8px`. Contains field stats + export buttons.
**Reusable:** Could generalize as a sticky footer toolbar.

---

## Category 2: Navigation

### 2.1 Sidebar Nav Items

**Component:** Sidebar Nav Item
**Classes:** `.sb-nav-item`, `.sb-nav-item.active`, `.sb-nav-icon`, `.sb-nav-badge`
**Demos:** Triage, Record Inspector, Contract Generator, Admin (secondary sidebar)
**Visual:** `display:flex; align-items:center; gap:10px; padding:7px 16px; font-size:12px; color:var(--text-secondary); border-left:3px solid transparent`.
Active: `color:var(--accent-blue); background:rgba(0,145,234,0.06); border-left-color:var(--accent-blue); font-weight:600`.
Hover: `background:var(--bg-card)`. Badge: `font-size:9px; background:var(--accent-blue); color:#fff; padding:1px 6px; border-radius:8px`.
**Interaction:** Left-border indicator on active. Hover fill.
**Reusable:** Yes — shared across 4 demos.

**Component:** Global Nav Item (Admin sidebar)
**Classes:** `.nav-item`, `.nav-item.active`, `.nav-label`, `.sidebar-icon`, `.sidebar-icon-wrap`
**Demos:** Admin
**Visual:** `display:flex; align-items:center; gap:12px; padding:8px 20px; font-size:12.5px; color:var(--sidebar-text, #8EA2C8); border-left:3px solid transparent`. Active: `box-shadow:inset 3px 0 0 var(--accent-orange, #FF7A18); color:white`. Collapsed: icon-only with `border-radius:12px; padding:10px`. Icon: `width:22px; height:22px; stroke:currentColor; stroke-width:1.8`.
**Interaction:** Active state uses an orange left-border inset shadow. Collapsed mode shows only icon.
**Reusable:** Admin nav only.

**Component:** Org Overview Nav Item
**Classes:** `.ov-nav-item`, `.ov-nav-item.active`
**Demos:** Org Overview
**Visual:** `display:flex; align-items:center; gap:8px; padding:8px 10px; border-radius:var(--ov-radius-sm)` (6px); `border-left:3px solid transparent; margin-bottom:2px`. Active: `background:var(--ov-accent-dim); color:var(--ov-accent); border-left-color:var(--ov-accent); font-weight:600`. Uses `border-radius` (different from other sidebar items that have no radius).
**Reusable:** Slight variant on `.sb-nav-item`.

---

### 2.2 Section Labels

**Component:** Sidebar Section Label
**Classes:** `.sb-section-label`
**Demos:** Triage, Record Inspector, Contract Generator, Admin
**Visual:** `font-size:9px; font-weight:700; color:var(--text-muted); text-transform:uppercase; letter-spacing:0.8px; padding:10px 16px 4px`.
**Reusable:** Yes — identical across all demos.

**Component:** Org Overview Section Label
**Classes:** `.ov-section-label`
**Demos:** Org Overview
**Visual:** `font-size:10px; font-weight:700; color:var(--ov-text-muted); text-transform:uppercase; letter-spacing:0.8px; margin-bottom:8px`. Same intent, 1px larger font than `.sb-section-label`.
**Reusable:** Slight variant.

---

### 2.3 Tab Bars / Pills

**Component:** Tab Bar (Record Inspector)
**Classes:** `.tab-bar`, `.tab-pill`, `.tab-pill.active`, `.tab-badge`
**Demos:** Record Inspector
**Visual:** `.tab-bar`: `display:flex; align-items:center; gap:4px; height:40px; padding:0 16px; border-bottom:1px solid var(--border); position:sticky; top:51px; background:var(--bg-panel); z-index:10`. `.tab-pill`: `padding:5px 12px; border-radius:5px; font-size:12px; border:1px solid var(--border)`. Active: `background:var(--accent-blue); color:#fff; border-color:var(--accent-blue)`. Badge variants: `.amber` = `rgba(255,152,0,0.25); color:var(--amber)`, green-check = plain icon.
**Interaction:** Click sets active state. Sticky on scroll.
**Reusable:** Could generalize as a horizontal tab component.

**Component:** Preview Tabs (Contract Generator)
**Classes:** `.preview-tabs`, `.preview-tab`, `.preview-tab.active`
**Demos:** Contract Generator
**Visual:** `margin-left:auto; display:flex; gap:2px`. `.preview-tab`: `padding:3px 12px; border-radius:var(--radius-sm)` (4px); `font-size:11px; border:1px solid transparent`. Active: `background:var(--bg-card); color:var(--text-primary); border-color:var(--border)`. Subtly different from record inspector tab style.
**Reusable:** Tab pattern variant.

**Component:** Admin Tab Content (hidden/shown)
**Classes:** `.ad-tab-content`, `.ad-tab-content.active`
**Demos:** Admin
**Visual:** `display:none` → `display:block` on active. No visual styling — pure visibility toggle.
**Interaction:** Driven by `.sb-nav-item` clicks.

---

### 2.4 Mode Toggle (Segmented Control)

**Component:** Triage Mode Toggle
**Classes:** `.triage-mode-toggle`, `.triage-mode-btn`, `.triage-mode-btn.active`
**Demos:** Triage
**Visual:** `display:flex; background:var(--bg-card); border:1px solid var(--border); border-radius:6px; overflow:hidden`. Button: `padding:6px 18px; font-size:12px; font-weight:600; color:var(--text-secondary); border:none`. Active: `background:var(--accent-blue); color:#fff`.
**Interaction:** Toggle between Dashboard / Work Queue views.
**Reusable:** Yes — generic segmented control.

**Component:** Role Mode Toggle (Admin sidebar footer)
**Classes:** `.mode-toggle`, `.mode-btn`, `.mode-btn.active`
**Demos:** Admin
**Visual:** `display:flex; gap:4px; flex-wrap:wrap`. Button: `padding:6px 12px; font-size:0.75em; border:1px solid var(--sidebar-border); border-radius:4px`. Active: `background:var(--accent-blue, #1565c0); color:white`. Dark palette context (sidebar background).
**Reusable:** Semantically equivalent to Triage mode toggle but in sidebar context.

---

### 2.5 Filter Chips

**Component:** Filter Chip (Triage)
**Classes:** `.triage-filter-chip`, `.triage-filter-chip.active`, `.chip-count`
**Demos:** Triage
**Visual:** `padding:4px 12px; border-radius:14px; border:1px solid var(--border); font-size:11px`. Active: `background:var(--accent-blue); color:#fff; border-color:var(--accent-blue)`. `.chip-count`: `font-family:var(--mono); font-size:10px; opacity:0.8; margin-left:4px`.
**Interaction:** Click filters contract list. Hover: `background:var(--bg-card)`.
**Reusable:** Yes.

**Component:** Signal Pills (Org Overview)
**Classes:** `.ov-signal-pill`, `.ov-signal-pill.active`
**Demos:** Org Overview
**Visual:** `padding:6px 14px; border-radius:16px; font-size:11px; font-weight:600; border:1px solid var(--ov-border)`. Active: `background:var(--ov-accent-dim); color:var(--ov-accent); border-color:var(--ov-accent)`. Larger padding than Triage chips.
**Reusable:** Same concept, slight style difference.

---

### 2.6 Breadcrumbs / Back Button

**Component:** Back Button
**Classes:** `.back-btn`
**Demos:** Record Inspector, Contract Generator
**Visual:** `color:var(--text-secondary); font-size:12px; cursor:pointer`. Arrow character prefix. No border, no background.
**Interaction:** Click navigates back.
**Reusable:** Yes.

---

### 2.7 Type Pills (Top Bar)

**Component:** Type Pills (Contract type selector)
**Classes:** `.type-pills`, `.type-pill`, `.type-pill.active`
**Demos:** Contract Generator
**Visual:** `display:flex; gap:6px`. Pill: `padding:4px 14px; border-radius:16px; border:1px solid var(--border); font-size:12px`. Active: `background:var(--accent-blue); border-color:var(--accent-blue); color:#fff`. Hover: `background:var(--bg-card)`.
**Reusable:** Equivalent to filter chips but for top-bar context.

---

## Category 3: Data Display

### 3.1 Cards (Generic)

**Component:** Lane Card (Triage health summary)
**Classes:** `.lane-card`
**Demos:** Triage
**Visual:** `background:var(--bg-card); border:1px solid var(--border); border-radius:8px; padding:16px`. Grid: `grid-template-columns:1fr 1fr; gap:16px`. Title: `font-size:12px; font-weight:700; text-transform:uppercase; letter-spacing:0.5px`. Total: `font-size:28px; font-weight:700`. Stat row: `display:flex; justify-content:space-between; font-size:12px`. Value: `font-family:var(--mono); font-weight:600`.
**Reusable:** Yes — generic stat summary card.

**Component:** Readiness Card
**Classes:** `.readiness-card`, `.readiness-card.active`
**Demos:** Triage
**Visual:** `background:var(--bg-card); border:1px solid var(--border); border-radius:8px; padding:14px; cursor:pointer`. Active: `border-color:var(--accent-blue); box-shadow:0 0 0 1px var(--accent-blue)`. Grid: `repeat(5, 1fr); gap:12px`.
**Interaction:** Click activates/deactivates section filter.
**Reusable:** Yes.

**Component:** Admin Metric Card
**Classes:** `.ad-metric-card`, `.ad-metric-value`, `.ad-metric-label`, `.ad-metric-trend`
**Demos:** Admin
**Visual:** `flex:1; background:var(--bg-panel); border:1px solid var(--border); border-radius:var(--radius-lg)` (10px); `padding:16px; cursor:pointer`. Value: `font-family:var(--mono); font-size:24px; font-weight:700`. Label: `font-size:10px; font-weight:700; text-transform:uppercase; letter-spacing:0.5px; margin-top:4px`. Trend `.up` = `color:var(--green)`, `.down` = `color:var(--red)`, `.neutral/.clean` = `color:var(--text-muted)/var(--green)`.
**Interaction:** Hover: `border-color:var(--border-light)`. Click switches admin tab.
**Reusable:** Yes — KPI metric card.

**Component:** Admin Panel
**Classes:** `.ad-panel`, `.ad-panel-header`, `.ad-panel-title`, `.ad-panel-action`, `.ad-panel-body`
**Demos:** Admin
**Visual:** `background:var(--bg-panel); border:1px solid var(--border); border-radius:var(--radius-lg)` (10px); `margin-bottom:12px; overflow:hidden`. Header: `display:flex; align-items:center; justify-content:space-between; padding:10px 14px; border-bottom:1px solid var(--border)`. Title: `font-size:11px; font-weight:700; color:var(--text-muted); text-transform:uppercase; letter-spacing:0.5px`. Action: `font-size:10px; color:var(--accent-blue); font-weight:600`. Body: `padding:14px`.
**Reusable:** Yes — primary content panel/card for Admin.

**Component:** Org Overview Entity Card
**Classes:** `.ov-entity-card`, `.ov-entity-card.expanded`, `.ov-entity-header`, `.ov-entity-name`, `.ov-entity-badge`, `.ov-entity-table`
**Demos:** Org Overview
**Visual:** `flex:1; background:var(--ov-bg-card); border:1px solid var(--ov-border); border-radius:var(--ov-radius-md)` (8px); `overflow:hidden`. Expanded: `border-color:var(--ov-accent)`. Badge `.distribution`: `background:rgba(0,145,234,0.15); color:var(--ov-accent)`.
**Interaction:** Click header to expand/collapse (chevron rotates 90deg). Expanded shows row table below.
**Reusable:** Specific to Org Overview entity grouping pattern.

**Component:** Alert Panel (Org Overview)
**Classes:** `.ov-alert-panel`, `.ov-alert-icon`, `.ov-alert-title`, `.ov-alert-count`, `.ov-alert-breakdown`
**Demos:** Org Overview
**Visual:** `flex:1; background:var(--ov-bg-card); border:1px solid var(--ov-border); border-radius:var(--ov-radius-md); padding:16px`. Count: `font-size:24px; font-weight:800`. `.rfi .ov-alert-count` = `color:var(--ov-amber)`, `.correction` = `color:var(--ov-blue)` (#42a5f5), `.anomaly` = `color:var(--ov-red)`.
**Interaction:** Hover: `background:var(--ov-bg-card-hover)`. Click opens modal.
**Reusable:** Dashboard KPI card variant.

**Component:** Document Card (Structured data in Record Inspector)
**Classes:** `.dcard`, `.dcard-title`, `.dcard-body`, `.drow`, `.dl`, `.dv`
**Demos:** Record Inspector
**Visual:** `background:var(--bg-card); border:1px solid var(--border); border-radius:8px; margin:14px 0; overflow:hidden`. Title: `font-size:11px; font-weight:800; letter-spacing:0.8px; padding:9px 16px; border-bottom:1px solid var(--border)`. Row: `display:flex; padding:4px 16px; gap:10px`. Label `.dl`: `min-width:120px; font-size:12px; color:var(--text-muted)`. Value `.dv`: `font-size:12px; font-weight:500; color:var(--text-primary)`. Mono variant: `font-family:var(--mono)`. Key variant: `color:var(--accent-blue); font-weight:600`.
**Reusable:** Record Inspector specific but pattern is reusable.

---

### 3.2 Tables

**Component:** Work Queue Table
**Classes:** `.wq-table`, `.wq-row`, `.wq-rank`, `.wq-name`, `.wq-issues`
**Demos:** Triage
**Visual:** `width:100%; border-collapse:collapse`. `thead th`: `font-size:10px; font-weight:700; color:var(--text-muted); text-transform:uppercase; letter-spacing:0.5px; padding:8px 12px; border-bottom:1px solid var(--border)`. `td`: `padding:10px 12px; font-size:12px`. Row hover: `background:rgba(0,145,234,0.03)`.
**Reusable:** Standard data table.

**Component:** Admin Table
**Classes:** `.ad-table`, `.ad-table .mono`, `.ad-table .bar-cell`, `.ad-table .inline-bar`
**Demos:** Admin
**Visual:** Same structure as `.wq-table`. Adds `.inline-bar`: `position:absolute; top:0; left:0; height:100%; opacity:0.12; border-radius:0 3px 3px 0` — a background fill indicator behind the cell text.
**Reusable:** Admin-specific but pattern generalizes.

**Component:** Dashboard Group / Contract Table (Triage)
**Classes:** `.dashboard-group`, `.group-header`, `.group-contracts`, `.contract-row`, `.contract-row-header`
**Demos:** Triage
**Visual:** Group: `background:var(--bg-card); border:1px solid var(--border); border-radius:8px; overflow:hidden`. Header: `display:flex; align-items:center; gap:12px; padding:14px 16px; cursor:pointer`. Contract row: `display:grid; grid-template-columns:1fr 80px 80px 100px 100px 60px; padding:10px 16px 10px 32px; font-size:12px`.
**Interaction:** Click header expands/collapses (chevron rotates 90deg). Hover: `background:var(--bg-card-hover)`.
**Reusable:** Triage-specific grouped list.

**Component:** Org Overview Entity Table Rows
**Classes:** `.ov-entity-table`, `.ov-etbl-header`, `.ov-etbl-row`, `.ov-etbl-col`, `.ov-etbl-category`
**Demos:** Org Overview
**Visual:** `.ov-etbl-header`: `display:flex; padding:8px 16px; font-size:10px; font-weight:700; text-transform:uppercase; background:rgba(0,0,0,0.15)`. `.ov-etbl-row`: `display:flex; padding:8px 16px; font-size:11px; border-top:1px solid var(--ov-border); cursor:pointer`. Category sub-header: `background:rgba(0,145,234,0.05); padding:8px 16px; font-size:10px; cursor:pointer`. Hover: `background:rgba(0,145,234,0.1)`.
**Interaction:** Category headers collapse rows. Row hover.
**Reusable:** Org Overview specific.

---

### 3.3 Badges / Status Pills

**Component:** Health Band Badge
**Classes:** `.health-band`, `.health-band.critical`, `.health-band.at-risk`, `.health-band.review`, `.health-band.healthy`
**Demos:** Triage
**Visual:** `font-size:10px; font-weight:600; padding:2px 8px; border-radius:3px; white-space:nowrap`.
- `.critical`: `background:rgba(244,67,54,0.15); color:var(--red)`
- `.at-risk`: `background:rgba(255,152,0,0.15); color:var(--amber)`
- `.review`: `background:rgba(0,145,234,0.15); color:var(--accent-blue)`
- `.healthy`: `background:rgba(76,175,80,0.15); color:var(--green)`
**Reusable:** Yes — standard across multiple demos under slight naming variations.

**Component:** Contract Severity Badge (Triage)
**Classes:** `.cr-severity.critical / .at-risk / .review / .healthy`
**Demos:** Triage
**Visual:** Same color pattern as `.health-band` but inside contract rows. `font-size:10px; font-weight:600; padding:2px 8px; border-radius:3px`.
**Reusable:** Yes — variant of health-band.

**Component:** Status Badge (Record Inspector / Field status)
**Classes:** `.sbadge`, `.sbadge.pass`, `.sbadge.review`, `.sbadge.failed`
**Demos:** Record Inspector
**Visual:** `font-size:10px; font-weight:700; padding:2px 8px; border-radius:3px; letter-spacing:0.3px`.
- `.pass`: same green pattern
- `.review`: same amber pattern
- `.failed`: `background:rgba(244,67,54,0.15); color:var(--red)`
**Reusable:** Yes — direct equivalent of `.health-band`.

**Component:** Count Pills (Record Inspector left panel header)
**Classes:** `.count-pills`, `.cpill`, `.cpill.pass`, `.cpill.review`, `.cpill.fail`
**Demos:** Record Inspector
**Visual:** `font-size:11px; font-weight:600; padding:2px 8px; border-radius:4px`. Colors same as above.
**Reusable:** Yes.

**Component:** Admin Status Pills (Pipeline status)
**Classes:** `.ad-status-pill`, `.ad-status-pill.queued`, `.ad-status-pill.downloading`, `.ad-status-pill.extracting`, `.ad-status-pill.ready`, `.ad-status-pill.failed`, `.ad-status-pill.review`, `.ad-status-pill.system_pass`
**Demos:** Admin
**Visual:** `display:flex; flex-direction:column; align-items:center; padding:10px 14px; border-radius:var(--radius-md)`(6px); `min-width:80px`. Count: `font-family:var(--mono); font-size:20px; font-weight:700`. Label: `font-size:9px; font-weight:700; text-transform:uppercase`. Colors:
- `.queued` / `.review`: `background:var(--amber-dim); color:var(--amber)`
- `.downloading`: `background:var(--accent-blue-dim); color:var(--accent-blue)`
- `.extracting`: `background:var(--teal-dim); color:var(--teal)`
- `.ready`: `background:var(--green-dim); color:var(--green)`
- `.failed`: `background:var(--red-dim); color:var(--red)`
- `.system_pass`: `background:var(--purple-dim); color:#ce93d8`
**Reusable:** Pattern is generalizable as a KPI status chip.

**Component:** Role Badge (Admin users)
**Classes:** `.ad-role-badge`, `.ad-role-badge.admin`, `.ad-role-badge.analyst`, `.ad-role-badge.verifier`
**Demos:** Admin
**Visual:** `font-size:9px; font-weight:700; padding:2px 8px; border-radius:8px; text-transform:uppercase; letter-spacing:0.3px`.
- `.admin`: `background:var(--purple-dim); color:#ce93d8`
- `.analyst`: `background:var(--accent-blue-dim); color:var(--accent-blue)`
- `.verifier`: `background:var(--teal-dim); color:var(--teal)`
**Reusable:** Yes.

**Component:** Feed Event Pills (Admin audit log)
**Classes:** `.ad-feed-pill`, `.ad-feed-pill.extraction`, `.ad-feed-pill.contract`, `.ad-feed-pill.batch`, `.ad-feed-pill.system`, `.ad-feed-pill.analyst`
**Demos:** Admin
**Visual:** `font-size:9px; font-weight:700; padding:2px 8px; border-radius:8px; text-transform:uppercase; letter-spacing:0.3px`. Same color-coding approach as `.ad-role-badge`.
**Reusable:** Yes — event type labeling system.

**Component:** Beta Badge
**Classes:** `.beta-badge`
**Demos:** Triage, Record Inspector, Contract Generator
**Visual:** `font-size:9px; background:var(--teal); color:#fff; padding:1px 6px; border-radius:3px; font-weight:700; letter-spacing:0.5px`.
**Reusable:** Yes.

**Component:** Topbar Beta Badge (Admin)
**Classes:** `.topbar-beta`
**Demos:** Admin
**Visual:** `font-size:9px; font-weight:700; padding:2px 6px; border-radius:3px; background:rgba(38,166,154,0.15); color:#26a69a`. Teal text on dim background instead of solid teal. Slightly different treatment.
**Reusable:** Variant of `.beta-badge`.

**Component:** Sandbox Badge
**Classes:** `.sandbox-badge`
**Demos:** Triage, Record Inspector, Contract Generator
**Visual:** `font-size:9px; background:rgba(255,152,0,0.15); color:var(--amber); padding:1px 6px; border-radius:3px; font-weight:700`.
**Reusable:** Yes — environment indicator.

**Component:** Topbar Environment Badge (Admin)
**Classes:** `.topbar-env-badge`
**Demos:** Admin
**Visual:** `padding:2px 10px; border-radius:4px; font-size:9px; font-weight:600; background:rgba(76,175,80,0.15); color:#4caf50; text-transform:uppercase; letter-spacing:0.5px`. Green instead of amber (production-safe vs. sandbox framing difference).

**Component:** Section Type Badge (Contract Generator)
**Classes:** `.section-type-badge`, `.section-type-badge.boilerplate`, `.section-type-badge.field-driven`, `.section-type-badge.conditional`, `.section-type-badge.composite`
**Demos:** Contract Generator
**Visual:** `font-size:9px; font-weight:700; text-transform:uppercase; letter-spacing:0.3px; padding:2px 7px; border-radius:3px`.
- `.boilerplate`: `background:var(--teal-dim); color:var(--teal)`
- `.field-driven`: `background:var(--accent-blue-dim); color:var(--accent-blue)`
- `.conditional`: `background:var(--purple-dim); color:#ce93d8`
- `.composite`: `background:var(--amber-dim); color:var(--amber)`

**Component:** Status Chip (Document status)
**Classes:** `.status-chip`, `.status-chip.draft`, `.status-chip.complete`
**Demos:** Contract Generator
**Visual:** `font-size:10px; font-weight:700; padding:2px 10px; border-radius:4px; letter-spacing:0.3px`.
- `.draft`: `background:var(--amber-dim); color:var(--amber)`
- `.complete`: `background:var(--green-dim); color:var(--green)`

**Component:** New/Ready Badges (Work Queue)
**Classes:** `.new-badge`, `.ready-badge`
**Demos:** Triage
**Visual:** `display:inline-block; font-size:9px; font-weight:700; padding:1px 6px; border-radius:3px; margin-left:6px; vertical-align:middle`. `.new-badge`: `background:var(--accent-blue); color:#fff`. `.ready-badge`: `background:rgba(76,175,80,0.15); color:var(--green)`.

**Component:** OCR Badge
**Classes:** `.ocr-badge`
**Demos:** Record Inspector
**Visual:** `font-size:9px; font-weight:700; background:rgba(76,175,80,0.15); color:var(--green); padding:2px 8px; border-radius:3px`.

**Component:** Signature Badge
**Classes:** `.signature-badge`, `.signature-badge.show`
**Demos:** Record Inspector
**Visual:** `display:none → inline-flex; align-items:center; gap:4px; background:rgba(76,175,80,0.15); color:var(--green); padding:3px 10px; border-radius:4px; font-size:11px; font-weight:600`. Hidden by default; shown via JS.

**Component:** Hinge Badge (Field tier)
**Classes:** `.hinge-badge`
**Demos:** Record Inspector
**Visual:** `font-size:9px; color:var(--teal); background:rgba(38,166,154,0.12); padding:1px 6px; border-radius:3px; font-weight:700; display:inline-flex; align-items:center; gap:3px`.

**Component:** Tier Badge (Field drawer)
**Classes:** `.tier-badge`, `.tier-badge.hinge`, `.tier-badge.primary`, `.tier-badge.supporting`
**Demos:** Record Inspector
**Visual:** `font-size:9px; padding:1px 6px; border-radius:3px; font-weight:700`.
- `.hinge`: `background:rgba(38,166,154,0.12); color:var(--teal)`
- `.primary`: `background:var(--accent-blue-dim); color:var(--accent-blue)`
- `.supporting`: `background:rgba(107,114,128,0.12); color:var(--text-muted)`

**Component:** Org Overview Feed Badges
**Classes:** `.ov-feed-badge`, `.ov-feed-badge.rfi-badge`, `.ov-feed-badge.correction-badge`, `.ov-feed-badge.anomaly-badge`
**Demos:** Org Overview
**Visual:** `font-size:9px; font-weight:700; padding:1px 6px; border-radius:8px; text-transform:uppercase`.
- `.rfi-badge`: amber dim
- `.correction-badge`: `rgba(66,165,245,0.15); color:var(--ov-blue)` (#42a5f5)
- `.anomaly-badge`: red dim

**Component:** Entity Badge (Org Overview modal)
**Classes:** `.ov-modal-row .entity-tag`
**Demos:** Org Overview
**Visual:** `font-size:9px; color:var(--ov-accent); background:var(--ov-accent-dim); padding:1px 6px; border-radius:8px; margin-left:6px`.

---

### 3.4 Progress Bars

**Component:** Health Bar (generic inline bar)
**Classes:** `.health-bar`, `.bar-fill`
**Demos:** Triage (Readiness Cards), Record Inspector (Left Panel)
**Visual:** `height:4px; background:var(--border); border-radius:2px; overflow:hidden`. Fill: `height:100%; border-radius:2px; transition:width 0.3s`. Fill color is set inline (red/amber/green).
**Reusable:** Yes — universal mini health bar.

**Component:** Sidebar Mini Progress Bar (3-segment)
**Classes:** `.sb-mini-bar`, `.sb-bar-done`, `.sb-bar-review`, `.sb-bar-todo`
**Demos:** Triage, Record Inspector
**Visual:** `display:flex; height:4px; border-radius:2px; overflow:hidden; margin:4px 16px 2px; background:var(--border)`. Segments: done=`var(--green)`, review=`var(--amber)`, todo=`var(--text-muted)`. Width set inline as percentage.
**Reusable:** Yes — sidebar workflow progress indicator.

**Component:** Topbar Progress Bar (Record Inspector)
**Classes:** `.progress-bar-top`, `.fill`
**Demos:** Record Inspector
**Visual:** `width:130px; height:5px; background:#2a3040; border-radius:3px; overflow:hidden`. Fill: `height:100%; border-radius:3px; transition:width 0.4s, background 0.4s`. Color changes dynamically (amber→green).
**Reusable:** Yes.

**Component:** Group Health Mini Bar (Triage dashboard groups)
**Classes:** `.gh-health-bar`, `.gh-health-fill`
**Demos:** Triage
**Visual:** `width:40px; height:4px; background:var(--border); border-radius:2px; overflow:hidden`. Fill color inline.
**Reusable:** Yes.

**Component:** Horizontal Bar Row (Admin analytics)
**Classes:** `.ad-hbar-row`, `.ad-hbar-label`, `.ad-hbar-track`, `.ad-hbar-fill`, `.ad-hbar-value`
**Demos:** Admin
**Visual:** `display:flex; align-items:center; gap:10px; padding:6px 0`. Label: `font-size:11px; min-width:100px`. Track: `flex:1; height:6px; background:var(--border); border-radius:3px; overflow:hidden`. Fill: `height:100%; border-radius:3px; background:var(--accent-blue)`. Value: `font-family:var(--mono); font-size:11px; min-width:36px; text-align:right`.
**Reusable:** Yes — labeled progress bar for analytics.

**Component:** Progress Pill (Record Inspector, floating)
**Classes:** `.progress-pill`, `.progress-pill.amber`, `.progress-pill.green`, `.progress-pill.red`, `.pp-text`, `.pp-pct`, `.pp-bar`, `.pp-fill`
**Demos:** Record Inspector
**Visual:** `position:fixed; bottom:18px; right:22px; background:rgba(30,36,54,0.85); backdrop-filter:blur(12px); border:1px solid var(--border); border-radius:10px; padding:8px 16px; display:flex; align-items:center; gap:12px; box-shadow:0 4px 24px rgba(0,0,0,0.5); z-index:50`. Mini bar inside: `width:60px; height:4px`. Color variants change border and text/fill:
- `.amber`: `border-color:rgba(255,152,0,0.4)`
- `.green`: `border-color:rgba(76,175,80,0.4); box-shadow:0 0 20px rgba(76,175,80,0.15)`
- `.red`: `border-color:rgba(244,67,54,0.4)`
**Interaction:** Click toggles `.pp-popover`. Frosted glass backdrop.
**Reusable:** Yes — floating section-progress indicator.

**Component:** Org Overview Progress Bar (Sidebar)
**Classes:** `.ov-progress-bar`, `.ov-progress-fill`
**Demos:** Org Overview
**Visual:** `height:3px; background:var(--ov-border); border-radius:2px; overflow:hidden`.
**Reusable:** Thinner variant of health bar.

---

### 3.5 Section Dots

**Component:** Section Dots (Triage group header)
**Classes:** `.section-dots`, `.section-dot`, `.section-dot.pass`, `.section-dot.review`, `.section-dot.fail`
**Demos:** Triage
**Visual:** `display:flex; gap:3px`. Dot: `width:8px; height:8px; border-radius:50%`. pass=`var(--green)`, review=`var(--amber)`, fail=`var(--red)`.
**Reusable:** Yes — compact section status indicator.

**Component:** Status Dot (Org Overview)
**Classes:** `.ov-status-dot`, `.ov-status-dot.fail`, `.ov-status-dot.review`, `.ov-status-dot.pass`, `.ov-status-dot.failed`
**Demos:** Org Overview
**Visual:** `width:8px; height:8px; border-radius:50%`. `.failed`: `border:1px dashed var(--ov-red); width:7px; height:7px` — dashed border to indicate a processing failure vs. a reviewed failure.
**Reusable:** Yes.

**Component:** Section Div / Section Divider (Document content)
**Classes:** `.sec-div`, `.sd-ico`, `.sd-lbl`, `.sd-dot`, `.sd-dot.confirmed`, `.sd-dot.unreviewed`
**Demos:** Record Inspector
**Visual:** `display:flex; align-items:center; gap:10px; margin:22px 0 4px; padding:7px 12px; height:36px; border-top:1px solid var(--border); border-radius:4px`. Label: `font-size:12px; font-weight:700; color:var(--text-muted); text-transform:uppercase; letter-spacing:0.8px`. Dot: `width:10px; height:10px; border-radius:50%; margin-left:auto`. `.confirmed` = `var(--accent-blue)`, `.unreviewed` = `var(--text-muted)`.
**Reusable:** Document section header component.

---

### 3.6 Lifecycle Rail

**Component:** Lifecycle Rail / Stage Cards
**Classes:** `.lc-rail`, `.lc-stage`, `.lc-stage.active`, `.lc-arrow`, `.lc-label`, `.lc-count`, `.lc-pct`
**Demos:** Triage
**Visual:** `display:flex; align-items:center; gap:0; margin-bottom:20px`. Stage: `flex:1; background:var(--bg-card); border:1px solid var(--border); border-radius:8px; padding:14px 16px; text-align:center`. Active: `border-color:var(--accent-blue); box-shadow:0 0 0 1px var(--accent-blue)`. Label: `font-size:11px; font-weight:700; text-transform:uppercase`. Count: `font-size:22px; font-weight:700`. Pct: `font-size:11px; color:var(--text-muted)`. Arrow: `color:var(--text-muted); font-size:18px; padding:0 6px`.
**Interaction:** Click filters contracts by lifecycle stage.
**Reusable:** Domain-specific but generalizable as a workflow stage rail.

---

### 3.7 Charts / Sparklines

**Component:** Sparkline Chart (Admin throughput)
**Classes:** `.ad-sparkline`
**Demos:** Admin
**Visual:** `display:block; width:100%; height:80px`. SVG `<polygon>`: `fill:rgba(0,145,234,0.15)` (area). SVG `<polyline>`: `fill:none; stroke:var(--accent-blue); stroke-width:1.5` (line).
**Reusable:** Yes — inline SVG sparkline.

**Component:** Confidence Distribution Bar (Admin)
**Classes:** `.ad-conf-bar`, `.ad-conf-seg`, `.ad-conf-seg.high`, `.ad-conf-seg.medium`, `.ad-conf-seg.low`, `.ad-conf-labels`
**Demos:** Admin
**Visual:** `display:flex; height:8px; border-radius:4px; overflow:hidden; margin:8px 0`. Segments: `.high`=`var(--green)`, `.medium`=`var(--amber)`, `.low`=`var(--red)`. Labels: `display:flex; gap:12px; font-size:10px`. Dot legend: `width:6px; height:6px; border-radius:50%`.
**Reusable:** Yes — confidence distribution visualization.

---

### 3.8 Metrics / Stats

**Component:** Stat Row (Admin analytics)
**Classes:** `.ad-stat-row`, `.ad-stat`, `.ad-stat-value`, `.ad-stat-label`
**Demos:** Admin
**Visual:** `display:flex; gap:20px; margin-bottom:12px; flex-wrap:wrap`. Value: `font-family:var(--mono); font-size:16px; font-weight:700; color:var(--text-primary)`. Label: `font-size:10px; color:var(--text-muted); text-transform:uppercase; letter-spacing:0.3px`.
**Reusable:** Yes.

**Component:** Sidebar Progress Rows (batch count display)
**Classes:** `.sb-progress-row`, `.sb-prog-count`, `.sb-prog-label`
**Demos:** Triage, Record Inspector
**Visual:** `display:flex; align-items:center; gap:8px; padding:3px 16px; font-size:12px`. Count: `font-weight:700; font-size:14px/18px; min-width:22px/28px`. Color: `.todo`=`var(--text-muted)`, `.review`=`var(--amber)`, `.done`=`var(--green)`.
**Reusable:** Yes.

**Component:** Tier Separators (Field list grouping)
**Classes:** `.tier-sep`, `.tier-sep.t0`, `.tier-sep.t1`, `.tier-sep.t2`
**Demos:** Record Inspector
**Visual:** `display:flex; align-items:center; gap:8px; padding:10px 16px 4px; font-size:10px; font-weight:700; letter-spacing:1px; text-transform:uppercase`. Lines (`.ln`): `flex:1; height:1px`. `.t0` = teal, `.t1` = accent-blue, `.t2` = text-muted.
**Reusable:** Yes — section separator with label.

**Component:** Admin Quality Grid
**Classes:** `.ad-quality-grid`, `.ad-quality-item`
**Demos:** Admin
**Visual:** `display:grid; grid-template-columns:1fr 1fr; gap:8px`. Item: `display:flex; justify-content:space-between; align-items:center; padding:8px 12px; background:var(--bg-card); border-radius:var(--radius-md)`. `.qlabel`: `font-size:11px; color:var(--text-secondary)`. `.qvalue`: `font-family:var(--mono); font-size:14px; font-weight:700`.
**Reusable:** Yes — key-value stat grid.

---

### 3.9 Feed / Activity Items

**Component:** Admin Feed Item (Audit log row)
**Classes:** `.ad-feed-item`, `.ad-feed-item.error-event`, `.ad-feed-ts`, `.ad-feed-pill`, `.ad-feed-body`, `.ad-feed-title`, `.ad-feed-meta`, `.ad-feed-id`
**Demos:** Admin
**Visual:** `display:flex; align-items:flex-start; gap:12px; padding:10px 14px; border-bottom:1px solid var(--border)`. Error: `border-left:3px solid var(--red)`. Timestamp: `font-family:var(--mono); font-size:10px; min-width:60px`. Body title: `font-size:12px; font-weight:600`. Meta: `font-size:11px; display:flex; gap:6px`.
**Interaction:** Hover: `background:var(--bg-card)`.
**Reusable:** Yes.

**Component:** Org Overview Feed Item
**Classes:** `.ov-feed-item`, `.ov-feed-dot`, `.ov-feed-body`, `.ov-feed-title`, `.ov-feed-meta`, `.ov-feed-badge`, `.ov-feed-arrow`
**Demos:** Org Overview
**Visual:** `display:flex; align-items:flex-start; gap:12px; padding:12px 16px; background:var(--ov-bg-card); border:1px solid var(--ov-border); border-radius:var(--ov-radius-md); margin-bottom:6px`. Dot: `width:10px; height:10px; border-radius:50%`. Arrow fades in on hover. `.anomaly` / `.escalation` dots pulse with keyframe animation.
**Interaction:** Hover: left border jumps to 3px `var(--ov-accent)` with padding adjustment. Arrow becomes visible.
**Reusable:** Yes — richer variant of Admin feed item.

---

## Category 4: Forms & Input

### 4.1 Buttons

**Component:** Header Button
**Classes:** `.header-btn`, `.header-btn.primary`, `.header-btn.gate`
**Demos:** Triage, Record Inspector, Contract Generator
**Visual:** `padding:5px 14px; border-radius:5px; border:1px solid var(--border); background:transparent; color:var(--text-primary); font-size:12px; font-weight:500`. `.primary`: `background:var(--accent-blue); border-color:var(--accent-blue); color:#fff`. `.gate`: `border-color:var(--accent-blue); color:var(--accent-blue); font-weight:700`.
**Interaction:** Hover: `background:var(--bg-card)`.
**Reusable:** Yes — primary CTA button family.

**Component:** Action Buttons (Field drawer)
**Classes:** `.action-btns`, `.abtn`, `.abtn.confirm`
**Demos:** Record Inspector
**Visual:** `display:flex; gap:8px; margin-top:10px`. Button: `padding:6px 18px; border-radius:5px; border:1px solid var(--border); font-size:12px; font-weight:500`. `.confirm`: `background:var(--accent-blue); border-color:var(--accent-blue); color:#fff`. Hover `.confirm`: `background:#0081d4`.
**Reusable:** Yes.

**Component:** Export Button (Contract Generator)
**Classes:** `.export-btn`, `.export-btn.primary`
**Demos:** Contract Generator
**Visual:** Identical to `.header-btn` with extra `background:var(--bg-card)` default state. `.primary` same as `.header-btn.primary`.
**Reusable:** Yes — alias for `.header-btn`.

**Component:** Add User Button (Admin)
**Classes:** `.ad-btn-add`
**Demos:** Admin
**Visual:** `display:inline-flex; align-items:center; gap:4px; padding:6px 14px; font-size:11px; font-weight:600; color:var(--accent-blue); background:var(--accent-blue-dim); border:1px solid rgba(0,145,234,0.3); border-radius:var(--radius-md)`. Hover: `background:rgba(0,145,234,0.25)`.
**Reusable:** Yes — a subtle "add action" button variant.

**Component:** Accept Button (Suggested fields)
**Classes:** `.accept-btn`
**Demos:** Record Inspector
**Visual:** `font-size:10px; padding:2px 10px; border-radius:3px; border:1px solid var(--green); color:var(--green); background:transparent`. Hover: `background:rgba(76,175,80,0.1)`.
**Reusable:** Yes.

**Component:** View / Mode Buttons
**Classes:** `.view-btn` (demo chrome)
**Demos:** Triage, Record Inspector, Contract Generator
Already documented under View Switcher. Not production components.

**Component:** Counterparty Action Buttons
**Classes:** `.cp-btn`, `.cp-btn.purple`
**Demos:** Record Inspector
**Visual:** `padding:5px 14px; border-radius:5px; border:1px solid var(--border); background:transparent; font-size:11px; font-weight:500`. `.purple`: `background:var(--purple); border-color:var(--purple); color:#fff`.

**Component:** Otto Send Button
**Classes:** `.kc-send`
**Demos:** Triage, Record Inspector
**Visual:** `width:32px; height:32px; border-radius:50%; background:linear-gradient(135deg, #00C2FF, #3CB88B); border:none; color:#fff; font-size:14px; cursor:pointer`.
**Reusable:** Yes — circular icon action button.

---

### 4.2 Text Inputs / Search

**Component:** Search Input (Top Bar)
**Classes:** `.rp-search`, `.rp-search input`
**Demos:** Triage, Record Inspector
**Visual:** `display:flex; align-items:center; gap:6px; background:var(--bg-card); border:1px solid var(--border); border-radius:5px; padding:4px 10px`. Input: `background:transparent; border:none; color:var(--text-primary); font-size:12px; outline:none; width:140px/90px`.
**Reusable:** Yes.

**Component:** Field Input (Contract Generator form)
**Classes:** `.field-input`, `.field-input.filled`
**Demos:** Contract Generator
**Visual:** `width:100%; padding:7px 10px; background:var(--bg-input); border:1px solid var(--border); border-radius:var(--radius-md)` (6px); `color:var(--text-primary); font-size:12px; outline:none; transition:border-color 0.15s, box-shadow 0.15s`. Focus: `border-color:var(--border-focus); background:var(--bg-input-focus); box-shadow:0 0 0 2px var(--accent-blue-dim)`. `.filled`: `border-color:var(--green)`. Placeholder: `color:var(--text-muted)`.
**Interaction:** Focus ring on focus. Green border when filled.
**Reusable:** Yes — standard form field.

**Component:** Counterparty Form Input
**Classes:** `.cp-field-input`, `.cp-field-input.extracted`
**Demos:** Record Inspector
**Visual:** `padding:5px 8px; border-radius:4px; border:1px solid var(--border); background:var(--bg-page); font-size:11px`. `.extracted`: `border-left:3px solid var(--purple-light)` — indicates AI-extracted values.
**Reusable:** Yes — form input with extraction indicator.

**Component:** Admin Search Input
**Classes:** `.ad-search-input`
**Demos:** Admin
**Visual:** `font-size:11px; padding:4px 8px; border-radius:var(--radius-sm)` (4px); `background:var(--bg-input); border:1px solid var(--border); width:180px`. Focus: `border-color:var(--border-focus)`.
**Reusable:** Yes.

**Component:** Otto Chat Input
**Classes:** `.kc-input`
**Demos:** Triage, Record Inspector
**Visual:** `flex:1; background:var(--bg-card); border:1px solid var(--border); border-radius:8px; padding:8px 12px; font-size:12px`. Focus: `border-color:var(--accent-blue)`.
**Reusable:** Yes.

**Component:** Org Overview Otto Input
**Classes:** `.ov-otto-input input`
**Demos:** Org Overview
**Visual:** `width:100%; background:var(--ov-bg-card); border:1px solid var(--ov-border); border-radius:var(--ov-radius-sm); padding:8px 12px; font-size:12px`. Focus: `border-color:var(--ov-accent)`.
**Reusable:** Equivalent to `.kc-input`.

---

### 4.3 Selects / Dropdowns

**Component:** Field Select (Contract Generator)
**Classes:** `select.field-input`
**Demos:** Contract Generator
**Visual:** Same as `.field-input` plus `appearance:none; background-image:url(SVG chevron); background-position:right 10px center; padding-right:28px`.
**Reusable:** Yes.

**Component:** Admin Filter Select
**Classes:** `.ad-filter-select`
**Demos:** Admin
**Visual:** `font-size:11px; padding:4px 8px; border-radius:var(--radius-sm); background:var(--bg-input); border:1px solid var(--border)`.

**Component:** Org Overview Select
**Classes:** `.ov-select`, `.ov-analyst-select`
**Demos:** Org Overview
**Visual:** `background:var(--ov-bg-card); border:1px solid var(--ov-border); border-radius:var(--ov-radius-sm); font-size:11px; padding:6px 10px`. Focus: `border-color:var(--ov-accent)`.

---

### 4.4 Toggles

**Component:** Toggle Switch (Primary)
**Classes:** `.toggle`, `.toggle.on`
**Demos:** Triage, Record Inspector, Contract Generator, Admin
**Visual:** `width:34px; height:18px; border-radius:9px; background:var(--border); position:relative; cursor:pointer; transition:background 0.2s`. `.on`: `background:var(--accent-blue)` (Triage/ContractGen) or `background:var(--green)` (Record Inspector/Admin). Knob `::after`: `width:14px; height:14px; border-radius:50%; background:#fff; position:absolute; top:2px; left:2px; transition:left 0.2s`. On: `left:18px`.
**Interaction:** Click class toggle. Slides knob with CSS transition.
**Reusable:** Yes — system-wide toggle.

**Component:** Dark Mode Toggle (Admin)
**Classes:** `.dark-mode-switch`, `.dark-mode-slider`
**Demos:** Admin
**Visual:** `width:40px; height:22px` (slightly larger than `.toggle`). Input hidden. Slider: `border-radius:11px`. Knob: `16px x 16px`. Checked: `background:var(--accent-bright, #7DB3FF)`.
**Reusable:** Variant toggle with native checkbox input.

**Component:** Mini Toggle (NYE fields in Record Inspector)
**Classes:** `.mini-toggle`, `.mini-toggle.on`
**Demos:** Record Inspector
**Visual:** `width:30px; height:16px; border-radius:8px` — smallest toggle. Knob `::after`: `width:12px; height:12px`. On: `background:var(--green); left:16px`.
**Reusable:** Yes — compact toggle for inline use.

**Component:** Toggle Track / Thumb (Contract Generator AI Polish)
**Classes:** `.toggle-track`, `.toggle-track.on`, `.toggle-thumb`
**Demos:** Contract Generator
**Visual:** `width:28px; height:16px; border-radius:8px; background:var(--border)`. On: `background:var(--accent-blue)`. Thumb: `width:12px; height:12px`. — another mini variant.

**Component:** Org Overview Toggle
**Classes:** `.ov-toggle`, `.ov-toggle.on`, `.ov-toggle-knob`
**Demos:** Org Overview
**Visual:** `width:32px; height:18px; border-radius:9px`. `.on`: `background:var(--ov-accent)`. Knob: `width:14px; height:14px`.

---

### 4.5 Sliders

**Component:** Range Slider (Royalty Rate)
**Classes:** `.slider-row`, `.slider-val`
**Demos:** Contract Generator
**Visual:** `display:flex; align-items:center; gap:10px`. Input: `flex:1; accent-color:var(--accent-blue); height:4px`. Value label: `font-family:var(--mono); font-size:13px; font-weight:600; color:var(--accent-blue); min-width:40px; text-align:right`.
**Interaction:** Input event updates value label and preview in real-time.
**Reusable:** Yes.

---

### 4.6 Checkbox Groups

**Component:** Checkbox Pill Group (Distribution Scope)
**Classes:** `.checkbox-group`, `.checkbox-pill`, `.checkbox-pill.checked`, `.check-icon`
**Demos:** Contract Generator
**Visual:** `display:flex; flex-wrap:wrap; gap:6px`. Pill: `display:flex; align-items:center; gap:5px; padding:4px 10px; border:1px solid var(--border); border-radius:16px; font-size:11px`. `.checked`: `background:var(--accent-blue-dim); border-color:var(--accent-blue); color:var(--accent-blue)`. Check icon: `width:14px; height:14px; border:1.5px solid var(--text-muted); border-radius:3px`. Checked icon: `background:var(--accent-blue); border-color:var(--accent-blue); color:#fff`.
**Interaction:** Click toggles `.checked` class and check icon content.
**Reusable:** Yes.

---

### 4.7 Role Pill Selectors

**Component:** Role Pills (Sidebar bottom)
**Classes:** `.sb-rpill`, `.sb-rpill.active`
**Demos:** Triage, Record Inspector
**Visual:** `padding:3px 10px; border-radius:12px; border:1px solid var(--border); background:transparent; color:var(--text-secondary); font-size:10px`. Active: `background:var(--accent-blue); color:#fff; border-color:var(--accent-blue)`.
**Reusable:** Yes.

**Component:** Role Pills (Left Panel, Record Inspector)
**Classes:** `.rpill`, `.rpill.active`
**Demos:** Record Inspector
**Visual:** `padding:4px 12px; border-radius:4px; font-size:11px; font-weight:600; border:1px solid var(--border)`. Active: same. Slightly larger than `.sb-rpill`.
**Reusable:** Yes.

**Component:** Org Overview Role Pill (display only)
**Classes:** `.ov-role-pill`
**Demos:** Org Overview
**Visual:** `display:inline-block; font-size:11px; font-weight:600; color:var(--ov-accent); background:var(--ov-accent-dim); padding:4px 12px; border-radius:12px`. Non-interactive display badge.

---

### 4.8 Clause Selector (Contract Generator)

**Component:** Clause Selector
**Classes:** `.clause-selector`, `.clause-name`, `.clause-preview`, `.clause-meta`, `.clause-meta .version`
**Demos:** Contract Generator
**Visual:** `background:var(--bg-input); border:1px solid var(--border); border-radius:var(--radius-md); padding:8px 10px`. Name: `font-size:11px; font-weight:600; color:var(--teal); font-family:var(--mono)`. Preview: `font-size:11px; color:var(--text-secondary); line-height:1.6; -webkit-line-clamp:3; overflow:hidden`. Meta version: `background:var(--bg-card); padding:1px 5px; border-radius:2px`.
**Reusable:** ContractGen-specific.

---

### 4.9 Section Cards (Contract Generator)

**Component:** Section Card (Builder)
**Classes:** `.section-card`, `.section-card.open`, `.section-card.has-values`, `.section-card.has-gaps`, `.section-header`, `.section-body`, `.section-chevron`
**Demos:** Contract Generator
**Visual:** `background:var(--bg-panel); border:1px solid var(--border); border-radius:var(--radius-lg)`(10px); `margin-bottom:8px`. `.has-values`: `border-left:3px solid var(--green)`. `.has-gaps`: `border-left:3px solid var(--amber)`. Header: `display:flex; align-items:center; gap:8px; padding:10px 14px; cursor:pointer`. Body: `display:none → block` when `.open`. Chevron: `font-size:10px; transition:transform 0.2s`. Open: `rotate(90deg)`.
**Interaction:** Click header toggles open. Left border changes based on form state.
**Reusable:** Generalizable as a collapsible form section.

---

## Category 5: Feedback

### 5.1 Toast Notifications

**Component:** Toast
**Classes:** `.toast`, `.toast.show`
**Demos:** Triage, Record Inspector
**Visual:** `position:fixed; bottom:70-80px; left:50%; transform:translateX(-50%); background:var(--bg-card); border:1px solid var(--accent-blue)/var(--border); color:var(--text-primary); padding:8px 20px; border-radius:8px; font-size:12px; opacity:0; transition:all 0.3s; z-index:300; pointer-events:none`. Show: `opacity:1; transform:translateX(-50%) translateY(0)`.
**Interaction:** Show/hide via JS. Auto-dismisses after 2000ms.
**Reusable:** Yes.

**Component:** Org Overview Toast
**Classes:** `.ov-toast`
**Demos:** Org Overview
**Visual:** `position:fixed; bottom:80px; left:50%; transform:translateX(-50%); background:var(--ov-bg-card); border:1px solid var(--ov-accent); border-radius:var(--ov-radius-md); padding:12px 20px; font-size:12px; box-shadow:0 4px 20px rgba(0,0,0,0.4); animation:toastIn 0.3s`. Uses entry animation rather than opacity toggle. Slightly more padded.
**Reusable:** Yes.

---

### 5.2 Loading / Shimmer

**Component:** Shimmer Overlay (Contract Generator)
**Classes:** `.shimmer-overlay`, `.shimmer-overlay.active`
**Demos:** Contract Generator
**Visual:** `position:absolute; inset:0; background:linear-gradient(90deg, transparent 0%, rgba(0,145,234,0.06) 50%, transparent 100%); background-size:200% 100%; animation:shimmer 1.5s infinite`. Hidden by default; activated on generate action.
**Interaction:** Appears during async operations.
**Reusable:** Yes.

**Component:** Admin Loading State
**Classes:** `.ad-loading`
**Demos:** Admin
**Visual:** `color:var(--text-muted); font-size:12px; padding:20px 0`.
**Reusable:** Yes — simple text loading state.

---

### 5.3 Warning/Alert Banners

**Component:** Warning Banner
**Classes:** `.warn-banner`, `.wt`, `.wb`
**Demos:** Record Inspector
**Visual:** `margin:8px 16px; padding:10px 12px; background:rgba(255,152,0,0.08); border:1px solid rgba(255,152,0,0.25); border-radius:6px`. Title `.wt`: `font-size:12px; font-weight:700; color:var(--amber); display:flex; align-items:center; gap:6px`. Body `.wb`: `font-size:11px; color:var(--text-secondary); padding-left:20px`.
**Reusable:** Yes — amber inline alert.

**Component:** Active Filter Bar
**Classes:** `.active-filter-bar`, `.clear-filter`
**Demos:** Triage
**Visual:** `display:flex; align-items:center; gap:8px; padding:8px 14px; margin-bottom:12px; background:rgba(0,145,234,0.06); border:1px solid rgba(0,145,234,0.2); border-radius:6px; font-size:12px; color:var(--accent-blue)`. Clear: `margin-left:auto; font-size:11px; opacity:0.7`. Hover: `opacity:1; text-decoration:underline`.
**Reusable:** Yes — active filter indicator.

---

### 5.4 Tooltips / Popovers

**Component:** Progress Pill Popover
**Classes:** `.pp-popover`, `.pp-popover.open`, `.pp-row`
**Demos:** Record Inspector
**Visual:** `position:fixed; bottom:56px; right:22px; background:rgba(30,36,54,0.95); backdrop-filter:blur(12px); border:1px solid var(--border); border-radius:8px; padding:12px 16px; width:250px; box-shadow:0 4px 24px rgba(0,0,0,0.5)`. `display:none → block` on `.open`. Row: `display:flex; align-items:center; gap:8px; padding:4px 0; font-size:12px`.
**Interaction:** Toggled by clicking progress pill. Click outside closes.
**Reusable:** Yes.

---

### 5.5 Scroll Progress

**Component:** Scroll Progress Bar (Document)
**Classes:** `.scroll-progress`, `.scroll-progress .bar`
**Demos:** Record Inspector
**Visual:** `position:sticky; top:0; height:3px; background:transparent; z-index:12`. Bar: `height:100%; background:var(--accent-blue); width:0; transition:width 0.1s`.
**Interaction:** Width updated on scroll event via JS.
**Reusable:** Yes.

---

## Category 6: Overlays

### 6.1 Context Menus

**Component:** Static Context Menu (Demo showcase)
**Classes:** `.static-ctx`, `.ctx-sel`, `.ctx-group`, `.ctx-row`, `.ctx-row.hovered`, `.cr-icon`, `.cr-label`, `.cr-key`, `.ctx-divider`
**Demos:** Record Inspector
**Visual:** `position:relative; width:260px; background:rgba(26,31,46,0.95); backdrop-filter:blur(16px); border:1px solid var(--border-light); border-radius:8px; overflow:hidden; box-shadow:0 8px 32px rgba(0,0,0,0.6)`. Row: `padding:5px 14px; display:flex; align-items:center; gap:8px; font-size:12px`. Hover/active: `background:rgba(0,145,234,0.1)`. Group label: `font-size:9px; font-weight:700; text-transform:uppercase; letter-spacing:0.5px; color:var(--text-muted)`. Divider: `height:1px; background:var(--border); margin:2px 0`.
**Reusable:** Yes — rich right-click context menu.

**Component:** Dynamic Context Menu (Right-click)
**Classes:** `.ctx-menu`, `.ctx-menu.open`, `.ctx-header`, `.ctx-item`, `.ci-icon`, `.ci-label`, `.ci-key`, `.ctx-divider`
**Demos:** Record Inspector
**Visual:** `position:fixed; z-index:200; background:rgba(26,31,46,0.96); backdrop-filter:blur(16px); border:1px solid var(--border-light); border-radius:8px; padding:6px 0; min-width:240px; max-width:300px; box-shadow:0 8px 32px rgba(0,0,0,0.6); display:none`. Item: `padding:5px 14px; font-size:12px`. Hover: `background:rgba(0,145,234,0.1)`. Key shortcut: `font-size:10px; color:var(--text-muted); font-family:var(--mono)`.
**Interaction:** `contextmenu` event positions at cursor. Keyboard shortcut support via `keydown` listener. Click outside closes.
**Reusable:** Yes.

---

### 6.2 Modal

**Component:** Alert Modal (Org Overview)
**Classes:** `.ov-modal-overlay`, `.ov-modal-overlay.open`, `.ov-modal`, `.ov-modal-header`, `.ov-modal-body`, `.ov-modal-row`, `.close-btn`
**Demos:** Org Overview
**Visual:** Overlay: `position:fixed; inset:0; background:rgba(0,0,0,0.6); z-index:1000; display:flex; align-items:center; justify-content:center; opacity:0; pointer-events:none; transition:opacity 0.2s`. Open: `opacity:1; pointer-events:auto`. Modal: `background:var(--ov-bg-card); border:1px solid var(--ov-border); border-radius:var(--ov-radius-lg)`(12px); `width:560px; max-height:70vh; display:flex; flex-direction:column; overflow:hidden; box-shadow:0 20px 60px rgba(0,0,0,0.5)`. Row: `display:flex; align-items:flex-start; gap:10px; padding:10px 8px; border-bottom:1px solid; cursor:pointer; border-radius:var(--ov-radius-sm); transition:background 0.1s`. Close button: `width:28px; height:28px; border-radius:50%; border:1px solid var(--ov-border); font-size:14px`.
**Interaction:** Click overlay closes. Click row navigates to contract. Fade animation on open/close.
**Reusable:** Yes.

---

### 6.3 Otto Chat Panel (AI Assistant Drawer)

**Component:** Otto FAB (Floating Action Button)
**Classes:** `.otto-fab`
**Demos:** Triage, Record Inspector
**Visual:** `position:fixed; bottom:24px; right:24px; width:48px; height:48px; border-radius:50%; background:linear-gradient(135deg, #00C2FF, #3CB88B); border:none; box-shadow:0 4px 16px rgba(0,194,255,0.3); z-index:49`.
**Interaction:** Hover: `transform:scale(1.08); box-shadow: 0 6px 24px rgba(0,194,255,0.4)`. Click toggles chat panel.
**Reusable:** Yes.

**Component:** Otto Chat Panel
**Classes:** `.otto-chat-panel`, `.otto-chat-panel.open`, `.kc-header`, `.kc-header-text`, `.kc-title`, `.kc-sub`, `.kc-close`, `.kc-body`, `.kc-msg`, `.kc-msg.system`, `.kc-msg.user`, `.kc-msg.otto`, `.kc-field-ref`, `.kc-footer`, `.kc-input-row`, `.kc-input`, `.kc-send`, `.kc-powered`
**Demos:** Triage, Record Inspector
**Visual:** `position:fixed; bottom:84px; right:24px; width:380px; height:480px; background:var(--bg-panel); border:1px solid var(--border); border-radius:12px; box-shadow:0 8px 40px rgba(0,0,0,0.6); z-index:55; display:none → flex`. Header: `padding:12px 16px; background:linear-gradient(135deg, rgba(0,194,255,0.08), rgba(60,184,139,0.08))`. Message variants:
- `.system`: centered, `background:rgba(0,194,255,0.06); border:1px solid rgba(0,194,255,0.15); border-radius:8px`
- `.user`: right-aligned, `background:var(--accent-blue); color:#fff; border-bottom-right-radius:4px`
- `.otto` (Otto message): left-aligned, `background:var(--bg-card); border:1px solid var(--border); border-bottom-left-radius:4px`
Field ref chip: `background:rgba(0,145,234,0.1); color:var(--accent-blue); padding:2px 8px; border-radius:4px; font-size:11px`.
**Reusable:** Yes — full AI chat panel.

**Component:** Org Overview Otto Chat
**Classes:** `.ov-otto-chat`, `.ov-otto-header`, `.ov-otto-body`, `.ov-otto-input`
**Demos:** Org Overview
**Visual:** `position:fixed; bottom:80px; right:24px; width:360px; height:480px; background:var(--ov-bg-sidebar); border:1px solid var(--ov-border); border-radius:var(--ov-radius-lg); box-shadow:0 8px 32px rgba(0,0,0,0.5)`. More minimal than the Triage/Dossier version — no styled message bubbles, just prose content.
**Reusable:** Simplified variant.

---

### 6.4 Drawers / Expandable Rows

**Component:** Field Drawer (Record Inspector)
**Classes:** `.fdrawer`, `.fdrawer.open`, `.fdrawer.hinge-drawer`, `.fd-label`, `.fd-title`, `.fd-text`, `.fd-meta`, `.evidence-box`, `.conf-bar`, `.action-btns`, `.next-unresolved`
**Demos:** Record Inspector
**Visual:** `background:var(--bg-card); border-left:3px solid var(--accent-blue); padding:0; max-height:0; overflow:hidden; transition:max-height 0.25s ease, padding 0.25s ease`. Open: `max-height:500px; padding:12px 16px`. Hinge variant: `border-left-color:var(--teal)`. Evidence box: `background:var(--bg-page); border-left:3px solid var(--accent-blue); padding:8px 12px; font-style:italic; border-radius:0 4px 4px 0`. Confidence bar: `height:4px`.
**Interaction:** Slide-down animation via max-height transition. Triggered by clicking `.frow`.
**Reusable:** Yes — field detail expand panel.

**Component:** Dashboard Group Contracts (Triage)
**Classes:** `.group-contracts`, `.group-contracts.visible`
**Demos:** Triage
**Visual:** `display:none → block`. No animation — instant toggle.
**Reusable:** Yes.

---

### 6.5 Spotlight Overlay Mode

**Component:** Spotlight / Focus Mode
**Classes:** `.doc-content.spotlight`, `.dpara.focused`, `.focus-hl`, `.exit-focus`, `.match-counter`
**Demos:** Record Inspector
**Visual:** When `.spotlight` active: all `.dpara`, `.sec-div`, `.dcard` get `opacity:0.2; filter:blur(0.8px)`. `.focused` paragraph: `opacity:1; filter:none; border:1.5px solid var(--accent-blue); border-radius:6px; padding:14px 16px; background:rgba(0,145,234,0.04); animation:spotlight-flash 1.2s`. Flash keyframe goes from `rgba(0,145,234,0.25)` to dim. Exit button: `position:sticky; top:56px; float:right; background:var(--bg-card); border-radius:16px; padding:5px 14px; font-size:11px`. Match counter: `position:absolute; top:-12px; right:12px; background:var(--bg-card); border-radius:12px; padding:2px 10px`.
**Interaction:** Highlights specific field's source paragraph. All else blurred.
**Reusable:** Dossier-specific interaction mode.

---

### 6.6 Counterparty Resolution Card

**Component:** CP Card (New Counterparty Warning)
**Classes:** `.cp-card`, `.cp-header`, `.cp-alert`, `.cp-name`, `.cp-sub`, `.cp-actions`, `.cp-form`, `.cp-form-grid`, `.cp-form-grid.g3`, `.cp-field`, `.cp-field-label`, `.cp-field-input`
**Demos:** Record Inspector
**Visual:** `margin:8px 16px; padding:14px; background:var(--bg-card); border:1px solid rgba(255,152,0,0.3); border-left:3px solid var(--amber); border-radius:6px`. Header: `font-size:13px; font-weight:700; color:var(--amber)`. Form inside: `background:var(--bg-card); border:1px solid var(--border); border-radius:6px; padding:12px`. Grid: `grid-template-columns:1fr 1fr` (or `1fr 1fr 1fr` for `.g3`).
**Reusable:** Domain-specific new-entity creation pattern.

---

## Category 7: Domain-Specific

### 7.1 Gate Indicator

**Component:** Gate Button
**Classes:** `.header-btn.gate`
**Demos:** Record Inspector
**Visual:** `border-color:var(--accent-blue); color:var(--accent-blue); font-weight:700`. Looks like a standard header button but styled as an action gate checkpoint.
**Reusable:** Domain-specific CTA for workflow gating.

---

### 7.2 Health Scores

**Component:** Health Score Display (inline numeric)
**Classes:** Inline styling via `.cr-health`, `.lh-total`, `.lc-count`
**Demos:** Triage, Org Overview
**Visual:** `.cr-health`: `font-family:var(--mono); font-size:11px; color:var(--text-secondary)`. Health color coded: <35 → `var(--red)`, 35-49 → `var(--amber)`, 50-69 → `var(--accent-blue)`, ≥70 → `var(--green)`.
**Reusable:** Yes — health coloring utility.

---

### 7.3 Confidence Bars

**Component:** Confidence Bar (Field drawer)
**Classes:** `.conf-bar`, `.conf-bar .track`, `.conf-bar .fill`, `.conf-bar .pct`
**Demos:** Record Inspector
**Visual:** `display:flex; align-items:center; gap:8px; margin:6px 0`. Track: `flex:1; height:4px; background:var(--border); border-radius:2px; overflow:hidden`. Fill: `height:100%; border-radius:2px`. Color: ≥80% = `var(--green)`, ≥50% = `var(--amber)`, <50% = `var(--red)`. Percentage label: `font-size:11px; min-width:28px`.
**Reusable:** Yes.

**Component:** Admin Confidence Distribution Bar
Already documented in 3.7 as `.ad-conf-bar`.

---

### 7.4 Field Rows (Record Inspector pattern)

**Component:** Field Row
**Classes:** `.frow`, `.frow.expanded`, `.frow.hinge`
**Demos:** Record Inspector
**Visual:** `display:flex; align-items:center; padding:7px 16px; gap:8px; cursor:pointer; border-left:3px solid transparent; transition:background 0.1s`. Hover: `background:var(--bg-card)`. `.expanded`: `background:var(--bg-card); border-left-color:var(--accent-blue)`. `.hinge`: `border-left-color:var(--teal)`. Field name: `flex:1; font-size:13px`. Value: `font-size:12px; max-width:100px; overflow:hidden; text-overflow:ellipsis`.
**Interaction:** Click expands/collapses field drawer. Right-click opens context menu.
**Reusable:** Domain-specific but generalizable.

---

### 7.5 Start Here Indicator

**Component:** Start Here Pulse Badge
**Classes:** `.start-here`
**Demos:** Record Inspector
**Visual:** `font-size:10px; font-weight:700; color:var(--teal); background:rgba(38,166,154,0.12); padding:2px 8px; border-radius:3px; animation:pulse 2s ease-in-out infinite; white-space:nowrap; box-shadow:0 0 8px rgba(38,166,154,0.3)`. Pulse: alternates opacity and glow shadow.
**Reusable:** Domain-specific workflow guidance indicator.

---

### 7.6 Document Typography (Contract rendering)

**Component:** Document Content Renderer
**Classes:** `.doc-content`, `.doc-title`, `.dpara`, `.dpara .ent`, `.dpara .dt`, `.sec-div`, `.sec-summary`
**Demos:** Record Inspector
**Visual:** `max-width:780px; margin:0 auto; padding:20px 28px 80px`. Title: `font-size:22px; font-weight:800; letter-spacing:-0.3px`. Para: `font-size:13.5px; line-height:1.75; color:var(--text-secondary)`. Entity highlights `.ent`/`.dt`: `color:var(--accent-blue); text-decoration:underline; text-decoration-color:rgba(0,145,234,0.4); font-weight:600`. Heatmap mode adds colored backgrounds at 3 intensities via `--heatmap-1/2/3`.
**Interaction:** Right-click → dynamic context menu. Heatmap overlay. Spotlight mode.
**Reusable:** Specific to document rendering.

**Component:** Contract Preview Document (ContractGen)
**Classes:** `.preview-doc`, `.doc-title`, `.doc-subtitle`, `.doc-section-title`, `.doc-para`, `.doc-var`, `.doc-var.filled`, `.doc-var.empty`, `.doc-sig-block`, `.doc-sig-party`
**Demos:** Contract Generator
**Visual:** `max-width:720px; margin:0 auto`. `.doc-var.filled`: `background:var(--green-dim); color:var(--green); border-radius:3px; font-family:var(--mono); font-size:12px`. `.doc-var.empty`: `background:var(--amber-dim); color:var(--amber); border:1px dashed var(--amber)`. Sig block: `grid-template-columns:1fr 1fr; gap:32px`.
**Interaction:** Click empty var to jump to field in builder.
**Reusable:** Domain-specific document template renderer.

---

### 7.7 Suggested Fields Section

**Component:** Suggested Fields Section
**Classes:** `.suggested-sec`, `.sug-row`, `.sug-row .icon`, `.sug-row .name`, `.sug-row .pct`, `.sug-row .val`, `.accept-btn`
**Demos:** Record Inspector
**Visual:** `padding:10px 16px; border-top:1px solid var(--border)`. Section `h4`: `font-size:11px; font-weight:700; color:var(--text-muted); text-transform:uppercase`. Row: `display:flex; align-items:center; padding:4px 0; gap:6px; font-size:12px`. Accept button: `font-size:10px; padding:2px 10px; border-radius:3px; border:1px solid var(--green); color:var(--green)`.
**Reusable:** Domain-specific AI suggestion interface.

---

### 7.8 Needs Attention / Not Yet Extracted Sections

**Component:** Needs Attention Section
**Classes:** `.needs-att`, `.na-row`
**Demos:** Record Inspector
**Visual:** `padding:8px 16px; border-top:1px solid var(--border)`. Row: `display:flex; align-items:center; padding:4px 0; gap:8px; font-size:12px`. Icon: `color:var(--amber)`.
**Reusable:** Domain-specific workflow guidance.

**Component:** Not Yet Extracted Section
**Classes:** `.nye-section`, `.nye-row`
**Demos:** Record Inspector
**Visual:** Same padding/structure as `.needs-att`. Row names: `color:var(--text-muted)`. Each row has a `.mini-toggle` for optional extraction.
**Reusable:** Domain-specific.

---

### 7.9 Logo Components

**Component:** Logo Icon (gradient cube)
**Classes:** `.logo-icon-svg` (Triage/Dossier), `.logo-icon` (ContractGen), `.topbar-logo-icon` (Admin)
**Demos:** Triage, Record Inspector, Contract Generator, Admin
**Visual:** `width:26px; height:26px; background:linear-gradient(135deg, #26a69a 0%, #4fc3f7 40%, #7c4dff 100%); border-radius:6px`. Diamond `::after`: `width:10px; height:10px; border:2px solid rgba(255,255,255,0.7); border-radius:2px; transform:rotate(45deg)`. Admin version is 28px x 28px.
**Reusable:** Yes — brand mark.

**Component:** Logo Text
**Classes:** `.logo-text`, `.topbar-brand`
**Demos:** All
**Visual:** `font-weight:700; font-size:14px; color:var(--text-primary); letter-spacing:-0.3px`.
**Reusable:** Yes.

---

### 7.10 User Info / Avatar

**Component:** User Avatar (Sidebar)
**Classes:** `.sb-avatar`, `.user-avatar`, `.ov-avatar`, `.topbar-avatar`, `.ad-user-avatar`
**Demos:** All
**Visual:** Typically `width:26-32px; height:26-32px; border-radius:50%; display:flex; align-items:center; justify-content:center; font-size:9-11px; font-weight:700; color:#fff`. Background varies by user (gradient or solid color). Admin `.topbar-avatar`: 32px, `background:#6a1b9a`.
**Reusable:** Yes.

**Component:** Online Status Indicator
**Classes:** `.ad-user-status`, `.ad-user-status.online`, `.ad-user-status.offline`, `.ov-team-dot`, `.ov-team-dot.active`, `.ov-team-dot.idle`
**Demos:** Admin, Org Overview
**Visual:** `width:8px; height:8px; border-radius:50%`. `.online`/`.active`: `background:var(--green)`. `.offline`/`.idle`: `background:var(--text-muted)`. Org Overview team dot is 6px.
**Reusable:** Yes.

---

## Reusability Summary

| Component | Triage | Admin | ContractGen | OrgOverview | Dossier |
|---|---|---|---|---|---|
| `.top-bar` / `.app-topbar` | Yes | Yes | Yes | Yes | Yes |
| `.sidebar` / `.nav-sidebar` | Yes | Yes | Yes | Yes | Yes |
| `.sb-nav-item` | Yes | Yes | Yes | - | Yes |
| `.sb-section-label` | Yes | Yes | Yes | - | Yes |
| `.toggle` | Yes | Yes | Yes | Yes | Yes |
| `.health-band` / `.sbadge` | Yes | - | - | - | Yes |
| `.ad-status-pill` | - | Yes | - | - | - |
| `.otto-fab` + panel | Yes | - | - | Yes | Yes |
| `.toast` | Yes | - | - | Yes | Yes |
| `.ctx-menu` (context menu) | - | - | - | - | Yes |
| `.field-input` | - | - | Yes | - | - |
| `.header-btn` | Yes | - | Yes | - | Yes |
| `.beta-badge` | Yes | Yes | Yes | - | Yes |
| Health/confidence bars | Yes | Yes | - | Yes | Yes |
| Avatar / user info | Yes | Yes | Yes | Yes | Yes |