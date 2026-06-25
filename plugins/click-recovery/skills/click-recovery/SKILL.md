---
name: click-recovery
version: "2.0"
description: |
  Find pages search engines trust but users ignore — high impressions, low CTR. Get a prioritized list of titles and meta descriptions to rewrite, then apply them by editing the local source file with a before/after diff (or output a paste-ready table when only URLs are available).
  Triggers: click recovery, improve CTR, low CTR, fix CTR, wasted impressions, title optimization, meta description optimization, SERP optimization.
  Requires: Google Search Console MCP server (the data source that finds CTR opportunities). Optional: a local repo to apply edits, Keywords Everywhere API for volume/intent validation.
  Workflow: Analyze → Recommend → Approve → Apply (edit local files with diff, or output table).
  Command: /click-recovery
  Modes: /click-recovery:analyze (report only), /click-recovery (full workflow with apply).
---

# Click Recovery

Find pages where a search engine already ranks you but users scroll past. These are your fastest wins — no need to build authority, just fix the pitch.

This skill is **stack-agnostic**. The pages it fixes can live in a markdown/MDX file, an HTML file, a component holding the `<title>`/`<meta name="description">`, a head config, or a CMS. GSC tells us *which* pages are underperforming; the apply step edits *wherever those pages actually live*.

**How it applies changes:**
- **In a repo (apply mode)** — for each approved page, the skill LOCATES the source that produces its title and description (front-matter `title:`/`description:`, a `<title>`/`<meta name="description">` in a component or `.html` file, or a head/SEO config), EDITS the file to the approved values, and always shows a **before/after diff for approval BEFORE writing**.
- **No repo (read-only mode)** — GSC plus URLs only: OUTPUT a table of approved new titles/descriptions per URL for the user to apply manually. No file edits.

---

## Prerequisites

- **Required:** Google Search Console MCP server. This is the one skill where GSC is effectively required — without it there is no way to find high-impression / low-CTR opportunities.
- **Optional (to apply edits):** a local repo path, or a CMS MCP if the site is served from a connected CMS rather than local files.
- **Optional:** Keywords Everywhere API — for search volume, intent, and demand validation.

See CLAUDE.md for standard MCP discovery (all MCPs optional *except GSC for this skill*), config loading, priority buckets, activity log, and abort guard.

⚡ GUARD — **GSC MCP unavailable:**
This skill needs GSC to find CTR opportunities. If GSC is not connected:
- Tell the user: "Click Recovery finds opportunities from Search Console data, so it needs GSC connected. Please connect the GSC MCP and run again."
- **Still offer a fallback:** "If you give me a specific URL you want to optimize, I can analyze its current title/description and propose improved meta tags (and apply them with a diff if you're in a repo) — just without the impression/CTR data to prioritize across pages."
- Otherwise stop.

---

## Modes

| Mode | Command | Use Case |
|------|---------|----------|
| **Full** | `/click-recovery` | Analyze → recommend → approve → apply (edit local files with diff, or output table). |
| **Analyze only** | `/click-recovery:analyze` | Report only — no changes made. Returns CTR analysis and recommendations. |

If **Full** or no mode specified: run Phases 0–5. If **Analyze only**: run Phases 0–4, output the report, stop.

---

## Workflow

```
[MODE SELECT] → FETCH → FILTER → VALIDATE → PRIORITIZE → RECOMMEND → [APPROVE → APPLY]
```

Phases in brackets only run in full mode. After presenting recommendations, ask which pages to update. On approval, apply by editing the local source files (with a before/after diff) — or, if there is no repo, output a paste-ready table.

---

## Threshold Rationale

These thresholds are based on industry benchmarks and can be adjusted:

| Threshold | Value | Rationale |
|-----------|-------|-----------|
| Min impressions (page) | 500 | Enough data for statistical significance |
| Min impressions (query) | 100 | Lower bar for query-level insights |
| Low CTR | < 2% | Below typical position 1–10 averages |
| Position cutoff | ≤ 20 | Realistic click potential; beyond page 2 clicks are rare |
| Ranking opportunity | 21–50 | Visible to the search engine but not to users |
| Min search volume | 50/month | Below this, effort may not be worth it |

---

## Conditional Guards (Global)

These guards apply throughout the skill execution.

