---
name: audit
version: "1.2"
description: |
  Quick SEO & AEO audit from a public URL. Pre-sale discovery tool that runs evidence-based checks across 4 dimensions (Content, Technical, Authority, Measurement) and reports each as a plain-language status with opportunities. Produces a client-ready report with opportunities, quick wins, and prioritized next steps — designed to present findings and prepare a proposal. No maturity score.
  Triggers: audit, site audit, seo audit, aeo audit, discovery audit.
  Requires: Web access (WebFetch). No MCP servers needed.
  Workflow: Parse → Prompt → Fetch → Check → Assess → Report → Save.
  Command: /audit {url}
---

# Quick Audit Skill

Pre-sale SEO and AEO audit from a public URL. Runs evidence-based checks on public signals across 4 dimensions, and outputs a client-ready report with prioritized opportunities and quick wins.

This report is designed to present to a client and prepare a proposal. It frames findings as opportunities, quantifies business impact, and leads naturally to an engagement recommendation.

This skill is **read-only** and requires no MCP servers — just a URL.

## Prerequisites

- **Required**: Web access (WebFetch tool)
- **No MCP servers needed** — quick audit runs on public signals only

## How findings are organized

No maturity score and no 1-to-5 levels. The audit reports four dimensions as a plain-language status and sorts findings into priority tiers:

- **Dimensions**: **Content**, **Technical**, **Authority**, **Measurement**. Each gets a qualitative status word, not a number: Strong / Solid / Needs work / At risk.
- **Priority tiers** for findings: Must fix, High impact, Nice to have, by observable criteria.

---

## Critical Guards

⚡ GUARD — **URL not accessible:**
- If WebFetch fails: "Could not fetch {url}. Check the URL is correct and publicly accessible."
- Stop execution.

⚡ GUARD — **Platform detection:**
- The audit is platform-agnostic — it runs on any public URL (Webflow, WordPress, Shopify, Framer, Next.js, a custom React/Vite build, or hand-coded HTML).
- Detect the platform from the homepage HTML for context only; never gate a check on it and never give platform-specific how-to in the report.

⚡ GUARD — **User requests abort:**
- Confirm, exit cleanly, output any partial results.

---

## Phase 0: PARSE & PROMPT

### 0.1 Parse URL & Set {domain}

Extract the domain from the provided URL. Normalize: strip `www.`, lowercase.
- `https://www.checklist-seo.com/blog/guide` → `checklist-seo.com`

The `{domain}` is used for:
- **Report save path**: `./{domain}/reports/audit-quick-YYYY-MM-DD.md`
- **Latest pointer**: `./{domain}/reports/latest-quick.md`
- **Activity log**: `./{domain}/reports/activity-log.md`

Determine the homepage URL from the domain (for sitemap and site-wide checks).

### 0.2 Review Activity Log

Check `./{domain}/reports/activity-log.md`:
- If it exists: show recent activity summary (last 10 entries)
- **Redundancy check**: if `/audit` was run in the last 7 days → warn: "You ran `/audit` on [date]. Run again anyway?"
- If not found: proceed silently

### 0.3 Prompt for Optional URLs

After parsing `{url}`, prompt:

```
Optional inputs (press Enter to skip each):

1. Topic pillar URL — a main topic page (e.g., /blog/seo-guide)
2. Blog post example URL — a typical blog article
```

Do not ask for credentials. Quick audit runs on public signals only.

---

## Phase 1: FETCH

Fetch these resources in a deterministic order:

| # | Resource | How | Required |
|---|----------|-----|----------|
| 1 | Home page HTML | WebFetch `https://{domain}/` | Yes — stop if fails |
| 2 | Pillar page HTML | WebFetch pillar URL (if provided) | No |
| 3 | Blog post HTML | WebFetch blog URL (if provided) | No |
| 4 | sitemap.xml | WebFetch `https://{domain}/sitemap.xml` | No — try `/sitemap_index.xml` as fallback |
| 5 | robots.txt | WebFetch `https://{domain}/robots.txt` | No |
| 6 | PageSpeed | Run PageSpeed via MCP on home URL (and pillar if provided) | No — skip if unavailable |

