---
name: audit-deep
version: "2.5"
description: |
  Deep SEO & AEO audit for any website. Crawls live pages and runs evidence-based checks across four dimensions (Content, Technical, Authority, Measurement), organizes findings into three action buckets (Technical · Content · Authority) and three priority tiers, and attaches a business-impact forecast - estimated incremental clicks/month and equivalent ad spend - to each opportunity. Discovers sibling subdomains/GSC properties, and - when Keywords Everywhere is connected - adds volume-backed keyword research, content-gap and topic-cluster analysis (current + opportunity), and a backlink profile. Works on any platform - Webflow, WordPress, Shopify, Framer, Next.js, or custom code. GSC, PageSpeed, a CMS MCP, and Keywords Everywhere layer on as optional enrichments; none are required. Folds in client documents (crawl exports, paid-media or keyword strategy, analytics) and pauses for more inputs before assembling a structured nine-section report: summary, methodology, dimension summary, findings by bucket, keyword strategy, forecast, prioritized recommendations, roadmap, appendices. Then generates per-bucket working documents: a technical fix checklist, a topic-cluster overview and an editorial content-brief sheet, and backlink-opportunity and E-E-A-T improvement lists.
  Triggers: audit deep, deep audit, deep seo audit, deep aeo audit, post-sale audit, implementation audit.
  Requires: Web access (WebFetch). Optional: GSC MCP, PageSpeed MCP, a CMS MCP (Webflow/etc.), Keywords Everywhere (volume + backlinks).
  Workflow: Discover (incl. document intake) -> Crawl -> Enrich -> Check -> Prioritize -> Checkpoint for more inputs -> Assemble structured report -> Save -> Companion deliverables (per-bucket working docs).
  Command: /audit:deep {url}
---

# Deep Audit Skill

Platform-agnostic deep SEO and AEO audit. The backbone is a **live multi-page crawl** - the skill pulls the sitemap, selects a representative set of pages, fetches each, and runs evidence-based checks across the real page set. It assesses four dimensions (Content, Technical, Authority, Measurement), sorts findings into priority tiers, and produces a client-ready engagement baseline with quantified opportunities, a phased roadmap, and supporting data tables.

This works on **any setup** - Webflow, WordPress, Shopify, Framer, a Next.js or React/Vite app, or a hand-coded static site. The audit never assumes a platform and never requires a CMS connection.

Whatever else is connected makes the audit deeper, not different:

- **Google Search Console MCP** → search analytics for query-page alignment, cannibalization, CTR gaps, branded trend, and indexation cross-reference.
- **PageSpeed MCP** → Core Web Vitals on the key pages.
- **A CMS MCP** (Webflow or any other) → field-completeness and template-level checks against the source of truth instead of inferring from rendered HTML.
- **A crawl-tool export** (Screaming Frog / Sitebulb) → structural confirmation across the full page set.

This skill is **read-only** - it never modifies any site or CMS content.

## Prerequisites