⚡ GUARD — **Load SEO Copilot config:**
At the start of execution, check if `.claude/seo-copilot-config.json` exists:
- If yes: load and apply settings:
  - `brandVoice.tone` → match tone in title/description rewrites
  - `brandVoice.avoid` → filter out forbidden words from suggestions
  - `seo.metaTitleFormat` → apply brand suffix/prefix to titles
  - `audience.primary` → frame messaging for the target audience
- If no: proceed with defaults, note once: "No SEO Copilot config found. Run `/getting-started` for personalized recommendations."

⚡ GUARD — **User requests abort:**
If the user says "stop", "cancel", "abort", or "nevermind" at any phase, follow the abort guard in CLAUDE.md: confirm, then exit cleanly with a summary and any partial results (e.g., analysis done but not applied).

⚡ GUARD — **No repo / no write target available:**
If running in full mode but there is no local repo (and no connected CMS MCP) to apply edits to:
- Run analysis and recommendations (Phases 0–4)
- Skip the file-edit step in Phase 5
- Output the approved recommendations as a paste-ready table (URL → new title / new description) so the user can apply them manually.
- Note: "No repo or CMS connected — recommendations generated as a table for manual application."

⚡ GUARD — **Keywords Everywhere unavailable:**
If the KE API is not available:
- Proceed without volume/intent data
- Note in the report: "Search volume and intent data unavailable. Recommendations based on GSC impressions only."
- Suggest the user add the KE API for richer insights.

---

## Phase 0: Mode Selection & Prerequisites

### 0.1 Choose Mode

At the start, determine which mode to run:

```
How would you like to run Click Recovery?

1. Full workflow (recommended) — Analyze, recommend, and apply approved changes
2. Analyze only — Generate report with recommendations, no changes
```

- If **Full workflow** or no mode specified: run Phases 0–5
- If **Analyze only**: run Phases 0–4, output the report, stop

### 0.2 Prerequisites Check

Run MCP discovery per CLAUDE.md. Specifically:

**GSC MCP (required):**
- Search: `+gsc search analytics`
- If missing: apply the GSC guard above (instruct the user to connect GSC; offer the single-URL fallback).

**Apply target (required only to write changes in full mode):**
- Determine whether we are inside a repo (apply mode) or have a connected CMS MCP. If neither, full mode falls back to outputting a table (see the no-repo guard). Note: "No write target — running in recommend-and-table mode."

**Keywords Everywhere (optional but check):**
- Search: `+keywords everywhere volume`
- If missing: continue, but ADD to the report header:
  ```
  ⚠️ Keywords Everywhere not connected.
  Recommendations based on GSC impressions only.
  Connect KE for search volume and intent data.
  ```

---

## Phase 1: Fetch GSC Data

### 1.1 Identify Target Site & Set {domain}

Ask the user which property to analyze if multiple GSC properties are available. Extract the domain from the selected property URL (e.g., `https://www.example.com/` → `example.com`).

### 1.2 Review Activity Log

Check `.claude/reports/{domain}/activity-log.md`:
- If it exists: read the last 10 entries and surface a brief summary:
  ```
  Recent activity on {domain}:
  - 2026-02-10 /weekly-report — Week W06 report. Health: STABLE.
  - 2026-02-05 /click-recovery — Updated meta titles on 5 pages.
  ```
- **Redundancy check:** if `/click-recovery` was run in the last 7 days → warn: "You ran `/click-recovery` on [date]. CTR changes typically take 2–3 weeks to show in GSC. Run again anyway?"
- **Content refresh check:** scan all entries for `/refresh-content` runs in the last 14 days. Extract any page URLs mentioned. Store as `recently_refreshed_pages` — used in Phase 5.4 to warn before overwriting titles that were just set by a full content refresh.
- If the log doesn't exist: proceed silently.

Use recent activity as context (e.g., if `/weekly-report` flagged specific CTR issues, prioritize those pages).

### 1.3 Pull Performance Data

Use the GSC MCP to fetch the last 90 days of data:

**Query-level data:**
- All queries with impressions > 100
- Include: query, page, impressions, clicks, CTR, position

**Page-level data:**
- All pages with impressions > 500
- Include: page, impressions, clicks, CTR, average position

### 1.4 Calculate Benchmarks

Compute site-wide averages:
- Average CTR by position bracket (1–3, 4–10, 11–20)
- Overall average CTR
- Median impressions per page

These benchmarks help identify underperformers relative to the site's own performance.

---

## Phase 2: Filter Opportunities

### 2.1 CTR Opportunity Filter

Flag pages/queries meeting these criteria:

