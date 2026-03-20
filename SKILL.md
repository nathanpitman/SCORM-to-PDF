# SCORM → PDF Tool — Development Context

## What this is

A single-file, zero-dependency static web app (`index.html`) that extracts text content from SCORM eLearning packages and produces a downloadable PDF. It is deployed to GitHub Pages and requires no build step, no server, and no backend.

The tool was built specifically for iHasco (part of Citation Group) to support an AI course creation workflow: SCORM packages are converted to readable PDF documents which can then be ingested by an AI course creator.

**Deployed URL:** `https://nathanpitman.github.io/scorm-to-pdf`  
**Repository:** Single file — `index.html` at the repo root.

---

## Architecture

Everything lives in one `index.html` file:
- Vanilla HTML/CSS/JS — no framework, no bundler, no build step
- One external dependency: **JSZip 3.10.1** loaded from cdnjs
- Two Google Fonts: **Geist** (UI) and **Newsreader** (wordmark)
- **Anthropic API** called directly from the browser for the AI rewrite feature (user supplies their own key)
- PDF is generated entirely client-side using a hand-rolled PDF 1.4 writer (no jsPDF or other library)

---

## Design System

The visual style is inspired by Claude.ai — light, warm, and editorial. Do not deviate from these tokens without good reason.

### CSS Variables (defined in `:root`)
```
--bg:           #FAF9F7   warm parchment page background
--surface:      #FFFFFF   card backgrounds
--surface-alt:  #F5F3EF   inset areas, drop zones, inactive inputs
--border:       #E8E4DC   default border colour
--border-mid:   #D4CFC5   hover/emphasis borders
--text:         #1A1814   primary text
--text-mid:     #5C5751   secondary/body text
--text-dim:     #9C9690   hints, labels, placeholders
--accent:       #CC785C   terracotta — primary interactive colour
--accent-light: #F5EDE8   accent tinted backgrounds
--accent-dark:  #A85840   accent hover state
--green:        #3D7A5A   success states
--green-light:  #EBF5EF   success backgrounds
--red:          #C0392B   error states
--amber:        #B45309   warning states
--shadow-sm:    subtle card shadow
--shadow:       elevated shadow
--radius-sm:    6px
--radius:       10px
--radius-lg:    16px
--font-sans:    'Geist', -apple-system, 'Helvetica Neue', sans-serif
--font-serif:   'Newsreader', 'Georgia', serif
```

### Component patterns
- **Cards** use `.card` > `.card-section`. Multiple sections are separated by a 1px border.
- **Step indicators** use `.step-badge` with states: default (muted circle), `.active` (filled dark), `.done` (filled green).
- **Buttons** use `.btn` base class. Variants: `.btn-primary` (dark fill), `.btn-accent` (terracotta fill).
- **Status bar** uses `.status-bar` with a `.dot` that has states: default (grey), `.busy` (amber, animated pulse), `.ok` (green), `.err` (red).
- **Progress bar** uses `.prog-wrap` / `.prog-bar` — hidden by default, shown with `.show` class.
- **Toast notifications** are the `.toast` element — shown with `.show` class, auto-dismissed after 2.8s.
- **Size warning** uses `.size-warn` — amber panel, shown with `.show` class.

---

## Global State Object

```js
var G = {
  ipOk:      false,     // IP ownership checkbox confirmed
  slides:    [],        // raw extracted slides: [{title, texts[]}]
  rewritten: [],        // AI-rewritten sections: [{title, prose}]
  meta:      {},        // course metadata: {title, schema, description, type, fileSize}
  ready:     false,     // extraction complete, PDF can be downloaded
  rwDone:    false      // AI rewrite complete
}
```

---

## User Flow (3 steps)

### Step 1 — Upload
1. User must tick the **IP ownership checkbox** before the drop zone activates
2. File is validated: must be `.zip`, hard limit 500 MB, soft warning above 80 MB
3. ZIP is loaded with JSZip, then the package type is auto-detected
4. Content is extracted into `G.slides`

