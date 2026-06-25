---
name: content-model
version: "2.0"
description: |
  Review or scaffold an SEO/AEO content model on any stack. Detects how content is structured today — markdown/MDX front-matter (Next.js, Vite, Astro blogs), a code-defined content type (TS interface / schema file), or a connected headless/traditional CMS (Contentful, Sanity, WordPress, Webflow) — scores it field-by-field against a recommended five-group schema with weighted scoring, and lists what to add. Or proposes a complete content model for the chosen system: a front-matter template + matching JSON-LD, a TS type + example, or CMS fields. Local files are the backbone; a CMS MCP is optional.
  Triggers: content model, content schema, front-matter schema, seo fields, blog schema, cms fields, review content model, create content model, content type audit, frontmatter keys, jsonld fields.
  Requires: A repo path (markdown/MDX articles, a schema/type file) OR a connected CMS MCP. All MCPs optional — local files are the backbone, no MCP required.
  Command: /content-model — with :review (audit only) and :create (scaffold only) variants.
  Workflow: Detect system → Choose mode → Review or Scaffold → Confirm.
---

# Content Model Skill

Review an existing content model against the recommended SEO/AEO schema — or scaffold a new one from scratch with all the fields that matter for rankings, authority, and sharing. Platform-agnostic: works whether your articles live in markdown front-matter, a TypeScript content type, a headless CMS, or a traditional CMS like WordPress or Webflow.

> **Renamed from `cms-collection-setup`.** The Webflow-only original audited/created a Webflow CMS collection. This is the platform-agnostic analog: the same five-group field model and weighted scoring, generalized to any content system. "Collection field" becomes "front-matter key", "schema field", or "content-type field" depending on where your content lives.

## Why the Schema Matters

Most blogs are missing fields that unlock real SEO/AEO leverage: a dedicated H1 override, OG image, primary keyword, pillar-page link, structured FAQ. Without them, `/refresh-content` can't populate them, `/click-recovery` can't read meta fields, and JSON-LD schema markup has no source data to pull from.

The field model below is the valuable core. The "where these fields live" layer is what changes between stacks — front-matter keys, a TS type, or CMS fields — but the fields and their weights are constant.

This skill fixes the foundation.

---

## Recommended Schema

The target schema covers five groups. Every field has a purpose — nothing decorative. The middle column gives the **abstract field**; the right column shows how it surfaces in each system.

### Core (weight 30%)
Fields that control how the article is found and indexed.

| Field | Markdown front-matter key | Code content type | Headless / traditional CMS |
|-------|---------------------------|-------------------|----------------------------|
| Title / Name | `title` | `title: string` | Built-in name/title field |
| Slug | `slug` (or derived from filename) | `slug: string` | Built-in slug field |
| H1 override | `h1` | `h1?: string` | PlainText field — decouples page H1 from title |
| Meta title | `metaTitle` / `seoTitle` | `metaTitle: string` | PlainText. ≤60 chars. Shown in SERP. |
| Meta description | `metaDescription` / `description` | `metaDescription: string` | PlainText. ≤155 chars. Shown in SERP. |

### Content (weight 25%)
Fields that control what appears on the page.

| Field | Markdown front-matter key | Code content type | Headless / traditional CMS |
|-------|---------------------------|-------------------|----------------------------|
| Body | the markdown body itself / `body` | `body` / MDX | RichText |
| Main image | `image` / `cover` / `hero` | `image: string` | Image field |
| Image alt | `imageAlt` / `coverAlt` | `imageAlt: string` | PlainText (alt for main image) |
| Reading time | `readingTime` | `readingTime?: number` | Number (minutes — computed or set) |
| Updated date | `updated` / `lastModified` | `updated?: string` | DateTime (shown to readers + used in schema) |

### Authority (weight 20%)
Fields that build trust signals and topical structure.

| Field | Markdown front-matter key | Code content type | Headless / traditional CMS |
|-------|---------------------------|-------------------|----------------------------|
| Author | `author` / `authors` | `author: string \| string[]` | Reference/MultiReference to Authors |
| Category | `category` | `category: string` | Reference to Categories |
| Tags | `tags` (list) | `tags: string[]` | Option list or MultiReference to Tags |
| Pillar page | `pillar` | `pillar?: string` | Reference to the pillar article this supports |

### Enhancement (weight 15%)
Fields that power SEO/AEO tooling and internal linking.