**High-impression, low-CTR pages:**
- Impressions > 500
- CTR < site average for that position bracket
- Position ≤ 20 (realistic click potential)
- **OR** Position 21–30 with impressions > 1000 (high-volume ranking opportunities)

**Wasted impression queries:**
- Impressions > 100
- CTR < 2% (or < half the position-bracket average)
- Position ≤ 10 (should be getting clicks)

**Ranking opportunities (separate category):**
- Position 21–50 with impressions > 1000
- These pages rank but need content improvement to reach page 1
- Flag for `/refresh-content` rather than just meta tag fixes

### 2.2 Quick Win Filter

Prioritize opportunities where small changes yield big impact:
- Position 1–3 with CTR < 5% → title/description mismatch with intent
- Position 4–10 with CTR < 2% → being outcompeted by better snippets
- High-impression queries not reflected in the current title → keyword gap

### 2.3 Group by Page

Consolidate query-level findings to page level:
- List all underperforming queries per page
- Sum total wasted impressions per page
- Identify the primary keyword (highest impressions) for each page

### 2.4 Capture Current Meta Tags

**Do this BEFORE making recommendations.** You need the current title and description for each flagged page to build the before/after comparison.

**Apply mode (repo or CMS connected):** for each flagged page URL, locate the source that produces its meta tags and read the current values.
1. Map the URL/slug to its source file or content item. Search by:
   - front-matter `slug:`/`permalink:`, or the filename, in a markdown/MDX content folder
   - a `<title>` / `<meta name="description">` in a matching `.html` file or component
   - a route entry in a head/SEO config or a CMS item matching the path
2. Read the current **meta title** (or the page `<h1>`/`title` field if no dedicated meta title) and current **meta description** (or a summary/excerpt field if no dedicated field).
3. Note where each value lives (file path + the exact line/field) so Phase 5 can edit precisely.

**Read-only mode (URL only):** crawl the live URL to capture the current `<title>` and `<meta name="description">`.

⚡ GUARD — **No meta description present:**
If a page has no dedicated meta description field/tag:
- Use the closest available summary/excerpt if one exists.
- If none exists: note in the report "No meta description found. Consider adding a `description` field / `<meta name=\"description\">` to this page."
- Still generate a recommended meta description for manual use.

⚡ GUARD — **No opportunities found:**
If filtering returns zero pages:
- Inform the user: "No significant CTR opportunities found. Your titles and meta descriptions are performing well relative to your positions."
- Suggest: "Consider running `/refresh-content` on older articles to improve rankings first, then re-check CTR."

⚡ GUARD — **Low traffic site:**
If total impressions < 1000 in 90 days:
- Warn: "Limited data available. Results may not be statistically significant."
- Proceed but note confidence is lower.

Store all current values — they are needed for the before/after comparison in recommendations.

---

## Phase 3: Validate with Keywords Everywhere

**IMPORTANT:** do not silently skip this phase. Either (1) use KE data to enrich recommendations, or (2) explicitly note in the report that KE was unavailable (see Prerequisites Check).

If the Keywords Everywhere MCP is available:

### 3.1 Enrich Query Data

For each flagged query, fetch:
- Monthly search volume
- CPC (indicates commercial intent)
- Competition score
- Trend data (rising/falling)

### 3.2 Intent Classification

Classify each query by intent:
- **Informational:** how, what, why, guide, tutorial
- **Commercial:** best, top, review, vs, compare
- **Transactional:** buy, price, discount, deal, near me
- **Navigational:** brand names, specific product names

### 3.3 Validate Demand

Filter out queries with:
- Search volume < 50/month (unless highly commercial)
- Falling trend > 30% YoY (dying queries)

Flag high-value queries:
- Volume > 1000/month
- CPC > $2 (commercial intent)
- Rising trend

---

## Phase 4: Prioritize & Recommend

### 4.1 Ranking Model

Rank pages by CTR opportunity using these signals (higher = do first):
1. **Wasted impressions** — pages with the most impressions getting no clicks
2. **CTR gap** — how far below the expected CTR for their position bracket
3. **Position** — positions 1–10 have higher recovery potential than 11+
4. **Keyword value** — volume × CPC as a tiebreaker when KE is connected
5. **Quick fix** — title missing the primary keyword = bump up

### 4.2 Rank Opportunities

