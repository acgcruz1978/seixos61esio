---
name: pdf
description: Generate a PDF from any content — markdown, HTML, code, or structured data — and deliver it as a downloadable file. Use when users want to: (1) export a document or report as PDF, (2) convert markdown or HTML to PDF, (3) save conversation output as a PDF file, (4) create a formatted printable document, or (5) prepare a submission-ready PDF. Triggered by phrases like 'generate a PDF', 'export as PDF', 'save this as PDF', 'create a PDF report', 'download as PDF', or 'convert to PDF'. Input: content (file path or inline text); Output: PDF file delivered via SendUserFile.
---

# PDF Skill

## Overview

This skill generates a PDF from any content source — markdown files, HTML, plain text, code listings, or structured data — using the best available local renderer and delivers the file directly to the user.

**Input**: A file path, an inline block of text, or a description pointing to the current working file.

**Output**: A PDF file delivered via `SendUserFile`, plus a brief confirmation of where it was saved.

**No external services used**: The PDF is rendered locally and sent only through the session file-transfer mechanism.

---

## When to Use This Skill

Use the **pdf** skill when the user asks to:

**Export / Download**
- "Generate a PDF from this file"
- "Export this report as a PDF"
- "Save this as a PDF"
- "Download this as a PDF"
- "Create a PDF of this document"

**Convert**
- "Convert this markdown to PDF"
- "Convert this HTML to PDF"
- "Turn this into a PDF"

**Create / Write**
- "Create a PDF report on [topic]"
- "Write a PDF summary of this"
- "Make a printable version of this"

---

## Workflow Overview

```
Step 1: Identify content source (file or inline text)
    ↓
Step 2: Check renderer availability (pandoc → weasyprint → wkhtmltopdf)
    ↓
Step 3: Render PDF with best available renderer
    ↓
Step 4: Verify output is non-empty
    ↓
Step 5: Deliver file via SendUserFile
```

**Quality Gate**: After rendering, verify the output file exists and is > 0 bytes before delivery.

---

## Step 1: Identify the Content

### From a file

If the user points to a file (e.g., "convert `docs/report.md` to PDF"), read it with the `Read` tool and note its format (markdown, HTML, plain text, code).

### From inline conversation text

If the content is inline in the conversation (e.g., a code block or pasted document), use it directly — write it to a temporary file first if the renderer requires a file path.

### Ambiguous source

If the source is genuinely unclear, ask once:

> "Which content should I convert to PDF — the current file, or should I generate a new document from our conversation?"

Do not guess.

---

## Step 2: Check Renderer Availability

Run these checks in parallel to find what is installed:

```bash
command -v pandoc      && pandoc --version 2>&1 | head -1
command -v weasyprint  && weasyprint --version 2>&1 | head -1
command -v wkhtmltopdf && wkhtmltopdf --version 2>&1 | head -1
```

### Renderer priority

| Priority | Tool | Best for |
|----------|------|----------|
| 1st | `pandoc` | Markdown, plain text, RST, LaTeX |
| 2nd | `weasyprint` | HTML, CSS-styled documents |
| 3rd | `wkhtmltopdf` | HTML with complex layouts / JavaScript |

If **no renderer is available**, report which packages to install and stop:

| OS | Command |
|----|---------|
| Ubuntu/Debian | `sudo apt install pandoc` or `pip install weasyprint` |
| macOS | `brew install pandoc` or `pip install weasyprint` |
| Windows | `winget install JohnMacFarlane.Pandoc` or `pip install weasyprint` |

Do **not** install packages without the user's explicit approval.

---

## Step 3: Render the PDF

### Pandoc (markdown / plain text)

```bash
pandoc input.md \
  --pdf-engine=xelatex \
  -V geometry:margin=1in \
  -V fontsize=12pt \
  -o output.pdf
```

**Fallback chain if LaTeX engine missing**:
1. Try `--pdf-engine=xelatex`
2. Fall back to `--pdf-engine=pdflatex`
3. Fall back to `--pdf-engine=lualatex`
4. If all LaTeX engines missing, try `--pdf-engine=weasyprint` (pandoc can call it)

**Code listings**: pass `--highlight-style=tango` for syntax highlighting.

### WeasyPrint (HTML / CSS)

```bash
weasyprint input.html output.pdf
```

If the source is markdown, convert to HTML first:

```bash
pandoc input.md -o _tmp.html && weasyprint _tmp.html output.pdf
```

### wkhtmltopdf (HTML with complex layout)

```bash
wkhtmltopdf --enable-local-file-access input.html output.pdf
```

### Code files (non-HTML, non-markdown)

Wrap in a minimal HTML template first, then render with `weasyprint` or `wkhtmltopdf`:

```html
<!DOCTYPE html>
<html><head>
<meta charset="utf-8">
<style>
  body { font-family: monospace; font-size: 12px; margin: 2cm; }
  pre { white-space: pre-wrap; }
</style>
</head><body><pre>CONTENT_HERE</pre></body></html>
```