| Field | Markdown front-matter key | Code content type | Headless / traditional CMS |
|-------|---------------------------|-------------------|----------------------------|
| Primary keyword | `primaryKeyword` / `keyword` | `primaryKeyword: string` | PlainText — the one keyword this article targets |
| Secondary keywords | `secondaryKeywords` (list) | `secondaryKeywords: string[]` | PlainText (comma-separated) or list |
| Related posts | `related` (list of slugs) | `related?: string[]` | MultiReference to other items |
| FAQ | `faq` (list of `{q, a}`) | `faq?: {q: string; a: string}[]` | MultiReference to FAQ items, or inline RichText |

### Social (weight 10%)
Fields that control how the article looks when shared.

| Field | Markdown front-matter key | Code content type | Headless / traditional CMS |
|-------|---------------------------|-------------------|----------------------------|
| OG title | `ogTitle` | `ogTitle?: string` | PlainText (can differ from meta title) |
| OG description | `ogDescription` | `ogDescription?: string` | PlainText. ≤200 chars. |
| OG image | `ogImage` | `ogImage?: string` | Image. Recommended 1200×630px (1.91:1). |

> **JSON-LD note (markdown/code stacks):** Several of these fields also surface in the rendered page's structured data. When the body renders `Article` (or `BlogPosting`) + `FAQPage` JSON-LD, it should pull `headline` from title/H1, `description` from meta description, `image`/`ogImage`, `author`, `dateModified` from updated date, and `mainEntity` from the FAQ list. A field "exists" for scoring purposes when it's present in the front-matter/type **or** demonstrably emitted in JSON-LD.

---

## Prerequisites

- **Backbone (always)**: a repo path containing markdown/MDX articles, a content schema/type file, or both. See `references/detection.md` for how each system is detected.
- **Optional**: a connected CMS MCP (Webflow, or any headless CMS exposed via MCP) if the content lives in a hosted CMS rather than in the repo.

No MCP is required. If no repo and no CMS MCP are available, the skill still runs in advisory mode using the recommended schema and a generic template.

See the package shared `CLAUDE.md` for MCP discovery, config loading (`.claude/seo-copilot-config.json`), and activity-log conventions — this skill follows them rather than redefining them.

---

## Skill Modes

| Mode | Command | Use Case |
|------|---------|----------|
| **Full** | `/content-model` | Guided: detect system, then choose review or create at runtime |
| **Review** | `/content-model:review` | Audit the existing content model against the recommended schema |
| **Create** | `/content-model:create` | Scaffold a new content model (template + JSON-LD, TS type, or CMS fields) |

---

## Conditional Guards (Global)

⚡ GUARD — **Load SEO Copilot config:**
At the start of execution, check if `.claude/seo-copilot-config.json` exists:
- If yes: load settings — `business.siteUrl` / domain helps identify the site for logging
- If no: proceed silently. No config is needed to run this skill.

⚡ GUARD — **No detectable content model:**
If no markdown/MDX articles, no schema/type file, and no connected CMS MCP can be found:
- Don't stop. Switch to advisory mode: present the recommended schema and a generic front-matter template, and ask the user where their content lives so detection can be retried.

⚡ GUARD — **User requests abort:**
If user says "stop", "cancel", "abort", or "nevermind" at any phase:
- Confirm: "Stop the workflow?"
- If confirmed: exit cleanly. In create mode, note which files/fields were already written.

---

## Phase 0: Detect the Content System

Before anything else, figure out where the content model lives. Follow the detection ladder in `references/detection.md`:

1. **Markdown / MDX** — Glob for article files (common roots: `client/src/blog/posts/*.md`, `server/content/articles/*.md`, `content/**/*.{md,mdx}`, `src/content/**/*.{md,mdx}`, `posts/*.md`, `app/blog/**/*.mdx`). If found, Read 3–8 representative files and parse the front-matter block. This is the most common case.
2. **Code-defined content type** — Glob for a schema/type file (e.g. `**/content.config.ts`, `**/contentlayer.config.*`, `**/*frontmatter*.ts`, `**/types/*post*.ts`, Astro `src/content/config.ts`, Sanity `schemas/*.{js,ts}`). Read it and extract the declared fields.
3. **Connected CMS MCP** — if a CMS MCP is available (Webflow `data_cms_tool`/`collections_list`, or another), list collections/content types and fetch the field list of the chosen one.
4. **None** — advisory mode (see guard above).

Set `{system}` = `markdown` | `code` | `cms` | `none`, and capture `{domain}` for logging (from config, the site URL, the repo, or the CMS).

