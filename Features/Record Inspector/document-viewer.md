# Record Inspector -- Document Viewer

## Purpose

The Document Viewer renders contract PDFs inside the Orchestrate panel and overlays extraction annotations directly on the document surface. It is the bridge between raw source material and structured field data, allowing users to verify extracted values against their original context.

---

## Access Points

| Trigger | Behavior |
|---|---|
| "Document" tab in Orchestrate panel | Switches from Record Inspector to Document Viewer within the same panel |
| Click file attachment in contract channel | Auto-opens Document Viewer with that file loaded |
| Click evidence context in a field card drawer | Switches to Document Viewer and scrolls to the source region |

---

## Layout Anatomy

The Document Viewer is divided into three vertical zones.

### 1. Page Navigation Bar (Top)

| Element | Behavior |
|---|---|
| Page indicator | Displays "Page X of Y" -- updates on scroll |
| Prev / Next buttons | Navigate one page at a time |
| Page thumbnails strip | Horizontal scrolling strip of page thumbnails; click to jump to page |
| Zoom controls | Three preset modes plus custom input |

**Zoom Modes**

| Mode | Description |
|---|---|
| Fit Width | Scales page to fill horizontal space (default) |
| Fit Page | Scales entire page to fit within viewport |
| Custom % | Manual zoom level via text input or +/- controls |

### 2. PDF Render Area (Center)

- Full rendered PDF pages with continuous vertical scroll.
- Scroll position drives the page indicator in the navigation bar.
- Supports smooth scrolling and scroll-snap to page boundaries (optional, can be toggled).

### 3. Annotation Overlay (On Document Surface)

Semi-transparent colored highlights rendered on top of the PDF at extracted field locations.

| Highlight Color | Field Status | Meaning |
|---|---|---|
| Green | Pass | Field extracted and validated successfully |
| Amber | Review | Field extracted but requires human review |
| Red | Fail | Field extraction failed validation |
| Cyan | AI-detected | Pattern identified by AI that is not tied to a predefined anchor |

Annotations are interactive -- clicking one opens the corresponding field card drawer in Record Inspector.

---

## Cross-Linking

### Document to Record Inspector

| Action in Document Viewer | Effect in Record Inspector |
|---|---|
| Click annotation highlight | Opens the corresponding field card and expands its drawer |
| Right-click annotation | Context menu with triage and extraction options (see below) |

### Record Inspector to Document

| Action in Record Inspector | Effect in Document Viewer |
|---|---|
| Click field card (evidence context) | Document Viewer scrolls to the page and region where the field was extracted |
| Spotlight mode activated | If side-by-side is active, Document Viewer scrolls to the spotlighted field's source |

The linking is bidirectional: selecting a field in either view highlights it in the other.

---

## Side-by-Side Mode

When Artifact Focus is active, the Orchestrate panel can split into two panes:

| Left Pane | Right Pane |
|---|---|
| Document Viewer | Record Inspector |

- Split is horizontal (left/right), not vertical.
- Draggable divider between panes.
- Both panes scroll independently but maintain field linking.
- Exiting Artifact Focus collapses back to single-pane tabbed view.

---

## Corruption Detection

| Condition | UI Response |
|---|---|
| Mojibake detected in extracted text | Warning banner at top of render area |
| Control characters in document | Warning banner at top of render area |

**Banner format:** "X corruption issues detected on Y pages"

- Banner is dismissible but reappears if user navigates to an affected page.
- Corruption regions are outlined with a dashed red border on the document surface.
- Affected field cards in Record Inspector show a corruption indicator icon next to the confidence badge.

---

## Scroll Sync

| Event | Behavior |
|---|---|
| User scrolls Document Viewer | Tab bar updates a scroll progress indicator (thin progress bar or page fraction) |
| User clicks page thumbnail | Document jumps to that page; progress indicator updates |
| Field link triggered from Record Inspector | Document scrolls to target; progress indicator updates |

Scroll sync is one-directional for the progress indicator (Document Viewer drives it). Field linking is bidirectional.

---

## Context Menu (Right-Click)

Right-clicking on the document surface or an annotation opens a context menu.

| Menu Item | Action |
|---|---|
| Create triage item for this region | Opens triage item creation with the selected region pre-attached as context |
| Extract text from selection | Runs OCR or text extraction on the user-selected area and copies to clipboard |
| Add annotation | Creates a manual annotation (user-defined color and label) at the clicked location |

Context menu items are only active when the click target is valid (selection exists for "Extract text", region is identifiable for "Create triage item").

---

## Keyboard Shortcuts

| Key | Action |
|---|---|
| Left Arrow | Previous page |
| Right Arrow | Next page |
| `+` / `=` | Zoom in |
| `-` | Zoom out |
| `Ctrl+F` / `Cmd+F` | Open find-in-document search bar |
| `Escape` | Close find bar or exit side-by-side mode |

---

## State Management

| State | Stored Where | Persistence |
|---|---|---|
| Current page number | Local component state | Resets on file change |
| Zoom level | User preferences | Persists across sessions |
| Scroll position | Local component state | Resets on file change |
| Annotation visibility | Local component state | Defaults to visible; toggled per session |
| Side-by-side active | Layout state (Artifact Focus) | Resets when Artifact Focus exits |
