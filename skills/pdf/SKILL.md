# PDF Skill

Generate a PDF from any content — markdown, HTML, code, reports, or structured data — and deliver it as a downloadable file.

## Triggers

Invoke this skill when the user asks to:

- "generate a PDF", "export as PDF", "save as PDF"
- "create a PDF report / document / summary"
- "download this as a PDF"
- "convert this to PDF"

## What this skill does

1. Determines the source content (current file, conversation output, or user-supplied text).
2. Chooses an appropriate rendering path:
   - **Markdown / plain text** → render via `pandoc` (preferred) or `weasyprint`.
   - **HTML** → render via `weasyprint` or `wkhtmltopdf`.
   - **Code listings** → wrap in a styled HTML template first, then render.
3. Writes the PDF to a sensible output path (same directory as the source, or a `dist/` folder when no source file exists).
4. Sends the file to the user with `SendUserFile`.

## Step-by-step instructions

### 1. Identify the content

- If the user points at a file, read it with the `Read` tool.
- If the content is inline in the conversation, use it directly.
- Clarify once if the source is genuinely ambiguous; do not guess.

### 2. Pick a renderer and check availability

Run the following checks in parallel:

```bash
command -v pandoc      && pandoc --version | head -1
command -v weasyprint  && weasyprint --version
command -v wkhtmltopdf && wkhtmltopdf --version | head -1
```

Use the first available tool that matches the content type (order: pandoc → weasyprint → wkhtmltopdf).
If none are installed, tell the user which package to install and stop.

### 3. Render the PDF

**Markdown with pandoc:**

```bash
pandoc input.md \
  --pdf-engine=xelatex \         # or pdflatex / lualatex
  -V geometry:margin=1in \
  -V fontsize=12pt \
  -o output.pdf
```

If `xelatex` is unavailable, fall back to `--pdf-engine=pdflatex`.

**HTML with weasyprint:**

```bash
weasyprint input.html output.pdf
```

**HTML with wkhtmltopdf:**

```bash
wkhtmltopdf --enable-local-file-access input.html output.pdf
```

### 4. Verify the output

```bash
ls -lh output.pdf
```

Confirm the file is non-empty (> 0 bytes). If the render failed, show the error and suggest a fix.

### 5. Deliver the file

Use `SendUserFile` with `status: "normal"` and a short caption describing the document.

## Output path convention

| Situation | Output path |
|-----------|-------------|
| Source is a file `src/report.md` | `src/report.pdf` |
| Content is inline, no project | `./output.pdf` |
| Project has a `dist/` directory | `dist/<title>.pdf` |

Use lowercase, hyphen-separated names derived from the document title when creating a new filename.

## Error handling

| Error | Action |
|-------|--------|
| No renderer available | List missing packages (`apt`, `brew`, or `pip` instructions per OS); stop. |
| LaTeX engine missing | Retry with `pdflatex`; if that also fails, switch to `weasyprint`. |
| Render exits non-zero | Show the last 20 lines of stderr; suggest the most likely fix. |
| Output file is 0 bytes | Treat as a render failure and follow the row above. |

## Constraints

- Do **not** install packages without asking the user first.
- Do **not** upload the PDF to any external service — deliver it only via `SendUserFile`.
- Keep temporary intermediate files (e.g. `_tmp.html`) in the system temp directory and clean them up after delivery.
- If the document contains sensitive data the user flagged as private, remind them to handle the output file accordingly.
