---
name: monthly-report
version: "2.0"
description: |
  Monthly SEO report on any site or stack — what changed, why it matters, and what to do next. Compares Month M vs M-1 with a 3-month trend, scores recommendations by impact/confidence/effort, and outputs an actionable report for founders and executives.
  Triggers: monthly report, seo report, monthly review, performance report, traffic report, monthly analysis.
  Requires: Google Search Console MCP server recommended (primary data source). Optional: a CMS MCP server (Webflow, WordPress, etc.) for content-type enrichment, Keywords Everywhere API for volume/intent enrichment. When GSC is absent, degrades to a live crawl + baseline.json with reduced scope.
  Command: /monthly-report (full report), /monthly-report:quick (executive summary + action plan only).
  Workflow: Configure → Fetch → Analyze → Score → Report → Save.
---

# Monthly Report Skill

A decision-making tool for founders and executives. Compares this month vs last month, identifies what changed and why, scores every recommendation, and outputs a prioritized action plan. Platform-agnostic — works on any site or stack.

This skill is **read-only** — it analyzes GSC and site data but never modifies any content. It points you to `/click-recovery` and `/refresh-content` for execution.

## Prerequisites

- **Recommended**: Google Search Console MCP server (primary data source). GSC is central. If missing, the skill degrades to a live crawl + the latest `baseline.json` and notes the reduced scope in the report.
- **Optional**: A CMS MCP server (Webflow, WordPress, Sanity, etc.) — optional enrichment for content-type inventory and metadata. Not required: metadata is audited from a live crawl or local source files.
- **Optional**: Keywords Everywhere MCP server (needs API key — for search volume and intent enrichment on top queries).

See the shared `CLAUDE.md` for the conventions this skill follows: file-based operation, all MCP servers optional, config at `.claude/seo-copilot-config.json`, and the activity log at `.claude/reports/{domain}/activity-log.md`.

## Data Sources & Degradation

GSC is the main data source. Metadata and page inventory come from one of three sources, in priority order:

1. **Live multi-page crawl** — discover URLs from the sitemap (`/sitemap.xml`) and fetch each page via WebFetch to read `<title>`, meta description, headings, and internal links.
2. **Local source files** — when running inside a repo, read page templates / content types / source files directly (Markdown, MDX, HTML, `.tsx`/`.jsx` components, front matter).
3. **CMS MCP (optional)** — if a CMS MCP is connected, use it to enrich the inventory with structured content-type data.

When **GSC is unavailable**, fall back to: crawl + the latest `baseline.json` from `./{domain}/reports/` (produced by `audit:deep`). Note clearly in the report header that scope is reduced (no period-over-period click/impression deltas; analysis is structural + baseline-relative only).

## Skill Modes

| Mode | Command | Use Case |
|------|---------|----------|
| **Full** | `/monthly-report` | Complete 7-section report with all analysis and recommendations. |
| **Quick** | `/monthly-report:quick` | Executive summary + action plan only. Fast overview for busy founders. |

## Workflow Overview

```
[PREREQUISITES] → CONFIGURE → FETCH → ANALYZE → SCORE → REPORT → SAVE
```

- **PREREQUISITES**: MCP discovery, config loading
- **CONFIGURE**: Mode select, date boundaries, site confirmation
- **FETCH**: GSC performance data (M and M-1), page inventory (crawl / source files / optional CMS MCP), optional KE enrichment
- **ANALYZE**: Build data for all 7 report sections
- **SCORE**: Impact/Confidence/Effort scoring → Priority buckets
- **REPORT**: Render full Markdown report
- **SAVE**: Display in terminal + save to `./{domain}/reports/`

---

## Conditional Guards (Global)

These guards apply throughout the skill execution.

⚡ GUARD — **Load SEO Copilot config:**
At the start of execution, check if `.claude/seo-copilot-config.json` exists:
- If yes: Load and apply settings:
  - `business.name` → Used for branded query detection
  - `brandVoice.tone` → Match tone in recommendation copy
  - `seo.metaTitleFormat` → Reference in metadata audit
  - `audience.primary` → Frame recommendations for target audience
  - `seo.competitors` → Compare if mentioned in GSC queries
- If no: Proceed with defaults, note: "No SEO Copilot config found. Run `/getting-started` for personalized recommendations."

