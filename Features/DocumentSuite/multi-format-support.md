# Multi-Format Document Support — Brainstorm

> **Status:** BRAINSTORM — Expanding document ingestion beyond PDF-only.

> **Context:** OrcestrateOS currently processes only PDFs. Airlock needs to handle the full spectrum of business documents that customers send — PowerPoints, Excel spreadsheets, Word docs, plain text, Markdown, and even EPUBs.

---

## The Problem

Customers don't always send PDFs. Real-world contract packages include:
- Word documents (.docx) — drafts, redlines, templates
- Excel spreadsheets (.xlsx, .csv) — financial schedules, rate cards, pricing matrices
- PowerPoint presentations (.pptx) — deal summaries, proposals, pitch decks
- Plain text (.txt) — email body exports, notes
- Markdown (.md) — technical documentation, specs
- EPUBs (.epub) — digital publications, long-form agreements
- Rich text (.rtf) — legacy documents
- Images (.png, .jpg, .tiff) — scanned documents, signatures

## Format Support Matrix

| Format | Extension(s) | Extraction Strategy | Text Extraction Library | Preview Strategy | Priority |
|--------|-------------|--------------------|-----------------------|-----------------|----------|
| PDF | .pdf | OCR + text layer extraction | PDF.js (text layer), Tesseract (OCR) | PDF.js viewer | P0 (exists) |
| Word | .docx, .doc | XML parsing (docx), LibreOffice convert (doc) | python-docx, mammoth | Convert to HTML → TipTap render | P1 |
| Excel | .xlsx, .xls, .csv | Cell-by-cell parsing, structured data extraction | openpyxl, pandas | HTML table render | P1 |
| PowerPoint | .pptx, .ppt | Slide-by-slide text + notes extraction | python-pptx | Slide carousel viewer | P2 |
| Plain Text | .txt | Direct text ingestion | Built-in | Monospace viewer | P1 |
| Markdown | .md | Direct text ingestion with structure preservation | markdown-it | TipTap render (Markdown mode) | P1 |
| EPUB | .epub | HTML chapter extraction | ebooklib | Chapter-by-chapter viewer | P3 |
| RTF | .rtf | Convert to text/HTML | striprtf, LibreOffice | TipTap render | P3 |
| Images | .png, .jpg, .tiff | OCR only | Tesseract, EasyOCR | Image viewer with OCR overlay | P2 |

## Conversion Pipeline

### Architecture

```
Upload → Format Detection → Converter → Normalized Text → Extractors → Preflight
                                |
                                +→ Preview Render (format-specific viewer)
```

### Format Detection

Use python-magic (libmagic) for MIME type detection, not file extension alone. Extensions can lie.

```python
import magic

def detect_format(file_bytes: bytes, filename: str) -> str:
    mime = magic.from_buffer(file_bytes, mime=True)
    FORMAT_MAP = {
        'application/pdf': 'pdf',
        'application/vnd.openxmlformats-officedocument.wordprocessingml.document': 'docx',
        'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet': 'xlsx',
        'application/vnd.openxmlformats-officedocument.presentationml.presentation': 'pptx',
        'text/plain': 'txt',
        'text/markdown': 'md',
        'application/epub+zip': 'epub',
        'application/rtf': 'rtf',
        'image/png': 'png',
        'image/jpeg': 'jpg',
        'image/tiff': 'tiff',
    }
    return FORMAT_MAP.get(mime, 'unknown')
```

### Conversion to Normalized Text

Every format converts to a common intermediate: plain text (for extraction) + structured HTML (for preview).

| Format | Text Extraction | HTML Conversion | Notes |
|--------|----------------|-----------------|-------|
| PDF | PDF.js text layer + OCR fallback | N/A (PDF.js renders natively) | Existing pipeline |
| DOCX | python-docx → paragraph text | mammoth → clean HTML | Preserves headings, lists, tables |
| XLSX | openpyxl → cell values | pandas → HTML table | Each sheet = separate page |
| PPTX | python-pptx → slide text + notes | Custom HTML template | Slide layout preserved |
| TXT | Direct read | Wrap in `<pre>` | Encoding detection via chardet |
| MD | Direct read | markdown-it → HTML | Preserves structure |
| EPUB | ebooklib → chapter HTML → text | Chapter HTML directly | Respect chapter boundaries |
| RTF | striprtf → text | LibreOffice → HTML | Legacy format support |
| Images | Tesseract/EasyOCR → text | Image tag + OCR overlay | Same as existing OCR pipeline |

