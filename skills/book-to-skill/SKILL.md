---
name: book-to-skill
description: Convert a book, guide, manual, or long-form document into a reusable Claude Code SKILL.md. Use when a user wants to teach Claude a repeatable workflow from a book or document, capture expert procedures from written material as a callable skill, or package domain knowledge from a PDF/ePub/text into a skill others can invoke. Triggered by phrases like "turn this book into a skill", "make a skill from this document", "create a skill based on this guide", "convert this manual into a skill", or "package this book's knowledge as a skill".
---

# Book-to-Skill Converter

Extract the core repeatable procedures from a book, guide, or long-form document and produce a well-structured `SKILL.md` ready to drop into a Claude Code skills directory.

## When to Use This Skill

- "Turn this book into a skill"
- "Make a skill from this guide / manual / whitepaper"
- "Create a skill based on this PDF"
- "Convert [book title] into a reusable skill"
- "Package this document's knowledge as a skill"
- "I want Claude to follow the process described in this book"

## What This Skill Does

1. **Reads the source material** — accepts a file path (PDF, ePub, Markdown, plain text) or inline pasted text
2. **Identifies the primary workflow** — finds the central repeatable procedure the book teaches
3. **Extracts supporting knowledge** — gathers trigger phrases, examples, constraints, and tips from the text
4. **Drafts the SKILL.md** — writes a complete skill file following the standard format
5. **Asks one clarifying question** — confirms the skill name and target audience before finalising
6. **Writes the file** — saves to `skills/<name>/SKILL.md` and reports what was created

## How to Use

### From a local file

```
Turn docs/clean-code.pdf into a skill.
```

```
Create a skill from the book at ~/Books/pragmatic-programmer.epub
```

### From pasted text

```
Make a skill from this content: [paste chapter text here]
```

### Specifying the output name

```
Convert refactoring-guide.md into a skill called "refactor".
```

### Targeting a specific chapter or section

```
Build a skill from chapter 3 of docs/system-design.pdf — the one on capacity planning.
```

## Example

**User**: "Turn `docs/the-pragmatic-programmer.pdf` into a skill."

**Claude**:
```
Reading docs/the-pragmatic-programmer.pdf…
Identified primary workflow: iterative project hygiene — broken-window fixes, tracer bullets, orthogonality checks.
Proposed skill name: pragmatic-dev

Does that sound right, or would you like a different name or focus?
```

**User**: "Yes, that's right."

**Claude**:
```
Writing skills/pragmatic-dev/SKILL.md…
Done — skill saved. Invoke it with /pragmatic-dev.
```

## Extraction Strategy

### What to look for

| Source element | Maps to SKILL.md section |
|----------------|--------------------------|
| Book title / subtitle | `name`, `description` frontmatter |
| "How to use this book" / intro | `When to Use This Skill` |
| Step-by-step chapters | `How to Use` (steps / code blocks) |
| Worked examples | `Example` section |
| Rules, heuristics, maxims | `Tips` + `Constraints` |
| Anti-patterns, warnings | `Error Handling` or `Constraints` |
| Reference tables / checklists | inline tables in the skill body |

### Scope rules

- Focus on the **one core workflow** the book is built around; if there are several, ask the user to pick one or generate one skill per major part.
- Keep each step **action-oriented**: start with a verb ("Read", "Run", "Check", "Write").
- Omit biographical anecdotes, marketing copy, and repeated filler — skills are dense and procedural.
- If the book is behind a paywall or DRM-locked, work only with content the user has already supplied; never attempt to fetch or scrape external sources.

## Output Format

The generated SKILL.md follows this structure exactly:

```markdown
---
name: <slug>
description: <one sentence — when to trigger, key capability>
---

# <Title>

<One-paragraph summary of what the skill does.>

## When to Use This Skill
<bullet list of trigger phrases>

## What This Skill Does
<numbered steps>

## How to Use
<usage examples with code blocks>

## Example
<concrete before/after>

## Tips
<bullet list>

## Constraints
<bullet list>
```

## File Placement

| Scenario | Output path |
|----------|-------------|
| Skill name provided | `skills/<name>/SKILL.md` |
| Name inferred from book title | `skills/<slugified-title>/SKILL.md` |
| User specifies a directory | `<dir>/<name>/SKILL.md` |

The parent directory is created if it does not exist.

## Error Handling

| Problem | Action |
|---------|--------|
| File not found | Report the missing path; ask user to confirm location |
| File is DRM-locked / unreadable | Ask user to paste the relevant text directly |
| No clear repeatable workflow found | Ask user which section or chapter to focus on |
| Book covers multiple unrelated topics | Generate one skill per major topic and ask which to keep |
| Output file already exists | Confirm before overwriting |

## Constraints

- Never fetch or download books from the internet — work only with files or text the user supplies
- Do not reproduce large verbatim excerpts from copyrighted works in the skill body — paraphrase and distil
- Keep the generated SKILL.md under 200 lines; if the source is vast, focus on the core workflow and add a "Further Reading" pointer
- Do not install packages or run code during extraction — this is a reading and writing task only