- **Required**: Web access (WebFetch). A publicly reachable URL is the only hard requirement.
- **Optional - GSC MCP** ([server](https://github.com/sofianbettayeb/gsc-mcp-server)): adds search-analytics-backed checks and raises the confidence ceiling.
- **Optional - PageSpeed MCP**: Core Web Vitals (LCP, TBT, CLS, mobile/desktop scores).
- **Optional - a CMS MCP** (e.g. [Webflow MCP](https://developers.webflow.com/mcp/reference/overview)): field completeness and template-level schema/metadata checks against the CMS itself.
- **Optional - Keywords Everywhere MCP** ([server](https://github.com/hithereiamaliff/mcp-keywords-everywhere)): volume/intent enrichment.

## Confidence Tiers

Confidence scales with the data available - not the platform:

| Data available | Confidence ceiling |
|----------------|--------------------|
| Live crawl only (sitemap + multi-page WebFetch) | Low–Medium |
| Crawl + PageSpeed | Low–Medium |
| Crawl + GSC | Medium |
| Crawl + GSC + (CMS MCP or crawl-tool export) | Medium–High |

State the actual confidence in the report header based on what ran.

## How findings are organized

No maturity score and no 1-to-5 levels. The audit reports four dimensions (Content, Technical, Authority, Measurement) as a plain-language summary (what is working, the biggest opportunity), sorts every finding into three action buckets (Technical, Content, Authority) and three priority tiers (Must fix, High impact, Nice to have) using observable criteria, and attaches a business-impact forecast to each opportunity. The priority tiers and forecast, not a score, drive the roadmap.

---

## Critical Guards

GUARD - **Homepage not reachable:**
- If WebFetch on the homepage fails: "Could not reach {url}. Check the URL is correct and publicly accessible." Stop.

GUARD - **No sitemap found:**
- Fall back to crawling links discovered on the homepage (collect same-host `<a href>` targets, dedupe, cap the set). Note in the report: "No sitemap found - page set discovered by following homepage links."

GUARD - **GSC not connected:**
- Do **not** stop. Proceed in crawl mode. Mark all GSC-dependent checks `N/A - no search-analytics data` and note: "Connect GSC MCP for search-analytics-backed findings (cannibalization, CTR gaps, indexation cross-reference) and higher confidence."

GUARD - **CMS MCP not connected (or doesn't match the site):**
- Do **not** stop. Field-completeness and template-level checks run against rendered HTML from the crawl instead of the CMS. Note that the source of truth for these checks was the rendered page, not the CMS.

GUARD - **Low traffic (< 500 impressions in 90 days, GSC mode only):**
- Warn: "Low traffic - statistical confidence on search-data findings is reduced." Proceed with caveats.

GUARD - **User requests abort:**
- Confirm, exit cleanly, output any partial results.

---

## Phase 0: DISCOVER

### 0.1 MCP Discovery

Search for tools BEFORE starting. **None are required** - note availability and adapt scope.

- **GSC MCP**: Search `+gsc search analytics`. If present → GSC enrichment on. If missing → crawl mode, note it.
- **PageSpeed MCP**: Search `+pagespeed performance audit`. If present → Core Web Vitals on. If missing → skip that section.
- **CMS MCP**: Search `+webflow data cms` (and any other CMS MCP the user has). If present → CMS enrichment on. If missing → rendered-HTML proxy.
- **Keywords Everywhere**: Search `+keywords everywhere volume`. If present → volume enrichment. If missing → note, proceed.

### 0.2 Parse URL & Set {domain}

Take the URL from the command, or prompt for one if not given. Extract and normalize the domain (strip `www.`, lowercase). Determine the homepage URL.

`{domain}` is used for:
- **Report save path**: `./{domain}/reports/audit-deep-YYYY-MM-DD.md`
- **Latest pointer**: `./{domain}/reports/latest-deep.md`
- **Activity log**: `./{domain}/reports/activity-log.md`

If GSC is connected, confirm the matching GSC property (e.g. `sc-domain:{domain}`). If multiple match, present a numbered list and ask. If none match the URL, note it and continue in crawl mode.

### 0.3 Platform Detection (informational only)

Fetch the homepage and detect the platform from HTML signals. This is **informational** - it adapts the report's wording and which enrichment is most useful, but it never gates a check.

| Signal | Platform |
|--------|----------|
| `/wp-content/` or `wp-json` | WordPress |
| `data-wf-site` attribute or `assets.website-files.com` | Webflow |
| `cdn.shopify.com` | Shopify |
| `static.wixstatic.com` | Wix |
| `squarespace-cdn.com` | Squarespace |
| `framerusercontent.com` | Framer |
| `/_next/` static chunks | Next.js |
| Vite/React build fingerprints (`/assets/index-*.js`, `id="root"`) | Custom / code-based |
| No match | Custom / Unknown |

Record `{platform}`. If a CMS MCP is connected but the platform doesn't match it (e.g. Webflow MCP connected but the site is Next.js), note: "CMS MCP connected but the site is {platform} - CMS checks fall back to rendered-HTML signals."

### 0.4 Review Activity Log

Check `./{domain}/reports/activity-log.md`:
- If it exists: show the last 10 entries.
- **Redundancy check**: if `/audit:deep` ran in the last 30 days → warn with date.
- If `./{domain}/reports/latest-quick.md` exists, note it for comparison.
- If not found: proceed silently.

### 0.5 Load Config

Load `.claude/seo-copilot-config.json` if present. Extract:
- `business.name` → branded query detection (GSC mode)
- `seo.competitors` → competitor mentions
- `audience.primary` → framing

If not found: proceed with defaults, note "Run `/getting-started` for personalized recommendations."

### 0.6 Sibling Property & Subdomain Discovery

A brand often has more than one web property - a separate showroom/landing subdomain, a regional site, a microsite on a different host. These are usually **invisible to the main GSC property** (a URL-prefix property only covers its own host) and are easy to miss, yet they frequently hide the worst technical problems. Always check:

1. **GSC properties**: call `gsc_list_sites` and list every property whose domain or host is related to `{domain}` (same root domain, subdomains, regional variants). Note which are verified.
2. **Subdomains in the wild**: scan the sitemap, homepage links, nav/footer, and `robots.txt` for links to other hosts on the same root domain (e.g. `go.`, `shop.`, `blog.`, `<city>.`). Note any landing-page-builder hosts (HubSpot `*.hs-sites.com`, Unbounce, Instapage).
3. For each related property/subdomain found, run a **mini technical pass** (fetch homepage + `technical_seo_audit`): title, meta, canonical, schema, indexability. Flag any that are **off the audited GSC property** (a measurement blind spot) or **self-sabotaging** (see the canonical-integrity check in Phase 3).

Record the full property set in the report header ("Properties analyzed") and give any broken sibling its own finding - a landing page with a wrong canonical or placeholder metadata can silently waste an entire regional market.

### 0.7 Additional Documents and Context (intake)

The audit should fold in any documents the client or team can provide, not just the live data this skill gathers. These often reframe priorities (a paid-media strategy can re-order the whole keyword plan) or confirm a finding (a crawl export confirming an access problem).

1. **Check the project sources folder.** Look in `./{domain}/sources/` for any files already dropped there. Read and use whatever is present.
2. **Prompt once for more** (do not block; proceed if the user has none):
   > "Any documents or context to fold in before I build the report? Common ones: a crawl export (Screaming Frog / Sitebulb), a paid-media or keyword strategy, brand or tone guidelines, an analytics export, a competitor list, or a previous audit. Share a path or paste it, or say skip."
3. **Save** anything provided into `./{domain}/sources/` (copy the file, keep the original name) so the audit is reproducible and the inputs travel with the report.
4. **Map each document to the section it informs**, and let it adjust the analysis before scoring:

| Document type | Informs | How it changes the analysis |
|---------------|---------|------------------------------|
| Crawl export (Screaming Frog / Sitebulb) | Technical (4.1), Appendix | Confirms or quantifies indexation, status codes, metadata, crawl access; takes precedence over inferred estimates |
| Paid-media / keyword strategy | Keyword strategy (5), Recommendations (7) | Re-orders keyword clusters and priorities to match commercial intent; align organic to paid |
| Brand / tone guidelines | Authority (4.3), and any copy recommendations | Shapes E-E-A-T and voice guidance |
| Analytics export (GA4 / Adobe) | Measurement (4.4), Forecast (6) | Adds conversion rate and value so the forecast can be expressed in revenue, not just clicks |
| Competitor list | Keyword strategy (5), Authority (4.3) | Frames gaps and backlink targets against named competitors |
| Previous audit / report | Executive summary (1), Dimension summary (3) | Becomes the baseline for "what changed since" |

This intake is the first chance to add inputs. There is a second, deliberate chance to add more right before the final document is assembled (Phase 5.2), so the report is never finalized prematurely.

---

## Phase 1: CRAWL (universal backbone)

This phase always runs and produces the page set every check operates on.

### 1.1 Discover the page set

1. Fetch `https://{domain}/sitemap.xml`. If it fails, try `/sitemap_index.xml`, then `/sitemap-index.xml`, then the `Sitemap:` line in `robots.txt`. Expand any index into child sitemaps.
2. If no sitemap exists, fall back to following same-host links from the homepage.
3. Collect all discovered URLs. Record the total count (this feeds the sitemap-vs-live check).

### 1.2 Classify URLs into page types

Group URLs by path pattern to detect content types without a CMS:
- `/` → homepage
- `/blog/*`, `/articles/*`, `/guides/*`, `/learn/*` → editorial
- `/product/*`, `/pricing`, `/features/*`, `/solutions/*` → commercial / product
- `/docs/*` → documentation
- `/about`, `/team`, `/authors/*` → authority
- Everything else → other

Record the detected types and their URL counts. These stand in for "CMS collections / templates" on platforms without a CMS.

### 1.3 Select a representative sample

Cap the crawl to keep it bounded (target ≤ 20–25 fetches):
- Always include the homepage.
- Up to 3 pages per detected page type.
- If GSC is connected, prioritize the top pages by impressions within each type; otherwise take the first N per type from the sitemap (prefer shortest paths = likely hubs).
- Always include at least one editorial page and one commercial page if they exist.

**Prefer GSC's actual ranking pages as the page set.** When GSC is connected, the page-level export *is* the real, complete set of pages Google ranks - use it as the spine of the analysis rather than relying on a sample crawl, and reserve the WebFetch crawl for the head-tag/schema details GSC doesn't expose. A sample crawl is the fallback when GSC is absent.

Record which URLs were selected and why. Note the sample size and total page count in the report footnote.

### 1.4 Fetch and parse each selected page

WebFetch each selected URL. For each page, extract and store:
- `<title>`, meta description, canonical, viewport, Open Graph tags
- H1 count + text, heading hierarchy (H1→H2→H3 order)
- Visible word count (strip nav/footer/scripts)
- JSON-LD blocks and the `@type` of each
- Internal link count (same-host), outbound non-social citation count
- FAQ / question-pattern headings
- Author byline / credential markers
- Freshness date (`<time>`, "updated", structured date)
- Analytics/tracking scripts - detect the full range, not just GA: GA4 (`gtag(`/`G-`), GTM (`GTM-`), **Adobe Analytics** (`AppMeasurement`, `s_code`, `assets.adobedtm.com`, `*.sc.omtrdc.net`, Omniture), PostHog, Plausible, Fathom, Segment (`analytics.js`), Matomo, Hotjar, Mixpanel. List every tool found (an enterprise stack like Adobe Analytics is a strong Measurement signal).

Also fetch `robots.txt` once (directives, sitemap reference, blocked paths).

---

## Phase 2: ENRICH (optional, only what's connected)

### 2.1 GSC Data (90 days) `[skip if GSC not connected]`

- **Page-level**: all pages with impressions - URL, clicks, impressions, CTR, position.
- **Query-level**: all queries with impressions > 5 - query, page, clicks, impressions, CTR, position.
- **URL inspection** (representative set): top 20 pages + one per page type - index status, last crawl, mobile usability.

**Large dataset handling:** don't read thousands of rows into context. Extract targeted slices with `jq`/`python3`: top 100 queries by impressions, top 50 by clicks, cannibalization candidates (queries where 2+ pages appear), and a 200-row sample for patterns. Note any sampling.

### 2.2 PageSpeed Data `[skip if PageSpeed MCP not connected]`

Run on the homepage + top 2 pages (by GSC impressions, or by likely importance in crawl mode). Record: mobile + desktop scores, LCP, TBT, CLS, and key diagnostics (render-blocking, image sizes, unused JS). Mobile score below 50 is a "Must fix" finding.

### 2.3 CMS Data `[skip if no CMS MCP, or CMS doesn't match site]`

When a CMS MCP is connected and matches the platform, read the source of truth instead of inferring:
- Collections/content types, fields, template SEO title/description/OG mapping.
- Per content collection, score field completeness against five groups (Core 30% / Content 25% / Authority 20% / Enhancement 15% / Social 10%) using alias matching (`seo-title`, `meta-title`, `og-title` all count). Compute per-group and weighted overall score; collections below 60% are a "Must fix".

If no CMS MCP: derive the equivalent signals from the crawled HTML across each detected page type (title/desc/OG/schema presence per type). Note that the source was rendered HTML, not the CMS.

### 2.4 Crawl-tool export (optional)

Ask once:
> "Do you have a Screaming Frog or Sitebulb export? Share the path to the issues-overview CSV and I'll fold it in."
> Options: 1. Yes - provide path  2. No - skip

If provided, map issue types to checks (Missing H1 → H1 coverage; Missing Meta Description → metadata completeness; Over 100 KB images → performance; Noindex → indexation; Low Content Pages → thin content; etc.). Crawl-tool data takes precedence over inferred estimates. Note: "Structural issues confirmed by [Tool] crawl of [N] pages."

### 2.5 Keyword Research & Backlinks (Keywords Everywhere - optional)

When Keywords Everywhere is connected, run a proper keyword and link layer (check `get_credits` first; these endpoints cost credits). This data powers report sections 5–7.

**Keyword volume enrichment** (`get_keyword_data`, country + currency for the target market):
- Enrich the top GSC head terms and every content-gap candidate with **monthly search volume, CPC, competition, and 12-month trend**.
- Flag high-CPC terms - they signal high commercial intent and margin (e.g. a service term at £3+ CPC is worth a dedicated page even at modest volume).
- Volume reframes priority: a term where the site sits at position 11 means far more if it's 18,000/mo than 180/mo.

**Ranking-keyword universe** (`get_domain_keywords`, target country):
- Pull what the domain *already* ranks for with estimated traffic and SERP position - cross-reference against GSC to confirm and to surface terms GSC truncated.

**Backlink profile** (`get_unique_domain_backlinks`, and `get_domain_backlinks` for anchor/url detail):
- Summarize **referring domains, anchor-text distribution, and target URLs**.
- **Segment quality vs spam**: genuine editorial/authority links (relevant publications, design/interiors blogs, trade directories) vs link-scheme footprints (repeated cookie-cutter directory pages, PBN article networks, identical anchors across throwaway domains).
- Healthy = mostly branded/natural anchors + topically relevant sources. Flag a heavy spam footprint as a **toxic-link review / disavow** candidate (note: usually ignored by Google, only act on a manual action or unexplained drop).

If Keywords Everywhere is **not** connected: derive keyword research and content gaps from GSC impressions alone (real demand, no absolute volume), skip the backlink section, and note both limitations in the report.

---

## Phase 3: CHECK

Run every check across the **crawled page set** (Phase 1), upgraded by any enrichment data (Phase 2). Every check produces: **Pass/Fail**, **Evidence** (snippet/count/value), **Source** (which page/template).

**IMPORTANT:** Check IDs (CD1, TD1, etc.) are internal only. They must NEVER appear in the client-facing report - use plain language and data evidence.

The "Source" column says where each check gets its strongest signal. When the listed enrichment is absent, the check still runs on the crawl with the noted fallback (or is marked `N/A` if there is no proxy).

### Content Checks

| # | Check | Rule | Source (fallback) |
|---|-------|------|-------------------|
| CD1 | Query-to-page alignment | Each top query has a relevant page (query in title/desc/H1) | GSC (N/A without GSC) |
| CD2 | No cannibalization | No query cluster has 2+ pages within 5 positions | GSC (N/A without GSC) |
| CD3 | No thin content | Every crawled page ≥ 300 visible words | Crawl |
| CD4 | Topic coverage | Each detected page type / query cluster has supporting pages | Crawl + GSC |
| CD5 | Field/metadata completeness | Title + description + OG present across each page type | CMS MCP (fallback: crawled HTML per type) |
| CD6 | Editorial → commercial linking | Each editorial page has ≥1 in-content link to a product/service page | Crawl |
| CD7 | FAQ / answer structure | Pages use FAQ blocks or question-format headings | Crawl |

### Technical Checks

| # | Check | Rule | Source (fallback) |
|---|-------|------|-------------------|
| TD1 | Indexation cross-reference | All sitemap pages are indexed | GSC (fallback: sitemap vs crawlable/robots, mark partial) |
| TD2 | Metadata not falling back to bare name/slug | Titles/descriptions are intentional, not auto from page name | CMS MCP (fallback: detect identical title==H1==slug across crawl) |
| TD3 | Schema coverage per page type | Each content type carries appropriate JSON-LD (Article on editorial, etc.) | Crawl (CMS MCP confirms template rules) |
| TD4 | Crawl freshness | Top pages crawled within 30 days | GSC (N/A without GSC) |
| TD5 | Sitemap vs live pages | Sitemap URL count matches discoverable pages (within 10%) | Crawl |
| TD6 | No duplicate titles | Zero duplicate `<title>` across the crawled set | Crawl (GSC/crawl-tool extends coverage) |
| TD7 | Metadata length | Titles 30–60 chars, descriptions 70–155 chars across the set | Crawl |
| TD8 | HTTPS + viewport + canonical | Present across the crawled set | Crawl |
| TD9 | Core Web Vitals | Mobile PageSpeed ≥ 50, LCP < 4s, CLS < 0.1 | PageSpeed MCP (N/A without it) |
| TD10 | Canonical integrity | Every canonical resolves to a URL on the **same host** - never an external domain (e.g. `http://google.com`) or a `#`/empty value | Crawl (**critical** - a foreign canonical de-indexes the page; always check sibling subdomains/landing pages) |
| TD11 | No placeholder content shipped live | No `lorem ipsum`, `blog-placeholder`, "Your title here", or default builder text in title/meta/OG/body | Crawl (common on landing-page-builder pages) |

`TD10` and `TD11` are cheap, high-severity, and most often fail on **sibling subdomains / landing-page-builder pages** (HubSpot, Unbounce, Instapage) - run them on every property found in Phase 0.6, not just the main site.

### Authority Checks

| # | Check | Rule | Source (fallback) |
|---|-------|------|-------------------|
| AD1 | Branded query volume | Branded queries (containing `business.name`) have clicks | GSC (fallback: brand entity present on-page) |
| AD2 | Branded trend positive | Branded clicks not declining month-over-month | GSC (N/A without GSC) |
| AD3 | Author attribution coverage | Author byline on ≥ 80% of editorial pages | Crawl (CMS MCP confirms) |
| AD4 | Author / about entity exists | About or author page discoverable with credentials | Crawl |
| AD5 | E-E-A-T consistency | Experience/credential markers consistent across content | Crawl |
| AD6 | External citations | Content pages cite non-social authoritative sources | Crawl |

### Measurement Checks

| # | Check | Rule | Source (fallback) |
|---|-------|------|-------------------|
| MD1 | Analytics on all page types | Tracking script present on every crawled page type | Crawl |
| MD2 | GSC data freshness | GSC has ≥ 90 days, most recent within 3 days | GSC (N/A without GSC) |
| MD3 | Existing reports | `./{domain}/reports/` contains weekly/monthly reports | Filesystem |
| MD4 | Conversion tracking | GA4 events / goal config detected | Crawl |

---

## Phase 4: PRIORITISE

No maturity score, no 1-to-5 levels. Turn the checks into a plain-language dimension summary, prioritized findings, and a forecast. The priority tiers and the forecast, not a score, drive the roadmap.

### Dimension summary

For each of the four dimensions (Content, Technical, Authority, Measurement), summarize what the checks found: what is working, and the biggest opportunity. Give each a qualitative status word, never a number or level:

| Status | Meaning |
|--------|---------|
| Strong | Few or no gaps; an asset to build on |
| Solid | Working, with clear room to improve |
| Needs work | Real gaps holding back results |
| At risk | A blocker actively costing traffic or trust |

Choose the word from the weight of evidence across that dimension's checks. Name the strongest and the weakest dimension. When a dimension's key signals depend on an absent enrichment (for example no GSC), say so and lower the stated confidence rather than guess. A foreign canonical or placeholder content shipped live puts Technical "At risk" regardless of the rest.

### Confidence

```
confidence = per the Confidence Tiers table (state which tier applied)
```

### Prioritization (Impact/Confidence/Effort)

Score every finding: Impact 1–5, Confidence 1–5, Effort 1–5 (1 = metadata fix … 5 = architecture change).

```
Priority = (Impact × Confidence) / Effort
```

| Bucket | Criteria |
|--------|----------|
| **Must fix** | Priority ≥ 8.0 |
| **High impact** | Priority ≥ 4.0 |
| **Nice to have** | Priority ≥ 1.5 |

### Business Impact Forecast `[needs GSC for clicks; CPC from Keywords Everywhere for £ value]`

Translate each opportunity into **estimated incremental clicks/month** and the **equivalent paid-search spend** it would replace. This is what connects findings to money - and it often *re-orders* priorities (a low-volume, high-CPC service term can outvalue a high-volume cheap one).

**Reference organic CTR-by-position curve** (blended desktop+mobile; an estimate - state this):

| Position | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 | 11–20 | 21–30 | 31+ |
|----------|----|----|----|----|----|----|----|----|----|----|-------|-------|-----|
| CTR | 28% | 15% | 10% | 7% | 5.5% | 4.3% | 3.5% | 3.0% | 2.6% | 2.3% | 1.2% | 0.6% | 0.3% |

**Per-opportunity formula** (use GSC's actual data, normalized to monthly - divide 90-day figures by 3):

```
monthly_impressions   = gsc_impressions_period / months_in_period
current_clicks_mo     = gsc_clicks_period / months_in_period
projected_clicks_mo   = monthly_impressions × CTR(target_position)
incremental_clicks_mo = max(0, projected_clicks_mo − current_clicks_mo)
ad_equivalent_£_mo    = incremental_clicks_mo × CPC        # what Google Ads would charge for those clicks
annual_value_£        = ad_equivalent_£_mo × 12
```

**Target-position rules (stay conservative - this is a forecast, not a promise):**
- Page-2 head terms (current pos 11–20) → target **6–8**, not #1.
- Striking distance (current pos 4–10) → target **3**.
- Deep results (pos 21+) → target **10** (page-1 entry).
- New/untargeted terms (no current impressions, e.g. a content gap) → forecast separately and label **"build + rank, longer horizon"** using KE volume × CTR(target); never mix into the "existing impressions" total.
- **Cap impressions at current** (don't inflate for query expansion) - this deliberately under-counts, so the number is defensible.
- A second lever is **CTR recovery at the same position** (better titles/snippets): `incremental = monthly_impressions × (CTR_benchmark_for_current_pos − current_CTR)`. Use for pages with high impressions and below-curve CTR (e.g. a strong position with a weak/generic title).

**Roll up by bucket.** Sum the per-opportunity forecasts under each of the three action buckets (Technical / Content / Authority) so each bucket carries its own clicks/mo and £/mo total, then a site total. Always state: "Estimates from GSC impressions × benchmark CTR × CPC; directional, not guaranteed."

---

## Phase 5: REPORT & SAVE

### 5.1 Report Principles

A professional client deliverable, the foundation for an engagement:

1. No check IDs. Plain language only ("content gaps", "indexation coverage", "branded search trend").
2. No platform assumptions beyond the detected platform, and no platform-specific how-to. Findings are about the site, not the tool.
3. Opportunities, not failures. Quantified upside.
4. Describe what and why with data. The how is the engagement.
5. Quantify with real data when available; in crawl-only mode use crawl counts and state confidence honestly.
6. Lead to a prioritized action plan.
7. No em dashes and no emojis in the output.
8. No internal tooling or workflow references in the client-facing report. Never name internal skills or processes (for example a humanizer step, this audit skill, or internal QA notes). Keep those reminders in internal working docs, not the deliverable.

### 5.2 Assemble the report

**First, a checkpoint (room for more inputs).** Before writing the final document, pause:
- Summarize the working findings in a few lines (overall level, the top 2 to 3 issues, the headline forecast).
- Ask once: "Anything else to fold in before I assemble the final report? More documents, another analysis, a competitor to compare, or a specific angle to emphasize? Otherwise I will build it now."
- If the user adds inputs, run each through the Phase 0.7 mapping (which section it informs), update that section's analysis, and ask again. Loop until the user says proceed.
- Only then assemble the final document. Never bolt late additions onto the bottom; fold each into the section it belongs to so the report stays one clean structure.

**Then output a single markdown file in this fixed nine-section structure.** Omit a section only if it has no data; never leave an empty header; never append new analysis at the end.

Header:
```
# SEO and AEO Audit: {domain}

**Prepared for:** {business name or domain}
**Date:** YYYY-MM-DD
**Platform:** {platform}
**Assessment type:** Deep
**Confidence:** {tier}
**Period analyzed:** {date range}
**Properties analyzed:** {main + any siblings from Phase 0.6; flag any not in the GSC property}
**Analytics detected:** {tools, e.g. GSC + Adobe Analytics}
**Data sources:** {only those that ran, including any documents folded in from /sources/}
```

Follow the header with a short Contents list of the nine sections.

**1. Executive summary.** Five to eight sentences for a decision-maker: the two or three things that matter most; the headline forecast (incremental clicks and ad-equivalent value); one line on what is already strong. If a paid-media or strategy document was provided, state the alignment principle here. Do not assign a maturity level or score.

**2. Methodology and data sources.** What was analyzed (lead with GSC ranking pages and the crawl), the documents folded in from /sources/, how findings are organized (four dimensions, three action buckets, three priority tiers, plus a forecast - no maturity score), the confidence tier, and the caveats (anything a blocked crawl or missing API could not assess).

**3. Dimension summary.** A 4-dimension table (Content, Technical, Authority, Measurement) with the qualitative status word (Strong / Solid / Needs work / At risk), what is working, and the biggest opportunity. Name the strongest and weakest dimension. No numeric scores, no levels, no check IDs.

**4. Findings by bucket.** A data-backed narrative per bucket. Lead with strengths, then gaps with numbers.
- 4.1 Technical: indexation and crawl access (including whether non-Google crawlers are blocked), canonical integrity and placeholder content across all properties, Core Web Vitals, sitemap and robots, schema, thin templates.
- 4.2 Content: cannibalization, thin commercial pages, topic-cluster structure (fragmentation, editorial-to-commercial linking), coverage (point to Section 5).
- 4.3 Authority: branded presence, E-E-A-T depth, backlink quality (authority vs spam).
- 4.4 Measurement: analytics stack detected, GSC data quality, conversion tracking, any property not covered by GSC.

**5. Keyword strategy.** The consolidated keyword universe in one place, grouped into named clusters (for example: Brand, core head terms, each location, premium/bespoke, product and material lines, services). Each cluster gets a small table: keyword, volume, CPC, current position, status (ranking / page 2 / near page 1 / GAP). If a paid-media or keyword strategy document was provided, organize the clusters to mirror it and align organic to paid (intent over volume). End with a "deliberately deprioritised" list (broad or off-strategy terms, and any zero-volume brand-only terms) so the choices are explicit. All keyword analysis lives here; do not scatter it across separate research, content-gap, and cluster sections.

**6. Business impact forecast.** The method note, the core-terms forecast (existing impressions: incremental clicks/month and ad-equivalent value, rolled up by bucket), and a separate build-and-rank upside table for the gap clusters. If an analytics export was provided, add a revenue line (clicks to conversions to value). Always close with the directional disclaimer.

**7. Prioritized recommendations.** The action plan, ordered so work starts at the top. One table: number, recommendation, bucket, cluster, effort (S/M/L), expected impact. Foreign canonicals and placeholder content always sit at the top. This is the section the client acts on.

**8. Roadmap.** Group the recommendations into phases (Foundation weeks 1 to 2, Core capture weeks 3 to 6, Specialist depth weeks 7 to 12, plus ongoing tracking), each with an expected outcome. Close with the expected shift per dimension in plain language (for example "Technical moves from At risk to Solid once the canonical and crawl-access fixes ship"), not a maturity target.

**9. Appendices.** Supporting tables only, no new arguments: top pages (GSC), highest-impression query gaps, keyword research dataset reference, backlink profile, crawl summary, properties and subdomains, business-impact forecast detail, and the list of source documents in /sources/. Include an appendix only when its data exists.

### 5.3 Save to File

- `./{domain}/reports/audit-deep-YYYY-MM-DD.md` (timestamped)
- `./{domain}/reports/latest-deep.md` (overwrite each run - other skills read this)

Save supporting datasets under `./{domain}/reports/data/`:
- `crawl-pages.json` - the crawled page set with per-page extracted signals
- `content-gaps.json`
- `cannibalization.json` `[GSC mode]`
- `metadata-audit.json`
- `keyword-research.json` `[Keywords Everywhere - volume/CPC/position per term]`
- `topic-clusters.json` `[current clusters + opportunity pillars]`
- `backlinks.json` `[Keywords Everywhere - referring domains, quality segment, anchors]`
- `properties.json` `[sibling subdomains/GSC properties from Phase 0.6 + their issues]`
- `forecast.json` `[per-opportunity click/£ forecast + bucket and total roll-up]`
- **`baseline.json`** - snapshot for `/weekly-report` deltas:

```json
{
  "domain": "{domain}",
  "date": "YYYY-MM-DD",
  "platform": "{platform}",
  "mode": "crawl | crawl+gsc | crawl+gsc+cms",
  "analytics": [],
  "properties": [ {"url": "", "in_gsc_property": false, "critical_issues": []} ],
  "forecast": {
    "incremental_clicks_mo": { "technical": 0, "content": 0, "total": 0 },
    "ad_equivalent_gbp_mo": { "technical": 0, "content": 0, "total": 0 },
    "annual_value_gbp": 0,
    "upside_clicks_mo": 0,
    "basis": "GSC impressions × benchmark CTR-by-position × CPC; directional"
  },
  "dimensions": { "content": "", "technical": "", "authority": "", "measurement": "" },
  "crawl": { "pages_total": 0, "pages_sampled": 0, "page_types": [] },
  "search_footprint": {
    "total_impressions_90d": null,
    "total_clicks_90d": null,
    "avg_ctr": null,
    "avg_position": null,
    "pages_with_impressions": null,
    "queries_with_impressions": null
  },
  "top_pages": [ {"url": "", "impressions": 0, "clicks": 0, "ctr": 0.0, "position": 0.0} ],
  "pagespeed": { "mobile_score": null, "desktop_score": null, "lcp_mobile": null, "tbt_mobile": null, "cls_mobile": null }
}
```

`search_footprint` fields are `null` in crawl-only mode. Create dirs as needed: `mkdir -p ./{domain}/reports/data/`.

### 5.4 Activity Log

Append to `./{domain}/reports/activity-log.md`:

```
| YYYY-MM-DD | /audit:deep | Deep audit ({platform}, {mode}). Dimensions - Content: {status}, Technical: {status}, Authority: {status}, Measurement: {status}. Must-fix: N. Forecast: +{clicks}/mo (~{currency}{value}/mo). [Mobile PageSpeed: XX/100.] |
```

Log even on early exit.

---

## Phase 6: COMPANION DELIVERABLES (working documents per bucket)

The report explains the situation. These are the documents the team executes from. Generate them after the report (offer them, do not force all of them) and save under `./{domain}/deliverables/`. Each maps to one action bucket and to the prioritized recommendations in report Section 7. No em dashes, no emojis.

### 6.1 Technical: fix checklist (`tech-fix-checklist.md`)

A developer-ready checklist, grouped by area, every item a checkbox with a clear done-definition.

```
| Done | Fix | Area | Severity | Owner | Effort | Done when |
|------|-----|------|----------|-------|--------|-----------|
| [ ] | ... | Canonical / Indexation / CWV / Crawl access / Sitemap / Schema / Tracking | Critical/High/Med | Dev / SEO | S/M/L | observable test |
```

Pull every Technical finding (4.1) plus the measurement/tracking setup items. Critical first (foreign canonical, placeholder content, crawler blocking). Make each "Done when" testable (for example: "canonical on /edinburgh/ resolves to its own URL", "LCP under 2.5s on mobile in PageSpeed", "Bing/AI user-agents return 200 not 403").

### 6.2 Content: topic-cluster overview + content briefs

**`topic-clusters.md`** - the cluster map. One block per cluster (from Section 5): the pillar page, the supporting content that already exists (with current position), what is missing, and the commercial target each links to. Mark current vs opportunity, so the team sees consolidation wins versus net-new builds.

**`content-briefs.csv`** - an editorial production sheet, modeled on a standard SEO editorial tracker. Include **every page that needs content work**, not only new pages: existing thin, weak, or cannibalised pages to rewrite; decent pages that only need light optimisation; and net-new pages to build. Columns:

```
Content action, Search Volume, Impact, Owner, Publication date,
Target Keyword, Secondary Keywords, Target URL, Intent, Meta Title, Meta Description,
H1, H2 outline, Internal links, Top words, Word count target, Notes
```

- **Content action** (the type of work): `Content creation` (new page), `Content update` (substantial rewrite or expansion of an existing page, including cannibalisation consolidation), `Content refresh` (light optimisation of an existing page: metadata, internal links, freshen dates and copy).
- **Internal links**: the internal linking plan for the page, which pages should link in and which it should link out to. Capture the editorial-to-commercial and pillar-to-shoppable links explicitly (for example a rug-type guide links to its shoppable category; a category links to its parent and to relevant guides).
- Intent: Transactional / Commercial / Informational.
- Impact: High/Med/Low from the forecast and the paid alignment.
- For creation and update rows, draft the Meta Title, Meta Description, H1, and an H2 outline so the brief is ready to write from. For refresh rows, the specific edit goes in Notes (for example "add internal links to the new shoppable pages; add a named author") and the existing meta can be left in place. Cover all pages flagged in Section 4 (Content and Technical) and Section 7, grouped by content action. Do not include a Market column.

### 6.3 Authority: backlink opportunities + E-E-A-T improvements

**`backlink-opportunities.md`** - a prospect list, not a wish list. Columns: Prospect domain, Type (publication / interiors blog / trade directory / supplier / press), Why relevant, Suggested target URL, Approach (guest piece / digital PR / listing / reclamation), Priority. Seed it from the genuine referring domains already earned (replicate what works), competitor link gaps, and the brand's niche. Include any unlinked-mention or broken-link reclamation found.

**`eeat-improvements.md`** - structured by the four E-E-A-T pillars, not a flat list. Each item: action, where to apply, why it lifts that pillar, priority.
- **Experience (first-hand):** sourcing stories with original photos, "inspected and graded by us" notes, original product and showroom photography and video, real customer/home-trial evidence, per-item provenance.
- **Expertise:** named author and specialist profiles with credentials and specialisms, bylines linked to those profiles, "reviewed by [name, date]" lines, any valuation/appraisal/authentication services, a knowledge base or glossary.
- **Authoritativeness:** a real "as featured in" press strip, genuine trade memberships and awards, entity-building (complete Google Business Profile per location, founder and brand profiles, sameAs, Wikidata), and the earned-link plan.
- **Trustworthiness:** real reviews with AggregateRating, clear policies (guarantee, authenticity, returns, aftercare), unmistakable physical-showroom proof (NAP, hours, map, photos), pricing transparency, multiple contact paths.
- **Schema that encodes it:** Person/author, enriched Organization (founder, memberOf, award, hasCredential), LocalBusiness per location, AggregateRating/Review, Article with linked author and dateModified, Product with material/origin.
- **AEO angle:** attribute quotable answers to named experts and keep entity signals consistent so AI engines can cite a recognised authority; note that this requires crawl access (the WAF must not block AI crawlers).
- Tie items to the pages that already rank (they gain most), and add an integrity note: use only real, verifiable signals; never fabricate reviews, ratings, press logos, memberships, awards, or credentials.

### 6.4 Measurement note

Fold conversion-tracking setup into the Technical checklist (6.1): confirm the analytics tool actually present (do not assume a vendor; detect GA4, Google Analytics, GTM, or whatever the crawl shows), and define the real conversion events for the business model (for a lead-gen business: form submissions, calls, booking requests). This is what lets the forecast later be expressed in revenue.

Create the folder as needed: `mkdir -p ./{domain}/deliverables/`. Log generated deliverables in the activity-log entry.

---

## Integration with Other Skills

| Finding | Skill | When |
|---------|-------|------|
| Low CTR / bad meta tags | `/click-recovery` | Quick metadata fixes (GSC) |
| Outdated content | `/refresh-content {url}` | Full content refresh |
| Striking distance keywords (positions 4–30) | `/keywords-opportunity:striking` | GSC + volume data |
| Content gaps / new topics | `/keywords-opportunity:discover` | New keyword discovery |
| Low AEO scores on key pages | `/aeo-optimize {url}` | Add FAQ, schema, direct answers, question H2s |
| AI/LLM visibility baseline | `/ai-visibility` | Track citations across ChatGPT/Claude/Perplexity/AIO |
| Missing config | `/getting-started` | First-time setup |
| Quick pre-sale assessment | `/audit {url}` | Before connecting any MCP |
| Ongoing monitoring | `/weekly-report` | Track progress weekly |

**Engagement workflow:**
1. `/audit {url}` for pre-sale discovery (public signals).
2. `/audit:deep {url}` for the engagement baseline - runs on any site; deepens automatically with GSC/PageSpeed/CMS.
3. Quick wins with `/click-recovery` and `/refresh-content`.
4. `/weekly-report` to track and demonstrate progress.
5. Re-run `/audit:deep` quarterly to measure progress against this baseline.
