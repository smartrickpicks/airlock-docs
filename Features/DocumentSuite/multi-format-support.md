# Multi-Format Document Support

> Beyond PDF: DOCX, XLSX, PPTX, TXT, MD, EPUB, RTF, images with per-format converters, viewers, and feature flags.

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

## Per-Format Conversion Details

### DOCX Conversion

**Pipeline:** `python-docx` extracts paragraphs, tables, and headers → `mammoth` produces clean HTML for preview.

| Element | Text Extraction | Notes |
|---------|----------------|-------|
| Paragraphs | Direct text, preserving line breaks | Style info (heading level) used for structure |
| Tables | Row-by-row, cells joined with `\t`, rows with `\n` | Table headers detected via bold/shading |
| Headers/Footers | Extracted separately, marked as `[HEADER]`/`[FOOTER]` | Stripped from extraction input to avoid false matches |
| Track changes | Accepted text only (ignore deletions) | Revision history available in preview but not extracted |
| Embedded images | OCR via Tesseract if `ENABLE_FORMAT_IMAGE` is on | Images extracted to temp files, processed separately |
| Hyperlinks | Link text extracted, URL stored in metadata | Links not extracted as field values |

**Edge case — `.doc` (legacy Word):** No direct Python parser. Convert to DOCX via LibreOffice headless: `libreoffice --headless --convert-to docx input.doc`. If LibreOffice is unavailable, reject with triage item "Legacy .doc format — please convert to .docx".

### XLSX Conversion

**Pipeline:** `openpyxl` reads cells → `pandas` renders HTML tables for preview.

| Element | Text Extraction | Notes |
|---------|----------------|-------|
| Cell values | All cells with data, row-by-row | Numbers formatted as strings |
| Formulas | Evaluated result (not formula text) | Formula text available in metadata |
| Sheet names | Used as section headers: `[Sheet: Rate Card]` | Each sheet = separate extraction page |
| Merged cells | Unmerged with value propagated to all cells | Preserves table structure |
| Named ranges | Range name used as field label | Useful for anchored extraction |
| Charts | Skipped (not text) | Chart title text extracted if available |

**Sheet ordering:** Sheets processed in workbook order. First sheet is primary. If a sheet name contains keywords ("Schedule", "Rates", "Pricing", "Terms"), it's promoted to the top of the extraction text.

**Multi-sheet strategy:** Each sheet produces its own text block with a clear delimiter:
```
=== Sheet: Deal Terms (1 of 3) ===
[cell data]

=== Sheet: Rate Card (2 of 3) ===
[cell data]
```

### PPTX Conversion

**Pipeline:** `python-pptx` extracts slide text + speaker notes in presentation order.

| Element | Text Extraction | Notes |
|---------|----------------|-------|
| Slide text | All text frames, concatenated per slide | Title, subtitle, bullet points |
| Speaker notes | Appended after slide text with `[NOTES]` prefix | Often contains more detail than slides |
| Tables on slides | Same as DOCX table extraction | Rendered as text grid |
| SmartArt | Underlying text extracted | Visual layout lost |
| Embedded media | Skipped | Video/audio ignored |

**Slide ordering:** Always in presentation order (slide 1, 2, 3...). Each slide is a "page" in the extraction text.

### Image OCR Strategy

**Pipeline:** Tesseract for printed text, EasyOCR as fallback for handwriting.

| Scenario | Strategy | Library |
|----------|----------|---------|
| **Scanned PDF (no text layer)** | Detect via empty text layer → OCR all pages | Tesseract |
| **Mixed PDF (some pages scanned)** | Per-page text layer check → OCR only blank pages | Tesseract |
| **Standalone image files** | Full OCR | Tesseract (first), EasyOCR (fallback) |
| **Low-quality scans** | Pre-process: deskew, contrast enhancement, noise removal | Pillow + Tesseract |
| **Handwritten content** | EasyOCR with handwriting model | EasyOCR |

**OCR confidence thresholds:**

