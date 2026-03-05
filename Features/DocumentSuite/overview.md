# Document Suite

> Parent module for all document-related functionality in Airlock. Provides the shared document engine that Contract Generator, Record Inspector, Patch Authoring, and the Documents module all consume.

## Decision

**Full template system (maximum)** -- Not just tables and headers, but template variables, live data binding, version diffing inline, and annotation overlays. The contract generator and document viewer merge into one composable engine.

## Framework Choice

| Engine | Library | Role | License |
|--------|---------|------|---------|
| **Viewer** | PDF.js (Mozilla) | Render uploaded contract PDFs with annotation overlay | Apache 2.0 |
| **Editor** | TipTap (ProseMirror-based) | Rich text editing, contract generation, clause insertion | MIT |

### Why TipTap

- Free, MIT license, excellent React/Next.js support
- Custom block types (financial tables, clause blocks, annotation blocks)
- Exports to HTML/Markdown/JSON (matches contract generator output format)
- Extensible -- custom node types for everything unique to Airlock
- Collaborative editing support (future feature)
- Themeable -- can match the Airlock OLED palette
- Active community and ecosystem of extensions

### Why PDF.js

- Mozilla's open-source PDF renderer, industry standard
- Text layer for search and selection
- Annotation overlay support
- react-pdf wrapper available for React/Next.js integration
- Page-by-page rendering with zoom controls

## Two Engines, Multiple Consumers

```
Document Suite
  |
  +-- PDF Viewer Engine (PDF.js)
  |     |-- Record Inspector: Document Viewer tab
  |     |-- Batch Processing: PDF preview
  |     |-- Export: Annotated PDF generation
  |
  +-- Rich Editor Engine (TipTap)
        |-- Contract Generator: Live preview panel
        |-- Patch Workflow: Action Focus editor
        |-- Documents Module: Document creation/editing
        |-- Templates: Template variable system
```

## Embeddable Components (Custom TipTap Blocks)

These are custom node types that can be inserted into any document within the editor:

| Component | Description | Used By |
|-----------|-------------|---------|
| **Financial Table** | Royalty rates, splits, payment terms in structured table | Contract Generator, Record Inspector |
| **Section Header** | Contract section header with clause ID reference | Contract Generator |
| **Signature Block** | Signatory fields with date/title/name | Contract Generator |
| **Clause Block** | Pre-approved legal prose from unified v2 clause library (188 clauses), with risk badge (`standard`/`elevated`/`critical`/`high`). Clause IDs use `CATEGORY-SECTION-TYPE-VERSION` format (e.g., `GEN-RECITALS-DIST-V1`). Variables use `{{CHECK_CODE}}` syntax. See `ContractGenerator/overview.md` for full schema. | Contract Generator |
| **Template Variable** | `{{territory}}`, `{{term}}` -- live-bound to form data | Contract Generator, Templates |
| **Data Binding Field** | Pulls value from extraction results, updates live | Record Inspector, Patch Workflow |
| **Annotation Marker** | Colored highlight with linked field card reference | Record Inspector (Document Viewer) |
| **Diff Block** | Before/after comparison with cyan highlights | Patch Workflow (diff view) |
| **Chart Embed** | Sparkline, bar chart, or confidence distribution | Admin Dashboard, Record Inspector |
| **Formula Field** | Calculated value from other fields (e.g., total = rate * quantity) | Contract Generator |
| **Conditional Section** | Shows/hides based on contract type or field values | Contract Generator |
| **Version Marker** | Inline version indicator for diff tracking | Patch Workflow |

## Template Variable System

Template variables are the bridge between form data and generated documents.

### Syntax

```
{{VARIABLE_NAME}}        -- Simple substitution
{{VARIABLE_NAME|default}} -- With default value
{{#IF condition}}...{{/IF}} -- Conditional block
{{#EACH items}}...{{/EACH}} -- Repeating block (for schedules, parties)
```

### Variable Sources

