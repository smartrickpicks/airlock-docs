# Theme

## Summary

Airlock's visual theme evolves the shared token system used across all four existing demos. The demos already share identical CSS tokens with minor naming variations. Airlock introduces an OLED-darker palette, a refined typography stack, and standardized effects for AI elements and governance indicators.

## Palette Evolution

The existing demos use a shared dark palette. Airlock shifts every surface darker toward true OLED black while increasing accent vibrancy.

### Surface Colors

| Purpose | Demo Token | Demo Value | Airlock Token | Airlock Value | Delta |
|---------|-----------|------------|---------------|---------------|-------|
| Page background | `--bg-page` | #12161f | `--airlock-bg` | #0B0E14 | Darker |
| Panel / sidebar | `--bg-panel` | #1a1f2e | `--airlock-surface` | #0F1219 | Darker |
| Card / elevated | `--bg-card` | #1e2436 | `--airlock-card` | #151923 | Darker |
| Border / divider | (varies) | (varies) | `--airlock-border` | #1E2330 | Standardized |

### Accent Colors

| Purpose | Demo Token | Demo Value | Airlock Token | Airlock Value | Delta |
|---------|-----------|------------|---------------|---------------|-------|
| Primary accent | `--accent-blue` | #0091ea | `--airlock-cyan` | #00D1FF | Brighter, shifted cyan |

### Text Colors

| Purpose | Airlock Token | Value |
|---------|---------------|-------|
| Primary text | `--airlock-text` | #E2E8F0 |
| Secondary / muted text | `--airlock-muted` | #64748B |

### Gate Colors

Governance gate indicators use a fixed semantic palette. These colors are not configurable by theme.

| State | Token | Value | Usage |
|-------|-------|-------|-------|
| Blocked / Critical | `--gate-red` | #EF4444 | Gate dots, alert borders, reject buttons |
| Warning / Needs Attention | `--gate-yellow` | #EAB308 | Gate dots, SLA warning borders |
| In Review / Pending | `--gate-purple` | #A855F7 | Gate dots, review status indicators |
| Clear / Approved | `--gate-green` | #22C55E | Gate dots, approval indicators, health good |
| SLA Urgent / Amber alerts | `--gate-amber` | #F59E0B | SLA countdown, urgency borders, Artifact Focus glow |

---

## Typography

### Font Stack

| Role | Font Family | Fallback | Usage |
|------|-------------|----------|-------|
| Monospace | Fira Code | `monospace` | Timestamps, IDs, SLA countdowns, code blocks, health percentages |
| UI Text | Fira Sans | `-apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif` | All labels, body text, headers, buttons, navigation |

### Type Scale

| Element | Font | Size | Weight | Color |
|---------|------|------|--------|-------|
| Module header | Fira Sans | 15px | 600 (semibold) | `--airlock-text` |
| Section header | Fira Sans | 13px | 600 (semibold) | `--airlock-text` |
| Group header | Fira Sans | 11px | 500 (medium), uppercase, letter-spacing 0.5px | `--airlock-muted` |
| Body text | Fira Sans | 13px | 400 (regular) | `--airlock-text` |
| Secondary text | Fira Sans | 12px | 400 (regular) | `--airlock-muted` |
| Timestamp | Fira Code | 11px | 400 (regular) | `--airlock-muted` |
| SLA countdown (large) | Fira Code | 15px | 700 (bold) | Contextual (green/amber/red) |
| Badge text | Fira Sans | 10-11px | 700 (bold) | White on red |
| Button text | Fira Sans | 13px | 600 (semibold) | Contextual |

---

## Icon Set

**Lucide React** is the sole icon library. No emojis in production UI.

| Category | Icons Used |
|----------|-----------|
| Module Bar | FileText, Users, CheckSquare, Calendar, FolderOpen, Settings |
| Navigation | ChevronDown, ChevronRight, Search, Plus, PlusCircle, AlertTriangle |
| Control tabs | GitBranch, Clock, CheckCircle, ScrollText, Bot |
| Actions | Shield, X, Pin, ExternalLink, Copy, Archive, Trash |
| Status | Circle (filled, for gate dots), TrendingUp, TrendingDown |