For each fetch, record: URL attempted, success/fail, HTTP status if available.

---

## Phase 2: CHECK

Run evidence-based checks on the fetched HTML. Every check produces:
- **Pass/Fail** (boolean)
- **Evidence** (exact snippet, attribute value, or count)
- **Source URL** (which page the evidence came from)

No heuristic language in pass/fail. Heuristics may only appear in recommendations.

**IMPORTANT:** Check IDs (C1, T4, etc.) are for internal skill logic only. They must NEVER appear in the client-facing report. The report uses plain language descriptions instead.

### Content Checks

| # | Check | Rule | Evidence |
|---|-------|------|----------|
| C1 | Title exists | `<title>` tag present and non-empty | Tag content |
| C2 | Exactly one H1 | Count of `<h1>` elements = 1 | Count found, text of H1(s) |
| C3 | Heading hierarchy | No skipped levels (H1→H2→H3, not H1→H3) | First violation if any |
| C4 | FAQ section present | Regex match for "FAQ", "Frequently Asked", or ≥3 question-pattern headings (`?` in H2/H3/H4) | Matched text |
| C5 | Word count ≥ 500 | Visible text word count (strip nav/footer/scripts) | Exact count |
| C6 | Word count ≥ 800 | Same as above, higher threshold | Exact count |
| C7 | Word count ≥ 2000 | Same, pillar-level threshold | Exact count |
| C8 | Internal links present | Count of `<a href>` pointing to same host | Count |
| C9 | Related posts pattern | Links/section matching "related", "you may also like", "more articles" | Matched pattern |
| C10 | Freshness date detected | `<time>` tag, "Last updated", "Published", or structured date pattern (YYYY-MM-DD) in article body | Date found |

### Technical Checks

| # | Check | Rule | Evidence |
|---|-------|------|----------|
| T1 | HTTPS active | URL scheme is `https` | URL |
| T2 | Viewport meta present | `<meta name="viewport">` exists | Tag content |
| T3 | Canonical link present | `<link rel="canonical">` exists | Href value |
| T4 | Open Graph tags present | `<meta property="og:title">` at minimum | Tags found |
| T5 | JSON-LD present | `<script type="application/ld+json">` exists | Schema types detected |
| T6 | Schema: Organization | JSON-LD contains `@type: Organization` | Snippet |
| T7 | Schema: Article | JSON-LD contains `@type: Article` (or subtypes) | Snippet |
| T8 | Schema: FAQPage | JSON-LD contains `@type: FAQPage` | Snippet |
| T9 | Schema: BreadcrumbList | JSON-LD contains `@type: BreadcrumbList` | Snippet |
| T10 | Schema: Person | JSON-LD contains `@type: Person` | Snippet |
| T11 | robots.txt accessible | Fetch succeeded, not empty | Content summary |
| T12 | robots.txt not blocking key paths | No `Disallow: /` for major user agents | Disallow rules found |
| T13 | Sitemap accessible | sitemap.xml fetched and parseable | URL count if available |
| T14 | Clean URL structure | No query-string parameters on core pages (home, pillar, blog) | URLs checked |

### Authority Checks

| # | Check | Rule | Evidence |
|---|-------|------|----------|
| A1 | Author byline present | Text matching "By [Name]", `<span class="author">`, or similar patterns on blog post (if provided) or homepage | Matched text |
| A2 | About/team page discoverable | Internal link containing "about", "team", or "company" in href or text | Link found |
| A3 | Author bio markers | Credentials, role, or experience mentions near author name ("CEO", "years of experience", "certified") | Matched text |
| A4 | External citations present | Outbound links to non-social, non-navigation domains on content pages | Count + domains |

### Measurement Checks

| # | Check | Rule | Evidence |
|---|-------|------|----------|
| M1 | GA4 detected | Script containing `gtag(` or GA measurement ID pattern (`G-XXXXXXX`) | Script snippet |
| M2 | GTM detected | Script containing `GTM-` container ID | Container ID |
| M3 | GSC verification | `<meta name="google-site-verification">` present | Tag content |
| M4 | Other analytics tools | Patterns for PostHog (`posthog`), Segment (`analytics.js`), Hotjar, Mixpanel, Plausible, Fathom | Tools detected |

