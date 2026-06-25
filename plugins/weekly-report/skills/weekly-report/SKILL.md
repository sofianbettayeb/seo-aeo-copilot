---
name: weekly-report
version: "2.0"
description: |
  Weekly SEO pulse — what changed this week, how it tracks against the month, and what to do now. Compares Week W vs W-1, scores recommendations by impact/confidence/effort, and outputs an actionable report with monthly progress context. Works on any site or stack — GSC is the main data source, but when it is absent the skill degrades to a live crawl plus a baseline.json comparison (reduced scope, noted in the report).
  Triggers: weekly report, seo weekly, weekly review, weekly performance, traffic this week, weekly analysis.
  Requires: A site URL or a local repo path. GSC MCP server recommended (the primary data source). Optional: a CMS MCP for content-type enrichment, Keywords Everywhere API for volume/intent enrichment. Live crawl (sitemap + WebFetch) and local source files are the fallback backbone.
  Command: /weekly-report (full report), /weekly-report:quick (executive summary + action plan only).
  Workflow: Discover → Fetch → Analyze → Score → Output.
---

# Weekly Report Skill

A weekly SEO pulse for founders and operators. Compares this week vs last week, identifies what changed and why, scores every recommendation, and outputs a prioritized action plan — with monthly progress context when available.

This skill is **read-only** — it analyzes performance and metadata but never modifies the site. It points you to `/click-recovery` and `/refresh-content` for execution.

Works on **any platform or stack**. GSC is the central data source. Metadata is audited from a live multi-page crawl (sitemap + WebFetch) and/or local source files when running inside a repo. A connected CMS MCP is optional enrichment, not a requirement.

See the shared **CLAUDE.md** for conventions referenced throughout: file-based outputs, all MCPs optional, config at `.claude/seo-copilot-config.json`, activity log at `.claude/reports/{domain}/activity-log.md`, priority buckets, abort guard, and output-format guidance.

## Prerequisites

- **A site URL or a local repo path** (required — defines what gets crawled/read)
- **Recommended**: Google Search Console MCP server — the primary data source for clicks, impressions, CTR, position, and indexation
- **Optional**: A CMS MCP (Webflow, WordPress, Sanity, etc.) — enriches the metadata audit with content-type/field coverage when connected
- **Optional**: Keywords Everywhere MCP server (needs API key) — for search volume and intent enrichment on top queries

If GSC is **not** connected, the skill still runs: it crawls the live site (sitemap + WebFetch) and/or reads local source files, then compares against `baseline.json` from the last `/audit:deep`. This is a **reduced-scope** report — no week-over-week query data, no CTR/position trends — and the header flags it clearly.

## Skill Modes

Default is full report. Use `/weekly-report:quick` to output only the executive summary and action plan (skips rendering sections 2–5 but still runs all analysis phases).

## Workflow Overview

```
DISCOVER → FETCH → ANALYZE → SCORE → OUTPUT
```

---

## Critical Guards

These guards can stop execution. All other edge cases (no branded queries, zero declining pages, zero content gaps) are handled inline — skip the section and note why.

⚡ GUARD — **No site URL or repo path:**
- Stop execution: "Weekly Report needs a site URL or a local repo path to analyze. Provide one and try again."

⚡ GUARD — **GSC MCP unavailable:**
- Do **not** stop. Switch to reduced-scope mode:
  - "Google Search Console isn't connected. I'll run a reduced-scope report from a live crawl plus the baseline from your last `/audit:deep`. Week-over-week query, CTR, and position trends won't be available. Continue?"
- If declined: stop. If accepted: proceed, and flag reduced scope in the report header and every affected section.

⚡ GUARD — **Very low weekly traffic (<50 impressions, GSC mode only):**
- Warn: "Very low traffic this week (<50 impressions). Data may be insufficient for meaningful analysis. Proceed anyway?"
- If declined: stop execution.

⚡ GUARD — **User requests abort:**
- See shared CLAUDE.md abort guard. Confirm, then exit cleanly and output any partial results.

---

## Phase 0: DISCOVER

MCP tools, config, last reports, date ranges, site selection.

### 0.1 MCP Discovery

Search for tools BEFORE starting analysis (see shared CLAUDE.md MCP Discovery):