### Step 2 — Extracted content
- Metadata grid shows: Course title, Package type, File size, Slides extracted, Approx. words
- Content preview shows first 3 slides (raw text, 260 chars each)
- Slide index lists up to 24 slides with word counts

### Step 3 — Rewrite + Download
- Optional: user pastes their Anthropic API key and clicks Rewrite
- Claude rewrites each section into flowing prose (calls `/v1/messages` directly from browser)
- Download PDF works with or without the rewrite step
  - With rewrite: uses `G.rewritten` (prose)
  - Without rewrite: uses `G.slides` merged by title (raw text)

---

## SCORM Package Detection & Extraction

### How package type is detected
1. Check for `html5/data/js/` files in the ZIP
2. If more than 3 such files exist, read the first one
3. If it contains `globalProvideData` → **Articulate Storyline**
4. Otherwise → **Standard SCORM** (HTML-based)

### Articulate Storyline extraction (`extractArt`)
- Iterates `html5/data/js/*.js` files
- Skips library/framework files matching: `SKIP_JS = /\.(min\.js)$|ds-bootstrap|ds-slides|ds-frame|jquery|modernizr|require/i`
- Only processes files containing `globalProvideData('slide'` — this is critical. `data.js`, `frame.js` and `paths.js` also call `globalProvideData` but with types `'data'`, `'frame'`, `'paths'`. These contain player UI labels stored as raw HTML (e.g. `<p dir='ltr' align='left' style="...">Resume</p>`) which would leak into the output as rogue markup if not excluded.
- Extracts `"title"` and all `"altText"` values from the JSON payload
- Filters out: image filenames, shape names (Rectangle, Pentagon, Group etc.), `%_player` template variables, strings under 10 chars, strings identical to the slide title
- Dedupes by first 80 chars of each text string
- Skips slides with titles containing: `'Quiz Results'`, `'Results Slide'`, `'Help | '`, `'| %_player'`

### Standard SCORM extraction (`extractStd`)
- Reads `imsmanifest.xml` to find `<resource href="...html">` files
- Maps resource identifiers to item titles via `<item identifierref="">`
- Strips HTML tags from each file using regex (not DOM — sandbox safe)
- Falls back to scanning all `.html` files in the ZIP if manifest has no hrefs

### Memory optimisation for large files
Files over 80 MB show a warning. Files over 500 MB are rejected.

These file types are **never decompressed** by the tool:
```
SKIP_EXTS: .png .jpg .jpeg .gif .svg .webp .mp4 .mp3 .m4v .m4a .webm .ogg .wav .woff .woff2 .ttf .eot .otf .css .swf .flv
SKIP_PATHS: html5/lib/  lms/  mobile/  story_content/video_  story_content/audio_
```

---

## PDF Generation

The PDF is built entirely with a hand-rolled PDF 1.4 writer. **There is no PDF library.** Do not introduce jsPDF or similar — the hand-rolled approach was specifically chosen because jsPDF failed silently due to async loading issues in the browser sandbox.

### Page layout
```
Page size:  A4 (595 × 842 pt)
Margins:    Left/Right 62pt, Top 58pt, Bottom 72pt (extra for footer)
Fonts:      Helvetica (F1, body) and Helvetica-Bold (F2, headings) — standard PDF Type1
```

### Text rendering
- `line()` — renders a single line at current `y` position, advances `y`
- `para()` — wraps text to fit within `USABLE` width, calls `line()` per wrapped line
- `cpl(size)` — estimates characters per line: `Math.floor(USABLE / (size * 0.525))`
- `np()` — starts a new page (pushes current ops array, resets `y` to top)
- `hr()` — draws a hairline horizontal rule

### Document structure
1. Course title (22pt bold)
2. Description if present (11pt, muted)
3. Package type + schema + export date (10pt, dimmed)
4. Hairline rule
5. For each section: heading (13pt bold) + paragraphs (10pt)

### Footer (every page)
```
Left:   "Generated from a SCORM package using SCORM → PDF Tool"  (8pt, grey)
Below:  "https://nathanpitman.github.io/scorm-to-pdf"  (8pt, terracotta #CC785C)
Right:  "Page X of Y"  (8pt, grey)
Above:  hairline rule at y=40
```
**To change the footer URL**, update `TOOL_URL` inside `buildPdf()`.

