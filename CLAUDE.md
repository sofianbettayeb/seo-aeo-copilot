# SEO/AEO Copilot — Global Instructions

Platform-agnostic SEO & AEO skills. These work on **any** site or stack — Webflow, WordPress, Shopify, Framer, Next.js, a custom React/Vite build, or hand-coded static HTML. No MCP server is required; connected tools (GSC, PageSpeed, a CMS MCP, Keywords Everywhere) deepen the analysis when present.

---

## Shared Skill Conventions

All skills follow these conventions. Each skill references them instead of duplicating the logic.

### MCP Discovery (run once at skill start)

**All MCPs are optional.** Search for what's available and adapt scope — never stop execution because a tool is missing (the one exception: `/click-recovery` needs GSC to find CTR opportunities; it says so explicitly and offers a single-URL fallback).

```
GSC MCP             → search: +gsc search analytics      (search-analytics findings + higher confidence)
PageSpeed MCP       → search: +pagespeed performance     (Core Web Vitals)
CMS MCP (any)       → search: +webflow data cms          (read CMS as source of truth, if the site uses one)
Keywords Everywhere → search: +keywords everywhere volume (volume/intent enrichment)
```

When an optional tool is absent, fall back to the universal backbone — a **live multi-page crawl** (sitemap + WebFetch) and/or the **local source files** when the skill is run inside the site's repo — and note the reduced scope once.

### Where changes get applied

Skills that change content (`/refresh-content`, `/write-blog`, `/aeo-optimize`, `/click-recovery`) operate in one of two modes, detected automatically:

- **Repo mode** — when run inside the site's codebase, locate the source file for the target page (markdown/MDX front-matter, a `.tsx`/`.jsx`/`.vue`/`.astro` component, static HTML, or a head/SEO config), **edit it directly, and show a before/after diff for approval before writing.** Respect the project's existing format and conventions (detect them by reading neighboring files).
- **Read-only mode** — when only a public URL is available, **output the exact changes** (rewritten copy, meta tags, JSON-LD blocks) for the user to paste in. Never edit files you can't see.

When a CMS MCP is connected and matches the site, applying via that MCP is also acceptable — treat it as one more "apply" target.

### Sitemap maintenance

When a skill adds a new public page/article, remind the user (or do it, in repo mode) to update the sitemap source and, if GSC is connected, resubmit via `gsc_submit_sitemap`. A "new page" task isn't complete until the sitemap reflects it.

---

### Config Loading (run once at skill start)

Load `.claude/seo-copilot-config.json` if it exists. Extract what the skill needs — each skill specifies which fields it uses. Useful fields: `business.siteUrl`, `business.platform`, `project.repoPath`, `project.contentDir`, `brandVoice.*`, `seo.*`, `aeo.*`. If the file doesn't exist, proceed with defaults and note once: "No config found. Run `/getting-started` to personalize recommendations."

Do not block execution on missing config.

---

### Priority Buckets (use instead of a scoring formula)

Three buckets with observable, binary criteria. Do not compute a numeric score.

**Must do** — at least one of:
- A page in the sitemap is returning a 404
- GSC confirms the same keyword ranking on 2+ pages simultaneously
- A page getting impressions has no `<title>` tag
- An indexation error on a page with existing traffic
- A cannibalization pair confirmed by GSC data (not inferred)

**High value** — at least one of:
- A keyword cluster with 50+ monthly impressions (or top 20% of the site's keyword set) has no matching page
- A page's CTR is less than half the expected rate for its average position
- The primary keyword for a page doesn't appear in its title
- A cluster has 2+ support posts but no pillar page
- A page that exists in the site architecture is missing from GSC entirely

**Nice to have** — everything else (coverage gaps, metadata length issues with no confirmed traffic impact, orphan pages, thin programmatic pages).

Items that don't meet "Nice to have" criteria are excluded from the action plan.

---

### Activity Log (run at skill end)

After every skill execution, append to `.claude/reports/{domain}/activity-log.md`.

If the file doesn't exist, create it:
```markdown
# Activity Log — {domain}

| Date | Skill | Summary |
|------|-------|---------|
```

Append one row:
```
| YYYY-MM-DD | /skill-name | [one-line summary of what was analyzed, changed, or applied] |
```

Log even on early exits — note the reason (e.g., "Aborted: GSC MCP unavailable").

---

### Abort Guard (applies to all skills)

If the user says "stop", "cancel", "abort", or "nevermind":
- Confirm: "Stop the workflow? Progress will be lost."
- If confirmed: exit cleanly, output any partial results already computed.

---

### Output Format Guidance

Skills specify the **intent** of each output section, not the exact format. Adapt to the data available:
- If a table would have more than half its cells empty, use a list instead
- If there are fewer than 3 items in a section, collapse it into a sentence
- Never output a section header with nothing under it
- Prefer specificity over completeness — one concrete finding beats five vague ones

---

## Activity Logging (ad-hoc sessions)

For work done outside a named skill, append a row using `ad-hoc` as the skill name. Log any session where you analyzed the site's SEO/AEO/content, changed pages or content files, applied recommendations, or ran a GSC query. Do **not** log purely informational sessions with no action taken.
