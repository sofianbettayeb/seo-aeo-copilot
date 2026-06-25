---
name: deliverables
version: "1.1"
description: |
  Turn an audit's working documents into branded, PDF-ready HTML reports. Reads the deep audit and its companion docs for a {domain} (technical fix checklist, topic clusters and content briefs, backlink and E-E-A-T lists) and renders each into a clean, print-ready report using a brand-agnostic template (template.html). Prompts once for branding (brand name, logo, accent colour, title-band colour, fonts) and falls back to a neutral, unbranded look. Output is one self-contained HTML file per report, A4 portrait, Cmd+P to PDF. Works on any platform.
  Triggers: deliverables, build deliverables, client reports, branded report, pdf report, report pack, package the audit.
  Requires: an existing audit and/or its companion working docs in the project (run /audit:deep first). No MCP servers needed.
  Workflow: Discover sources -> Prompt for branding -> Build one report per source -> Save -> Open -> Log.
  Command: /deliverables
---

# Deliverables Skill

Packages the raw working documents from an audit into polished, client-ready HTML reports. The reports are light-background, print-to-PDF documents that share one design system.

Four things are non-negotiable on every deliverable this skill produces:

1. **Humanized content.** Run the humanizer pass before finalizing (draft, AI-tells audit, final rewrite). No em dashes, no emojis, no internal tooling or workflow references.
2. **On brand.** Apply the user's default brand profile automatically if one is set (see Phase 1); only prompt for branding when no default exists.
3. **Clear background.** Light/white background documents, readable and ink-friendly. Never a dark style for a read-or-print document.
4. **PDF-ready.** Exports cleanly with Cmd+P (A4 portrait, backgrounds print, sensible page breaks, one self-contained file). This is built into the template.

This skill does not analyze the site. It formats work that already exists. Run `/audit:deep` first; this turns its output into something you can send a client.

## Prerequisites

- **The template**: `template.html` in this skill folder (brand-agnostic, tokenized). Never edit a brand into the template itself; branding is applied per run via the BRANDING variables.
- **Source documents**: the audit report and/or its companion working docs for the `{domain}` (see Phase 0). If none are found, the skill says so and points to `/audit:deep`.
- **No MCP servers needed.** This is a local formatting step.

This skill is read-only on the source docs. It writes new HTML files; it never edits the source markdown or CSV.

---

## Critical Guards

GUARD - **No source documents found:**
- If neither a deep-audit report nor any companion working doc exists for `{domain}`: "No audit documents found for {domain}. Run `/audit:deep {url}` first, then `/deliverables`." Stop.

GUARD - **Template missing:**
- If `template.html` cannot be read: note it and stop. Do not invent a template inline.

GUARD - **User requests abort:**
- Confirm, exit cleanly, output any files already written.

---

## Phase 0: DISCOVER SOURCES

### 0.1 Set {domain}

Take `{domain}` from the command argument, or infer it from the working directory, or ask. Source documents live under `./{domain}/`.

### 0.2 Locate the source documents

Look in `./{domain}/reports/` and `./{domain}/deliverables/` for any of these. Each maps to one branded report:

| Report (output) | Built from (source docs, whichever exist) |
|-----------------|-------------------------------------------|
| 1. SEO and AEO Audit | `reports/latest-deep.md` (or the newest `reports/audit-deep-*.md`) |
| 2. Keyword and Content Plan | `deliverables/topic-clusters.md` + `deliverables/content-briefs.csv` (+ the keyword-strategy section of the audit) |
| 3. Technical Fix Checklist | `deliverables/tech-fix-checklist.md` |
| 4. Backlink and E-E-A-T Plan | `deliverables/backlink-opportunities.md` + `deliverables/eeat-improvements.md` |

List what was found. If only some exist, build only those, and say which were skipped and why. The set is not fixed: if other working docs exist (for example a `monthly-report` or a custom brief), offer to render them too, one report each, using the same template.

### 0.3 Ask which to build

Show the matched set and ask: "Build all of these, or a subset?" Default to all matched.

---

## Phase 1: BRANDING (apply the default, prompt only if none)

Branding is never hardcoded into the template. Resolve it per run in this order:

1. **Default brand profile**: check the user's standing instructions (global or project CLAUDE.md) and the `.claude/seo-copilot-config.json` `branding` block. If a default brand is set there, **apply it automatically without asking**, and state which brand you used (for example "Styled to {brand}."). Read the brand's source-of-truth file if one is referenced, so the colours, fonts, and logo are accurate. Do not prompt.
2. **Prompt only if there is no default**, or if the user named a different brand for this deliverable. Ask for these, all optional, with the neutral fallback shown:

   | Field | Maps to | Default if skipped |
   |-------|---------|--------------------|
   | Brand name | wordmark + footer | the domain |
   | Logo (URL or local path) | `.band` logo image | none (the wordmark shows instead) |
   | Accent colour (hex) | `--accent` (rules, badges, kicker, links) | `#475569` neutral slate |
   | Title-band colour (hex) | `--dark` (the dark header band) | `#1F2937` neutral |
   | Heading font | `--heading` | Georgia serif stack |
   | Body font | `--body` | system sans stack |
   | Client name | footer right side | the domain |
   | Prepared by | byline in the band meta | omitted |

3. **Offer to save**: "Save this branding to config so I do not ask next time?" If yes, write the `branding` block to `.claude/seo-copilot-config.json`.