⚡ GUARD — **User requests abort:**
If user says "stop", "cancel", "abort", or "nevermind" at any phase:
- Confirm: "Stop the workflow? Progress will be lost."
- If confirmed: Exit cleanly with summary of what was completed
- Output any partial results (e.g., data fetched but not analyzed)

⚡ GUARD — **GSC MCP unavailable:**
GSC is the primary data source. If unavailable:
- Do **not** stop. Degrade gracefully:
  - Run a live crawl (sitemap + WebFetch) and/or read local source files for the page inventory and metadata audit
  - Load the latest `baseline.json` from `./{domain}/reports/` for a reference point
  - Skip all period-over-period click/impression/CTR/position deltas (no GSC time series)
- Note prominently in the report header: "⚠️ GSC not connected — running in reduced-scope mode. Analysis is structural and baseline-relative only; no traffic deltas. Connect GSC for full month-over-month performance data."

⚡ GUARD — **CMS MCP unavailable:**
A CMS MCP is optional enrichment only. If unavailable:
- Audit metadata and page inventory from a live crawl (sitemap + WebFetch) and/or local source files instead
- Note: "No CMS MCP connected — page inventory and metadata audited via live crawl / local source files. Connect a CMS MCP for structured content-type enrichment."
- Continue — never stop for this.

⚡ GUARD — **Keywords Everywhere unavailable:**
If KE API is not available:
- Proceed without volume/intent enrichment
- Note in report: "Search volume and intent data unavailable. Analysis based on GSC impressions only."
- Suggest user add KE API for richer insights

---

## Phase 0: Prerequisites & Configuration

### 0.1 Choose Mode

At the start, determine which mode to run:

```
How would you like to run the Monthly Report?

1. **Full report** (recommended) — All 7 sections with detailed analysis and recommendations
2. **Quick summary** — Executive summary + action plan only (faster)
```

- If **Full report** or no mode specified: Run all phases, render all 7 sections
- If **Quick summary**: Run all phases but only render Section 1 (Executive Summary) and Section 7 (Action Plan)

### 0.2 MCP Discovery

**Search for these tools BEFORE starting analysis (all optional — never stop because one is missing):**

**GSC MCP** (recommended — primary data source):
- Search: `+gsc search analytics`
- If missing: switch to reduced-scope mode (see the GSC guard) — crawl + `baseline.json`, no traffic deltas.

**CMS MCP** (optional enrichment):
- Search: `+webflow data cms` (or the relevant CMS MCP for the stack)
- If missing: audit metadata / inventory from the live crawl and local source files instead.

**Keywords Everywhere** (optional but check):
- Search: `+keywords everywhere volume`
- If missing: Continue, but ADD to report header:
  ```
  ⚠️ Keywords Everywhere not connected.
  Analysis based on GSC impressions only.
  Connect KE for search volume and intent data.
  ```

### 0.3 Load Config

Load `.claude/seo-copilot-config.json` if it exists. Extract:
- `business.name` → for branded query split
- `brandVoice` → for recommendation tone
- `seo.metaTitleFormat` → for metadata audit reference
- `seo.competitors` → for competitor query detection
- `audience.primary` → for framing recommendations

### 0.4 Calculate Date Boundaries

Compute date ranges for comparison:

| Period | Label | Use |
|--------|-------|-----|
| Month M | Current/most recent complete month | Primary analysis |
| Month M-1 | Previous month | Comparison baseline |
| Month M-2 | Two months ago | 3-month trend (optional) |
| Month M-3 | Three months ago | 3-month trend (optional) |

If today is mid-month, use the most recent **complete** month as M.

### 0.5 Select Site & Set {domain}

Determine the site domain:
- If GSC is connected: list available GSC properties; if multiple, ask the user to select. Extract the domain from the selected property URL (e.g., `https://www.example.com/` → `example.com`).
- If GSC is absent: ask the user for the site URL, or infer it from the repo / config / sitemap.

The `{domain}` value is used for:
- **Report save path**: `./{domain}/reports/monthly-report-YYYY-MM.md`
- **Baseline lookup**: `./{domain}/reports/baseline.json` and `latest-deep.md`
- **Activity log**: `.claude/reports/{domain}/activity-log.md`

### 0.6 Review Activity Log

Check `.claude/reports/{domain}/activity-log.md`:
- If it exists: read the last 10 entries and surface a brief summary:
  ```
  Recent activity on {domain}:
  - 2026-02-10 /weekly-report — Week W06 report. Health: STABLE.
  - 2026-02-05 /click-recovery — Updated meta titles on 5 pages.
  - 2026-01-31 /monthly-report — January 2026 report. Health: UP.
  ```
