---
name: refresh-content
version: "2.0"
description: |
  Make an older post current again. Updates outdated facts, stats, tool references, and advice. Adds new sections for topics that have emerged since publication. Does not change the post's core topic or target keyword — that's write-blog's job. Works on any stack: edits the local source file (markdown/MDX, HTML, component meta tags, JSON-LD) with a before/after diff, or outputs paste-ready recommendations when only a URL is available.
  Triggers: refresh blog, update article, actualise content, update post, outdated content, content refresh, refresh my article, make this post current, update this article.
  Requires: A URL or a local repo path. All MCPs optional — GSC MCP for keyword/traffic data, Keywords Everywhere for related-term discovery, a CMS MCP if one is connected. Live crawl (WebFetch) + local files are the backbone.
  Workflow: Read → Audit → Propose → Approve → Update → Publish/Recommend.
---

# Refresh Content

Older posts go stale. Tools get renamed, stats become outdated, advice stops being valid, new developments make sections incomplete. This skill finds what's stale and proposes targeted updates — without changing the post's purpose or target keyword.

**What this skill does:** Updates facts, dates, tool references, statistics, and adds sections for topics that emerged after the post was published.

**What this skill does NOT do:** Change the target keyword, restructure the post's argument, or rewrite sections that are still accurate. For a new post, use `/write-blog`.

**How it applies changes:**
- **In a repo** — the skill READS and EDITS the actual source file for the article (markdown/MDX, HTML, the component holding the meta tags, JSON-LD in code, the sitemap source). It always shows a clear **before/after diff for approval BEFORE writing**. Use search to locate the file for a given URL or slug (front-matter `slug:`/`permalink:`, the filename, or a matching heading).
- **No repo (URL only)** — operate read-only: OUTPUT the exact changes to make (rewritten copy blocks, meta tags, schema snippets) for the user to paste in. No file edits.

---

## Prerequisites

- **Required:** a target — either a public URL (read-only mode) or a local repo path (apply mode)
- **Optional:** GSC MCP — to check if the post is still getting impressions (helps prioritize what to fix)
- **Optional:** Keywords Everywhere — to check if related terms have emerged worth adding
- **Optional:** a CMS MCP, if the site is served from a connected CMS rather than local files

All MCPs are optional. Never stop because an MCP is missing — degrade gracefully and note it. Live crawl (WebFetch) plus local files are the backbone.

See CLAUDE.md for standard MCP discovery (all MCPs optional), config loading, and abort guard.

---

## Modes

| Mode | Command | Use Case |
|------|---------|----------|
| **Full** | `/refresh-content` | Read → audit → propose → single approval → apply (edit files / output recommendations) → publish |
| **Audit only** | `/refresh-content:audit` | Analysis and proposed changes only — nothing written |

---

## Workflow

```
READ → AUDIT → PROPOSE → [ONE APPROVAL] → UPDATE → PUBLISH/RECOMMEND
```

There is **one approval gate** after PROPOSE. The user reviews all proposed changes at once, approves or adjusts, then execution runs without further interruption.

---

## Phase 0: Setup

Follow CLAUDE.md for: MCP discovery (all MCPs optional), config loading (use `brandVoice.tone`, `brandVoice.avoid`, `audience.expertiseLevel`), and abort guard.

**Get target article** — ask the user for a URL or article name if not provided:
- Determine the mode: are we inside a repo (apply mode) or working from a URL only (read-only mode)?
- **Apply mode:** locate the source file for this article. Search by front-matter `slug`/`permalink`, by filename, or by a matching title/H1. Confirm the file path with the user if there is any ambiguity. Read the full file: body content, front-matter (title, meta title/description, publish date), and any inline schema.
- **Read-only mode:** crawl the live URL (WebFetch) for body content, `<title>`, meta description, and visible publish date.
- Capture the **publish date** — this is the staleness baseline.

**Activity log check:**
- If `/refresh-content` ran on this URL in the last 30 days → warn: "Refreshed on [date]. Run again?"
- If `/click-recovery` (or any meta-tag edit) recently touched this page's title → note it to avoid overwriting

**GSC check (if available):**
Fetch last 90 days of queries for this page URL. Note impressions, primary query, position. If impressions < 10: "Low traffic — this post may have been demoted. A refresh helps, but check whether it's worth maintaining."

---

## Phase 1: Audit

Read the full post and check for staleness across four categories. For each issue found, record: location, what's stale, proposed fix.

### 1.1 Outdated facts and statistics

Flag:
- Statistics with a year (e.g., "In 2022, 43% of...") where the year is more than 18 months ago
- Claims about a tool's pricing, features, or market position
- "As of [date]" statements now outdated
- Past-tense predictions ("by 2025...") that can now be evaluated

Propose the updated fact, or mark "needs verification" for the user to fill in.

### 1.2 Tool and product references

Flag:
- Tools renamed, acquired, shut down, or significantly changed
- Pricing references that are likely outdated
- Step-by-step instructions tied to a UI that has changed
- Competitor comparisons using old positioning

### 1.3 Missing topics