**M3 limitation:** This check only detects HTML meta tag verification. Domain-level GSC properties use DNS TXT records, which are not visible in page HTML. If M3 fails, note in the report: "GSC verification not detected via HTML. If verified via DNS (domain-level property), this check does not apply." Do not penalize the score when there is reason to believe DNS verification is in use (e.g., analytics scripts suggest GSC awareness, or client confirms domain-level property).

---

## Phase 3: ASSESS

No maturity score and no levels. Turn the checks into a per-dimension status and a prioritized findings list.

### Dimension status

For each dimension, weigh its checks and assign one qualitative status word (judgement, not arithmetic):

| Status | When |
|--------|------|
| Strong | Few or no gaps; an asset to build on |
| Solid | Working, with clear room to improve |
| Needs work | Real gaps holding back results |
| At risk | A blocker actively costing traffic or trust |

Guidance per dimension:
- **Content**: title + single H1 + clean hierarchy + FAQ/answer structure + real depth leans Strong; missing FAQ or thin content is Needs work; no title or no H1 is At risk.
- **Technical**: HTTPS + viewport + canonical + OG + schema breadth (Organization, Article, Breadcrumb) leans Strong; missing canonical/OG or no JSON-LD is Needs work; a foreign or broken canonical, or blocked indexing, is At risk.
- **Authority**: author byline + about/credentials + external citations leans Strong; no author entity is Needs work.
- **Measurement**: analytics installed plus GSC awareness is Solid to Strong; nothing detected is Needs work. (See the M3 limitation note: do not mark down for a DNS-verified GSC property.)

Name the strongest and the weakest dimension.

### Confidence

Always "Low-Medium" in quick mode (public signals only). State it in the report.

### Priority tiers

Sort findings into Must fix / High impact / Nice to have using the observable criteria in the shared Priority Buckets convention. Do not compute a numeric score.

---

## Phase 4: REPORT & SAVE

### 4.1 Report Principles

The report is a **client-facing deliverable** designed to present findings and prepare a proposal. Follow these rules:

1. **No check IDs** — Never write C1, T4, M3, etc. in the report. Use plain language: "page title", "Open Graph tags", "search console verification".
2. **Opportunities, not failures** — Frame findings as untapped potential, not broken things. "You're leaving X on the table" not "X is broken".
3. **No implementation details** — Say **what** needs attention and **why** it matters. Never say **how** to fix it (e.g., don't write "In Webflow Designer, go to Page Settings → Open Graph"). The how is the work you sell.
4. **Business language** — Traffic, visibility, clicks, rankings, competitive advantage. Not "pass/fail", "boolean", "gates".
5. **Lead to engagement** — The report should make the client want to work with you. End with a clear next step.

### 4.2 Report Structure

Output a single markdown file with these sections in order:

---

**Header:**
```
# SEO and AEO Audit — {domain}

**Prepared for:** {domain}
**Date:** YYYY-MM-DD
**Assessment type:** Discovery (public signals)
**Confidence:** Low–Medium (based on publicly available data)
**Pages analyzed:** [list URLs checked]
```

---

**1. About This Assessment**

Brief methodology section (3-4 sentences):
- What was analyzed (homepage, pillar page if provided, blog post if provided, sitemap, robots.txt)
- How findings are organized (four dimensions reported as a plain-language status, findings sorted into priority tiers, evidence-based; no maturity score)
- What this assessment covers and what it doesn't (public signals only — no search analytics, no CMS data, no traffic numbers)
- Confidence caveat: "A deep audit with search analytics and CMS access would increase confidence and reveal data-backed opportunities."

---

**2. Executive Summary**

3-5 sentences that a decision-maker can read and understand immediately:
- The single biggest opportunity (framed as upside, not problem)
- What's already strong (give credit — builds trust)
- The one or two things most worth fixing first and what they would unlock
- Do not assign a maturity level or score

---

**3. Dimension Summary**

A one-line read on where the site stands overall, then a table with NO check IDs and NO scores or levels:

| Dimension | Status | What's Working | Biggest Opportunity |
|-----------|--------|----------------|---------------------|
| Content | Strong / Solid / Needs work / At risk | [strength] | [opportunity] |
| Technical | ... | [strength] | [opportunity] |
| Authority | ... | [strength] | [opportunity] |
| Measurement | ... | [strength] | [opportunity] |

Below the table:
- Strongest dimension and why it matters
- Weakest dimension and what it's costing

---

**4. Key Findings**

For each dimension, write a **narrative paragraph** (not a pass/fail table):
- Lead with what's working (builds credibility, shows you're fair)
- Then describe what's missing and why it matters in business terms
- Use evidence (quotes, counts, schema types found) but never check IDs
- End each dimension with its biggest opportunity (the outcome, not the how-to)

