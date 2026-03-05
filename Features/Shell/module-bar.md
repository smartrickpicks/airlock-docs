# Module Bar

## Summary

The Module Bar is the far-left vertical strip of the Airlock shell. It functions identically to Discord's server icon sidebar -- a narrow column of module icons that the user clicks to switch context. The active module determines what appears in the Channel Sidebar.

## Dimensions

| Property | Value |
|----------|-------|
| Width | 72px fixed |
| Height | 100vh |
| Position | Far left, pinned |
| Background | `--airlock-bg` (#0B0E14) |
| Border | 1px right border, `--airlock-border` (#1E2330) |

## Module Icons

Icons are stacked vertically, centered within the 72px column. Each icon sits inside a 48px circular container.

| Module | Lucide Icon | Order (default) |
|--------|-------------|-----------------|
| Contracts | FileText | 1 |
| CRM | Users | 2 |
| Tasks | CheckSquare | 3 |
| Calendar | Calendar | 4 |
| Documents | FolderOpen | 5 |
| Admin | Settings | 6 |

### Icon States

| State | Visual Treatment |
|-------|-----------------|
| Default | Icon in `--airlock-muted` (#64748B), circular container with `--airlock-surface` (#0F1219) background |
| Hover | Icon brightens to `--airlock-text` (#E2E8F0), container background lightens to `--airlock-card` (#151923), tooltip appears |
| Active | 3px left border in `--airlock-cyan` (#00D1FF), icon in `--airlock-cyan`, container background `--airlock-card` |
| Drag | Slight scale (1.05), drop shadow, reduced opacity on source position |

### Icon Container

- Size: 48px x 48px
- Border radius: 16px (default), transitions to 12px on hover (Discord-like squircle effect)
- Margin: 8px auto (vertically between icons)
- Transition: border-radius 150ms, background 150ms

## Active Module Indicator

A vertical cyan bar on the left edge of the active module icon.

| Property | Value |
|----------|-------|
| Width | 3px |
| Height | 36px (centered on the icon) |
| Color | `--airlock-cyan` (#00D1FF) |
| Border radius | 0 2px 2px 0 |
| Transition | height 200ms ease |

On hover over an inactive module, a shorter indicator (8px tall) appears as a preview.

## Tooltip

Appears on hover after a 300ms delay. Positioned to the right of the icon.

| Property | Value |
|----------|-------|
| Position | Right of icon, 8px gap |
| Background | `--airlock-card` (#151923) |
| Border | 1px solid `--airlock-border` (#1E2330) |
| Border radius | 6px |
| Padding | 8px 12px |
| Font | Fira Sans, 13px, `--airlock-text` |
| Shadow | 0 4px 12px rgba(0,0,0,0.4) |

Tooltip content: Module name on line 1. If unread badge count > 0, show count on line 2 in `--airlock-muted`.

Example:
```
Contracts
3 unread
```

## Notification Badge

A red indicator dot overlaid on the top-right corner of the module icon container.

| Property | Value |
|----------|-------|
| Position | Top-right of icon container, offset -4px/-4px |
| Size | 18px diameter (with count), 10px diameter (dot-only if count = 0 but has activity) |
| Background | #EF4444 |
| Text | White, Fira Sans, 11px bold |
| Border | 2px solid `--airlock-bg` (to create cutout effect) |
| Max display | "99+" for counts over 99 |

Badge visibility rules:
- Badge appears when there are unread notifications, pending actions, or gate alerts within that module
- Badge count aggregates across all channels in the module
- Clicking the module clears the badge after the user views the relevant content (not on click alone)

## Reordering

Module icons support drag-and-drop reordering.

- Long press (200ms) or explicit grab handle initiates drag
- Drop zones appear between other icons (2px cyan line indicator)
- Order persists to user preferences (local storage, synced to backend)
- Admin module always remains at the bottom (not reorderable)

## Bottom Section

The bottom of the Module Bar contains user controls, separated from module icons by a 1px horizontal divider in `--airlock-border`.

| Element | Details |
|---------|---------|
| User Avatar | 36px circle, shows user profile image or initials. Click opens user menu (profile, preferences, logout). |
| Settings Gear | 20px Lucide Settings icon in `--airlock-muted`. Click opens application settings panel in the triptych Orchestrate zone. |

## Responsive Behavior

The Module Bar does not collapse or hide. At all viewport widths where the application is supported (min 1024px), the Module Bar remains visible at 72px.