- **GSC MCP** (recommended): Search `+gsc search analytics`. If missing → reduced-scope mode (see guard above).
- **CMS MCP** (optional): Search for a connected CMS server (`+webflow data cms`, `+wordpress`, etc.). If present → use it to enrich content-type/field coverage. If missing → derive content types from the crawl/source files instead.
- **Keywords Everywhere** (optional): Search `+keywords everywhere volume`. If missing → note in report header, proceed without.

### 0.2 Load Config

Load `.claude/seo-copilot-config.json` if it exists. Extract:
- `business.name` → branded query split
- `brandVoice` → recommendation tone
- `seo.metaTitleFormat` → metadata audit reference
- `seo.competitors` → competitor query detection
- `audience.primary` → framing recommendations

If not found: proceed with defaults, note "Run `/getting-started` for personalized recommendations."

### 0.3 Select Site & Set {domain}

If GSC is connected, list available properties; if multiple, ask the user to select, then extract the domain from the property URL (e.g., `https://www.example.com/` → `example.com`).

If GSC is absent, derive `{domain}` from the provided URL or the repo's configured site URL.

The `{domain}` value is used throughout for:
- **Report save path**: `.claude/reports/{domain}/weekly-report-YYYY-WXX.md`
- **Deep audit / baseline lookup**: `.claude/reports/{domain}/latest-deep.md` and `.claude/reports/{domain}/baseline.json`
- **Monthly report lookup**: `.claude/reports/{domain}/monthly-report-*.md`

### 0.4 Review Activity Log

Once `{domain}` is set, check `.claude/reports/{domain}/activity-log.md`:
- If it exists: read the last 10 entries and surface a brief summary:
  ```
  Recent activity on {domain}:
  - 2026-02-10 /click-recovery — Updated meta titles on 5 pages
  - 2026-02-05 /weekly-report — Week W06 report. Health: STABLE.
  - 2026-02-01 /monthly-report — January 2026 report. Health: UP.
  ```
- **Redundancy check**: if `/weekly-report` was already run for the same week → warn: "You already ran `/weekly-report` for Week [XX] on [date]. Run again anyway?"
- If the log doesn't exist: proceed silently (first run for this domain).

Use recent activity as context for recommendations (e.g., "Since `/click-recovery` was run 2 days ago, CTR changes may not be visible yet").

⚡ GUARD — **Report already exists:**
Check if `.claude/reports/{domain}/weekly-report-YYYY-WXX.md` already exists:
- If yes: "A report for Week [XX] already exists. Overwrite?"
- If declined: stop.

### 0.5 Load Deep Audit & Baseline

Read the latest deep audit output (written by `/audit:deep`):
- `.claude/reports/{domain}/latest-deep.md` — extract the dimension summary (per-dimension status), the quantified opportunities and forecast, and any structural issues flagged.
- `.claude/reports/{domain}/baseline.json` — the metadata/crawl snapshot. In reduced-scope (no-GSC) mode, this is the comparison baseline for week-over-week deltas. In GSC mode, it still seeds the metadata audit and indexation cross-reference.
- If neither found: proceed without, note "No deep audit found. Run `/audit:deep` for a baseline and richer context."

### 0.6 Load Last Monthly Report

Search for the most recent `.claude/reports/{domain}/monthly-report-*.md`:
- If found: extract executive summary metrics (clicks, impressions, CTR, position), health trend, and action plan items.
- If not found: proceed without, note "No monthly report found. Run `/monthly-report` for month-level context."

### 0.7 Load Latest Keyword Opportunity Report

Check `.claude/reports/{domain}/latest-keywords-opportunity.md`:
- If found and < 30 days old:
  - Extract: report date, quick wins list, top action plan items (must-pursue + high-value).
  - Store as `keyword_opportunity_context` — reference it in Sections 3 and 5 instead of re-deriving the same findings.
  - Note in the report header: "Keyword opportunity report from [date] loaded — [N] striking distance pages, [N] new opportunities. Full detail in `latest-keywords-opportunity.md`."
  - In Section 5 (On-Page SEO Opportunities): surface the top 3 quick wins from the keywords-opportunity report, clearly labelled as sourced from that report. Do NOT re-run a full striking distance analysis — reference the existing findings instead.
- If not found or > 30 days old: run the standard striking distance analysis in Phase 2.6 as usual, and suggest: "Run `/keywords-opportunity` for a full volume-enriched analysis."

