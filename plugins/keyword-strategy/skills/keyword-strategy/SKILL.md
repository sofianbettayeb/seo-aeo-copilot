---
name: keyword-strategy
version: "1.0"
description: |
  Build a complete keyword strategy from just a URL. Analyzes the website, generates relevant seed keywords, identifies main competitors, pulls their ranking keywords via Keywords Everywhere, finds the keyword gap, explores new keywords, then defines the strategy — topic clusters, page type per keyword, and a phased roadmap based on search volume and difficulty.
  Triggers: keyword strategy, seo strategy, competitor keywords, competitor keyword analysis, keyword gap, keyword gap analysis, what keywords should this site target, keyword plan, market keyword research, keyword research from url, define keyword strategy, competitor gap, seo plan for a site.
  Requires: Keywords Everywhere MCP server (the core data source — degraded mode without it). Optional: GSC MCP (own-ranking enrichment), a CMS MCP (page inventory).
  Workflow: Discover → Analyze Site → Seeds → Competitors → Gap → Explore → Score → Strategy → Report → Save.
  Modes: /keyword-strategy {url} (full), /keyword-strategy:gap {url} (competitor gap analysis only), /keyword-strategy:explore {url} (no competitor analysis — site-derived keywords + expansion only).
---

# Keyword Strategy

`/keywords-opportunity` answers "what can I win from what I already rank for?" — it needs GSC and existing rankings. This skill answers the earlier question: **"what should this site target in the first place?"** It works from nothing but a URL, which makes it the right tool for a new engagement, a prospect analysis, a site with no GSC access, or a strategy reset.

The pipeline: understand the site → figure out who actually competes with it in search → measure the keyword gap with real data → expand into adjacent demand → turn the raw list into a strategy a human can execute (clusters, page types, priorities).

**This skill is read-only.** It produces a strategy document and hands off to `/content-brief` for keyword-to-page mapping, then `/write-blog`, `/refresh-content`, and `/aeo-optimize` for execution.

---

## Prerequisites