Sort pages by rank and group into the three priority buckets in CLAUDE.md:
- **Must do** — a page's CTR is less than half the expected rate for its average position (confirmed in GSC) AND impressions are above the site average
- **High value** — meaningful CTR gap with 50+ weekly impressions, or the primary keyword is missing from the title on a high-impression page
- **Nice to have** — small CTR gap, low impressions, or marginal improvement potential

### 4.3 Generate Recommendations

For each page, generate title and meta description suggestions:

**Titles (≤60 characters):**
- Lead with the primary keyword
- Include a compelling hook (number, year, power word)
- Match search intent (informational = "Guide", commercial = "Best", etc.)
- Differentiate from competitors in the SERP
- Apply the brand suffix from config if set (e.g., "| Brand Name")

**Meta descriptions (≤155 characters):**
- Include the primary keyword naturally
- Add a clear value proposition
- Include a call-to-action or curiosity hook
- Mention specifics (numbers, timeframes, outcomes)

**Intent-based framing:**
- Informational: "Learn how to...", "Complete guide to...", "X steps to..."
- Commercial: "Best X for [use case]", "X vs Y: Which is...", "Top X in [year]"
- Transactional: "Get X today", "Free X", "X% off", "Starting at $X"

### 4.4 Avoid AI Writing Tell-Tales

**Phrases to NEVER USE in titles/descriptions:**
- "Ultimate Guide" (overused)
- "Everything You Need to Know"
- "game-changer" / "revolutionary"
- "In [Year]" at the start (put the year elsewhere)
- "Discover how..."
- "Unlock the secrets..."

**Patterns to AVOID:**
- Identical structure across all suggestions ("Best X for Y" for every title)
- Generic value props: "save time and money"
- Exclamation marks in descriptions
- Starting every description with the brand name

**Human signals to ADD:**
- Specific numbers: "47 templates" not "dozens of templates"
- Honest qualifiers: "for small teams" instead of "for everyone"
- Casual confidence: "The only X you'll need" (sparingly)
- Outcome focus: "Rank higher in 30 days" not "Improve your SEO"

### 4.5 Output Report

Present findings as a structured report:

```
# Click Recovery Report

**Site**: [property URL]
**Period**: Last 90 days
**Total wasted impressions**: [sum across all flagged pages]
**Estimated recoverable clicks**: [impressions × expected CTR improvement]

**Data sources**:
- ✅ GSC: Connected
- ✅ Apply target: Local repo / CMS connected [or ⚠️ None — table output only]
- ✅ Keywords Everywhere: Connected [or ⚠️ Not connected — no volume/intent data]
- ✅ SEO Copilot Config: Loaded [or ℹ️ Not found — using defaults]

[If KE not connected, add:]
> ⚠️ Keywords Everywhere not connected. Recommendations based on GSC impressions only.

---

## Must Do

### 1. [Page Title]
**URL**: [page URL]
**Source**: [file path + field/line, or "URL only — manual apply"]
**Current performance**:
- Impressions: [X] | Clicks: [X] | CTR: [X]% | Avg Position: [X]
- Expected CTR for position: [X]% | CTR gap: [X]%

**Top underperforming queries**:
| Query | Impressions | CTR | Position | Volume | Intent |
|-------|-------------|-----|----------|--------|--------|
| ...   | ...         | ... | ...      | ...    | ...    |

**Diagnosis**: [Why is CTR low? Title mismatch? Weak description? Competitor snippets?]

**Recommended changes**:

| Field | Current | Suggested | Chars |
|-------|---------|-----------|-------|
| Meta title | [current title] | [suggested title] | [N]/60 |
| Meta description | [current desc or "❌ Missing"] | [suggested desc] | [N]/155 |

**Keyword framing**: [advice based on intent]
**Estimated impact**: +[X] clicks/month

---

[Repeat for each Must Do page]

## High Value
[Summary table with key metrics, current titles, suggested titles]

## Nice to Have
[Summary table with key metrics]

## Ranking Opportunities (Position 21–50)

These pages have high impressions but need more than meta tag fixes to reach page 1:

| Page | Impressions | Position | Recommendation |
|------|-------------|----------|----------------|
| [title] | [X] | [X] | Run `/refresh-content` for full optimization |

---

## Next Steps

1. Review and approve Must Do recommendations
2. Apply approved changes (this skill edits your source files with a diff, or hands you a table)
3. For ranking opportunities: run `/refresh-content [URL]`
4. Monitor CTR in GSC after 2–4 weeks
5. Re-run `/click-recovery` monthly
```

### 4.6 Competitor SERP Analysis (Optional)