- **Redundancy check**: if `/monthly-report` was already run for the same month → warn: "You already ran `/monthly-report` for [Month] on [date]. Run again anyway?"
- If the log doesn't exist: proceed silently (first run for this domain)

Use recent activity as context for the report (e.g., note which skills ran since last monthly report and what they changed).

⚡ GUARD — **Report already exists:**
Check if `./{domain}/reports/monthly-report-YYYY-MM.md` already exists for the target month:
- If yes: Ask user: "A report for [Month] already exists. Overwrite it?"
- If user declines: Stop execution

### 0.7 Load Latest Deep Audit & Baseline

Check `./{domain}/reports/`:
- **`baseline.json`** (from `audit:deep`): the structural + metric baseline. Load it as the reference point — especially important when GSC is unavailable, where it replaces period-over-period deltas.
- **`latest-deep.md`**: the most recent deep audit narrative. Extract its dimension summary (per-dimension status), forecast, and flagged issues so the monthly report builds on, rather than re-derives, that diagnosis.
- If neither exists: proceed, and suggest running `/audit:deep` to establish a baseline.

### 0.8 Load Last 4 Weekly Reports

Search for recent weekly reports at `.claude/reports/{domain}/weekly-report-*.md`:
- Load the last 4 by date (most recent first)
- For each, extract: week label, health trend, must-fix and high-impact action items
- Build a **carry-forward set**: action items that appeared in any of the last 4 weeklies AND were not subsequently logged as completed in the activity log
- These surface in Section 7 (Action Plan) labelled "[carried from W-XX]" — so open work from weekly reports doesn't get lost at month-end
- If no weekly reports found: proceed without, note "No weekly reports found. Run `/weekly-report` weekly for continuous tracking."

### 0.9 Load Latest Keyword Opportunity Report

Check `.claude/reports/{domain}/latest-keywords-opportunity.md`:
- If found and < 60 days old:
  - Extract: report date, striking distance findings, new opportunity clusters, action plan
  - Store as `keyword_opportunity_context` — reference in Sections 2.3 (content gaps) and 2.5 (striking distance) instead of independently re-deriving the same findings from GSC impressions
  - Note in report header: "Keyword opportunity report from [date] loaded — referenced in content gaps and striking distance sections."
- If not found or > 60 days old: run standard gap analysis in Phase 2 as usual, and suggest running `/keywords-opportunity` for enriched volume + intent data

---

## Phase 1: Fetch Data

### 1.1 GSC Performance Data

Fetch performance data for Month M and Month M-1 (and optionally M-2, M-3 for trends):

**Page-level data:**
- All pages with any impressions
- Include: page URL, impressions, clicks, CTR, average position
- Fetch for both M and M-1

**Query-level data:**
- All queries with impressions > 10
- Include: query, page, impressions, clicks, CTR, position
- Fetch for both M and M-1

If GSC is unavailable, skip this step and rely on the baseline + crawl (see the GSC guard).

### 1.2 Page Inventory

Build the full page inventory. Use whichever source is available, in this priority order:

**A. Live crawl (default, no repo needed):**
1. Fetch the sitemap (`/sitemap.xml`, or the path confirmed in config) to enumerate public URLs
2. For each URL (sampled per the large-site guard), fetch the page via WebFetch and extract: URL, `<title>`, meta description, H1/headings, publish/modified date if exposed, and internal links from the body

**B. Local source files (when in a repo):**
1. Locate page templates / content types / source files (Markdown, MDX, HTML, `.tsx`/`.jsx` components, front matter)
2. Extract: route/slug, title, meta description, body content, internal links

**C. CMS MCP (optional enrichment):**
1. If a CMS MCP is connected, list content types / collections and their items
2. Extract: item name, slug, published date, meta title, meta description, body content
3. Merge structured fields into the inventory built from A or B

⚡ GUARD — **Large site (>100 pages/items):**
If the crawl or a content type yields more than 100 URLs/items:
- Sample the first 100 (prioritize by GSC impressions when available)
- Note in report: "Large site ([N] URLs) — sampled first 100. Run again scoped to a section for a full audit."

### 1.3 GSC URL Inspection (Sampled)

For the top 20 pages by impressions, run URL inspection if available:
- Index status (indexed / not indexed / crawled but not indexed)
- Last crawl date
- Mobile usability issues

