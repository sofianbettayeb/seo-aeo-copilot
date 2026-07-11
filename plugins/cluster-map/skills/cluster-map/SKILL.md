---
name: cluster-map
version: "1.0"
description: |
  Render a keyword strategy as a visual site-structure tree — one self-contained, PDF-ready HTML file showing the domain at the root, the nav-level money pages, and every topic cluster as a branch (pillar → supporting pages) with volumes, difficulty, phase chips, and a CTA arrow from each cluster to the money page it feeds.
  Triggers: cluster map, keyword cluster map, cluster illustration, visualize clusters, visualize the keyword strategy, site structure map, topic cluster diagram, keyword map visual, illustrate the strategy, show me the site structure.
  Requires: a keyword strategy report (from /keyword-strategy) or a pasted cluster list. Content briefs (from /content-brief) deepen it with slugs, decisions, and week-level timing. No MCP needed.
  Workflow: Discover → Intake → Structure → Render → Verify → Save.
  Modes: /cluster-map (latest strategy for the current domain), /cluster-map {domain} (pick which site's reports to use).
---

# Cluster Map

`/keyword-strategy` produces the plan as tables; `/content-brief` produces the execution detail. This skill turns both into the picture a client or writer actually understands in ten seconds: **which pages exist or will exist, where they live in the site, which cluster they belong to, and which money page every cluster feeds.**

The output is a single HTML file: a site tree, not a card grid. Hierarchy is the whole point — a user looking at cards asked "I don't see the relationship between the nav and the cluster." The tree answers it structurally:

```
domain
├── money pages (nav level: home, feature pages, pricing, app store listing)
└── /blog (or the site's content root)
    ├── CLUSTER 1 [phase chip]
    │   └── pillar page (dark node)  · CTA → money page
    │       ├── supporting page (blue edge)
    │       └── keywords served as sections (muted note, NOT a page node)
    └── CLUSTER n …
```

**This skill is read-only on the site** — it renders a deliverable, it changes nothing.

---

## Conditional Guards

⚡ GUARD — **No strategy source:** if neither `.claude/reports/{domain}/latest-keyword-strategy.md` nor a pasted cluster list exists, say so and point to `/keyword-strategy {url}` first, or accept a pasted list (cluster name, pillar keyword + volume, supports, phase). Wait for the user.

⚡ GUARD — **Branding:** load `.claude/seo-copilot-config.json` `brandVoice.*` if present. If the user has a standing brand default, apply it; otherwise use the neutral palette in the template and note once that colors can be swapped. Never dark-background — this is a document to read and print.

⚡ GUARD — **User requests abort:** confirm, output the partial file if any.

---

## Phase 0: DISCOVER

1. Set `{domain}` from the argument, config `business.siteUrl`, or the most recent `.claude/reports/*/latest-keyword-strategy.md`.
2. Check the activity log for a recent `/cluster-map` run — if the strategy hasn't changed since, offer to just reopen the existing file.

## Phase 1: INTAKE

Read, in order of preference:
1. `.claude/reports/{domain}/latest-keyword-strategy.md` — clusters, pillar keywords, volumes, difficulty, phases, deliberately-avoided keywords
2. `.claude/reports/{domain}/latest-content-briefs.md` (if present) — real slugs, CREATE/REWORK/RETARGET decisions, week-level timing, CTA targets, keywords-as-sections decisions
3. The site's sitemap or the strategy's inventory — which money pages exist at nav level

Normalize to a tree model: `{domain, money_pages[{url, note, receives_from[], highlight?, phase?}], content_root, clusters[{id, label, phase, pillar{slug, keyword, vol, kd, action, timing, cta_target}, supports[{slug_or_note, vol, phase}], deferred?}]}`.

## Phase 2: STRUCTURE (the rules that make the map honest)

1. **Money pages sit at nav level, clusters under the content root.** External surfaces the strategy treats as owned (an App Store listing, a marketplace profile) appear at nav level, marked external.
2. **One dark pillar node per cluster.** Volume + difficulty on the node; action + timing as a note beside it.
3. **Supports are only pages.** A keyword served as a *section* of the pillar renders as one muted note-node ("X (vol) — sections inside the pillar"), never as a page node. Otherwise the map promises pages nobody planned.
4. **Every pillar carries `CTA → {money page}`** and every money page that receives clusters carries `← receives the CTA from {cluster ids}`. This is the nav-to-cluster relationship, stated twice so it can't be missed.
5. **Phase chips on clusters AND on later-phase supports.** Deferred/future clusters render grayed and dashed with a one-line condition for unlocking them.
6. **Existing pages keep their real slugs**; new pages get the brief's proposed slug; reworks/retargets say so ("existing — retitled…", "retargeted → {keyword}").
7. **A legend** (phase chips, dark = pillar, blue edge = support, CTA arrow) **and a "how to read this" footnote** with the internal-linking rules the strategy prescribes.

## Phase 3: RENDER

Produce ONE self-contained HTML file from the template below. Hard rules:
- Light/white background, ink-friendly; `@page` print CSS (A4 portrait) + `print-color-adjust: exact` so it exports via Cmd+P
- No external assets — no CDN fonts, no images, no scripts
- En dashes only, never em dashes; no emojis
- Node text stays short; anything longer than ~8 words becomes a `.note` beside the node, not text inside it (long inline text wraps badly against the tree connectors)

The tree connector CSS (the fiddly part — reuse, don't reinvent):

```css
ul.tree, .tree ul { list-style: none; padding-left: 26px; }
.tree li { position: relative; padding: 4px 0 4px 22px; }
.tree li::before { content: ""; position: absolute; left: 0; top: 0;
  border-left: 2px solid var(--line); height: 100%; }
.tree li:last-child::before { height: 21px; }   /* stop the rail at the elbow */
.tree li::after { content: ""; position: absolute; left: 0; top: 21px;
  width: 18px; border-top: 2px solid var(--line); }
.tree > li { padding-left: 0; }
.tree > li::before, .tree > li::after { display: none; }
```

Node classes: `.node` (base box) · `.root-node` (dark, domain) · `.money` (2px dark border, soft bg) · `.money.hot` (accent border, highlighted surfaces) · `.pillar` (dark filled) · `.support` (4px blue left edge) · `.node.muted` / `.future` (grayed, dashed) · `.chip.p1/.p2/.p3` (phase) · `.cta` (accent, bold) · `.note`, `.receives` (gray annotations) · `.cluster-label` (uppercase cluster heading).

Neutral palette (swap for the client's brand when one is set): `--dark: #1f2d3d; --accent: #b3132f; --accent-bright: #e11d48; --blue: #2f80c3; --gray: #6b7684; --line: #c9d2da; --bg-soft: #f5f7f9;` — body max-width ~980px, base font 13.5px, system-safe font stack.

## Phase 4: VERIFY (never ship unrendered HTML)

1. Serve the file locally (`python3 -m http.server {port}` in the output dir, background) — `file://` URLs don't load in the app browser.
2. Open it in the preview browser, screenshot, and check: connectors render at every level, no text wraps underneath the tree rails, chips and CTA arrows legible, nothing overflows.
3. Fix and re-check until clean. Kill the temp server.
4. Open the file for the user in their default browser (`open {path}` on macOS, or their stated preference).

## Phase 5: SAVE

- `.claude/reports/{domain}/keyword-cluster-map-YYYY-MM-DD.html` — timestamped
- `.claude/reports/{domain}/latest-keyword-cluster-map.html` — overwrite each run
- If the engagement has a client workspace/deliverables folder, copy it there too.

Print the path and a 2–3 line read of the map (how many clusters, which phase is live, which money page carries the most clusters).

---

## Integration with Other Skills

| Situation | Skill |
|-----------|-------|
| No strategy to visualize yet | `/keyword-strategy {url}` |
| Slugs/decisions/timing missing from the map | `/content-brief` first — the map is only as concrete as the briefs |
| Turn the map into a branded client report page | `/deliverables` |
| Verify the built cluster structure once pages ship | `/topic-map` |

Re-render whenever the strategy or briefs change — it's cheap.

---

## Error Handling

| Error | Action |
|-------|--------|
| No strategy report and nothing pasted | Guard fires — stop, point to `/keyword-strategy`. |
| Strategy exists but has no clusters section | Fall back to its keyword tables grouped by root phrase; say the clustering is inferred. |
| Briefs missing | Render from strategy alone; slugs marked as proposed, timing at phase level only. |
| Local server port busy | Try another port. |
| Preview browser can't load the page | Skip the screenshot check, open directly in the system browser, and say verification was visual-only. |

---

## Activity Log

Append to `.claude/reports/{domain}/activity-log.md` (create with the standard header if missing):

```
| YYYY-MM-DD | /cluster-map | Rendered {N} clusters ({N} P1 / {N} P2 / {N} P3) over {N} money pages. Saved keyword-cluster-map-YYYY-MM-DD.html. |
```

Log even on early exits, with the reason.