| Confidence | Behavior |
|------------|----------|
| >= 90% | Accept text, no warning |
| 60-89% | Accept text, triage item "Low OCR confidence on page N" |
| < 60% | Accept text with `[OCR_UNCERTAIN]` tag, triage item "Very low OCR confidence" |
| 0% (blank) | Skip page, triage item "Blank or unreadable page N" |

**Performance:** OCR is the bottleneck. For batch processing:
- Max 5 concurrent OCR jobs (matches batch processor semaphore)
- Timeout per page: 30 seconds
- Timeout per document: 5 minutes
- Oversized images (>10MP) are downsampled before OCR

---

## Encoding Detection & Normalization

All text-based formats (TXT, MD, RTF, CSV) go through encoding detection before extraction:

```python
import chardet

def normalize_text(raw_bytes: bytes) -> str:
    detected = chardet.detect(raw_bytes)
    encoding = detected['encoding'] or 'utf-8'
    confidence = detected['confidence']

    if confidence < 0.5:
        # Try common encodings in order
        for enc in ['utf-8', 'latin-1', 'cp1252', 'ascii']:
            try:
                return raw_bytes.decode(enc)
            except UnicodeDecodeError:
                continue
        # Last resort: decode with replacement chars
        return raw_bytes.decode('utf-8', errors='replace')

    return raw_bytes.decode(encoding)
```

**Mojibake detection:** Same as existing PDF pipeline — count control characters and encoding artifacts. If mojibake > 5% of text, create RED gate triage item.

---

## Performance SLA Targets

| Format | Target (p50) | Target (p95) | Timeout | Action on Timeout |
|--------|-------------|-------------|---------|-------------------|
| PDF (text) | 3s | 8s | 60s | Retry once, then fail |
| PDF (OCR) | 30s | 120s | 300s | Retry once, then fail |
| DOCX | 2s | 5s | 30s | Retry once |
| XLSX | 3s | 15s | 60s | Retry once |
| PPTX | 3s | 8s | 30s | Retry once |
| TXT/MD | 0.5s | 1s | 10s | Retry once |
| Images | 15s | 45s | 120s | Retry once |

**Monitoring:** Feature Control Plane shows per-format conversion success rate, average processing time, and failure count. Alert if success rate drops below 95% for any enabled format.

---

## EPUB Alternative (License Concern)

`ebooklib` is AGPL-3.0, which requires distributing Airlock's source code if used. Instead, use direct ZIP parsing:

```python
import zipfile
from lxml import etree

def extract_epub_text(file_bytes: bytes) -> str:
    """Extract text from EPUB without ebooklib (avoids AGPL)."""
    chapters = []
    with zipfile.ZipFile(io.BytesIO(file_bytes)) as zf:
        # Parse container.xml to find rootfile
        container = etree.parse(zf.open('META-INF/container.xml'))
        rootfile = container.find('.//{urn:oasis:names:tc:opendocument:xmlns:container}rootfile')
        opf_path = rootfile.get('full-path')

        # Parse OPF to get spine order
        opf = etree.parse(zf.open(opf_path))
        spine_items = opf.findall('.//{http://www.idpf.org/2007/opf}itemref')
        manifest = {item.get('id'): item.get('href')
                    for item in opf.findall('.//{http://www.idpf.org/2007/opf}item')}

        # Extract chapters in spine order
        for itemref in spine_items:
            idref = itemref.get('idref')
            href = manifest.get(idref)
            if href and href.endswith(('.xhtml', '.html', '.htm')):
                html = zf.read(os.path.join(os.path.dirname(opf_path), href))
                tree = etree.HTML(html)
                text = etree.tostring(tree, method='text', encoding='unicode')
                chapters.append(text.strip())

    return '\n\n=== Chapter Break ===\n\n'.join(chapters)
```

**Decision: Use this ZIP-based approach for EPUB.** Remove `ebooklib` from dependencies.

---

## Related Specs

- **[Document Suite](./overview.md)** — Parent document engine spec
- **[Record Inspector / Document Viewer](../Record%20Inspector/document-viewer.md)** — PDF viewer spec
- **[Feature Control Plane](../FeatureControlPlane/overview.md)** — Format-level toggles
- **[Batch Processing](../BatchProcessing/overview.md)** — Multi-format batch ingestion