Example tone: "The site has a strong FAQ section with 11 question-answer pairs and proper schema markup — this is exactly what AI answer engines look for when selecting sources. However, there's no mechanism for visitors to discover related content, which means each page visit is a dead end instead of a journey deeper into the site."

---

**5. Opportunities & Business Impact**

Table of the top 3-5 opportunities, framed as upside:

| # | Opportunity | Impact | What You're Leaving on the Table |
|---|-------------|--------|----------------------------------|
| 1 | [opportunity] | High/Medium/Low | [business consequence in plain language] |

Focus on:
- Traffic and visibility impact
- Competitive disadvantage
- AI/AEO readiness gaps
- User experience consequences

Do NOT include how to fix these — that's the engagement.

---

**6. Quick Wins**

2-3 items that can show visible results within the first week of an engagement. These build client confidence that progress is immediate.

For each quick win:
- What it is (outcome, not implementation)
- Why it matters (business impact)
- Effort level: Low (hours, not days)

Frame as: "These are the first things we'd address — you'll see changes within days, not months."

---

**7. Priorities**

The ordered action plan from the findings, grouped by priority tier (no levels).

| Priority | Dimension | What to do (outcome, not how-to) | Effort |
|----------|-----------|----------------------------------|--------|
| Must fix | [dim] | [outcome description] | Low/Medium/High |
| High impact | [dim] | [outcome description] | Low/Medium/High |
| Nice to have | [dim] | [outcome description] | Low/Medium/High |

Below the table: a brief narrative on what addressing the top priorities means for the business (more visibility, better AI coverage, competitive positioning).

---

**8. Recommended Next Steps**

This is the proposal bridge. 2-3 clear options:

**Option A: Deep Audit** (recommended)
- "A deep audit crawls a representative set of your live pages and runs the full check set across them — works on any platform, no setup required. If you can also connect search analytics (GSC), it quantifies these opportunities with traffic data, identifies content gaps, and builds a prioritized engagement plan with ROI estimates."

**Option B: Quick Wins First**
- "Start with the quick wins identified above, then reassess. This is a good option if you want to see results before committing to a larger engagement."

**Option C: Full Engagement**
- "Address all opportunities in a structured roadmap — quick wins in week 1, strategic improvements over 4-8 weeks, with weekly progress tracking."

End with a single call-to-action line.

---

### 4.3 Save to File

Save two files:
- `./{domain}/reports/audit-quick-YYYY-MM-DD.md` (timestamped)
- `./{domain}/reports/latest-quick.md` (overwrite each run — other skills read this)

Create directories if needed: `mkdir -p ./{domain}/reports/`

### 4.4 Activity Log

Append to `./{domain}/reports/activity-log.md`:

```
| YYYY-MM-DD | /audit | Quick audit. Dimensions - Content: {status}, Technical: {status}, Authority: {status}, Measurement: {status}. Must-fix: N. |
```

Log even on early exit (e.g., "Aborted: URL not accessible").

---

## Integration with Other Skills

| Finding | Skill | When |
|---------|-------|------|
| Need data-backed audit | `/audit:deep` | Multi-page crawl on any site; deepens with GSC/PageSpeed/CMS |
| Low CTR / bad meta tags | `/click-recovery` | After connecting GSC |
| Outdated content | `/refresh-content {url}` | For specific pages |
| Missing config | `/getting-started` | First-time setup |
| Ongoing monitoring | `/weekly-report` | After fixing issues |
