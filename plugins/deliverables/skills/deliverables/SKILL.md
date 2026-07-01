---
name: deliverables
version: "2.0"
description: |
  Produce a branded, PDF-ready client deliverable in the format you ask for. You say what you need (a report, a one-pager, a presentation/deck, or a document/memo) and what it is about, and this renders it into a clean, print-ready file using the matching format template under formats/. Pulls content from existing working docs for a {domain} (a deep audit, topic clusters, content briefs, backlink and E-E-A-T lists) or from content you provide directly. Applies the default brand automatically (TitanX unless told otherwise) and falls back to a neutral, unbranded look. Output is one self-contained HTML file per deliverable, Cmd+P to PDF. Works on any platform.
  Triggers: deliverables, build a deliverable, client report, one-pager, one pager, presentation, slide deck, deck, pitch deck, document, memo, brief, strategy memo, proposal, scope sheet, branded report, pdf report, package the audit.
  Requires: nothing mandatory. Content can come from existing working docs (run /audit:deep first for an audit-based deliverable) or be supplied directly. No MCP servers needed.
  Workflow: Choose format and source -> Apply branding -> Render into the format template -> Save -> Open -> Log.
  Command: /deliverables
---

# Deliverables Skill

Turns content into a polished, client-ready file in whatever format the moment calls for: a multi-page report, a single-page one-pager, a slide deck, or a prose document. Every output is a light-background, print-to-PDF file, and all four formats share one branding system and one component library, so a report and a deck for the same client look like they came from the same studio.

Four things are non-negotiable on every deliverable this skill produces:

1. **Humanized content.** Run the humanizer pass before finalizing (draft, AI-tells audit, final rewrite). No em dashes, no emojis, no internal tooling or workflow references.
2. **On brand.** Apply the user's default brand profile automatically if one is set (see Phase 1); only prompt for branding when no default exists.
3. **Clear background.** Light/white background documents, readable and ink-friendly. Never a dark style for a read-or-print document (a deck cover band is the one allowed dark accent).
4. **PDF-ready.** Exports cleanly with Cmd+P. Each format template has its own correct print setup baked in (portrait for report/one-pager/document, landscape for deck).

This skill formats content; it does not analyze a site. For an audit-based deliverable, run `/audit:deep` first and point this at its output. For anything else, give it the content (a draft, notes, or a working doc) and the format.

## Prerequisites

- **The format templates**: the `formats/` folder in this skill holds one template per format, all brand-agnostic and tokenized:
  - `formats/report.html` - multi-page A4 portrait
  - `formats/one-pager.html` - single A4 portrait page
  - `formats/deck.html` - 16:9 landscape slides
  - `formats/document.html` - A4 portrait, prose/memo
  Never edit a brand into a template; branding is applied per run via the BRANDING variables.
- **Content**: either existing working docs for the `{domain}` (see Phase 0) or content the user supplies in the request.
- **No MCP servers needed.** This is a local formatting step.

This skill is read-only on the source docs. It writes new HTML files; it never edits source markdown or CSV.

---

## Critical Guards

GUARD - **Format template missing:**
- If the chosen `formats/*.html` cannot be read: note it and stop. Do not invent a template inline.

GUARD - **No content to render:**
- If the user asked for an audit-based deliverable but no audit doc or working doc exists for `{domain}`, and they have not supplied content: say so and point to `/audit:deep {url}`, or ask them to provide the content. Do not fabricate findings.

GUARD - **One-pager overflow:**
- A one-pager must fit on a single page. If the content cannot fit, say so and offer either to trim it or to switch that deliverable to the report format. Never ship a one-pager that spills to a second page.

GUARD - **User requests abort:**
- Confirm, exit cleanly, output any files already written.

---

## Phase 0: CHOOSE FORMAT AND SOURCE

### 0.1 Set {domain}

Take `{domain}` from the command argument, or infer it from the working directory, or ask. Working docs, if any, live under `./{domain}/`.

### 0.2 Pick the format

Determine the format from the request. Map the user's words: "one-pager" -> one-pager, "deck / slides / presentation / pitch" -> deck, "memo / brief / letter / strategy doc / proposal" -> document, "report / audit / full write-up" -> report. If it is ambiguous, ask which of the four, with a one-line recommendation based on the content and audience.

