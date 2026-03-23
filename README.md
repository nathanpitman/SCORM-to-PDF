# SCORM-to-PDF

A zero-dependency browser tool that extracts text content from SCORM eLearning packages and generates downloadable PDF documents — with optional AI-powered prose rewriting via the Claude API.

Built for iHasco (Citation Group) to support AI course creation workflows where SCORM packages need to be converted to readable documents.

**Live tool:** https://nathanpitman.github.io/scorm-to-pdf

---

## What it does

1. **Extracts** text content from a `.zip` SCORM package uploaded by the user
2. **Displays** a preview of the extracted slides, course metadata, and word counts
3. **Optionally rewrites** the raw slide content into flowing prose using Claude
4. **Generates** a downloadable PDF of the extracted (or rewritten) content

No server, no build step, no installation — everything runs client-side in the browser.

---

## How it works

### Upload & extraction

The user uploads a `.zip` SCORM package. The tool auto-detects the package type:

- **Articulate Storyline** — detected by the presence of `html5/data/js/` files containing `globalProvideData`. Slide titles and alt-text are extracted from the JavaScript data files.
- **Standard SCORM** — reads `imsmanifest.xml` to find resource files, then strips HTML tags to extract text content.

Large asset files (images, video, audio, fonts, CSS) are skipped during decompression to keep memory usage low.

### AI rewrite (optional)

If the user supplies an Anthropic API key, each extracted section is sent to Claude (`claude-sonnet-4-5`) and rewritten from bullet-point slide content into coherent, readable paragraphs. Sections under 10 words are skipped.

### PDF generation

A PDF is generated entirely in JavaScript — no external PDF library is used. The tool implements a minimal PDF 1.4 writer that handles:

- A4 page layout with configurable margins
- Line wrapping and page breaks
- Course title, section headings, and body text
- A footer with the tool URL on every page

---

## Usage

### Online

Visit **https://nathanpitman.github.io/scorm-to-pdf** — no setup needed.

### Local

```bash
git clone https://github.com/nathanpitman/scorm-to-pdf.git
cd scorm-to-pdf
# Open index.html directly, or serve it:
python -m http.server 8000
```

Then open `http://localhost:8000` in your browser.

### Steps

1. Check the IP ownership confirmation checkbox
2. Drop or select a `.zip` SCORM package
3. Review the extracted content preview
4. *(Optional)* Enter an Anthropic API key and click **Rewrite** to generate prose
5. Click **Download PDF**

---

## Architecture

The entire application lives in a single `index.html` file (~860 lines). There is no build process, no framework, and no backend.

**External dependencies (CDN only):**
- [JSZip 3.10.1](https://stuk.github.io/jszip/) — ZIP file parsing
- Google Fonts (Geist, Newsreader) — UI typography

**Everything else is vanilla HTML, CSS, and JavaScript.**

### Key limits

| Constraint | Value |
|---|---|
| Hard file size limit | 500 MB |
| Soft warning threshold | 80 MB |
| Max tokens per AI section | 1024 |
| Input text truncation | 3200 chars per section |
| Slide preview (UI) | First 3 slides, 260 chars each |
| Slide index (UI) | Up to 24 slides |

### Known limitations

- Non-ASCII characters are replaced with spaces (Helvetica Type1 PDF limitation)
- Flash/canvas-only content returns no text
- JavaScript-rendered SCORM content is not captured (static HTML parsing only)
- Images and diagrams are not included in the PDF
- Quiz slides are intentionally filtered out of Articulate packages

---

## Privacy & security

- An IP ownership checkbox must be confirmed before any file is processed
- The Anthropic API key is only held in the input field for the session — it is never stored or logged
- All processing is client-side; no content is sent anywhere except to the Anthropic API when the rewrite feature is used
- PDF output sanitises all text to remove non-ASCII characters and escape PDF special characters

---

## Development

The application is documented in [`SKILL.md`](./SKILL.md), which covers the internal architecture, state management, extraction logic, PDF writer implementation, and design system in detail.

To deploy, push `index.html` to a GitHub repository with Pages enabled.