### 0.8 Calculate Date Boundaries

Compute 7-day windows (Mon–Sun). If today is mid-week, use the most recent complete window as W.

| Period | Label | Use |
|--------|-------|-----|
| Week W | Most recent complete Mon–Sun | Primary analysis |
| Week W-1 | Previous 7 days | Comparison baseline |
| Week W-2 | Two weeks ago | 4-week trend |
| Week W-3 | Three weeks ago | 4-week trend |

In reduced-scope mode, week boundaries still apply for labeling, but trend tables collapse to "current crawl vs baseline" since historical query data is unavailable.

---

## Phase 1: FETCH

Performance data, page inventory, indexation sample.

### 1.1 GSC Performance Data (if connected)

Fetch for W and W-1 (optionally W-2, W-3 for trend):

- **Page-level**: all pages with any impressions — URL, impressions, clicks, CTR, position
- **Query-level**: all queries with impressions > 5 — query, page, impressions, clicks, CTR, position

If GSC is absent, skip this step and rely on the crawl + baseline.json comparison.

### 1.2 Page Inventory (crawl + source files)

Build the page list from whatever is available, in this order of preference:

1. **Live crawl**: fetch the sitemap (`/sitemap.xml`), then WebFetch each URL (sample the first 100 if large). Extract: URL, `<title>`, meta description, H1, canonical, indexability signals.
2. **Local source files** (when running in a repo): read page templates, content types, and source files (markdown/MDX, HTML, component meta tags, JSON-LD) to extract the same fields. Use the repo's route/content list as the page inventory.
3. **CMS MCP (optional enrichment)**: if a CMS server is connected, pull content-type/field metadata to enrich the audit (e.g., which content types have a dedicated SEO-title field). This is additive — never required.

Reconcile the three sources into one normalized page inventory.

### 1.3 Indexation Check (Top 5, if GSC connected)

For the top 5 pages by impressions, run GSC URL inspection:
- Index status, last crawl date, mobile usability.
- If rate-limited: reduce to top 3, note in report.

If GSC is absent, infer indexability from crawl signals (robots meta, canonical, sitemap presence) and note the lower confidence.

### 1.4 Keywords Everywhere Enrichment (Optional)

If available, enrich top 30 queries: search volume, CPC, competition, trend.

---

## Phase 2: ANALYZE

Metrics, content performance, content-type/template audit, metadata audit, striking distance.

### 2.1 Executive Summary Metrics

Calculate W vs W-1 (or current crawl vs baseline in reduced-scope mode):
- Total clicks (absolute + %)
- Total impressions (absolute + %)
- Average CTR
- Average position
- Indexed page count (from inspection, GSC page count, or crawl signals)

**Health trend:**
- **UP**: clicks ≥ +5% AND (impressions up OR position improved)
- **DOWN**: clicks ≥ -5% AND (impressions down OR position worsened)
- **STABLE**: everything else

In reduced-scope mode, base the trend on crawl-derived signals (page count, metadata completeness vs baseline) and label it "directional (no GSC data)".

**Identify:** biggest win (largest click increase), biggest risk (largest click decrease), top 3 priorities (from scoring engine).

**Monthly context** (if monthly report available): month-to-date aggregates vs last monthly totals — on track / ahead / behind.

### 2.2 Performance Overview

- 4-week trend table: clicks, impressions, CTR, position for W-3 through W (GSC mode only)
- Branded vs non-branded split (using `business.name` from config)
- Note that conversion data requires GA4 or similar

### 2.3 Content Performance

- **Top 10 pages** by clicks in W (with W-1 comparison)
- **Top 5 growing** — largest click increase W vs W-1 (min 5 clicks in W)
- **Top 5 declining** — largest click decrease W vs W-1 (min 5 clicks in W-1)
- **Content gaps** — GSC queries with high impressions not matching any page title or description

In reduced-scope mode, replace click-based rankings with crawl-derived signals (e.g., pages new/changed vs baseline, pages missing or with thin metadata).

### 2.4 Content-Type / Template SEO Audit

For each content type / page template (from the crawl, source files, or CMS MCP), run a quick coverage check against the recommended fields:

**Core fields (highest priority):**
- Dedicated SEO title present? Flag templates/types that fall back to the item `name`/`H1` as the SEO title. Use alias matching — `seo-title`, `meta-title`, `titile` all count.
- Meta description present?
- H1 override present?