For Must Do pages, offer to analyze the SERP:
1. Web search the primary keyword
2. Extract titles and descriptions of the top 5 results
3. Identify patterns (what hooks are they using?)
4. Find differentiation opportunities

⚡ GUARD — **Analyze-only mode:**
If running in analyze-only mode (`/click-recovery:analyze`):
- Output the report
- End with: "Analysis complete. Run `/click-recovery` (full mode) to apply approved changes."
- Stop here — do not proceed to Phase 5.

---

## Phase 5: Apply

### 5.1 User Approval

After presenting the report, ask:

```
Which pages would you like to update?
1. All Must Do pages ([N] pages)
2. Select specific pages
3. Review only — don't apply yet
```

If the user selects option 3, stop here.

### 5.2 Locate Each Page's Source

For each approved page, confirm where its title and description actually live (this was captured in Phase 2.4; re-locate if needed):

- **Markdown / MDX:** front-matter `title:` / `description:` (or `meta_title` / `meta_description`).
- **HTML file:** the `<title>` element and `<meta name="description">`.
- **Component / template:** the head block or SEO component that renders these tags for the route.
- **Head / SEO config or route map:** the entry keyed by the page path or slug.
- **CMS (if a CMS MCP is connected):** the content item matching the path; identify the meta title and description fields.

⚡ GUARD — **Source not found:**
If a URL's source can't be located in the repo:
- Tell the user the URL and ask whether it's generated elsewhere (external, a different repo, or deleted).
- Skip that page and continue with the others — or, if the user confirms it's manual, include it in the read-only table output instead.

### 5.3 Present Changes for Final Approval (with Diff)

Before writing anything, show the exact changes.

**Apply mode** — show a per-page before/after **diff** of the source file (the exact lines/fields that will change), plus a length-check table:

```
## Ready to Apply

example.com/guide → src/content/guide.md
- title: Old Guide Title
+ title: New, Clearer Guide Title

- description: Old description text...
+ description: New description with the primary keyword and a clear hook...

| Page | Field | New | Status |
|------|-------|-----|--------|
| [title] | Meta title | [new] | ✅ 52 chars |
| [title] | Meta title | [new] | ⚠️ 68 chars — will truncate |
| [title] | Meta description | [new] | ✅ 142 chars |
| ...     | ...   | ... | ... |

Apply these edits to the source files? (yes/no)
```

**Read-only / no-repo mode** — output the paste-ready table only (no diff, no file writes):

```
| URL | New Meta Title | New Meta Description |
|-----|----------------|---------------------|
| ... | ...            | ...                 |
```

**Title length warnings:**
- ⚠️ **> 60 chars:** "Title will be truncated in search results. Consider shortening."
- ⚠️ **< 30 chars:** "Title is short — missing the chance to include keywords or a hook."
- ✅ **30–60 chars:** Optimal length

**Meta description warnings:**
- ⚠️ **> 155 chars:** "Description may be truncated."
- ⚠️ **< 70 chars:** "Description is short — add more value proposition."
- ✅ **70–155 chars:** Optimal length

If any warnings exist, ask: "Some titles/descriptions have length issues. Proceed anyway, or revise first?"

⚡ GUARD — **Page recently refreshed by `/refresh-content`:**
For each page in the approval list, check whether its URL appears in `recently_refreshed_pages` (built in Phase 1.2):
- If yes: add a row-level warning: "⚠️ `/refresh-content` ran on [date] — this title was updated as part of a full content refresh."
- Before editing that page, ask: "This page was recently refreshed — overwriting its title may undo that change. Proceed with this page anyway? (yes/no/skip)"
- If the user says skip: remove the page from the batch and continue with the others.

**Wait for explicit confirmation before writing any file.**

### 5.4 Apply the Changes

On confirmation:

**Apply mode (local files):** for each approved page, EDIT its source file in place — replace the current title/description with the approved values exactly as shown in the diff. Make one precise edit per field; do not reformat surrounding content. Log success or failure per file.

**Apply mode (CMS MCP):** update the matching content item's meta title and description fields via the connected CMS MCP. Log success/failure per item.

**Read-only / no-repo mode:** there is nothing to write — the table from 5.3 is the deliverable.

### 5.5 Publishing / Going Live