⚡ GUARD — **URL inspection rate limit:**
If rate-limited:
- Inspect top 10 pages only
- Note: "URL inspection limited to top 10 pages due to API rate limits."

### 1.4 Keywords Everywhere Enrichment (Optional)

If KE is available, enrich the top 30 queries by impressions:
- Monthly search volume
- CPC (commercial intent signal)
- Competition score
- Trend data (rising/falling)

---

## Phase 2: Analyze

Build the data structures needed for each report section.

### 2.1 Executive Summary Metrics

Calculate (from GSC where available; otherwise mark as baseline-relative / N/A):
- Total clicks M vs M-1 (absolute + %)
- Total impressions M vs M-1 (absolute + %)
- Average CTR M vs M-1
- Average position M vs M-1
- Total indexed pages (from URL inspection or GSC page count, or crawl count)

Determine health trend:
- **UP**: clicks increased ≥5% AND (impressions up OR position improved)
- **DOWN**: clicks decreased ≥5% AND (impressions down OR position worsened)
- **STABLE**: everything else
- **N/A (reduced scope)**: when GSC is absent — report structural findings vs baseline instead of a traffic trend

Identify:
- **Biggest win**: page with largest click increase M vs M-1
- **Biggest risk**: page with largest click decrease M vs M-1
- **Top 3 priorities**: preview from scoring engine (Phase 3)

### 2.2 Performance Overview

Build traffic table with 3-month trend:

| Metric | M-2 | M-1 | M | Trend |
|--------|-----|-----|---|-------|
| Clicks | ... | ... | ... | ↑/↓/→ |
| Impressions | ... | ... | ... | ↑/↓/→ |
| CTR | ... | ... | ... | ↑/↓/→ |
| Avg Position | ... | ... | ... | ↑/↓/→ |

**Branded vs Non-Branded Split:**
- Branded queries: contain `business.name` from config (case-insensitive)
- If no config: ask user for brand name, or skip split
- Calculate clicks/impressions for each group

⚡ GUARD — **No branded queries detected:**
If zero queries match brand name:
- Skip branded split
- Note: "No branded queries detected. Check that the brand name in config matches how users search."

### 2.3 Content Performance

**Top 10 traffic pages:**
- Sorted by clicks in Month M
- Include: URL, clicks M, clicks M-1, change, impressions, CTR, position

**Top 5 growing pages:**
- Largest click increase M vs M-1 (minimum 10 clicks in M to qualify)

**Top 5 declining pages:**
- Largest click decrease M vs M-1 (minimum 10 clicks in M-1 to qualify)

⚡ GUARD — **Zero declining pages:**
If no pages declined:
- Note: "No significant declines detected. All pages maintained or grew traffic."
- Still show top pages and growing pages

**Content gaps:**
Identify GSC queries with >500 impressions that don't match any page's meta title or meta description (from the crawl / source files / CMS inventory):
- These represent topics users search for but the site doesn't explicitly target
- Group by theme where possible

⚡ GUARD — **Zero content gaps:**
If all high-impression queries are covered:
- Note: "No major content gaps found. Existing pages cover your high-impression queries well."

### 2.4 Technical SEO Health

**Indexation cross-reference:**

1. Build set of the site's published pages (normalize: strip trailing slashes, lowercase) — from the crawl, source files, or CMS inventory
2. Build set of GSC pages with impressions (normalize: strip scheme + domain, lowercase)
3. Cross-reference:
   - **Not indexed**: published pages NOT appearing in GSC = potentially not indexed
   - **Indexed orphans**: GSC pages NOT in the site inventory = old/deleted pages still in Google's index

If GSC is absent, run the published-page set against the sitemap and `baseline.json` instead, and flag sitemap/crawl discrepancies (404s, missing canonical pages).

**URL normalization logic:**
- GSC URLs: `https://example.com/blog/my-post` → `/blog/my-post`
- CMS / content-type items: collection-or-type slug + item slug → `/blog/my-post`
- Static pages / routes: page path → `/about`, `/contact`
- Strip trailing slashes, lowercase everything

**Metadata audit:**
For all pages (from crawl / source files / CMS inventory), check:
- **Missing meta title**: no title set or using default/placeholder
- **Missing meta description**: empty or not set
- **Duplicate meta titles**: same title on multiple pages
- **Title too long**: > 60 characters
- **Title too short**: < 30 characters
- **Description too long**: > 155 characters
- **Description too short**: < 70 characters