---

## Step 4: Verify Output

```bash
ls -lh output.pdf
```

- If the file exists and is **> 0 bytes**: proceed to delivery.
- If the file is **missing or 0 bytes**: treat as a render failure — show the last 20 lines of stderr and suggest the most likely fix before retrying.

---

## Step 5: Deliver the File

Use `SendUserFile` with `status: "normal"` and a short caption:

```
caption: "report.pdf — converted from report.md (12 pages)"
```

Then clean up any temporary intermediate files (e.g., `_tmp.html`) from the system temp directory.

---

## Output Path Convention

| Situation | Output path |
|-----------|-------------|
| Source is `path/to/report.md` | `path/to/report.pdf` |
| Source is an HTML file | Same directory, `.pdf` extension |
| Inline content, no project | `./output.pdf` |
| Project has a `dist/` or `build/` directory | `dist/<title>.pdf` |

Use lowercase, hyphen-separated filenames derived from the document title when creating a new file.

---

## Quality Gate

Before calling `SendUserFile`, confirm all of the following:

- [ ] Output file exists on disk
- [ ] File size > 0 bytes
- [ ] No render errors in stderr
- [ ] Temporary files cleaned up

If any item fails, diagnose and fix before delivering.

---

## Error Handling

| Error | Action |
|-------|--------|
| No renderer available | List install commands per OS; stop and wait for user. |
| LaTeX engine missing | Retry with next engine in fallback chain. |
| `xelatex` font errors | Add `--pdf-engine=pdflatex` or specify `-V mainfont`. |
| Render exits non-zero | Show last 20 lines of stderr; suggest fix; retry once. |
| Output file is 0 bytes | Treat as render failure; show stderr. |
| CSS/images not loading in weasyprint | Use `--base-url` flag pointing to the source directory. |

---

## Decision Points

### Decision Point 1: Ambiguous source (Step 1)

**I ask once**: Which content to convert?
**User answers**: File path, inline content, or description.

### Decision Point 2: No renderer installed (Step 2)

**I report**: Which packages are missing and how to install them.
**User decides**: Install and retry, or provide an alternative path.

### Decision Point 3: Render failure after retry (Step 4)

**I report**: Exact error message and recommended fix.
**User decides**: Apply fix, switch renderer, or cancel.

---

## Example Usage

### User request
"Convert `docs/architecture.md` to a PDF."

### Skill response

1. **Read** `docs/architecture.md` — 450-line markdown document. ✓
2. **Check renderers** — `pandoc` found (v3.1.9), `xelatex` found. ✓
3. **Render**:
   ```bash
   pandoc docs/architecture.md \
     --pdf-engine=xelatex \
     -V geometry:margin=1in \
     -V fontsize=12pt \
     -o docs/architecture.pdf
   ```
4. **Verify** — `docs/architecture.pdf` exists, 284 KB. ✓
5. **Deliver** — `SendUserFile(["docs/architecture.pdf"], caption="architecture.pdf — 18 pages")`. ✓

---

## Tips for Best Results

### Provide clear source reference
- A file path is unambiguous; inline text works too.
- If converting a long conversation output, paste or save it to a file first.

### Prefer markdown over plain text
- Markdown gives pandoc enough structure to produce well-formatted PDFs with headings, tables, and code blocks.

### For styled documents, use HTML + WeasyPrint
- If you need custom fonts, colours, or layout, write or generate an HTML file with CSS and use WeasyPrint.

### Check LaTeX availability for best typography
- `xelatex` or `pdflatex` produce the highest-quality PDFs from markdown.
- Run `which xelatex` to confirm availability; install via `texlive-xetex` if needed.

---

## Constraints

- **Never upload the PDF to an external service** — deliver only via `SendUserFile`.
- **Never install packages silently** — always ask the user before running package managers.
- **Sensitive content**: If the user has flagged content as private, remind them to handle the output file accordingly before delivery.
- **Temporary files**: Write to the system temp directory (`/tmp` or `$TMPDIR`) and delete them after delivery.

---

## Limitations

- **Requires a local renderer**: If `pandoc`, `weasyprint`, and `wkhtmltopdf` are all absent, PDF generation is not possible without installation.
- **LaTeX required for best markdown output**: Without a LaTeX engine, pandoc falls back to weasyprint, which may produce different formatting.
- **Complex HTML**: Documents with heavy JavaScript or external fonts may not render correctly with weasyprint; use wkhtmltopdf in that case.
- **Very large documents**: Files > ~500 pages may take significant time; set user expectations accordingly.

---

## Summary

The **pdf** skill converts any local content to a PDF file through three steps: identify the source, render with the best available tool (pandoc → weasyprint → wkhtmltopdf), and deliver via `SendUserFile`. A quality gate verifies the output before delivery, and a clean error-handling table covers the most common failure modes.

**Typical time**: < 10 seconds for documents under 100 pages.