Default icon size: 16px inline, 20px in headers, 24px in collapsed panel mode, 48px containers in Module Bar.

---

## Effects

### AI Element Glow

AI-generated content (suggestions, agent messages, automated insights) receives a subtle cyan glow to distinguish it from human-authored content.

| Property | Value |
|----------|-------|
| Text shadow | `0 0 10px rgba(0, 209, 255, 0.3)` |
| Border | Left 3px solid `--airlock-cyan` (#00D1FF) on AI signal cards |
| Background | Slight cyan tint: `rgba(0, 209, 255, 0.03)` on AI message bubbles |

The glow is intentionally subtle. It should be perceptible but not distracting.

### Color-Coded Left Borders

Signal cards, channel items, and status indicators use left borders to convey semantic meaning.

| Context | Border Width | Color Source |
|---------|-------------|-------------|
| Signal cards | 3px | Card type color (see triptych.md Signal Card Left Border Colors) |
| Active channel | 2px | `--airlock-cyan` |
| Active module | 3px | `--airlock-cyan` |

### Gate Indicator Dots

Small filled circles used in channel items, signal cards, and the Governance Bar.

| Property | Value |
|----------|-------|
| Size | 8px diameter |
| Shape | Circle, filled |
| Color | Gate color based on state (red/yellow/purple/green) |
| Border | None (sits directly on card background) |

### SLA Countdown

| Property | Value |
|----------|-------|
| Font | Fira Code |
| Size | 15px bold (Governance Bar), 13px regular (Control panel), 11px (inline) |
| Color | Green (#22C55E) > 50%, Amber (#F59E0B) 25-50%, Red (#EF4444) < 25% |
| Pulse | When < 10% remaining, gentle pulse animation (opacity 0.8-1.0, 2s cycle) |

---

## Transitions and Motion

| Category | Duration | Easing | Usage |
|----------|----------|--------|-------|
| Micro-interactions | 150ms | ease | Hover states, button presses, icon color changes |
| Panel transitions | 200-300ms | ease-in-out | View state changes, panel resize, sidebar content switch |
| Tooltip appear | 300ms delay, 150ms fade | ease | Module Bar tooltips, hover info |
| Content crossfade | 150ms | ease | Panel content swaps during state changes |
| Notification enter | 300ms | ease-out | Signal cards sliding in |
| Collapse/expand | 200ms | ease | Channel groups, panel collapse to icon-only |

---

## Demo Shell Theming Strategy

### Problem

All four existing demos share identical CSS tokens with slight naming differences. Each demo duplicates the full token set.

### Solution

Extract a shared base and let Airlock override.

| File | Purpose | Contents |
|------|---------|----------|
| `theme.base.css` | Shared token foundation | All common tokens using demo naming (`--bg-page`, `--bg-panel`, etc.) |
| `theme.airlock.css` | Airlock overrides | Maps Airlock tokens (`--airlock-bg`, etc.) and overrides base values to the darker palette |

### Migration Path

1. Extract `theme.base.css` from existing demo CSS (tokens only, no component styles)
2. Each demo's existing CSS file imports `theme.base.css` and adds component-specific styles
3. Airlock links `theme.base.css` then layers `theme.airlock.css` on top
4. Component styles reference Airlock tokens (`--airlock-bg`) which resolve to the darker values
5. Existing demos continue working unchanged -- their tokens remain in `theme.base.css`

### Token Mapping

In `theme.airlock.css`, Airlock tokens are defined as overrides:

```
/* Conceptual -- not implementation code */
--airlock-bg:      #0B0E14;    /* overrides --bg-page: #12161f */
--airlock-surface:  #0F1219;    /* overrides --bg-panel: #1a1f2e */
--airlock-card:     #151923;    /* overrides --bg-card: #1e2436 */
--airlock-cyan:     #00D1FF;    /* overrides --accent-blue: #0091ea */
```

This approach ensures:
- Zero breaking changes to existing demos
- Single source of truth for shared tokens
- Clean override path for Airlock's darker palette
- Future themes (light mode, high-contrast) follow the same pattern
