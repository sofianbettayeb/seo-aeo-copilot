---
name: content-brief
version: "1.0"
description: |
  Turn a keyword list into an execution plan. Maps each keyword to the right URL on the site, decides whether to create a new page, rework an existing one, retarget it, or consolidate competing pages — then writes a ready-to-execute content brief per page (slug, title, H1, meta, outline, secondary keywords, internal links, and which skill executes it).
  Triggers: content brief, content briefs, keyword mapping, map keywords to pages, keyword to url mapping, which page should target this keyword, create or rework, page brief, editorial brief, brief for writers, content plan from keywords, keyword assignment.
  Requires: A keyword list — from /keyword-strategy, /keywords-opportunity, /topic-map, or pasted by the user. All MCPs optional: GSC (ranking evidence), Keywords Everywhere (volume enrichment), a CMS MCP (page inventory).
  Workflow: Discover → Intake → Inventory → Map → Decide → Brief → Save.
  Modes: /content-brief (all keywords from the latest strategy report), /content-brief {keyword} (single keyword), /content-brief:map (mapping + decisions only, no full briefs).
---

# Content Brief

A keyword strategy answers *what to target*. This skill answers *where and how*: for every keyword, which URL should own it, and does that mean creating a page, reworking one, retargeting one, or merging two that compete. Then it produces the brief a writer (or `/write-blog`) needs to execute without re-doing the research.

This is the bridge between research skills (`/keyword-strategy`, `/keywords-opportunity`, `/topic-map`) and execution skills (`/write-blog`, `/refresh-content`, `/aeo-optimize`, `/click-recovery`).

**This skill is read-only** — it plans changes but never applies them.

---

## Prerequisites