| Format | Geometry | Use it for |
|--------|----------|------------|
| **Report** | A4 portrait, multi-page | Full audits, content plans, anything with tables, briefs, and several sections. The detailed deliverable. |
| **One-pager** | A4 portrait, single page | Executive summary, scope sheet, snapshot, leave-behind. One page, no overflow. |
| **Deck** | 16:9 landscape slides | Pitch, proposal walkthrough, kickoff, anything presented on screen or projected. One idea per slide. |
| **Document** | A4 portrait, prose | Strategy memo, brief, proposal letter, scope note. Reading-first, light masthead, optional signature block. |

A single request can produce more than one format (for example a report plus a one-pager summary of it). Build each as its own file.

### 0.3 Pick the content source

Content can come from either place; decide which applies:

- **From working docs** (audit-based or other). Look in `./{domain}/reports/` and `./{domain}/deliverables/` for source docs. Common mappings, when the source exists:

  | Deliverable subject | Built from |
  |---------------------|-----------|
  | SEO and AEO audit | `reports/latest-deep.md` (or the newest `reports/audit-deep-*.md`) |
  | Keyword and content plan | `deliverables/topic-clusters.md` + `deliverables/content-briefs.csv` |
  | Technical fix checklist | `deliverables/tech-fix-checklist.md` |
  | Backlink and E-E-A-T plan | `deliverables/backlink-opportunities.md` + `deliverables/eeat-improvements.md` |

  Other working docs (a monthly report, a custom brief) are fair game too.

- **Supplied directly.** The user hands over the content in the request (a draft, notes, key points). Render that; do not go hunting for an audit.

List what you found or what you were given, and confirm the format + subject before building. Default to building what was asked; if multiple working docs match and the user was vague, show the matched set and ask which to build.

---

## Phase 1: BRANDING (apply the default, prompt only if none)

Branding is never hardcoded into a template. Resolve it per run in this order:

1. **Default brand profile**: the standing default is **TitanX** unless the user names another brand (see the user's global instructions). Apply it automatically without asking and state which brand you used (for example "Styled to TitanX."). Read the brand source-of-truth file (for TitanX, `titanx.io/brand/brand.md`) so colours, fonts, and logo are accurate. Also check `.claude/seo-copilot-config.json` `branding` block for an override. Do not prompt when a default applies.
2. **Prompt only if there is no default** and the user named no brand. Ask for these, all optional, with the neutral fallback shown:

   | Field | Maps to | Default if skipped |
   |-------|---------|--------------------|
   | Brand name | wordmark + footer | the domain |
   | Logo (URL or local path) | the logo image | none (the wordmark shows instead) |
   | Accent colour (hex) | `--accent` (rules, badges, kicker, links) | `#475569` neutral slate |
   | Title-band / cover colour (hex) | `--dark` | `#1F2937` neutral |
   | Heading font | `--heading` | Georgia serif stack |
   | Body font | `--body` | system sans stack |
   | Client name | footer | the domain |
   | Prepared by | byline | omitted |

3. **Offer to save**: "Save this branding to config so I do not ask next time?" If yes, write the `branding` block to `.claude/seo-copilot-config.json`.

If the user skips everything, render with the neutral defaults already in the templates. A skipped logo falls back to the text wordmark (every template handles this); never ship a broken image.

**Logo handling**: a URL is used as-is. A local path is referenced relative to the output folder (copy the image into `./{domain}/deliverables/assets/` and point at `assets/<file>`), or inlined as a data URI for a fully self-contained file. Prefer a real, verifiable logo; never hand-draw or invent one.

---

## Phase 2: RENDER INTO THE FORMAT TEMPLATE

Start from the chosen `formats/*.html`, replace the header tokens, and inject the rendered content into `{{CONTENT}}`.

### 2.1 Set the header tokens (all formats)

| Token | Value |
|-------|-------|
| `{{REPORT_TITLE}}` | the deliverable name (for example "Technical Fix Checklist", "Q3 Growth Proposal") |
| `{{REPORT_KICKER}}` | short label, for example "Technical &middot; Report" or "Proposal" |
| `{{REPORT_SUB}}` | one-line subtitle |
| `{{PREPARED_FOR}}` | client / business name |
| `{{PREPARED_BY}}` | the preparer, or omit if none |
| `{{DATE}}` | deliverable date |
| `{{LOGO_SRC}}` | logo URL/path, or leave empty to show the wordmark |
| `{{BRAND_NAME}}` | brand name |
| `{{CLIENT_NAME}}` | client name (footer) |
| `{{CONTENT}}` | the rendered body (2.3) |

Apply branding by overriding the BRANDING variables in the `:root` block of the chosen template (`--accent`, `--accent-ink`, `--dark`, `--heading`, `--body`). Set `--accent-ink` to a slightly darker shade of the accent for legible small text on white.

### 2.2 Build the body in the right shape for the format

- **Report** and **document**: a sequence of `<section>` blocks. In the report, number sections (`<h2><span class="n">01</span>Title</h2>` + `<div class="srule"></div>`) and put `class="avoid"` on sections and `.brief` cards that must not split across a page break. In the document, lead with prose under each `<h2>`; close with the `.sign` block for a signature or contact line if it fits the piece.
- **One-pager**: everything goes in `{{CONTENT}}` and must fit one page. Use `<p class="lead">` for the opening line, then the `.cols` two-column grid of short `.block` items, the `.stats` row for headline figures, compact `.card` and `.callout` blocks. Keep it tight. If it overflows, trim or escalate to a report (see the one-pager guard).
- **Deck**: each idea is a `<section class="slide">` (the cover is already in the template; add content slides into `{{CONTENT}}`). One idea per slide, a short `<h2>` headline, a `.srule`, then a `.body` with a few blocks (`.grid .g3`/`.g2`, `.stats`, `.points`, a `.callout`, or a small table). Slides are auto-numbered. Do not overfill a slide.

### 2.3 Render content into the shared classes

Convert markdown/CSV/notes into clean HTML using the classes the templates already style. Do not invent new styles; reuse these (availability noted where a format omits one):

- **Lead**: `<p class="lead">`. Body: plain `<p>`. Muted note: `<p class="muted">`.
- **Tables**: plain `<table><thead>...<tbody>...</table>`. Right-align numeric columns with `class="num"`. Monospace a URL/path with `class="pg"`. A keyword cell can use `class="kw"` (report).
- **Callout**: `<div class="callout"><b>Label.</b> text</div>`.
- **Stat figures** (one-pager, deck): `.stats` > `.stat` > `.v` (value) + `.l` (label).
- **Cards**: `.grid .g3`/`.g2` > `.card` (report, deck); compact `.card` inside `.cols` (one-pager).
- **Severity badges**: `b-crit` Critical, `b-high` High, `b-med` Medium. **Action badges** (report): `b-new`, `b-update`, `b-refresh`. **Priority badges**: `p1`, `p2`, `p3`.
- **Checklist checkbox** (report): `<span class="chk"></span>` as the first cell.
- **Numbered points**: the `.points` / `.point` / `.pn` pattern.
- **Content brief** (report): the `.brief` block with a `.bh` header and a `<dl>` of label/value pairs.
- **Document prose extras**: `blockquote` for a pulled quote, `.sign` for the closing signature/contact block.

Match the source faithfully. A checklist stays a checklist; a brief stays a brief; tables keep their columns. Do not add analysis; this step formats, it does not re-diagnose.

### 2.4 Output rules (client-facing)

1. No em dashes and no emojis anywhere. Use commas, colons or periods. The middot `&middot;` is fine in the kicker and footer.
2. No internal tooling or workflow references. Never name internal skills or processes (this skill, an audit skill, a humanizer step, QA notes).
3. No check IDs or internal codes. Plain language only.
4. Keep every claim tied to its source. Do not introduce new findings.
5. Run the prose through the humanizer pass before finalizing (no AI tells), keeping rules 1 to 4.

---

## Phase 3: SAVE, OPEN, LOG

### 3.1 Save

Write each file to `./{domain}/deliverables/` with a clear, format-tagged name:
- `report-seo-aeo-audit.html`
- `one-pager-executive-summary.html`
- `deck-growth-proposal.html`
- `document-strategy-memo.html`

(Adjust the subject to whatever was built.) Copy any local logo into `./{domain}/deliverables/assets/`.

### 3.2 Offer to open

Offer to open the files in the browser so the user can review and Cmd+P to PDF. Per-format print reminders:
- Report / one-pager / document: Layout **Portrait**, tick "Background graphics".
- Deck: Layout **Landscape**, Margins **None**, tick "Background graphics".
- Chrome gives the best result in all cases.

### 3.3 Activity log

Append to `./{domain}/reports/activity-log.md`:

```
| YYYY-MM-DD | /deliverables | Built [N] [format(s)] (PDF-ready): [list]. Branding: [brand name or "neutral default"]. |
```

---

## Integration with Other Skills

| Step | Skill |
|------|-------|
| Produce an audit and its working docs | `/audit:deep {url}` |
| Package those docs (or any content) into a branded deliverable | `/deliverables` |
| Humanize the prose before finalizing | the humanizer pass (built into this skill) |

Run order for an audit-based deliverable: `/audit:deep` to generate the analysis, then `/deliverables` to render it in the format you need. For anything else, run `/deliverables` directly with the content and the format.