### 2.5 On-Page SEO Opportunities

**High impressions, low CTR (CTR optimization):**
- Pages with impressions > 500 AND CTR < 2%
- Or pages where CTR is < half the site average for that position bracket
- These are candidates for `/click-recovery`

**Striking distance keywords (position 5-15):**
- Queries where position is between 5 and 15
- Sorted by impressions (highest potential)
- A small ranking improvement could move these to top positions

**Keyword mismatches:**
- Compare top GSC query for each page against the page's meta title
- If the #1 query (by impressions) doesn't appear in the meta title → mismatch
- These are quick wins for title optimization

(All of Section 2.5 requires GSC. If GSC is absent, note that on-page opportunities cannot be ranked by traffic and fall back to metadata findings from Section 2.4.)

### 2.6 Internal Linking & Structure

**Link graph from page body content:**
If body content is accessible (from the crawl, source files, or CMS rich-text fields):
1. Parse body content for internal links (hrefs containing the site domain or relative paths)
2. Build a simple link graph: which pages link to which

⚡ GUARD — **No body content accessible:**
If body content is not available or not in a parseable format:
- Skip link graph analysis
- Note: "Internal link analysis unavailable — page body content not accessible via crawl, source files, or CMS API."
- Still check for orphan pages using GSC + site inventory cross-reference

**Link distribution:**
- Average internal links per page
- Pages with most internal links (over-linked)
- Pages with fewest internal links (under-linked)

**Orphan pages:**
- Pages with zero internal links pointing to them (from the link graph)
- Cross-reference with the sitemap / site inventory

**Under-linked pages:**
- High-traffic pages (top 20 by clicks) with fewer than 3 internal links pointing to them
- These deserve more internal link support

**Pillar page health:**
- If identifiable from site structure (e.g., `/blog/topic` → `/blog/topic/subtopic`):
  - Check if pillar pages link to all child pages
  - Check if child pages link back to pillar

---

## Phase 3: Score Recommendations

### 3.1 Priority Buckets

Use the priority buckets from the shared `CLAUDE.md`. No numeric formula — assign buckets based on observable criteria.

**Monthly-specific additions:**

**Must do** (address this month) — structural or multi-month confirmed:
- Page in sitemap returning 404
- GSC confirms same keyword on 2+ pages simultaneously over multiple months
- Indexation error on a page with existing traffic
- Trend confirmed across 3+ consecutive months AND impressions > 100

**High value** (schedule within 2 weeks):
- Keyword cluster in top 20% of site impressions with no matching page
- Page CTR < half expected for its position bracket (confirmed in GSC this month)
- Primary keyword missing from title on a page with 100+ monthly impressions
- Cluster has 2+ support posts but no pillar page
- A page losing clicks 3+ months in a row with no content changes

**Nice to have** — everything else qualifying. Items with no data signal are excluded.

### 3.2 Recommendation Format

Each recommendation includes:

1. **Action-oriented title** — what to do (e.g., "Rewrite meta title for /blog/seo-guide to match top query")
2. **Why it matters** — 1-2 lines with supporting metrics (e.g., "This page lost 340 clicks (-45%) month-over-month. The #1 query 'seo guide 2026' doesn't appear in the title.")
3. **Expected impact** — qualitative + quantified (e.g., "High — recovering even 50% of lost clicks = ~170 clicks/month")
4. **Effort level** — Low / Medium / High with explanation
5. **Exact steps** — platform-agnostic instructions (e.g., "Open the page template / content type for seo-guide and update the meta title field to: ..." — adapt to the site's stack: CMS field, front matter, or source component)
6. **Skill shortcut** — which skill to run (e.g., "Run `/click-recovery`" or "Run `/refresh-content /blog/seo-guide`")

---

## Phase 4: Compile Report

Render the full Markdown report using the template below. In Quick mode, only render Section 1 and Section 7.

### Report Template

```markdown
# Monthly SEO Report — [Month Year]

**Site**: [domain / property URL]
**Period**: [Month M] vs [Month M-1]
**Generated**: [date]

**Data sources**:
- ✅ GSC: Connected [or ⚠️ Not connected — reduced scope, no traffic deltas]
- ✅ Page inventory: [Live crawl / Local source files / CMS MCP]
- ✅ Keywords Everywhere: Connected [or ⚠️ Not connected — no volume/intent data]
- ✅ Deep audit baseline: Loaded ([date]) [or ℹ️ None — run /audit:deep]
- ✅ SEO Copilot Config: Loaded [or ℹ️ Not found — using defaults]

---

## 1. Executive Summary

**Health trend**: 🟢 UP / 🟡 STABLE / 🔴 DOWN [or ℹ️ N/A — reduced scope]

| Metric | [M-1] | [M] | Change |
|--------|-------|-----|--------|
| Clicks | [X] | [X] | [+/-X] ([+/-X%]) |
| Impressions | [X] | [X] | [+/-X] ([+/-X%]) |
| CTR | [X%] | [X%] | [+/-X pp] |
| Avg Position | [X] | [X] | [+/-X] |
| Indexed Pages | [X] | [X] | [+/-X] |

**Biggest win**: [Page title] — [metric improvement]
**Biggest risk**: [Page title] — [metric decline]

**Top 3 priorities**:
1. [Priority 1 — one-line summary]
2. [Priority 2 — one-line summary]
3. [Priority 3 — one-line summary]

---

## 2. Performance Overview

### Traffic Trend (3 months)

| Metric | [M-2] | [M-1] | [M] | Trend |
|--------|-------|-------|-----|-------|
| Clicks | [X] | [X] | [X] | ↑/↓/→ |
| Impressions | [X] | [X] | [X] | ↑/↓/→ |
| CTR | [X%] | [X%] | [X%] | ↑/↓/→ |
| Avg Position | [X] | [X] | [X] | ↑/↓/→ |

### Branded vs Non-Branded

| Segment | Clicks | Impressions | CTR | % of Total |
|---------|--------|-------------|-----|------------|
| Branded | [X] | [X] | [X%] | [X%] |
| Non-Branded | [X] | [X] | [X%] | [X%] |

> **Note**: Branded queries use "[business name]" from config. Non-branded = everything else.

### Conversions

> Conversion data is not available via GSC API. If you track conversions in GA4 or another tool, cross-reference the top pages below with your conversion data.

---

## 3. Content Performance

### Top 10 Pages by Traffic

| # | Page | Clicks (M) | Clicks (M-1) | Change | Impressions | CTR | Position |
|---|------|-----------|-------------|--------|-------------|-----|----------|
| 1 | [title] | [X] | [X] | [+/-X%] | [X] | [X%] | [X] |
| ... | ... | ... | ... | ... | ... | ... | ... |

### Top 5 Growing Pages

| Page | Clicks (M-1) | Clicks (M) | Change | Why |
|------|-------------|-----------|--------|-----|
| [title] | [X] | [X] | [+X%] | [brief reason] |

### Top 5 Declining Pages

| Page | Clicks (M-1) | Clicks (M) | Change | Likely Cause | Action |
|------|-------------|-----------|--------|--------------|--------|
| [title] | [X] | [X] | [-X%] | [reason] | [skill to run] |

### Content Gaps

Queries with >500 impressions not matching any page title or description:

| Query | Impressions | Clicks | Position | Opportunity |
|-------|-------------|--------|----------|-------------|
| [query] | [X] | [X] | [X] | [Create new page / Update existing page] |

---

## 4. Technical SEO Health

### Indexation Status

| Status | Count | Pages |
|--------|-------|-------|
| ✅ Indexed & in site inventory | [X] | — |
| ⚠️ Not indexed (live, not in GSC) | [X] | [list] |
| ⚠️ Indexed orphans (in GSC, not on site) | [X] | [list] |

### Metadata Audit

| Issue | Count | Pages |
|-------|-------|-------|
| ❌ Missing meta title | [X] | [list] |
| ❌ Missing meta description | [X] | [list] |
| ⚠️ Duplicate meta titles | [X] | [list] |
| ⚠️ Title too long (>60 chars) | [X] | [list] |
| ⚠️ Title too short (<30 chars) | [X] | [list] |
| ⚠️ Description too long (>155 chars) | [X] | [list] |
| ⚠️ Description too short (<70 chars) | [X] | [list] |

---

## 5. On-Page SEO Opportunities

### High Impressions, Low CTR

Pages where Google shows you but users don't click:

| Page | Impressions | CTR | Position | Top Query | Action |
|------|-------------|-----|----------|-----------|--------|
| [title] | [X] | [X%] | [X] | [query] | Run `/click-recovery` |

### Striking Distance Keywords (Position 5-15)

A small ranking push could move these to top positions:

| Query | Page | Position | Impressions | Potential |
|-------|------|----------|-------------|-----------|
| [query] | [title] | [X] | [X] | [estimated click gain] |

### Keyword Mismatches

Top query for the page doesn't appear in the meta title:

| Page | Meta Title | Top Query (by impressions) | Impressions |
|------|-----------|---------------------------|-------------|
| [title] | [current title] | [query] | [X] |

---

## 6. Internal Linking & Structure

### Link Distribution

| Metric | Value |
|--------|-------|
| Average internal links per page | [X] |
| Most linked page | [title] ([X] links) |
| Least linked page (with traffic) | [title] ([X] links) |

### Orphan Pages

Pages with zero internal links pointing to them:

| Page | Clicks | Impressions | Action |
|------|--------|-------------|--------|
| [title] | [X] | [X] | Add internal links from related pages |

### Under-Linked High-Traffic Pages

Top pages that deserve more internal link support:

| Page | Clicks | Internal Links | Recommended |
|------|--------|---------------|-------------|
| [title] | [X] | [X] | Add [X] more links from related content |

### Pillar Page Health

| Pillar | Child Pages | Links to Children | Links Back | Status |
|--------|-------------|-------------------|------------|--------|
| [title] | [X] | [X/X] | [X/X] | ✅ / ⚠️ Incomplete |

---

## 7. Action Plan

### Must Fix

[For each recommendation:]

#### [Action-oriented title]

**Why it matters**: [1-2 lines with metrics]
**Expected impact**: [qualitative + quantified]
**Effort**: [Low/Medium/High — explanation]

**Steps**:
1. [Platform-agnostic step — CMS field / front matter / source component]
2. ...

**Shortcut**: Run `[/skill-name]`

---

### High Impact

[Same format as above]

---

### Nice to Have

[Condensed format — title, why, effort, shortcut only]

---

## Next Steps

1. Start with **Must Fix** items — they have the highest ROI
2. Use `/click-recovery` for CTR optimization tasks
3. Use `/refresh-content [URL]` for content refresh tasks
4. Re-run `/monthly-report` next month to track progress

## Monitoring Checklist

| When | What to check |
|------|---------------|
| **1 week** | Verify indexation changes (submit URLs via GSC if needed) |
| **2 weeks** | Check metadata updates are reflected in SERPs |
| **4 weeks** | Re-run `/monthly-report` to measure impact and find new opportunities |
```

---

## Phase 5: Output

### 5.1 Display in Terminal

Output the full rendered report in the terminal.

### 5.2 Save to File

Save the report to `./{domain}/reports/monthly-report-YYYY-MM.md` where YYYY-MM is the report month. This keeps the monthly report alongside `latest-deep.md` and `baseline.json` from `/audit:deep`.

⚡ GUARD — **Can't create reports directory:**
If `./{domain}/reports/` doesn't exist:
- Create it (Write creates parent directories automatically)
- If creation fails: Output report in terminal only, warn: "Could not create reports directory. Report displayed above but not saved to file."

⚡ GUARD — **File already exists:**
This was already handled in Phase 0 (overwrite confirmation). If we reach this point, overwrite is approved — proceed with saving.

### 5.3 Suggest Next Skills

After saving, suggest next actions based on findings:

```
## What to Do Next

Based on this report, here are the recommended next steps:

[If CTR issues found:]
→ Run `/click-recovery` to fix meta titles and descriptions for [N] underperforming pages

[If declining content found:]
→ Run `/refresh-content [URL]` on your top declining pages:
  - [page 1 URL]
  - [page 2 URL]

[If content gaps found:]
→ Consider creating new content targeting: [top gap queries]

[Always:]
→ Re-run `/monthly-report` next month to track progress
```

---

## Phase 6: Follow-Up

Provide a monitoring checklist at the end of the report:

| When | What to Check | How |
|------|---------------|-----|
| **1 week** | Indexation of any new/updated pages | Check GSC Coverage report or `site:URL` in Google |
| **2 weeks** | Metadata changes reflected in SERPs | Search brand + key terms, verify updated titles/descriptions |
| **4 weeks** | Re-run monthly report | Run `/monthly-report` to measure changes and find new opportunities |

---

## Error Handling

| Error | Action |
|-------|--------|
| GSC MCP not connected | Switch to reduced-scope mode: crawl + baseline.json, no traffic deltas; note in header |
| CMS MCP not connected | Audit metadata / inventory via live crawl or local source files, continue |
| GSC property not found | List available properties, ask user to select |
| No data for date range | Try shorter range (28 days), warn about limited data |
| Multiple GSC properties | List all, ask user to select |
| Keywords Everywhere API error | Proceed without KE data, note in report |
| Rate limit hit (URL inspection) | Reduce sample size, note limitation |
| Large site (>100 pages/items) | Sample first 100 (prioritize by impressions), note in report |
| No body content accessible | Skip internal link analysis, note in report |
| Sitemap not found | Ask user for the sitemap URL or fall back to local source files |
| No baseline.json found | Proceed without baseline, suggest running /audit:deep |
| Low traffic site (<1000 impressions) | Warn about statistical significance, proceed with caveats |
| Report directory creation fails | Display in terminal only, warn user |
| Report file already exists | Ask overwrite confirmation (Phase 0 guard) |
| No branded queries found | Skip branded split, note in report |
| Zero declining pages | Note positive trend, skip decline section |
| Zero content gaps | Note good coverage, skip gaps section |
| URL normalization mismatch | Log mismatched URLs, attempt fuzzy matching on slug |

---

## Key Data Combination Logic

### URL Normalization

GSC returns full URLs. Sites use slugs/routes. Normalize both to match:

```
GSC:           https://example.com/blog/my-post  →  /blog/my-post
CMS / type:    type "blog" + slug "my-post"      →  /blog/my-post
Static / route: page path "/about"               →  /about
```

- Strip scheme and domain from GSC URLs
- Combine type/collection slug + item slug for CMS items
- Use page path / route directly for static pages
- Strip trailing slashes
- Lowercase everything

### Indexation Cross-Reference

```
Site published pages (crawl / source files / CMS)  ←→  GSC pages with impressions

Match     = indexed and live (good)
Site only = potentially not indexed (investigate)
GSC only  = indexed orphan — old/deleted page still in Google (clean up)
```

### Content Gaps

```
GSC queries with >500 impressions
  MINUS
Queries that appear in any page's meta title OR meta description
  EQUALS
Content gaps — topics users search for but the site doesn't explicitly target
```

### Branded Query Split

```
Branded = queries containing business.name (from config), case-insensitive
Non-branded = everything else

If no config or no business.name: ask user or skip split
```

---

## Integration with Other Skills

Monthly Report is **read-only** — it identifies issues and recommends actions. Other skills execute those actions.

| Finding | Recommended Skill | Why |
|---------|-------------------|-----|
| No baseline / first run | `/audit:deep` | Establish the structural + metric baseline (baseline.json) the monthly report builds on |
| Low CTR pages | `/click-recovery` | Fix meta titles and descriptions |
| Declining content | `/refresh-content [URL]` | Full content refresh with updated keywords |
| Missing metadata | `/click-recovery` | Quick metadata fixes |
| Content gaps / new keyword opportunities | `/keywords-opportunity:discover` | Uncover new topics to target with volume + intent data |
| Striking distance keywords (positions 4-30) | `/keywords-opportunity:striking` | Page 1-3 rankings with traffic upside — prioritized by volume |
| Full keyword strategy refresh | `/keywords-opportunity` | Striking distance + new discovery in one report |
| Keyword mismatches | `/click-recovery` | Align titles with actual search queries |

**Workflow:**
1. Run `/audit:deep` once to establish a baseline
2. Run `/monthly-report` monthly to assess overall SEO health
3. Use the Action Plan to prioritize work
4. Execute with `/click-recovery` and `/refresh-content`
5. Re-run `/monthly-report` next month to measure impact

---

## Activity Log

After every execution, append a row to `.claude/reports/{domain}/activity-log.md` (per the shared `CLAUDE.md` convention).

**Determining `{domain}`**: Extract from the selected GSC property URL (e.g., `https://www.example.com/` → `example.com`), or from the site URL / repo / config when GSC is absent. Same logic as `/weekly-report`.

If the file doesn't exist, create it with the header:

```markdown
# Activity Log — {domain}

| Date | Skill | Summary |
|------|-------|---------|
```

Then append a row:

```
| YYYY-MM-DD | /monthly-report | [one-line summary: e.g., "February 2026 report. Health: UP. 4 must-fix, 6 high-impact items. Saved to monthly-report-2026-02.md"] |
```

Log even if the skill exits early due to a guard — note why (e.g., "Reduced scope: GSC unavailable, crawl + baseline only").