- **Core data source**: [Keywords Everywhere MCP server](https://github.com/hithereiamaliff/mcp-keywords-everywhere) — powers competitor keywords, volume, CPC, competition, and expansion. Without it the skill degrades to site-derived keyword suggestions with no volume or gap data.
- **Optional**: [Google Search Console MCP server](https://github.com/sofianbettayeb/gsc-mcp-server) — marks keywords the site already ranks for, so the strategy focuses on genuine gaps
- **Optional**: A CMS MCP (e.g. Webflow) — faster, more complete page inventory than crawling

---

## Skill Modes

| Mode | Command | Use Case |
|------|---------|----------|
| **Full** | `/keyword-strategy {url}` | Complete pipeline: site analysis, competitors, gap, expansion, strategy |
| **Gap** | `/keyword-strategy:gap {url}` | Competitor gap analysis only — skips expansion and full strategy doc |
| **Explore** | `/keyword-strategy:explore {url}` | No competitor analysis — seeds from the site + KE expansion. Cheaper on KE credits. |

---

## Workflow Overview

```
DISCOVER → ANALYZE SITE → SEEDS → COMPETITORS → GAP → EXPLORE → SCORE → STRATEGY → REPORT → SAVE
```

---

## Conditional Guards (Global)

⚡ GUARD — **No URL provided:**
Ask for the site URL before doing anything else. One question, then wait.

⚡ GUARD — **Load SEO Copilot config:**
Check `.claude/seo-copilot-config.json`:
- If yes: load `business.name` (branded-query filtering), `business.siteUrl` (default URL), `seo.primaryKeywords` (seed anchors), `seo.competitors` (competitor candidates), `audience.primary` and `business.markets` (relevance judgment + KE country)
- If no: proceed with defaults, note once: "Run `/getting-started` for personalized recommendations."

⚡ GUARD — **Keywords Everywhere unavailable:**
```
⚠️ Keywords Everywhere isn't connected.
Keyword Strategy relies on KE for competitor keywords, search volume, and competition data.

Options:
1. Connect it: https://github.com/hithereiamaliff/mcp-keywords-everywhere
2. Continue in degraded mode — site-derived keyword suggestions and clusters only,
   no volume, no difficulty, no competitor gap. The strategy will be directional, not data-backed.
```
Wait for the user's choice. In degraded mode: skip Phases 3–5 KE calls, mark every volume/competition cell "n/a", and label the report header prominently.

⚡ GUARD — **KE credit check:**
At start, call `get_credits`. A full run costs roughly: 1 domain-keywords pull per competitor + own domain (heaviest), plus keyword data for 50–150 keywords, plus a handful of related/PASF pulls. If credits look insufficient for the planned scope, tell the user the estimate and offer to reduce scope (fewer competitors, `:explore` mode) before spending.

⚡ GUARD — **Non-Google-friendly market:**
If the site's primary market isn't the default KE country (site language, ccTLD, config `business.markets`), set the KE country parameter accordingly and note it in the report header.

⚡ GUARD — **User requests abort:**
Confirm exit. Output any partial results already computed.

---

## Phase 0: DISCOVER

1. **MCP check** — search for Keywords Everywhere (core), GSC (optional), CMS MCP (optional). Apply guards above.
2. **Config load** — per global guard.
3. **Activity log** — check `.claude/reports/{domain}/activity-log.md`. If `/keyword-strategy` ran within 60 days, surface it: strategy documents have a shelf life of months, not weeks — ask whether the user wants a re-run or a review of the existing one (`latest-keyword-strategy.md`).
4. **Set `{domain}`** from the URL (registered domain, no `www`).

---

## Phase 1: ANALYZE SITE

Understand what the business does before touching keyword data — every later relevance judgment depends on this.

1. **Fetch the given URL** and parse: value proposition, offerings, audience signals, language/locale.
2. **Fetch the sitemap** (`/sitemap.xml`, fall back to robots.txt reference). Build the page inventory: URL, and for a sample of up to 15 representative pages (home, product/service pages, top-level category pages, 3–5 recent blog posts), fetch and extract title, H1, meta description.
3. **If a CMS MCP is connected** and matches the site: pull the page/CMS inventory from there instead of crawling — it's faster and complete.
4. **Summarize** in 4–6 lines: what the business sells, to whom, in which market, and what content already exists (page count by type: product/service, editorial, other).
5. **Infer the current keyword posture**: for each sampled page, note the apparent target keyword (from title/H1/slug). This is the "what they're already trying to rank for" baseline.

Show the summary to the user in one short block. If the business inference looks wrong, they'll correct it here — cheaper than correcting a whole strategy.

---

## Phase 2: SEED KEYWORDS

Generate 15–25 seed keywords internally (don't display the raw list — it primes the KE queries):

1. **Core offering terms** — the clearest 2–3 word descriptions of each product/service
2. **Audience-qualified variants** — "[offering] for [audience]"
3. **Problem/outcome terms** — what the customer searches before knowing the product category
4. **Category terms** — the market category name and its synonyms
5. **Config anchors** — `seo.primaryKeywords` if present

Exclude branded terms (own brand) — they're not strategy, they're navigation.

---

## Phase 3: COMPETITORS

**Skip in `:explore` mode.**

1. **Collect candidates** from, in order: user-provided names, config `seo.competitors`, competitors named on the site itself (comparison pages, "alternatives to" mentions), and your own knowledge of the category from Phase 1.
2. **Propose 3–5 candidates in one message** and ask the user to confirm, replace, or add. One round-trip, then proceed. Direct search competitors matter more than business competitors — a site can compete in the SERPs with publishers, not just rivals.
3. **Validate each confirmed competitor** via KE `get_domain_traffic`: a domain with no measurable organic traffic tells you nothing about the gap — drop it and say so.
4. Cap at 5 competitors. Each domain pull costs credits and past 5 the gap analysis stops changing.

---

## Phase 4: GAP ANALYSIS

**Skip in `:explore` mode.**

1. **Pull ranking keywords** via KE `get_domain_keywords` for the target domain and each validated competitor (top keywords by volume).
2. **Clean**: drop branded queries (any brand name in the set), duplicates, and keywords irrelevant to the target's business (use the Phase 1 summary as the relevance test — a competitor's careers page keywords are not a gap).
3. **Classify every remaining keyword:**

| Class | Condition | Meaning |
|-------|-----------|---------|
| **Gap** | 2+ competitors rank, target doesn't | Proven demand in this exact market, target absent |
| **Underperforming** | Target ranks below position 20, any competitor ranks above | Demand confirmed, page exists or partially exists, losing |
| **Watch** | Exactly 1 competitor ranks, target doesn't | Possibly that competitor's niche — validate volume before acting |
| **Held** | Target ranks top 20 | Already competing — exclude from strategy, note as strength |

4. **If GSC is connected**: cross-check "Gap" keywords against GSC queries. A keyword with GSC impressions isn't a pure gap — reclassify as Underperforming (Google already shows the site for it; it needs a better page, not a new topic).

---

## Phase 5: EXPLORE

Competitor gaps show where the market already is. Expansion shows where it's going.

1. **Related keywords** — KE `get_related_keywords` for the top 5 themes (by combined gap volume, or by seed importance in `:explore` mode)
2. **PASF** — KE `get_pasf_keywords` for the top 3 seeds: adjacent intent the audience actually expresses
3. **Seed validation** — KE `get_keyword_data` for all Phase 2 seeds not already covered by the gap data

**Filter out:** volume < 30/mo (unless CPC > $5), declining trend > 30% YoY, semantic drift (fails the Phase 1 relevance test), and anything already classified in Phase 4.

**Classify intent** for every surviving keyword:

| Intent | Signal | Maps to page type |
|--------|--------|-------------------|
| Informational | how to, what is, guide, checklist, tips | Editorial / blog |
| Commercial | best, vs, review, alternative, compare | Comparison / listicle |
| Transactional | buy, price, pricing, hire, [category] software/tool/service | Product / landing page |
| Navigational | brand names | Exclude |

---

## Phase 6: SCORE

Score every keyword (gap + underperforming + exploration) with the same formula the other skills in this suite use, so priorities are comparable across reports:

```
volume_score      = min(volume / 500, 5)
commercial_score  = min(CPC / 2, 5)
attainability     = (1 - competition) × 5      # KE competition 0–1
trend_modifier    = 1.2 if ↑ rising else (0.8 if ↓ declining else 1.0)
opportunity_score = round(((volume_score + commercial_score + attainability) / 3) × trend_modifier, 1)
```

Class modifiers — the class carries information the formula doesn't:
- **Underperforming**: ×1.2 (a page or partial page already exists — cheapest wins)
- **Gap**: ×1.0
- **Watch / exploration**: ×0.9 (demand less proven for this exact market)

KE's competition metric is an ads-competition proxy, not true organic difficulty. Treat attainability as directional; where a keyword's top SERP is visibly dominated by high-authority domains (you'll often know this from the domain pulls), say so in the notes column rather than trusting the number.

Priority buckets (consistent with CLAUDE.md):
- **Must pursue** — score ≥ 8.0, or Underperforming with volume ≥ 200/mo
- **High value** — score ≥ 4.0
- **Worth tracking** — score ≥ 1.5
- Below 1.5: cut from the strategy entirely.

---

## Phase 7: STRATEGY

This is the deliverable — everything before it is input. Turn the scored list into a plan:

### 7.1 Topic clusters

Group keywords sharing a root phrase or intent into clusters (aim for 4–8 clusters; a strategy with 20 clusters is a list, not a strategy). For each cluster:
- **Cluster name** and the **pillar keyword** (highest-volume broad keyword)
- Supporting keywords with volume, competition, intent, class
- **Existing coverage**: which of the site's current pages (Phase 1 inventory) already sit in this cluster

### 7.2 Page type mapping

Assign every Must-pursue and High-value keyword a page type:

| Page type | When |
|-----------|------|
| **Pillar / hub** | The cluster's broad head term — one per cluster |
| **Editorial / blog** | Informational keywords |
| **Comparison / listicle** | Commercial-investigation keywords (best, vs, alternatives) |
| **Product / landing** | Transactional keywords |
| **FAQ / glossary** | Definitional long-tail worth owning but not worth a full article |

### 7.3 Phased roadmap

Sequence into three phases by dependency and ROI, not just score:
- **Phase 1 (weeks 1–4)** — Underperforming fixes (existing pages, fastest wins) + the pillar page of the single highest-opportunity cluster
- **Phase 2 (weeks 5–12)** — supporting pages for that cluster + second cluster pillar
- **Phase 3 (quarter 2)** — remaining clusters, Watch keywords that validated

Rationale in one line per phase. Cap Phase 1 at 5 items — a roadmap nobody can execute is worse than a short one.

---

## Phase 8: REPORT

```
# Keyword Strategy — {domain}

**Date:** YYYY-MM-DD
**Market:** {country / language}
**Data sources:**
- ✅ Keywords Everywhere: {N} keywords analyzed, {K} competitor domains [or ⚠️ degraded mode — no volume data]
- ✅ GSC: own rankings cross-checked [or ℹ️ not connected]
- Site analysis: {N} pages inventoried

**The site in one paragraph:** {Phase 1 summary}

**Headline finding:** {1–2 sentences — the single most important strategic fact.
e.g., "Competitors own 340 relevant keywords this site doesn't touch; 80% of the gap
sits in two clusters the site has zero pages for."}

## Competitive Landscape
| Competitor | Est. organic traffic | Ranking keywords (relevant) | Overlap with {domain} |
|---|---|---|---|

## Keyword Gap
### Must pursue  — table: Keyword | Volume/mo | CPC | Competition | Trend | Intent | Class | Score | Notes
### High value   — same table
### Worth tracking — condensed table

## New Keyword Opportunities (expansion)
{same table structure, grouped by source theme}

## Topic Clusters
{per cluster: pillar keyword, supporting keyword table, existing coverage, page types}

## Roadmap
{Phase 1 / 2 / 3 tables: Keyword → Page type → Cluster → Action (create/rework) → Volume → Score}

## Next Steps
→ Run `/content-brief` to map every roadmap keyword to a URL and get per-page briefs
→ Run `/topic-map` after the first pages ship to verify cluster structure
```

Follow the Output Format Guidance in CLAUDE.md — collapse near-empty tables and sections.

---

## Phase 9: SAVE

- `.claude/reports/{domain}/keyword-strategy-YYYY-MM-DD.md` — timestamped
- `.claude/reports/{domain}/latest-keyword-strategy.md` — overwrite each run

Create directories if needed. After saving, print the path and the headline finding, then:

```
→ Next: /content-brief — maps these keywords to URLs and decides create vs rework per page.
```

---

## Integration with Other Skills

| Situation | Skill |
|-----------|-------|
| Turn this strategy into per-page briefs and create/rework decisions | `/content-brief` |
| Deep-dive keywords for one specific new page or product | `/keywords-opportunity:new-page` / `:new-product` |
| You have GSC and want wins from existing rankings | `/keywords-opportunity` |
| Verify cluster structure once pages exist | `/topic-map` |
| Execute: write a new article | `/write-blog` |
| Execute: fix an underperforming page | `/refresh-content {url}` |

Cadence: re-run quarterly, or when the business adds an offering or enters a new market. Rankings data shifts weekly; strategy shouldn't.

---

## Error Handling

| Error | Action |
|-------|--------|
| URL unreachable | Ask user to verify the URL. Stop. |
| No sitemap found | Crawl from the homepage nav links instead (up to 15 pages). Note reduced inventory. |
| KE not connected | Degraded-mode guard — user chooses. |
| KE credits exhausted mid-run | Stop KE calls, finish with data collected so far, mark incomplete sections, tell the user. |
| `get_domain_keywords` empty for target domain | Normal for new/small sites — note "no measurable rankings yet; every relevant keyword is effectively a gap" and proceed. |
| `get_domain_keywords` empty for a competitor | Drop that competitor, note it, continue with the rest. |
| All proposed competitors rejected by user and none supplied | Fall back to `:explore` mode, note why. |
| GSC connected but property missing | Proceed without GSC enrichment, note it. |

---

## Activity Log

After every execution, append to `.claude/reports/{domain}/activity-log.md` (create with the standard header if missing):

```
| YYYY-MM-DD | /keyword-strategy | {N} keywords analyzed vs {K} competitors. Gap: {N} must-pursue, {N} high-value. {N} clusters defined. Top cluster: "{name}". |
| YYYY-MM-DD | /keyword-strategy:gap | Gap analysis vs {K} competitors: {N} gap keywords, {N} underperforming. |
| YYYY-MM-DD | /keyword-strategy:explore | Site-derived research: {N} keywords, {N} clusters. No competitor data. |
```

Log even on early exits, with the reason.