### 0.1 Choose Mode (full mode only)

If running `/content-model` (no sub-mode):

```
Detected content system: {system}  (e.g. "markdown front-matter — 47 posts in client/src/blog/posts")

What would you like to do?

1. Review — score the current content model against the recommended SEO/AEO schema
2. Create — scaffold a complete content model for this system
```

- If **Review**: go to Phase 1R
- If **Create**: go to Phase 1C

---

## Phase 1R: Review — Gather the Current Model

### 1R.1 Collect the field set

- **markdown**: union the front-matter keys across the sampled files. A key counts as "present" if it appears in the majority of sampled posts (note keys that appear in only some — partial coverage). Also scan the rendering layer (layout/template/component) for JSON-LD emission to credit fields surfaced there.
- **code**: list the fields declared in the content type / schema, with their types and optional/required status.
- **cms**: fetch the collection/content-type field list (slugs + types) via the CMS MCP.

Record, per recommended field, what was found and where.

---

## Phase 2R: Review — Audit Against Schema

### 2R.1 Map Existing Fields

For each field in the recommended schema, attempt to find a match. Use `references/field-aliases.md` for the alias lists.

**Matching rules (in order of confidence):**

1. **Exact match** — front-matter key / type field / CMS slug matches the canonical name (e.g. `metaTitle`, `meta-seo-title`).
2. **Alias match** — a common alternative name maps to the same purpose (e.g. `description` → meta description; `cover` → main image; `lastmod` → updated date). See `references/field-aliases.md`.
3. **JSON-LD / render inference** (markdown/code only) — the field isn't in front-matter but is demonstrably emitted in the page's JSON-LD or template (e.g. `dateModified` rendered from file mtime). Count as a half-match and flag it as "computed/rendered, not authored".
4. **Type inference** (code/cms) — no name match, but a field of the right type sits unassigned.
5. **No match** — field is missing.

### 2R.2 Check Type / Shape Compatibility

For each matched field, verify the type/shape is sensible:

| Expected shape | OK | Flag ⚠️ |
|----------------|-----|---------|
| string | string | list, number |
| list | array / comma-separated string | single scalar where a list is expected |
| image ref | string path / asset ref / Image field | rich text blob |
| number (reading time) | number | string that isn't numeric |
| date (updated) | ISO date / DateTime | freeform text |
| reference (author/pillar/related/FAQ) | slug, ref, or MultiReference | unmatched scalar |

