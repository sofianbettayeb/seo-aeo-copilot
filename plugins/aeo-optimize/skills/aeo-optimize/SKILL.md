---
name: aeo-optimize
version: "2.0"
description: |
  Transform any page into an AEO-ready answer. Audits 12 AEO dimensions, proposes a full rewrite, gets approval, then applies the changes by editing the local content file directly (with a before/after diff) — or outputs ready-to-paste blocks when only a URL is available.
  Triggers: aeo optimize, aeo optimization, optimize for ai, answer engine optimization, aeo audit, optimize for chatgpt.
  Requires: nothing mandatory — works on a local repo or a live URL. Optional: GSC MCP for keyword discovery, PageSpeed MCP, a CMS MCP for hosted platforms.
  Command: /aeo-optimize (full workflow), /aeo-optimize:audit (score only).
  Modes: /aeo-optimize (full workflow), /aeo-optimize:audit (score only).
---

# AEO Optimize

Transform any page into an AEO-ready answer. Audits 12 dimensions, proposes targeted rewrites, gets one approval, then applies them — directly to the local source file when working inside a codebase, or as ready-to-paste snippets when you only have a URL.

Works on any stack: Webflow, WordPress, Shopify, Framer, Next.js, custom React/Vite, hand-coded HTML. No platform assumption, no required MCP.

**Modes:**
- `/aeo-optimize` — full workflow: audit → propose → approve → apply
- `/aeo-optimize:audit` — score only, no changes proposed

See CLAUDE.md for standard conventions: file-based operation, all MCPs optional, config at `.claude/seo-copilot-config.json`, activity log at `.claude/reports/{domain}/activity-log.md`, and the abort guard.

**Data sources (all optional, degrade gracefully):** GSC MCP for keyword discovery, PageSpeed MCP, a CMS MCP if the site is hosted on a platform that exposes one. The backbone is always either the local content file or a live crawl of the page — the skill never stops because an MCP is missing.

---

## Phase 0 — Setup

### 0.1 Discovery & config
Follow CLAUDE.md standard pattern. Note which optional tools are available (GSC, PageSpeed, any CMS MCP) — never stop because one is missing. Load `.claude/seo-copilot-config.json` — use `business.name` and author info for schema generation (Phase 3.9). If config is absent, proceed with defaults and ask the user for the author/business name when generating schema.

### 0.2 Determine the apply mode (critical)
This decides how the optimizations get applied at the end:

- **Repo mode** — you are working inside a codebase (a local repository with source files). Edits will be made directly to the **local source file** for the page, with a before/after diff shown for approval before writing.
- **URL-only mode** — you have only a public URL and no local repo. The skill operates **read-only** and **outputs the exact optimized blocks** (FAQ markup, JSON-LD, rewritten headings, meta tags) for the user to apply manually.

If unsure, ask: "Are you working inside the site's codebase (I'll edit the source files directly), or do you just have the URL (I'll output ready-to-paste blocks)?"

Store `apply_mode = "repo"` or `apply_mode = "url"`.

### 0.3 Page input
Ask the user for the target page:
- A full URL (e.g. `https://example.com/guide/disable-subdomain-indexing`)
- Or a slug / local path (e.g. `disable-subdomain-indexing` or `content/guides/disable-subdomain-indexing.md`)

### 0.4 Locate the source (repo mode)
In **repo mode**, find the file that produces this page. Search the repo (filename, slug, and matching title/headings inside files):
- **Markdown / MDX article** — e.g. `content/**/*.md`, `posts/**/*.mdx`, `_articles/*.md`. Common locations: `content/`, `posts/`, `articles/`, `src/content/`, `server/content/`, `blog/`.
- **Component file** — `.tsx` / `.jsx` / `.vue` / `.astro` rendering the page (routes under `pages/`, `app/`, `src/routes/`, `src/pages/`).
- **Static HTML** — `*.html` matching the slug.
- **Data file** — JSON/YAML feeding a template (e.g. `src/data/*.json`).

Read the file. Confirm with the user: "Found the source at `{path}` — is this the right file?" Store `source_path`. If you cannot find it, ask the user for the path, or fall back to URL-only mode.

In **URL-only mode**, skip this — you'll crawl the live page in Phase 1.3.

