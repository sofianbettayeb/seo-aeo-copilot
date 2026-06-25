# Content System Detection

How `/content-model` figures out where the content model lives. Walk the ladder top to bottom; the first hit wins, but record secondary signals (e.g. markdown front-matter AND a TS type both present).

Local files are the backbone. A CMS MCP is optional and only relevant for hosted content.

---

## 1. Markdown / MDX front-matter (most common)

Glob for article files. Common roots, in rough priority:

```
client/src/blog/posts/*.md        # Vite/React blogs (e.g. AEO-Copilot)
server/content/articles/*.md      # server-rendered blogs (e.g. Sofian-Personal)
content/**/*.{md,mdx}             # generic
src/content/**/*.{md,mdx}         # Astro content collections
app/blog/**/*.{md,mdx}            # Next.js app router
posts/**/*.{md,mdx}
_posts/*.md                       # Jekyll-style
```

When found:
- Read 3–8 representative files (mix of old and recent).
- Parse the leading `---` … `---` YAML/TOML front-matter block in each.
- Union the keys. A key is "present" if it appears in the majority of sampled files; flag keys present in only some as partial coverage.
- Also Read the rendering layer (layout component, `[slug]` page, MDX provider) and scan for JSON-LD emission (`application/ld+json`, `Article`, `BlogPosting`, `FAQPage`). Fields emitted there but absent from front-matter count as rendered/inferred matches in scoring.

Set `{system} = markdown`.

---

## 2. Code-defined content type

Glob for a schema/type file that declares the content shape:

```
**/content.config.ts              # Astro
**/contentlayer.config.*          # Contentlayer
src/content/config.ts             # Astro content collections
**/*frontmatter*.{ts,js}
**/types/*{post,article,blog}*.{ts,d.ts}
**/lib/*{posts,content}*.{ts,js}  # often holds the Post type + loader
schemas/*.{js,ts}                 # Sanity schema dir
**/sanity.config.{js,ts}
```

When found:
- Read the file and extract declared fields: name, type, optional (`?`) vs required.
- For Sanity/Contentful-style schemas, parse the `fields` array (name + type).

Set `{system} = code`.

---

## 3. Connected CMS (via MCP)

Only if a CMS MCP is available (the shared `CLAUDE.md` MCP-discovery step found one):

- **Webflow**: `data_cms_tool` / `collections_list` to list collections, `collections_get` for the field list (slugs + types).
- **Other headless CMS over MCP**: list content types, fetch fields of the chosen one.

If multiple collections/types exist, ask the user which to review or extend.

Set `{system} = cms`.

---

## 4. None → advisory mode

If nothing above resolves:
- Do not stop.
- Present the recommended five-group schema and the generic front-matter template (`references/scaffold-templates.md`).
- Ask the user where their content lives (repo path, schema file, or a CMS to connect), then retry detection.

Set `{system} = none`.

---

## Capturing `{domain}` for logging

Resolve in this order:
1. `business.siteUrl` / domain in `.claude/seo-copilot-config.json`
2. A site URL the user provided
3. The repo directory name
4. The CMS site name/domain

Used only for the activity-log path `.claude/reports/{domain}/activity-log.md`.