| Source | Example Variables | Binding |
|--------|------------------|---------|
| **Form fields** | `{{OPP_TERRITORY}}`, `{{OPP_TERM_LENGTH}}` | User input in builder panel |
| **Extraction results** | `{{EXTRACTED.territory}}`, `{{EXTRACTED.effective_date}}` | Preflight extraction data |
| **Entity resolution** | `{{ENTITY.legal_name}}`, `{{ENTITY.counterparty}}` | Resolver match results |
| **System** | `{{TODAY}}`, `{{WORKSPACE_NAME}}`, `{{AUTHOR}}` | Auto-populated |
| **Calculated** | `{{TOTAL_ADVANCE}}`, `{{NET_RATE}}` | Formula from other variables |

### Live Data Binding

When viewing a contract in the Record Inspector, template variables in the generated document update live from extraction results:
- Green highlight: variable matches extraction (confirmed)
- Amber highlight: variable differs from extraction (needs review)
- Red highlight: variable has no extraction data (missing)

## Version Diffing

Inline diff view for comparing document versions:
- **Original** vs **Revised** side-by-side or inline
- Cyan highlights for additions
- Red strikethrough for deletions
- Amber background for modifications
- Each diff block links to the patch that caused the change
- Version timeline scrubber (slider to view historical versions)

## Annotation Overlay (PDF Viewer)

For uploaded PDFs rendered via PDF.js:
- Semi-transparent colored rectangles over extracted field locations
- Color mapping: green (pass), amber (review), red (fail), cyan (AI-detected)
- Click annotation to open field card drawer in Record Inspector
- Bidirectional linking: click field card to scroll PDF to extraction location
- Corruption indicators: warning banner for mojibake/control char pages
- Context menu: "Create triage item", "Extract text from selection", "Add annotation"

## Integration Points

| Feature | Uses Viewer | Uses Editor | Custom Blocks |
|---------|------------|-------------|---------------|
| Contract Generator | No | Yes (preview panel) | Clause, Financial Table, Template Variable, Conditional, Signature |
| Record Inspector | Yes (Document tab) | No | Annotation Marker (overlay on PDF) |
| Patch Workflow | No | Yes (diff preview) | Diff Block, Version Marker, Data Binding Field |
| Documents Module | Yes (file browser) | Yes (creation) | All blocks available |
| Export | Yes (annotated PDF) | Yes (formatted output) | All blocks for rendering |

## OrcestrateOS Source Mapping

| OrcestrateOS Feature | Document Suite Component |
|---------------------|------------------------|
| Contract Generator preview panel | TipTap editor with clause blocks + template variables |
| Record Inspector document viewer | PDF.js viewer with annotation overlay |
| Patch diff view | TipTap editor with diff blocks + version markers |
| Financial tables in contracts | Financial Table custom block |
| Extraction highlights | Annotation overlay on PDF.js |

## Theming

The editor and viewer must match the Airlock theme:
- Editor background: `--airlock-card` (#151923)
- Editor text: `--airlock-text` (#E2E8F0)
- Editor borders: `--airlock-border` (#1E2330)
- Code/monospace: Fira Code
- Body text: Fira Sans
- Clause blocks: subtle border with risk-level color coding
- Template variables: inline badge style (like a chip/pill)

## Feature Control Plane

| Capability | Toggle | Calibration |
|-----------|--------|-------------|
| PDF viewer | On/off | Zoom default, annotation opacity |
| Rich editor | On/off | Max document size, block types enabled |
| Template variables | On/off | Allowed variable sources |
| Clause library | On/off | Risk level filtering |
| Diff view | On/off | Diff algorithm (character/word/line) |
| Live data binding | On/off | Refresh interval, binding sources |

## Dependencies

- `@tiptap/react` + `@tiptap/starter-kit` -- Core editor
- `@tiptap/extension-table` -- Table support
- `@tiptap/extension-placeholder` -- Template variable placeholders
- `@tiptap/extension-highlight` -- Annotation highlights
- `pdfjs-dist` -- PDF rendering
- `react-pdf` -- React wrapper for PDF.js
- `diff` or `jsdiff` -- Text diffing for version comparison