### 0.5 Activity log check
Read `.claude/reports/{domain}/activity-log.md`. Check the last 10 entries for:
- Recent work on this specific page (avoid re-doing changes < 30 days old)
- Prior `/aeo-optimize` runs on this page
- Recent content/CTR work that affected this page

---

## Phase 1 — Page & Query Discovery

### 1.1 Fetch GSC queries for this page (optional)
If GSC MCP is available, call it with `dimensions: ["query"]`, filtered to this specific page URL, 90-day window.

```
startDate: 90 days ago
endDate: today
dimensions: ["query"]
page: {full page URL}
rowLimit: 50
```

If GSC returns < 5 rows or 0 impressions, expand to 180 days. If still empty — or GSC isn't connected — ask the user to provide the primary keyword manually.

### 1.2 Identify primary and secondary queries
- **Primary query:** highest impression count among clean queries (apply anomaly filter: strip `^\d+[:\,]`, `%[0-9A-Fa-f]{2}`, JSON fragments). If no GSC, use the keyword the user provided.
- **Secondary queries:** top 3 by impressions after the primary (or ask the user for 2–3 related variations).
- Store: `primary_query`, `secondary_queries[]`, `primary_position`, `primary_impressions` (positions/impressions only if GSC available).

Show the user:
```
Primary query: "{primary_query}" — {impressions} impressions, position {position}
Secondary: {q2}, {q3}, {q4}
```
(Omit impressions/position if GSC wasn't used.)

Ask: "Is this the right keyword to optimize for? Or would you like to target a different one?"

Wait for confirmation before proceeding.

### 1.3 Get the page content
- **Repo mode:** use the content already read from `source_path` in Phase 0.4. If the page is assembled from multiple files (layout + content + data), read each relevant part and note which file holds the body, which holds meta tags, and which holds (or should hold) the JSON-LD.
- **URL-only mode:** crawl the live page. Extract the rendered HTML — H1, headings, body text, meta title, meta description, existing `<script type="application/ld+json">`, and internal links.

Identify where each editable element lives:
- **Body content** — main article/page text
- **Meta title** — `<title>` or frontmatter `title` / SEO-title field
- **Meta description** — `<meta name="description">` or frontmatter `description`
- **JSON-LD** — existing `<script type="application/ld+json">` block(s); note `schema_exists = true/false`

### 1.4 Extract content structure
Parse the body content to identify:
- **H1** — first heading or title field
- **H2 list** — all second-level headings
- **Intro** — first 100 words of body
- **Paragraphs** — count, average word count per paragraph
- **Bullet lists** — count
- **Tables** — count
- **Steps / numbered lists** — count
- **FAQ section** — detect by H2/H3 containing "FAQ", "Frequently Asked", "Questions"
- **Summary section** — detect by H2 containing "Summary", "Conclusion", "Key Takeaways"
- **Internal links** — extract all links pointing to the same domain
- **Existing schema** — detect `<script type="application/ld+json">`

---

## Phase 2 — AEO Audit

Score each dimension. Output a table showing pass ✅ / partial ⚠️ / fail ❌ with a one-line note.

| # | Dimension | Check | Weight |
|---|-----------|-------|--------|
| 1 | Query alignment | Primary query in H1 AND in first 100 words AND in meta title | High |
| 2 | Direct answer (≤100 words) | Body opens with a self-contained answer to the primary query | High |
| 3 | Question-based sections | ≥60% of H2s phrased as questions | Medium |
| 4 | Bullet lists | Dense paragraphs (>4 lines) replaced with bullets where content permits | Medium |
| 5 | Tables | Comparison or multi-attribute content formatted as tables | Medium |
| 6 | Step-by-step sections | Procedural content uses numbered lists or step format | Medium |
| 7 | Definition sentences | Key concepts have explicit "X is Y" or "X refers to Y" definitions | Medium |
| 8 | Short paragraphs | No paragraph exceeds 4 sentences; avg ≤ 2.5 sentences | Medium |
| 9 | FAQ section | FAQ exists with ≥ 5 questions relevant to the primary query cluster | High |
| 10 | Schema markup | FAQPage + Article JSON-LD present (HowTo if steps exist) | High |
| 11 | Internal links | 3–5 relevant internal links with descriptive anchor text | Medium |
| 12 | Summary section | Page ends with a summary, conclusion, or key takeaways block | Low |

**Scoring:**
- ✅ Pass = 2 pts
- ⚠️ Partial = 1 pt
- ❌ Fail = 0 pts

Total = `/24`. Express as percentage and maturity level:
- 20–24 (83–100%) — **AEO Ready**
- 14–19 (58–83%) — **Partially Optimized**
- 8–13 (33–58%) — **Needs Work**
- 0–7 (0–33%) — **Not AEO Ready**

In `/aeo-optimize:audit` mode: output the audit table + score + top 3 gaps ranked by impact. Stop here.

In `/aeo-optimize` mode: proceed to Phase 3.

---

## Phase 3 — Proposal Generation

Generate the full rewrite proposal. Each element below must be written, not just described.

Apply humanizer rules to all generated content before presenting:
- No AI vocabulary (seamless, robust, delve, comprehensive, leverage, etc.)
- No em-dash overuse
- No rule-of-three padding
- No inflated symbolism
- Short, direct sentences

Write each block in the **format the source uses** so it can drop straight in:
- Markdown source → Markdown headings, lists, and frontmatter
- JSX/TSX/Vue/Astro component → JSX/HTML matching the existing markup conventions
- Static HTML → plain HTML
- URL-only → clean HTML (or Markdown if the user prefers), clearly labeled for manual paste

### 3.1 Rewritten intro (first 100 words)
Write a new opening paragraph that:
- Directly answers the primary query in the first 2 sentences
- States what the page covers
- Uses the primary query naturally in sentence 1 or 2
- Does not exceed 100 words
- Reads like it was written by a practitioner, not an AI

### 3.2 H1 update (if needed)
If the current H1 doesn't contain the primary query, propose a new H1:
- Contains the primary query
- Under 60 characters
- Framed as a statement or question (not keyword-stuffed)

### 3.3 H2 restructuring
For each existing H2, propose a question-phrased version where the content supports it.

Format:
```
Current: "How to Implement HTTPS"
Proposed: "How Do You Implement HTTPS on Your Site?"
```

Don't force every H2 into a question — only where it makes sense. Keep section meaning intact.

### 3.4 Content reformatting
Identify the 2–3 densest paragraphs. Propose bullet or table replacements.

For each:
```
CURRENT (dense paragraph):
[paste the paragraph]

PROPOSED (reformatted):
[bullet list or table]
```

For procedural content, propose numbered step format. Each step: one sentence + one optional detail sentence.

**When a table is the right format for comparison or multi-attribute content:**
Write the table in the source's native format:
- Markdown source → a Markdown table.
- JSX/HTML/static HTML → a semantic `<table>` with `<thead>`/`<tbody>`.
- If the source pipeline strips raw tables (some CMS rich-text fields do), fall back to a per-item structured list as the body content, and provide the full HTML table separately for manual insertion:
```html
<p><strong>Option Name</strong></p>
<ul>
  <li><strong>Attribute 1:</strong> Value</li>
  <li><strong>Attribute 2:</strong> Value</li>
</ul>
```
Keep table styling minimal and consistent with the site's existing CSS; only use inline styles when the platform offers no class hook. Vary the visual treatment per page where you control it — don't reuse a fixed template.

### 3.5 Definition sentences
For each technical term or concept in the primary keyword cluster, propose a one-sentence definition to insert at first use:
```
Concept: "subdomain indexing"
Definition: "Subdomain indexing is when Google crawls and indexes your staging URL alongside your live domain — treating them as separate, competing sites."
```

### 3.6 FAQ section
Write a complete FAQ section with 5–7 questions. Rules:
- Questions must match real GSC queries or natural query variations
- Each answer: 2–4 sentences, self-contained
- Ordered: most important first
- First question must directly address the primary query
- Avoid repeating content already in the body verbatim

Format (adapt headings to the source's syntax):
```
## Frequently Asked Questions

### {Question 1}
{Answer 1}

### {Question 2}
{Answer 2}
...
```

### 3.7 Summary section
Write a 3–5 bullet summary section placed at the end of the page:
```
## Summary

- {Key point 1}
- {Key point 2}
- {Key point 3}
- {Key point 4 if needed}
- {Key point 5 if needed}
```

Each bullet: one sentence, action or insight-oriented.

### 3.8 Internal links
Identify 3–5 pages on the site that are topically relevant to this content. For each:
- **Anchor text:** descriptive, not "click here" or "this page"
- **Target:** same domain only, scoped to same section/category where possible
- **Placement:** specific sentence in the body where the link fits naturally

Format for each:
```
Link {n}:
Anchor: "{anchor text}"
Target: {/path/to/page}
Placement: Insert in paragraph starting with "..."
```

To find relevant pages: in repo mode, search the content directory / route list for sibling pages and match by topic overlap with the primary query; in URL-only mode, use the site's sitemap or known section paths.

**Scope rules:**
- Maximum 5 new links
- Do not duplicate existing internal links already in the content
- Do not link to the same URL twice
- Prefer pillar pages and closely related items

### 3.9 Schema markup (JSON-LD)
Generate the complete schema block.

**Always include:** `Article`
```json
{
  "@context": "https://schema.org",
  "@type": "Article",
  "headline": "{H1}",
  "description": "{meta description}",
  "author": {
    "@type": "Person",
    "name": "{author name from config or ask user}",
    "url": "{author URL from config}"
  },
  "publisher": {
    "@type": "Organization",
    "name": "{business.name from config}",
    "url": "https://{domain}"
  },
  "datePublished": "{original publish date or today}",
  "dateModified": "{today}"
}
```

**Include `FAQPage` when FAQ is added:**
```json
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {
      "@type": "Question",
      "name": "{Question 1}",
      "acceptedAnswer": {
        "@type": "Answer",
        "text": "{Answer 1}"
      }
    }
    // ... repeat for each FAQ question
  ]
}
```

**Include `HowTo` when numbered steps are added:**
```json
{
  "@context": "https://schema.org",
  "@type": "HowTo",
  "name": "{H1}",
  "step": [
    {
      "@type": "HowToStep",
      "name": "{Step 1 heading}",
      "text": "{Step 1 description}"
    }
  ]
}
```

Output as a single `<script type="application/ld+json">` block combining all applicable types. Note where it should live for the source:
- **Markdown** — many static-site setups inject JSON-LD via frontmatter or a layout component; if the renderer supports a raw HTML block, embed the `<script>` directly, otherwise add the data to frontmatter and the layout.
- **JSX/TSX component** — render via a `<script type="application/ld+json" dangerouslySetInnerHTML={{ __html: JSON.stringify(...) }} />` (or framework equivalent, e.g. Next.js `<Head>` / metadata API).
- **Static HTML** — paste the `<script>` block into `<head>` or before `</body>`.
- **URL-only** — output the block and instruct the user to add it to the page head.

### 3.10 Meta title and description update (if needed)
If the primary query isn't in the current meta title:
- Propose a new meta title (50–60 chars, query-first)
- Propose a new meta description (140–155 chars, includes primary query + clear benefit)

Note the location: frontmatter fields, the component's `<head>`/metadata block, or the static `<title>`/`<meta>` tags.

---

## Phase 4 — Approval Gate

Present the full proposal to the user in a structured summary:

```
## AEO Optimization Proposal — {page slug}
Primary query: "{primary_query}"
Current AEO score: {score}/24 ({level})
Post-optimization estimated score: {estimated}/24
Apply mode: {repo — edits {source_path} | url — outputs blocks for manual paste}

### Changes proposed:
1. ✏️  Intro rewrite (first 100 words)
2. 🏷️  H1 update: "{new H1}"
3. 📋  {N} H2s restructured as questions
4. 📝  {N} content blocks reformatted (bullets/tables/steps)
5. 💬  {N} definition sentences added
6. ❓  FAQ section (7 questions)
7. 📌  Summary section (4 bullets)
8. 🔗  {N} internal links added
9. 🧩  Schema markup (Article + FAQPage{+ HowTo})
10. 🏷️  Meta title + description updated

[Full proposal follows below]
```

Then show the complete content for each section (3.1–3.10).

Ask: "Do you want to proceed with all changes? Or adjust specific sections before I apply them?"

**If user approves all:** proceed to Phase 5.
**If user wants edits:** incorporate feedback, re-present the affected sections, confirm again.
**If user cancels:** stop. Do not modify anything.

---

## Phase 5 — Apply

### 5.1 Merge content
Take the original page body and apply all approved changes:
- Replace intro with 3.1
- Update H1 with 3.2 (if changed)
- Replace H2 text with 3.3 versions
- Replace/reformat dense paragraphs with 3.4 content
- Insert definition sentences at first use per 3.5
- Append FAQ section (3.6) before Summary
- Append Summary section (3.7) at end
- Insert internal links per 3.8 placements
- Add the JSON-LD per 3.9 to the correct location
- Update meta title/description per 3.10
- Do not remove or rearrange content that wasn't flagged for change

### 5.2 Apply changes

**Repo mode — edit the local source file directly:**
1. Construct the exact edits to `source_path` (and any sibling files that hold meta tags or JSON-LD, e.g. a layout component or frontmatter).
2. **Show a before/after diff for every file you will change** and get explicit approval before writing. Example:
   ```
   --- {source_path} (before)
   +++ {source_path} (after)
   @@
   - {old line}
   + {new line}
   ```
3. On approval, apply the edits to the file(s). Preserve the source's formatting conventions (indentation, frontmatter style, JSX patterns).
4. Do **not** run build/deploy/publish commands. Tell the user the files are edited and they can preview, commit, and deploy through their normal workflow.
5. If a CMS MCP is connected and the user prefers to push through it instead of editing local files, offer that as an alternative — but the default is the local-file edit with diff.

**URL-only mode — read-only, output blocks for manual application:**
- Output each optimized block in a copy-ready form, clearly labeled with where it goes:
  - Rewritten intro / H1 / H2s
  - Reformatted content (bullets/tables/steps)
  - FAQ section markup
  - Summary section
  - Internal links (anchor + target + placement)
  - JSON-LD `<script>` block (state: add to page head)
  - New meta title + description
- Do not attempt to modify anything. Make clear these are for the user to apply in their editor/CMS.

---

## Phase 6 — Report & Log

### 6.1 Output completion summary
```
## ✅ AEO Optimization Complete — {page slug}

Primary query: "{primary_query}"
AEO score: {before}/24 → {after}/24 ({level_before} → {level_after})

Changes applied:
- ✏️  Intro rewritten (direct answer in first 100 words)
- 🏷️  H1 updated: "{new H1}"
- 📋  {N} H2s converted to questions
- 📝  {N} content blocks reformatted
- 💬  {N} definitions added
- ❓  FAQ added ({N} questions)
- 📌  Summary section added
- 🔗  {N} internal links inserted
- 🧩  Schema: Article + FAQPage{+ HowTo}
- 🏷️  Meta title + description updated

{apply note: "Edited {source_path} — preview & deploy through your normal workflow" OR "Blocks output above for manual application"}
```

### 6.2 Save report
Save to `.claude/reports/{domain}/aeo-optimize-{slug}-{YYYY-MM-DD}.md`

Include: audit scores (before/after per dimension), full proposal as reference, schema block, internal links added, and the file(s) edited (repo mode) or the blocks output (URL mode).

### 6.3 Activity log
Append to `.claude/reports/{domain}/activity-log.md`:
```
| {date} | /aeo-optimize | AEO optimization on /{slug}. Query: "{primary_query}". Score: {before}→{after}/24. FAQ added ({N}q), schema Article+FAQPage{+HowTo}, {N} internal links. {Edited {source_path} | Blocks output for manual apply}. |
```

---

## Error handling

| Error | Action |
|-------|--------|
| GSC unavailable or returns no data | Ask user to provide the primary keyword manually |
| Source file not found in repo | Ask user for the path, or fall back to URL-only mode |
| Page assembled from multiple files | Identify which file holds body / meta / JSON-LD; edit each with its own diff |
| Source pipeline strips raw tables | Use a structured list in the body; provide the HTML table separately for manual insertion |
| Body field not identifiable | Ask user to confirm which file/field holds the main content |
| Live page can't be crawled (URL mode) | Ask the user to paste the page content or provide the source file |
| Renderer doesn't support inline JSON-LD | Add schema via the layout/metadata mechanism the framework provides (frontmatter + layout, Next.js metadata, etc.) |

---

## Integration with other skills

| Situation | Recommended next step |
|-----------|----------------------|
| Page has outdated facts or stale content | Refresh/actualize the content first, then `/aeo-optimize` |
| Page has a CTR problem after AEO | Run a click-recovery pass — AEO changes may shift the optimal title |
| Want to find which pages to AEO-optimize | Run a keyword-opportunity pass — pages with striking-distance queries are best candidates |
| Site-wide AEO audit | Run a deep audit to identify all pages below the AEO threshold |