**Key SEO fields:**
- Primary keyword field present?
- Updated/modified date present?
- OG image present?
- Author reference present?

For each content type, output a brief line:
```
Blog Posts: Core ✅ | Keyword ❌ | OG image ❌ | Author ✅ — Schema score: ~60%
```

Flag content types below 70% coverage. When a CMS MCP is connected, recommend its own collection/field review to add missing fields. When running from source files, point to the template/component or content schema that needs the field. When only a URL is available, flag it as a paste-ready recommendation.

### 2.5 Metadata Audit

For all pages in the inventory (crawl + source files + optional CMS):
- Missing meta title / missing meta description
- Duplicate meta titles across pages
- Title too long (>60 chars) / too short (<30 chars)
- Description too long (>155 chars) / too short (<70 chars)

**Indexation cross-reference:**
- Normalize URLs (strip scheme + domain from GSC, build the public URL from source routes / content slugs, lowercase, strip trailing slashes)
- Pages in the inventory but not in GSC = potentially not indexed
- GSC pages not in the inventory = indexed orphans (old/deleted)

In reduced-scope mode, the cross-reference compares the current crawl against `baseline.json` (new pages, dropped pages, metadata regressions) instead of against GSC.

### 2.6 On-Page SEO Opportunities

**High impressions, low CTR (GSC mode):**
- Pages in the top 10% by impressions with CTR below site average for their position bracket.
- These are candidates for `/click-recovery`.

**Striking distance keywords (position 5–15, GSC mode):**
- Sorted by impressions (highest potential first).

**Keyword mismatches:**
- Top GSC query for each page vs meta title — flag if the #1 query doesn't appear in the title.
- In reduced-scope mode, flag pages whose configured primary keyword (from source/config) is missing from the title instead.

### Adaptive Thresholds

For sites with < 5,000 weekly impressions, replace absolute thresholds with relative ones:

| Threshold | Standard (≥5K impressions) | Small site (<5K impressions) |
|-----------|---------------------------|------------------------------|
| Content gaps | >100 impressions | Top 5% of queries by impressions |
| High impressions / low CTR | Top 10% by impressions + CTR below average | Same (already relative) |
| Min clicks for growing/declining | 5 clicks | Top 10 pages by click change (any direction) |
| Very low traffic warning | <50 total impressions | Same (absolute floor) |

---

## Phase 3: SCORE

Use the priority buckets from the shared CLAUDE.md. No numeric formula — assign buckets based on observable criteria.

**Weekly data note:** Single-week observations only qualify for High Value, not Must Do, unless the issue is structural (broken page, confirmed cannibalization). Must Do requires either a multi-week confirmed trend or a structural problem visible in the data.

### 3.1 Priority Buckets

See shared CLAUDE.md for full criteria. Weekly-specific additions:

**Must do** (this week) — structural or confirmed:
- Page in sitemap returning 404
- GSC confirms same keyword on 2+ pages simultaneously
- Indexation error on a page with existing traffic
- Trend confirmed across 3+ consecutive weeks AND impressions > 50

**High value** (schedule within 3 days):
- Keyword cluster in top 20% of site impressions with no matching page
- Page CTR < half expected for its position bracket (confirmed in GSC this week)
- Primary keyword missing from title on a page with 50+ weekly impressions
- Cluster has 2+ support posts but no pillar page

**Nice to have** — everything else qualifying. Items with no data signal are excluded.

### 3.2 Recommendation Format

Each recommendation:
1. **Action title** — specific and directive
2. **Why** — 1–2 lines with the observable metric that triggered it
3. **Expected impact** — what changes if this is fixed (qualitative + quantified where possible)
4. **Effort** — Low / Medium / High
5. **Steps** — concrete, stack-appropriate instructions (edit the source file/template, update via CMS MCP if connected, or paste-ready when URL-only)
6. **Skill shortcut** — which skill to run (e.g., `/click-recovery`)
7. **Cross-reference** — if flagged in the last monthly report or deep audit, note "Still open from [report]"

---

## Phase 4: OUTPUT

Show executive summary + action plan in terminal. Save full report to `.claude/reports/{domain}/`.

### 4.1 Report Structure

The saved report is a Markdown file with these sections:

**Header:**
- Site, period (W dates vs W-1 dates), generated date
- Mode: full GSC data **or** reduced-scope (crawl + baseline) — state clearly which
- Data sources: GSC (connected / not), crawl (sitemap / source files), CMS MCP (connected / not), KE (connected / not), config (loaded / not), deep audit (loaded / not), monthly report (loaded / not)

**Section 1 — Executive Summary:**
- Health trend (UP / STABLE / DOWN; or "directional" in reduced-scope mode)
- W vs W-1 metrics table (clicks, impressions, CTR, position, indexed pages)
- Biggest win, biggest risk, top 3 priorities
- Monthly context box: month-to-date vs last monthly totals, open action items count

**Section 2 — Performance Overview:**
- 4-week trend table (W-3 through W; GSC mode only)
- Branded vs non-branded split
- Conversions note (cross-reference with GA4)

**Section 3 — Content Performance:**
- Top 10 pages, top 5 growing, top 5 declining, content gaps

**Section 4 — Technical SEO Health:**
- Content-type / template SEO audit: field coverage per content type (Core/Keyword/OG/Author) with % score
- Indexation cross-reference (not indexed, indexed orphans, or vs baseline in reduced-scope mode)
- Metadata audit (missing, duplicate, length issues)

**Section 5 — On-Page SEO Opportunities:**
- High impressions / low CTR pages
- Striking distance keywords (position 5–15)
- Keyword mismatches (top query / primary keyword not in title)

**Section 6 — Action Plan:**
- Must Do — full format with steps and shortcuts
- High Value — full format
- Nice to Have — condensed (title, why, effort, shortcut)
- Cross-reference: open/completed items from the last monthly report and deep audit

**Footer:**
- Next steps: run `/click-recovery`, `/refresh-content`, re-run `/weekly-report` next week, `/monthly-report` at month-end
- Monitoring: 3 days (quick fixes live), 1 week (re-run weekly), month-end (monthly report)

In **quick mode** (`/weekly-report:quick`): display only Section 1 and Section 6 in terminal. Still save full report to file.

### 4.2 Save to File

Save to `.claude/reports/{domain}/weekly-report-YYYY-WXX.md` (ISO week number).

- Write the file directly — parent directories are created automatically.
- If the write fails: output in terminal only, warn the user.

### 4.3 Suggest Next Skills

Based on findings:
- CTR issues → `/click-recovery`
- Declining content → `/refresh-content [URL]`
- Content gaps → new content targeting top gap queries
- Always → re-run `/weekly-report` next week, `/monthly-report` at month-end

---

## Integration with Other Skills

Weekly Report is **read-only**. Other skills execute.

| Finding | Skill | Why |
|---------|-------|-----|
| Low CTR pages | `/click-recovery` | Fix meta titles and descriptions |
| Declining content | `/refresh-content [URL]` | Full content refresh |
| Missing metadata | `/click-recovery` | Quick metadata fixes |
| Content gaps | `/keywords-opportunity:discover` | Uncover new keyword topics to target |
| Striking distance keywords | `/keywords-opportunity:striking` | Page 1–3 rankings with traffic upside |
| Keyword mismatches | `/click-recovery` | Align titles with search queries |
| Content-type / template SEO gaps | CMS field review (if connected) or source-template edit | Add missing SEO fields |
| Need a scored baseline | `/audit:deep` | Writes `latest-deep.md` + `baseline.json` this skill reads |

**Cadence:**
1. `/audit:deep` once for a scored baseline (writes `latest-deep.md` + `baseline.json`)
2. `/weekly-report` every week for pulse
3. Execute with `/click-recovery` and `/refresh-content`
4. `/monthly-report` at month-end for the full picture

---

## Activity Log

After every execution, append a row to `.claude/reports/{domain}/activity-log.md` (see shared CLAUDE.md).

If the file doesn't exist, create it with the header:

```markdown
# Activity Log — {domain}

| Date | Skill | Summary |
|------|-------|---------|
```

Then append a row:

```
| YYYY-MM-DD | /weekly-report | [one-line summary: e.g., "Week W07 report. Health: UP. 2 must-fix, 3 high-impact items. Saved to weekly-report-2026-W07.md"] |
```

The `{domain}` is already set in Phase 0. Log even if the skill exits early due to a guard — note why (e.g., "Reduced-scope: GSC unavailable, crawl + baseline only").
