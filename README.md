# SEO/AEO Copilot

Claude Code skills for SEO and AEO on **any** site — keyword research, content creation, traffic recovery, deep audits, and weekly/monthly reporting, all through natural language.

Platform-agnostic by design. Works whether your site runs on Webflow, WordPress, Shopify, Framer, Next.js, a custom React/Vite build, or hand-coded static HTML. Content-changing skills **edit your local source files** (with a diff for approval) when run inside the repo, or **output paste-ready recommendations** when given only a URL.

> Looking for the Webflow-native version (CMS publishing via the Webflow MCP)? See [webflow-seo-copilot](https://github.com/sofianbettayeb/webflow-seo-copilot).

## Requirements

No MCP server is required. Each one deepens the analysis when connected:

| MCP Server | Adds |
|------------|------|
| [Google Search Console MCP](https://github.com/sofianbettayeb/gsc-mcp-server) | Search analytics — powers `/click-recovery` and the reports; optional enrichment for `/audit:deep`, `/refresh-content`, `/keywords-opportunity`, `/topic-map` |
| PageSpeed MCP | Core Web Vitals for `/audit:deep` |
| A CMS MCP (e.g. [Webflow](https://developers.webflow.com/mcp/reference/overview)) | Reads the CMS as source of truth, and lets content skills apply via the CMS instead of files |
| [Keywords Everywhere MCP](https://github.com/hithereiamaliff/mcp-keywords-everywhere) | Search volume and intent data |
| [AEO Copilot MCP](https://github.com/sofianbettayeb/aeo-copilot-mcp) | `/aeo-onboard` and `/ai-visibility` (LLM visibility tracking) |

`/audit {url}` and `/audit:deep {url}` run on any public URL with zero MCP.

## Quick Start

```bash
# Add the marketplace
/plugin marketplace add sofianbettayeb/seo-aeo-copilot

# Install the skills you want (examples)
/plugin install getting-started@seo-aeo-copilot
/plugin install audit@seo-aeo-copilot
/plugin install audit-deep@seo-aeo-copilot
/plugin install refresh-content@seo-aeo-copilot
/plugin install write-blog@seo-aeo-copilot
/plugin install aeo-optimize@seo-aeo-copilot
/plugin install click-recovery@seo-aeo-copilot
/plugin install keywords-opportunity@seo-aeo-copilot
/plugin install content-model@seo-aeo-copilot
/plugin install topic-map@seo-aeo-copilot
/plugin install weekly-report@seo-aeo-copilot
/plugin install monthly-report@seo-aeo-copilot
/plugin install ai-visibility@seo-aeo-copilot
/plugin install aeo-onboard@seo-aeo-copilot

# Run setup first — captures brand voice, platform, repo path, and SEO/AEO goals
/getting-started
```

## Skills

| Command | What it does |
|---------|--------------|
| `/getting-started` | One-time setup — business context, brand voice, platform, repo path, SEO/AEO constraints → `.claude/seo-copilot-config.json`. |
| `/audit {url}` | Quick pre-sale audit from any public URL — four dimensions as a plain-language status, prioritized opportunities. No MCP. |
| `/audit:deep {url}` | Deep audit — live multi-page crawl, deepened by GSC/PageSpeed/CMS when connected. Findings, forecast, and per-bucket working docs. |
| `/deliverables` | Turn an audit's working docs into branded, PDF-ready HTML reports. Prompts for branding; neutral default. |
| `/keywords-opportunity` | Striking-distance keywords + new-topic discovery, scored and routed. |
| `/topic-map` | Keyword exports + URLs + GSC → structured topic map, gaps, editorial calendar. |
| `/content-model` | Audit or design the content field model (front-matter / headless CMS / code type / CMS). |
| `/write-blog` | Write a new SEO+AEO article as a local source file (or a draft). Humanizer pass included. |
| `/refresh-content {url}` | Refresh an existing article — edits the source file with a diff, or outputs recommendations. |
| `/aeo-optimize {url}` | Make a page AEO-ready (FAQ, schema, question H2s, direct answers) by editing the source file. |
| `/click-recovery` | GSC-driven CTR fixes — better titles/descriptions applied to local files. |
| `/weekly-report` · `/monthly-report` | Performance pulse + prioritized action plan. |
| `/ai-visibility` · `/aeo-onboard` | LLM visibility baseline and brand bootstrap (via AEO Copilot MCP). |

Full docs live in each skill's `SKILL.md` under `plugins/<name>/skills/<name>/`.