### PDF string safety
All text passed to PDF streams goes through `safe()`:
```js
function safe(s){ return String(s).replace(/[^\x20-\x7E]/g,' ').replace(/\\/g,'\\\\').replace(/\(/g,'\\(').replace(/\)/g,'\\)'); }
```
This strips non-ASCII characters (replaced with space) and escapes PDF string delimiters. This is why non-Latin characters in course titles will appear as spaces — a known limitation.

---

## AI Rewrite

- Model: `claude-sonnet-4-5`
- Endpoint: `https://api.anthropic.com/v1/messages`
- Header required for direct browser calls: `'anthropic-dangerous-direct-browser-access': 'true'`
- `max_tokens`: 1024 per section
- Sections are merged before rewriting: consecutive slides with identical titles are combined into one section
- Sections with fewer than 10 words are skipped
- Each section's raw text is truncated to 3200 chars before sending
- On API error, the section falls back to raw extracted text (no hard failure)
- The API key is **never stored** — it exists only in the input field during the session

### Rewrite prompt structure
```
You are converting extracted eLearning slide content into a readable document section.

Course: "{course title}"
Section: "{section title}"

Raw extracted text:
---
{raw text, max 3200 chars}
---

Rewrite this as 2–4 clear flowing paragraphs of professional prose. Rules:
- No bullet points or lists
- Do not repeat the section title in your response
- Only use information present in the source text
- Neutral, informative tone suitable for workplace training documentation
- Write the paragraphs only, nothing else
```

---

## Known Limitations

1. **Non-ASCII characters** in course content are replaced with spaces in the PDF (limitation of Helvetica Type1 font). To fix this properly, embed a TrueType font with full Unicode coverage in the PDF.
2. **Characters-per-line estimation** is approximate (`size * 0.525` factor). Narrow characters (i, l, 1) will cause lines to wrap earlier than needed; wide characters (W, M) may very occasionally overflow by 1–2 characters.
3. **Flash/canvas-only content** — older SCORM 1.2 packages that use Flash or `<canvas>` for all rendering will return no text.
4. **JavaScript-rendered content** — content injected at runtime by JS will not be captured (static HTML only).
5. **Images and diagrams** are not included in the PDF. Only text is extracted.
6. **Quiz questions and results** are filtered out of Articulate Storyline packages (intentional).

---

## File Structure

```
/
└── index.html    ← entire application (HTML + CSS + JS, ~860 lines)
└── SKILL.md      ← this file
```

No `package.json`, no `node_modules`, no build config. Deploy by pushing `index.html` to the root of a GitHub repo with Pages enabled.

---

## Common Development Tasks

### Change the deployed URL in the footer
Find `TOOL_URL` inside `buildPdf()` (~line 795) and update the string.

### Add a new SCORM package format
Add a detection branch in `go()` after the Articulate Storyline check. Create a new `extractXxx(zip, ...)` async function following the same pattern as `extractArt` and `extractStd`. Return `[{title, texts[]}]`.

### Change the AI model
Find `model:'claude-sonnet-4-5'` inside `startRewrite()` and update. Always use a current Anthropic model string.

### Add new metadata fields to the results grid
Add entries to the `cells` array in `renderResults()`. Also set the value on `G.meta` during extraction in `go()`.

### Change PDF page size or margins
Update constants at the top of `buildPdf()`: `W`, `H`, `ML`, `MR`, `MB`. Adjust `footerY` if changing bottom margin.

### Change the file size limits
- Hard rejection: `MB > 500` in `go()`
- Soft warning: `MB > 80` in `go()`

### Adjust the AI rewrite prompt
Find the `prompt` variable inside the `for` loop in `startRewrite()`. The prompt is a template literal built from `G.meta.title`, `sec.title`, and `raw`.

### Add support for embedded TrueType fonts (fixes non-ASCII)
This requires embedding a font as a base64 stream in the PDF objects and replacing the Type1 font references in the page resource dictionary. This is a significant change to `buildPdf()`.