How changes go live depends on the stack — do not assume any one platform:
- **Local repo:** the edits are written to the working tree. Tell the user the files changed and that the change goes live via their normal deploy/build flow (commit + push, rebuild, etc.). Only commit/push if the user asks.
- **CMS MCP:** if the CMS distinguishes draft vs. published, ask: "Publish these items now or leave as drafts?" and act accordingly.
- **Read-only:** the user applies the table manually wherever the pages are managed.

### 5.6 Summary

Present a final summary:

```
## Click Recovery Complete

**Updated**: [N] pages
**How applied**: [Local file edits / CMS update / Table for manual apply]

| Page | Source | Status | Title Updated | Description Updated |
|------|--------|--------|---------------|---------------------|
| [title] | src/content/guide.md | ✅ Edited | ✅ | ✅ |
| [title] | components/Head.tsx  | ✅ Edited | ✅ | ✅ |
| [title] | (no desc field)      | ⚠️ Partial | ✅ | ❌ Add field manually |
| [title] | —                    | ❌ Not found | — | — |

## Re-check Schedule

| When | What to check |
|------|---------------|
| **2–3 weeks** | Verify changes are indexed (`site:URL` in your search engine). Check GSC for crawl errors. |
| **4–6 weeks** | Measure CTR impact. Compare clicks before/after. Note any position changes. |
| **Monthly** | Re-run `/click-recovery` to find new opportunities as rankings shift. |

## Next Steps
- For ranking opportunities (position 21+), use `/refresh-content [URL]`
- For pages missing a meta description field, add one to that page's source
```

---

## Error Handling

| Error | Action |
|-------|--------|
| GSC MCP not connected | Stop; instruct user to connect GSC. Offer single-URL meta optimization fallback. |
| GSC property not found | List available properties, ask the user to select |
| No data for date range | Try a shorter range (28 days), warn about limited data |
| Keywords Everywhere API error | Proceed without KE data, note in report |
| Rate limit hit | Pause, retry with exponential backoff |
| No repo / no CMS to apply to | Run analysis, output a paste-ready table for manual application |
| Page URL not found in repo | Ask if it's generated elsewhere; skip or move to the manual table |
| Meta field / tag not found | Try common field/tag name variations; ask the user if still not found |
| No meta description on page | Note in report, suggest adding one, still provide a recommendation |
| File edit fails | Show the error, leave the file untouched, continue with other pages |
| Meta title > 60 chars | Warn, suggest shortening before applying |
| Meta description > 155 chars | Warn, suggest shortening before applying |

---

## Success Metrics

After implementing recommendations, track at each milestone:

**2–3 weeks (indexing check):**
- Are updated pages indexed? (`site:URL`)
- Any crawl errors in GSC?

**4–6 weeks (impact measurement):**
- CTR change per page (compare to pre-update baseline)
- Click change per page
- Position stability (ensure rankings held or improved)

**Monthly (ongoing):**
- Total organic traffic change
- New CTR opportunities from ranking shifts
- Re-run `/click-recovery` to catch new issues

---

## Integration with Other Skills

Click Recovery focuses on **meta tags** (title + description) for CTR optimization.
Refresh Content does **full content refreshes** (body, keywords, FAQs, schema, internal links).

| Scenario | Use |
|----------|-----|
| Low CTR, content is fine | `/click-recovery` — fix the pitch, not the content |
| Outdated content, rankings dropping | `/refresh-content` — full content overhaul |
| Both CTR and content issues | Run `/click-recovery` first for quick wins, then `/refresh-content` for deeper fixes |

**Workflow:**
1. Run `/click-recovery` monthly to catch CTR opportunities
2. Approve and apply the meta tag updates (file edits or table)
3. For pages needing more than meta fixes, run `/refresh-content [URL]`

---

## Activity Log

After every execution, append a row to `.claude/reports/{domain}/activity-log.md` (see CLAUDE.md for the shared format).

**Determining `{domain}`**: extract from the GSC property URL used for analysis (e.g., `https://www.example.com/` → `example.com`).

If the file doesn't exist, create it with the header:

```markdown
# Activity Log — {domain}

| Date | Skill | Summary |
|------|-------|---------|
```

Then append a row:

```
| YYYY-MM-DD | /click-recovery | [one-line summary: e.g., "Analyzed 45 pages. Updated meta titles on 5 pages, descriptions on 3 via local file edits."] |
```

Log even if the skill exits early due to a guard — note why (e.g., "Aborted: GSC MCP unavailable"). Log in analyze-only mode too (e.g., "Analyze only — 8 CTR opportunities found, no changes applied.").