Based on the post's publish date and topic, identify what has emerged since publication:
- New tools, methods, or standards in this topic area
- Platform updates (e.g., a platform launched a feature, a search engine changed behavior, AI answer engines changed how they surface content)
- Developments that make the current coverage incomplete

If GSC data available: check for queries landing on this page that the post doesn't cover — those are explicit addition candidates.

If Keywords Everywhere available: run `get_related_keywords` on the primary keyword, flag high-volume terms the post doesn't address.

For each missing topic: propose a section title and 2–3 sentence summary of what it should cover. The user provides the actual expertise — this skill proposes the structure.

### 1.4 Outdated advice

Flag recommendations that are:
- No longer best practice
- Based on platform behavior that has changed
- Dependent on tools or services that no longer exist

### 1.5 Metadata and AEO signals

Check: does the meta title still reflect the primary keyword? Is the meta description still accurate? Flag if a year in the title needs updating (e.g., "Best Tools in 2023").

AEO checks (so answer engines can quote the post):
- Is there a clear, quotable one-sentence answer near the top for the post's core question?
- Are there FAQ-style Q&A pairs that match how people phrase the query?
- Is FAQ / Article schema present and valid (JSON-LD)? Flag missing or stale schema for proposal.

---

## Phase 2: Propose

Compile all findings into a single structured proposal. This is the only approval gate.

```
## Refresh Proposal — [Post Title]

**URL / File:** [url or source file path]
**Published:** [date] ([X months ago])
**GSC:** [X impressions / primary query: "[query]" / pos [X]] or [No GSC data]

### Outdated facts ([N])
1. [Section] — Current: "[quote]" → Proposed: "[update]"
2. [...]

### Tool references ([N])
1. Current: "[tool ref]" → Proposed: "[update or flag as needs verification]"

### Missing topics ([N])
1. Add section: "[Title]"
   What to cover: [2–3 sentences]
   Why: [e.g., "A relevant platform launched X after this post was published"]

### Outdated advice ([N])
1. [Section] — Current: "[excerpt]" → Proposed: "[fix or remove]"

### Metadata & AEO ([N])
| Field | Current | Proposed |
|-------|---------|----------|
| Meta title | [current] | [proposed] |
| Meta description | [current] | [proposed] |
| Quotable answer | [present/absent] | [proposed] |
| FAQ / Article schema | [present/absent/stale] | [proposed] |

**Total:** [N] facts · [N] tools · [N] new sections · [N] advice fixes · [N] metadata/AEO

Approve all, or tell me which to skip.
```

All approved copy goes through a **humanizer pass** before it is written or output — draft → AI-tells audit → final rewrite. Match `brandVoice.tone` and avoid `brandVoice.avoid` terms from config.

Use the CLAUDE.md priority buckets (Must do / High value / Nice to have) to order findings. Do not compute a numeric score.

⚡ GUARD — **Nothing stale found:**
Output: "This post looks current — no outdated facts, tool changes, or missing topics found." Stop. Do not propose cosmetic changes.

⚡ GUARD — **Audit only mode:** Output the proposal and stop. Do not edit files or write anything.

---

## Phase 3: Update

On approval, apply every approved change surgically.

**Apply mode (in a repo):**
- For each approved change, show a **before/after diff** of the exact source-file edit, then write it with a precise find-and-replace edit. Edit only the relevant sentences/sections — do not rewrite surrounding content or reformat the file.
- For new sections: insert after the most relevant existing section, or at the end. Add the heading and a short summary only — flag clearly that the user needs to add the substantive content.
- For metadata/schema: edit the front-matter fields, the component/template meta tags, or the JSON-LD block in source. Update the `lastmod`/updated date if the front-matter or schema carries one.
- If the article URL or any sitemap-relevant field changes, update the sitemap source so the URL and its `lastmod` are current (per CLAUDE.md sitemap maintenance).

**Read-only mode (URL only):**
- Output every approved change as paste-ready blocks: rewritten copy, exact meta tags, and schema (JSON-LD) snippets. Do not write any files.

After applying (or compiling) all changes, show a brief summary:
```
✅ Updated [N] outdated facts
✅ Updated [N] tool references
✅ Added [N] section(s): "[title]" — add your content here
✅ Updated meta title / description / schema
⏭️ Skipped: [N] items

[Apply mode] Ready to publish / commit?
[Read-only mode] Paste the blocks above into your site.
```

---

## Phase 4: Publish / Recommend

- **Apply mode, code-based site:** ask once: "Commit these edits, or leave them in the working tree?" Then per the project's deploy flow (e.g., push to main). Confirm the sitemap reflects any new/changed URL and, if a GSC MCP is connected, resubmit it via `gsc_submit_sitemap`.
- **Apply mode, CMS MCP connected:** ask once: "Push the update live now or save as draft?" and apply via the CMS MCP.
- **Read-only mode:** no publish step — the user pastes the recommended changes into their platform.

If no publish/commit path is available, note it and stop after the diff — never block on a missing MCP.

---

## Activity Log

Follow CLAUDE.md format. Log even on early exits, noting the reason. Example:
```
| YYYY-MM-DD | /refresh-content | [Post title]: updated 3 stats, 2 tool refs, added "[section]" section. Edited source file + sitemap. Committed. |
```