## Extraction Compatibility

The extraction pipeline (7 extractors + dispatcher) operates on **plain text**. As long as the conversion produces clean text, all extractors work without modification.

Key considerations:
- **Table data** (Excel, Word tables): Convert to structured text with clear delimiters so extractors can identify amounts, dates, etc.
- **Slide text** (PowerPoint): Concatenate slide text in order. Notes often contain more detail than slides.
- **Multi-sheet Excel**: Each sheet processed separately. Sheet names used as section headers.
- **Headers/footers**: Strip repeated headers/footers to avoid false extraction matches.
- **Encoding**: Use chardet for encoding detection. Convert everything to UTF-8 before extraction.

## Preview Viewers

Each format needs its own preview component in the Orchestrate panel:

| Format | Viewer Component | Features |
|--------|-----------------|----------|
| PDF | PDFViewer (existing) | Page navigation, zoom, text selection, annotation overlay |
| DOCX/RTF/MD | TipTapViewer | Read-only TipTap render, annotation overlay on converted HTML |
| XLSX | SpreadsheetViewer | Tab bar for sheets, sortable columns, cell highlighting for extracted values |
| PPTX | SlideViewer | Slide carousel, thumbnail strip, speaker notes panel |
| TXT | MonospaceViewer | Line numbers, search, syntax highlighting (if code) |
| EPUB | ChapterViewer | Table of contents sidebar, chapter navigation, bookmark support |
| Images | ImageViewer | Zoom, pan, OCR text overlay toggle |

## Feature Control Plane

Each format is independently toggleable:

| Flag | Default | Description |
|------|---------|-------------|
| ENABLE_FORMAT_PDF | true | PDF processing (core, always on) |
| ENABLE_FORMAT_DOCX | true | Word document processing |
| ENABLE_FORMAT_XLSX | true | Excel spreadsheet processing |
| ENABLE_FORMAT_PPTX | false | PowerPoint processing |
| ENABLE_FORMAT_TXT | true | Plain text processing |
| ENABLE_FORMAT_MD | true | Markdown processing |
| ENABLE_FORMAT_EPUB | false | EPUB processing |
| ENABLE_FORMAT_RTF | false | RTF processing |
| ENABLE_FORMAT_IMAGE | true | Image OCR processing |

## Python Dependencies (New)

| Library | Purpose | License |
|---------|---------|---------|
| python-docx | DOCX text extraction | MIT |
| mammoth | DOCX → clean HTML | BSD |
| openpyxl | XLSX cell parsing | MIT |
| python-pptx | PPTX slide extraction | MIT |
| ebooklib | EPUB chapter extraction | AGPL-3.0 (evaluate alternative) |
| striprtf | RTF → plain text | BSD |
| python-magic | MIME type detection | MIT |
| chardet | Encoding detection | LGPL |
| markdown-it-py | Markdown → HTML | MIT |
| pandas | Excel → HTML tables | BSD |

**Note on ebooklib:** AGPL license may be problematic. Alternative: use zipfile to extract EPUB (it's a ZIP of HTML files) and parse directly. EPUBs are just XHTML inside a ZIP container.

## Capacity Planning

| Format | Avg Processing Time | Memory Usage | Notes |
|--------|-------------------|-------------|-------|
| PDF (text) | 2-5s | Low | Existing baseline |
| PDF (OCR) | 15-60s per page | High (Tesseract) | CPU-intensive |
| DOCX | 1-3s | Low | Fast XML parsing |
| XLSX | 1-10s | Medium | Depends on sheet count and cell count |
| PPTX | 2-5s | Low | Slide count dependent |
| TXT/MD | <1s | Minimal | Direct read |
| EPUB | 2-8s | Low | Chapter count dependent |
| Images | 10-30s per image | High (OCR) | Same as PDF OCR |

## Document Loss / Fallback Strategy

What happens when conversion fails:
1. Store the original file regardless of conversion success
2. Log conversion failure as a triage item (severity: warning)
3. Allow manual text paste as fallback ("Conversion failed — paste the text content here")
4. Queue for retry with different conversion strategy
5. Feature Control Plane shows conversion success rate per format

---

## Related Specs

- **[Document Suite](./overview.md)** — Parent document engine spec
- **[Record Inspector / Document Viewer](../Record%20Inspector/document-viewer.md)** — PDF viewer spec
- **[Feature Control Plane](../FeatureControlPlane/overview.md)** — Format-level toggles
- **[Batch Processing](../BatchProcessing/overview.md)** — Multi-format batch ingestion