- **Required**: a keyword list from any of: `.claude/reports/{domain}/latest-keyword-strategy.md`, `latest-keywords-opportunity.md`, `latest-topic-map.md`, a user-pasted list, or a file the user points to
- **Optional**: [Google Search Console MCP server](https://github.com/sofianbettayeb/gsc-mcp-server) — ranking evidence turns mapping guesses into confirmed decisions
- **Optional**: [Keywords Everywhere MCP server](https://github.com/hithereiamaliff/mcp-keywords-everywhere) — fills in volume/CPC for user-pasted keywords that arrive bare
- **Optional**: A CMS MCP (e.g. Webflow) — complete page inventory without crawling

---

## Skill Modes

| Mode | Command | Use Case |
|------|---------|----------|
| **Full** | `/content-brief` | Map + decide + full brief for every actionable keyword |
| **Single** | `/content-brief {keyword}` | One keyword → one mapping decision → one brief |
| **Map only** | `/content-brief:map` | The mapping table and decisions, no per-page briefs. Good for a planning meeting. |

---

## Workflow Overview

```
DISCOVER → INTAKE → INVENTORY → MAP → DECIDE → BRIEF → SAVE
```

---

## Conditional Guards (Global)

⚡ GUARD — **Load SEO Copilot config:**
Check `.claude/seo-copilot-config.json`: `business.siteUrl` (domain), `project.repoPath` + `project.contentDir` (repo-mode inventory), `brandVoice.*` (brief tone guidance), `seo.primaryKeywords`. If missing: proceed, note `/getting-started` once.

⚡ GUARD — **No keyword source found:**
If no report files exist and the user gave no keywords:
```
I need a keyword list to work from. Either:
1. Run /keyword-strategy {url} first (recommended — briefs inherit clusters and page types), or
2. Paste keywords here (one per line; volume/intent columns welcome but optional)
```
Wait for the user.

⚡ GUARD — **Keyword list is large (> 40 actionable keywords):**
Full briefs for 40+ keywords bury the reader. Propose: full briefs for Must-pursue items, mapping-table-only for the rest — user can request any individual brief afterwards.

⚡ GUARD — **User requests abort:**
Confirm exit. Output any partial results already computed.

---

## Phase 0: DISCOVER

1. **MCP check** — GSC, Keywords Everywhere, CMS MCP. All optional; note what's absent once.
2. **Config load** — per guard.
3. **Set `{domain}`** — from config, the source report, or ask.
4. **Activity log** — check `.claude/reports/{domain}/activity-log.md`. Note recent `/write-blog` and `/refresh-content` runs: a page created last week may already cover a keyword in the list.

---

## Phase 1: INTAKE

Resolve the keyword list, in priority order:

1. **User-provided** — pasted list, file path, or single keyword argument
2. **`latest-keyword-strategy.md`** — take the roadmap and Must-pursue/High-value tables; inherit cluster, page type, intent, volume, score
3. **`latest-keywords-opportunity.md`** or **`latest-topic-map.md`** — take gap/new-opportunity keywords (skip striking-distance items already mapped to a page there)
4. None found → guard fires

Normalize every keyword to: `{keyword, volume, intent, cluster, suggested_page_type, score, source}`. Missing fields:
- **Volume**: if KE is connected, batch-fetch `get_keyword_data` for bare keywords. Otherwise leave blank — don't invent numbers.
- **Intent**: classify from the keyword's surface form (how-to/what-is → informational; best/vs/alternatives → commercial; buy/pricing/hire → transactional).

Confirm scope in one line before the heavy work: "Working from {source}: {N} keywords across {K} clusters. Mapping now."

---

## Phase 2: INVENTORY

Build the page inventory — every mapping decision depends on knowing what exists. Use the richest source available:

1. **Repo mode** (config `repoPath` set and readable): enumerate content source files (markdown/MDX front-matter, page components, static HTML). Extract per page: URL/slug, title, H1, meta description, and a rough body-topic signal (first heading levels).
2. **CMS MCP** connected: pull static pages + CMS items with their SEO fields.
3. **Otherwise crawl**: sitemap → fetch up to 30 pages (prioritize non-blog pages plus the most recent and most linked blog posts) → extract title, H1, meta.
4. **GSC connected**: pull page-level data (90 days) and query→page pairs. This is the strongest mapping evidence there is — Google's own opinion of which page answers which query.

Record per page: `{url, title, h1, meta, inferred_primary_keyword, gsc_top_queries[], gsc_position}`.

---

## Phase 3: MAP

For each keyword, find the page that should own it. Evidence, strongest first:

1. **GSC confirmation** — the page already gets impressions/clicks for this keyword or a close variant
2. **Explicit targeting** — keyword (or its root) appears in the page's title, H1, or slug
3. **Topical match** — the page covers the keyword's topic but targets a different phrase
4. **Cluster adjacency** — no page matches, but a page exists in the same cluster (candidate for expansion vs. new page)
5. **Nothing** — no page comes close

Assign each keyword a match: **confirmed** (evidence 1–2), **partial** (3–4), **none** (5), or **conflict** — two or more pages matching at strength 1–3 (cannibalization risk; if GSC shows the same keyword getting impressions on 2+ pages, the conflict is confirmed, not inferred).

---

## Phase 4: DECIDE

One decision per keyword. The decision matrix:

| Decision | When | Executes with |
|----------|------|---------------|
| **KEEP** | Confirmed match, GSC position ≤ 10, title aligned | Nothing — exclude from briefs, list as already served |
| **REWORK** | Confirmed/partial match but underperforming: position > 10, or thin coverage, or keyword absent from title | `/refresh-content {url}` (+ `/click-recovery` if it's purely a title/meta gap) |
| **RETARGET** | Page matches topically but its title/H1 target a worse keyword than this one (lower volume, wrong intent) | `/refresh-content {url}` with new primary keyword |
| **CREATE** | No match, or cluster-adjacent page is already committed to its own keyword | `/write-blog` (editorial) or new landing page |
| **CONSOLIDATE** | Conflict — pages splitting one keyword | Merge into the stronger page (GSC evidence decides which), redirect the other |

Judgment calls the matrix can't make for you:
- **REWORK vs CREATE at the boundary**: if the adjacent page would need to serve two distinct intents to cover this keyword, create. One page, one primary intent.
- **RETARGET is a trade**: the page gives up its current keyword. Only retarget when the new keyword is clearly better (volume, intent fit) AND the current one is either kept elsewhere or worth losing. Say which in the rationale.
- Every decision gets a one-line rationale citing its evidence ("GSC: 140 impressions, position 24, keyword absent from title"). No hedging.

---

## Phase 5: BRIEF

**Skip in `:map` mode.** For every keyword decided CREATE, REWORK, RETARGET, or CONSOLIDATE:

```
### {keyword} — {DECISION}

**Page**: {existing URL, or NEW → suggested slug `/{slug}`}
**Page type**: {pillar / editorial / comparison / product-landing / FAQ}
**Cluster**: {cluster name} | **Intent**: {intent} | **Volume**: {N}/mo
**Decision rationale**: {one line, evidence-based}

**Title tag** (50–60 chars): "{proposed title}"
**H1**: "{proposed H1}"
**Meta description** (140–155 chars): "{proposed meta}"

**Outline** (H2-level, 4–7 sections):
- {H2} — {one line on what it must cover}
- ...
{For REWORK: mark each section [exists — keep] / [exists — expand] / [add]}

**Secondary keywords**: {3–5, worked into H2s and body naturally}
**Internal links**:
- From: {2–3 existing pages that should link here, with anchor text}
- To: {cluster pillar + 1–2 related pages}

**Length guidance**: {match the depth the intent demands — a transactional page
isn't 2,000 words, a pillar isn't 600}
**Execute with**: {/write-blog | /refresh-content {url} | /click-recovery | manual merge + redirect}
```

For CONSOLIDATE, the brief covers the surviving page and explicitly lists: which page wins (and why), which sections migrate from the losing page, and the redirect to set.

Proposed titles and meta descriptions follow `brandVoice.*` from config when present. These briefs feed `/write-blog`, which runs its own humanizer pass — don't pre-polish copy here, just make targets and structure unambiguous.

---

## Phase 6: OUTPUT + SAVE

### Report structure

```
# Content Briefs — {domain}

**Date:** YYYY-MM-DD
**Source:** {keyword source} — {N} keywords
**Inventory:** {N} pages ({repo / CMS / crawl / GSC})

## Decision Summary
| Decision | Count | Keywords |
|---|---|---|
| CREATE | {N} | ... |
| REWORK | {N} | ... |
| RETARGET | {N} | ... |
| CONSOLIDATE | {N} | ... |
| KEEP (already served) | {N} | ... |

## Keyword → Page Map
| Keyword | Volume/mo | Intent | Cluster | Mapped page | Match | Decision | Execute with |
|---|---|---|---|---|---|---|---|

## Briefs
{Phase 5 blocks, ordered: CONSOLIDATE first (they unblock others), then REWORK, RETARGET, CREATE —
within each group by score/volume descending}

## Execution Order
{numbered list, 1 line each: the sequence that makes sense — consolidations and quick
reworks before new content, pillar pages before their supporting pages}
```

### Save

- `.claude/reports/{domain}/content-briefs-YYYY-MM-DD.md` — timestamped
- `.claude/reports/{domain}/latest-content-briefs.md` — overwrite each run

Create directories if needed. After saving, print the path, the decision summary counts, and the first 3 items of the execution order.

**Sitemap reminder** (per CLAUDE.md): every CREATE that ships means a sitemap update + GSC resubmission. The executing skill handles it, but list it in the execution order so it isn't lost.

---

## Integration with Other Skills

| Situation | Skill |
|-----------|-------|
| No keyword list yet | `/keyword-strategy {url}` first |
| Execute a CREATE brief | `/write-blog` (hand it the brief block) |
| Execute a REWORK or RETARGET brief | `/refresh-content {url}` |
| Title/meta-only gaps found during mapping | `/click-recovery` |
| Page needs answer-engine structure too | `/aeo-optimize {url}` after the rework |
| Verify cluster structure after executing | `/topic-map` |

Re-run after each execution batch — decisions shift as pages ship (yesterday's CREATE is today's KEEP).

---

## Error Handling

| Error | Action |
|-------|--------|
| No keyword source | Guard fires — ask user. |
| Source report exists but has no actionable keyword tables | Tell the user what was found, ask for keywords directly. |
| No page inventory obtainable (no repo, no CMS, no sitemap, URL unreachable) | Every keyword becomes CREATE by default — warn loudly that rework/conflict detection was impossible. |
| GSC not connected | Mapping falls back to title/H1/slug evidence. Conflicts become "suspected", never "confirmed". |
| KE not connected and keywords arrive bare | Proceed without volume; order briefs by cluster then intent instead of score. |
| Repo path in config but unreadable | Fall back to crawl, note it. |

---

## Activity Log

After every execution, append to `.claude/reports/{domain}/activity-log.md` (create with the standard header if missing):

```
| YYYY-MM-DD | /content-brief | {N} keywords mapped from {source}. Decisions: {N} create, {N} rework, {N} retarget, {N} consolidate, {N} keep. {N} briefs written. |
| YYYY-MM-DD | /content-brief:map | Mapping only: {N} keywords → {N} pages. {N} conflicts flagged. |
```

Log even on early exits, with the reason.