If the user skips everything, render with the neutral defaults already in the template. A skipped logo must fall back to the text wordmark (the template handles this); never ship a broken image.

**Logo handling**: a URL is used as-is. A local path should be referenced relative to the output folder (copy the image into `./{domain}/deliverables/assets/` and point at `assets/<file>`), or inlined as a data URI for a fully self-contained file. Prefer a real, verifiable logo; never hand-draw or invent a logo.

---

## Phase 2: BUILD ONE REPORT PER SOURCE

For each report to build, start from `template.html`, replace the tokens, and inject the rendered content.

### 2.1 Set the header tokens

| Token | Value |
|-------|-------|
| `{{REPORT_TITLE}}` | the report name (for example "Technical Fix Checklist") |
| `{{REPORT_KICKER}}` | short label, for example "Report 3 of 4 &middot; Technical" |
| `{{REPORT_SUB}}` | one-line subtitle |
| `{{PREPARED_FOR}}` | client / business name |
| `{{PREPARED_BY}}` | the preparer, or remove the whole "By" span if none |
| `{{DATE}}` | report date |
| `{{LOGO_SRC}}` | logo URL/path, or leave empty to show the wordmark |
| `{{BRAND_NAME}}` | brand name (wordmark + footer left) |
| `{{CLIENT_NAME}}` | client name (footer right) |
| `{{CONTENT}}` | the rendered section HTML (2.3) |

Apply branding by overriding the BRANDING variables in the `:root` block (`--accent`, `--accent-ink`, `--dark`, `--heading`, `--body`). Set `--accent-ink` to a slightly darker shade of the accent for legible small text on white.

### 2.2 Keep the report structure

Each report is a sequence of `<section>` blocks. A section is:

```html
<section class="avoid">
  <h2><span class="n">01</span>Section title</h2>
  <div class="srule"></div>
  ... content ...
</section>
```

Number the sections. Use `class="avoid"` on sections (and on `.brief` cards) that should not split across a page break.

### 2.3 Render the source content into the template classes

Convert the markdown/CSV into clean HTML using the classes the template already styles. Do not invent new styles; reuse these:

- **Lead paragraph**: `<p class="lead">`. Body: plain `<p>`. Muted note: `<p class="muted">`.
- **Tables**: plain `<table><thead><tr><th>...</th></tr></thead><tbody>...</tbody></table>`. Right-align numeric columns with `class="num"` on the `th`/`td`. Monospace a URL/path with `class="pg"`. A keyword cell can use `class="kw"`.
- **Callout** (a headline figure or a key point): `<div class="callout"><b>Label.</b> text</div>`.
- **Severity badges** (technical): `<span class="badge b-crit">Critical</span>`, `b-high` High, `b-med` Medium.
- **Action badges** (content plan): `<span class="badge b-new">New</span>`, `b-update` Update, `b-refresh` Refresh.
- **Priority badges** (authority / backlinks): `<span class="badge p1">P1</span>`, `p2`, `p3`.
- **Checklist checkbox**: `<span class="chk"></span>` as the first cell of a checklist row.
- **Legend** under a heading: `<div class="legend"> ... </div>`.
- **A page brief** (content plan detail): the `.brief` block with a `.bh` header row and a `<dl>` of label/value pairs (Meta title, Meta desc, H1, H2 outline, Internal links, Note).
- **Numbered points** (for example three key findings): the `.points` / `.point` / `.pn` pattern.

Match the source faithfully. A checklist stays a checklist; a brief stays a brief; tables keep their columns. Group long content plans by theme or by action, and put detailed briefs in their own section.

### 2.4 Output rules (client-facing)

1. No em dashes and no emojis anywhere. Use commas, colons or periods. The middot `&middot;` is fine in the kicker and footer.
2. No internal tooling or workflow references. Never name internal skills or processes (this skill, an audit skill, a humanizer step, QA notes). Those belong in the working docs, not the deliverable.
3. No check IDs or internal codes. Plain language only.
4. Keep every claim tied to the source doc. Do not add new analysis; this step formats, it does not re-diagnose.
5. Run the prose through the project's humanizer pass before finalizing (no AI tells), keeping rules 1 to 4.

---

## Phase 3: SAVE, OPEN, LOG

### 3.1 Save

Write each report to `./{domain}/deliverables/` with a clear name:
- `report-1-seo-aeo-audit.html`
- `report-2-keyword-content-plan.html`
- `report-3-technical-fix-checklist.html`
- `report-4-backlink-eeat-plan.html`

(Adjust names to whatever set was built.) Copy any local logo into `./{domain}/deliverables/assets/`.

### 3.2 Offer to open

Offer to open the files in the browser so the user can review and Cmd+P to PDF. Remind them: Layout Portrait, tick "Background graphics", Chrome for best results.

### 3.3 Activity log

Append to `./{domain}/reports/activity-log.md`:

```
| YYYY-MM-DD | /deliverables | Built N branded HTML reports (PDF-ready): [list]. Branding: [brand name or "neutral default"]. |
```

---

## Integration with Other Skills

| Step | Skill |
|------|-------|
| Produce the audit and its working docs | `/audit:deep {url}` |
| Package those docs into branded client reports | `/deliverables` |
| Build the matching pitch or proposal deck | (use the deck the documents support) |

Run order: `/audit:deep` to generate the analysis and working docs, then `/deliverables` to turn them into the branded report pack you hand to the client.