Flag shape mismatches as ⚠️ (exists but wrong shape — won't auto-populate cleanly).

### 2R.3 Score Coverage

Per group:

```
Group score = (exact matches + 0.5 × (alias matches + rendered/inferred matches)) / total fields in group × 100
```

Overall weighted score uses the group weights:

| Group | Weight |
|-------|--------|
| Core | 30% |
| Content | 25% |
| Authority | 20% |
| Enhancement | 15% |
| Social | 10% |

```
Overall = 0.30·Core + 0.25·Content + 0.20·Authority + 0.15·Enhancement + 0.10·Social
```

### 2R.4 Output Audit Report

```
# Content Model Audit — {site / collection / repo}

**System**: {markdown | code | cms}  ({where — e.g. "front-matter, 47 posts in client/src/blog/posts"})
**Overall SEO/AEO Schema Score**: [X]% ([rating])

Rating scale:
- 90–100% → Excellent — fully modeled
- 70–89%  → Good — minor gaps
- 50–69%  → Needs work — key fields missing
- <50%    → Foundation gaps — significant SEO/AEO risk

---

## Core Fields (30% weight) — Score: X%

| Recommended Field | Status | Found As | Shape OK? |
|-------------------|--------|----------|-----------|
| Title | ✅ Exact | title | ✅ string |
| Slug | ✅ Exact | (filename) | ✅ |
| H1 override | ❌ Missing | — | — |
| Meta title | ⚠️ Alias | seoTitle | ✅ string |
| Meta description | ✅ Exact | description | ✅ string |

## Content Fields (25% weight) — Score: X%
[same table format]

## Authority Fields (20% weight) — Score: X%
[same table — for reference-type fields, note the target collection / how related items are linked]

## Enhancement Fields (15% weight) — Score: X%
[same table format]

## Social Fields (10% weight) — Score: X%
[same table format]

---

## What to Add

Missing fields, by priority. Phrasing adapts to the system:
- **markdown** → "add front-matter key `X`" (+ "emit `Y` in the Article JSON-LD")
- **code** → "add `X: type` to the content type"
- **cms** → "add an `X` field (Type: …)"

### Must Add (Core)
- **H1 override** — Decouples the page H1 from the title for keyword optimization.
  - markdown: add `h1:` to front-matter; code: `h1?: string`; cms: PlainText field.
- **Meta title** — Required for SERP optimization. Currently falling back to `title`, which causes keyword mismatches.

### Should Add (Content / Authority)
- **Reading time** — Trust signal. Compute from word count or set manually.
- **Pillar page** — Powers internal-linking strategy. (markdown: `pillar:` slug; cms: Reference → suggest existing collections.)

### Nice to Add (Enhancement / Social)
- **Primary keyword** — Lets `/refresh-content` target keywords per article.
- **OG image** — Controls social-share appearance. Without it, platforms grab any image on the page.

---

## Fields Present, Not in Schema

| Field | Found As | Shape | Suggestion |
|-------|----------|-------|------------|
| draft | draft | bool | Keep — useful, not part of SEO schema |
| date | date | date | Can double as the "Updated date" source if no separate field |

---

## Next Steps

1. Add the missing fields: run `/content-model:create` and choose "Add missing fields to the existing model"
2. Or add them by hand (front-matter keys / type fields / CMS fields)
3. Once fields exist, run `/refresh-content [URL or path]` to populate them for an article
```

### 2R.5 Offer to Apply

After the report, ask:

```
Want me to add the missing fields now?

1. Yes — add all missing fields
2. Yes — let me choose which
3. No — I'll do it manually
```

If 1 or 2: jump into Phase 2C field/template generation, scoped to just the missing fields. For markdown, this means proposing front-matter additions (and a JSON-LD patch) on the existing template/example file. For code, proposing additions to the type. For CMS, creating the fields via MCP.

---

## Phase 1C: Create — Choose Target & Configure

### 1C.1 Confirm the Target System

If `{system}` was detected, confirm it. Otherwise ask:

```
Where should this content model live?

1. Markdown / MDX front-matter (+ matching JSON-LD) — for a Next.js / Vite / Astro blog
2. A code-defined content type (TypeScript interface or schema file)
3. A connected CMS (via MCP) — Webflow, Contentful, Sanity, WordPress, etc.
```

### 1C.2 Identity

```
1. Name: What's this content type called? (e.g. "Article", "Blog Post", "Insight")
2. Slug source:
   - markdown: filename-derived, or an explicit `slug` key?
   - code: `slug` field
   - cms: collection slug (auto-suggested, confirm or change)
```

### 1C.3 Configure Flexible Fields

Some fields have shape choices. Ask upfront.

```
A few decisions before I build the model:

**Author**
1. Single author — string (markdown), `author: string` (code), or Reference (cms)
2. Multiple authors — list / `string[]` / MultiReference
3. None — skip
[cms, if 1/2]: which Authors collection? (list, or "create later")

**Tags**
1. Fixed list — Option field (cms) / string literal union (code) — define the options
2. Flexible — list of strings (markdown/code) / MultiReference to a Tags collection (cms)
3. None — skip
[if fixed]: which tags? (comma-separated)
[cms, if flexible]: which Tags collection? (list, or "create later")

**Category**
1. Yes — string (markdown/code) / Reference (cms)
2. None — skip
[cms, if 1]: which Categories collection? (list, or "create later")

**Pillar page**
Links this article to the topic page it supports — internal-linking leverage.
1. Yes — slug/path (markdown/code) / Reference to a pillar collection (cms)
2. Self-reference (cms) — link to another item in this collection
3. None — skip
[cms, if 1]: which collection? (list, or "create later")

**FAQ**
1. Structured — list of {q, a} (markdown/code) / MultiReference to an FAQ collection (cms)
2. Inline — RichText (cms) / a freeform `faq` block (markdown)
3. None — skip
[cms, if structured]: which FAQ collection? (list, or "create later")
```

### 1C.4 Confirm Before Building

Present the plan as a table (group · field · representation-in-this-system · notes), then state what will be produced:

- **markdown** → a front-matter template file + a matching JSON-LD snippet (and, if a repo, an example post). Show the target paths.
- **code** → a TypeScript type (e.g. `Article`/`Post`) + one example object literal. Show the target file path.
- **cms** → the list of fields to create on the collection (via MCP).

**Wait for explicit user confirmation before writing files or creating fields.**

---

## Phase 2C: Create — Build It

Build in group order: Core → Content → Authority → Enhancement → Social. Log progress per field.

### 2C.1 markdown target

Produce two artifacts (see `references/scaffold-templates.md` for the full reference templates):

1. **Front-matter template** — all chosen fields with example/placeholder values and inline comments noting char limits (meta title ≤60, meta description ≤155, OG description ≤200) and image guidance (OG image 1200×630, 1.91:1).
2. **JSON-LD snippet** — an `Article`/`BlogPosting` block (plus `FAQPage` if FAQ chosen) that reads from those fields, so structured data stays in sync.

If in a repo, offer to **write the template/example file with a before/after diff** (e.g. a documented `_template.md` or an example post, and a `jsonld` snippet in the layout). Never overwrite an existing file without showing the diff and getting confirmation.

### 2C.2 code target

Produce a TypeScript type with all chosen fields (required vs optional matching the schema — Core/Content required, Enhancement/Social mostly optional) plus one fully-filled example object. Offer to write it to the detected/agreed path with a diff.

### 2C.3 cms target (MCP)

Create the collection (if new) then create fields in order, following the field/type table in `references/scaffold-templates.md`. Mirror the original's guards:

⚡ GUARD — **Slug conflict:** suggest `{slug}-2`, `{slug}-articles`; confirm before retry.
⚡ GUARD — **Reference target = "create later":** skip that Reference/MultiReference field, log it as "⚠️ Skipped — add after creating the target collection", don't error.
⚡ GUARD — **Self-reference (related posts):** always create last; if it fails, note in summary.
- Pause ~2s between field creations; on rate limit, pause 10s and retry once.
- On unknown field error: log, skip, continue; flag in summary.

---

## Phase 3: Summary

```
## Content Model {Created | Updated} — {name}

**System**: {markdown | code | cms}
**Where**: {paths written / collection id}
**Fields**: {X} of {Y} planned

### Field Summary

| Group | Field | Status | Representation |
|-------|-------|--------|----------------|
| Core | Title | ✅ | title |
| Core | H1 override | ✅ Added | h1 |
| Authority | Pillar page | ⚠️ Skipped | create target collection first (cms) |
| ... | ... | ... | ... |

### Skipped / Follow-up
[any skipped fields and why]

---

### What to Do Next
1. **markdown**: copy the template into new posts; confirm the layout emits the JSON-LD snippet.
2. **code**: import the type where posts are loaded; backfill existing content to satisfy required fields.
3. **cms**: bind new fields to the template page; create any "later" reference collections.
4. **Populate**: run `/refresh-content [URL or path]` to fill SEO fields on existing articles.
5. **Schema markup**: once fields are populated, `/refresh-content` injects Article + FAQ JSON-LD automatically.

### Skills That Read These Fields
| Skill | Fields Used |
|-------|-------------|
| `/refresh-content` | All — primary/secondary keywords, meta titles, FAQ, updated date |
| `/click-recovery` | Meta title, Meta description |
| `/monthly-report` | Meta title (template audit) |
```

---

## Error Handling

| Error | Action |
|-------|--------|
| No content model detected | Advisory mode — present recommended schema + generic template, ask where content lives |
| CMS MCP not connected (cms target chosen) | Fall back to markdown or code target, or ask user to connect the MCP — never hard-stop |
| Front-matter unparseable in some files | Sample more files; note inconsistent posts in the report |
| Schema/type file ambiguous | Report the fields found, ask the user to confirm which file is canonical |
| CMS slug already exists | Suggest alternatives, confirm before retry |
| Reference target not found / "create later" | Skip the reference field, log it, continue |
| Self-reference fails (related posts) | Always created last — note in summary if it fails |
| Field-creation rate limit | Pause 2s between fields; on persistent limit, pause 10s and retry once |
| Unknown error writing/creating a field | Log, skip, continue; flag in summary |

---

## Activity Log

After every execution, append a row to `.claude/reports/{domain}/activity-log.md` (see shared `CLAUDE.md`). Determine `{domain}` from config, the site URL, the repo name, or the CMS.

Create the file with the standard header if missing, then append:

**Review mode:**
```
| YYYY-MM-DD | /content-model | Reviewed content model ({system}). Schema score: X%. Missing: [key fields]. |
```

**Create mode:**
```
| YYYY-MM-DD | /content-model | Scaffolded content model ({system}): [template+JSON-LD | TS type | N CMS fields] across Core/Content/Authority/Enhancement/Social. |
```

Log even on partial runs (e.g., "Added 4 of 6 missing front-matter keys; JSON-LD patch pending").
