# Scaffold Templates

Reference artifacts for Phase 2C (Create). Use the one matching the target system. Adapt field presence to the user's Phase 1C choices (skip what they declined). Always show a diff before writing into a repo.

Char limits to surface as comments: meta title ≤60, meta description ≤155, OG description ≤200. OG image 1200×630px (1.91:1).

---

## A. Markdown / MDX front-matter template

```markdown
---
# --- Core (30%) ---
title: "Your article title"
slug: "your-article-slug"            # or derived from filename
h1: "On-page H1 (can differ from title)"
metaTitle: "SERP title ≤60 chars"
metaDescription: "SERP description ≤155 chars"

# --- Content (25%) ---
image: "/articles/images/your-cover.webp"
imageAlt: "Descriptive alt text for the cover image"
readingTime: 7                       # minutes; compute from word count
updated: 2026-06-23                  # feeds dateModified in JSON-LD

# --- Authority (20%) ---
author: "Sofian Bettayeb"            # or [list] for multiple
category: "SEO"
tags: ["seo", "aeo", "content"]
pillar: "your-pillar-slug"           # the topic page this supports

# --- Enhancement (15%) ---
primaryKeyword: "main target keyword"
secondaryKeywords: ["related term", "lsi term"]
related: ["other-slug-1", "other-slug-2"]
faq:
  - q: "A reader question?"
    a: "A direct answer."

# --- Social (10%) ---
ogTitle: "Share title (can differ from metaTitle)"
ogDescription: "Share description ≤200 chars"
ogImage: "/articles/images/your-og.webp"   # 1200×630, 1.91:1
---

Article body in markdown goes here.
```

### Matching JSON-LD snippet (rendered in the layout/template)

Keeps structured data in sync with the front-matter. Include the `FAQPage` block only if `faq` is used.

```html
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "BlogPosting",
  "headline": "{{ h1 || title }}",
  "description": "{{ metaDescription }}",
  "image": "{{ ogImage || image }}",
  "author": { "@type": "Person", "name": "{{ author }}" },
  "datePublished": "{{ date }}",
  "dateModified": "{{ updated }}",
  "keywords": "{{ [primaryKeyword, ...secondaryKeywords].join(', ') }}",
  "mainEntityOfPage": "{{ canonicalUrl }}"
}
</script>

<!-- only if faq exists -->
<script type="application/ld+json">
{
  "@context": "https://schema.org",
  "@type": "FAQPage",
  "mainEntity": [
    {% for item in faq %}
    { "@type": "Question", "name": "{{ item.q }}",
      "acceptedAnswer": { "@type": "Answer", "text": "{{ item.a }}" } }
    {% endfor %}
  ]
}
</script>
```

When writing into a repo: drop the template as a documented `_template.md` (or an example post) and add/patch the JSON-LD in the article layout component. Show diffs; never silently overwrite.

---

## B. Code-defined content type (TypeScript)

```ts
export interface Article {
  // --- Core (30%) ---
  title: string;
  slug: string;
  h1?: string;                 // page H1 override
  metaTitle: string;           // ≤60 chars
  metaDescription: string;     // ≤155 chars

  // --- Content (25%) ---
  image: string;               // cover/hero path
  imageAlt: string;
  readingTime?: number;        // minutes
  updated?: string;            // ISO date → dateModified

  // --- Authority (20%) ---
  author: string | string[];
  category: string;
  tags: string[];
  pillar?: string;             // slug of the pillar article

  // --- Enhancement (15%) ---
  primaryKeyword: string;
  secondaryKeywords?: string[];
  related?: string[];          // slugs
  faq?: { q: string; a: string }[];

  // --- Social (10%) ---
  ogTitle?: string;
  ogDescription?: string;      // ≤200 chars
  ogImage?: string;            // 1200×630, 1.91:1
}
```

Required vs optional follows the schema weights: Core + Content carriers required; Enhancement/Social mostly optional. Ship one fully-filled example object so authors see the intended shape.

---

## C. Headless / traditional CMS fields (via MCP)

Create fields in this order, group by group. Skip any field the user declined; skip reference fields whose target collection is "create later" (log them).

**Core**

| Field | Type | Display | Slug |
|-------|------|---------|------|
| H1 override | PlainText | H1 Override | h1-override |
| Meta title | PlainText | Meta Title | meta-title |
| Meta description | PlainText | Meta Description | meta-description |

(Title/Name and Slug are built-in in most CMSes — skip.)

**Content**

| Field | Type | Display | Slug | Notes |
|-------|------|---------|------|-------|
| Body | RichText | Body | body | |
| Main image | Image | Main Image | main-image | |
| Image alt | PlainText | Image Alt | image-alt | |
| Reading time | Number | Reading Time | reading-time | "In minutes" |
| Updated date | DateTime | Updated Date | updated-date | |

**Authority**

| Field | Type | Display | Slug | Condition |
|-------|------|---------|------|-----------|
| Author | Reference / MultiReference | Author | author | target = Authors collection |
| Category | Reference | Category | category | target = Categories |
| Tags | Option / MultiReference | Tags | tags | options list, or Tags collection |
| Pillar page | Reference | Pillar Page | pillar-page | target = pillar/topics collection |

**Enhancement**

| Field | Type | Display | Slug | Notes |
|-------|------|---------|------|-------|
| Primary keyword | PlainText | Primary Keyword | primary-keyword | |
| Secondary keywords | PlainText | Secondary Keywords | secondary-keywords | "Comma-separated" |
| Related posts | MultiReference | Related Posts | related-posts | self-reference — create LAST |
| FAQ | MultiReference / RichText | FAQ | faq | per 1C.3 choice |

**Social**

| Field | Type | Display | Slug | Notes |
|-------|------|---------|------|-------|
| OG title | PlainText | OG Title | og-title | |
| OG description | PlainText | OG Description | og-description | |
| OG image | Image | OG Image | og-image | "1200×630px (1.91:1)" |

Guards: slug conflict → suggest alternatives; reference target "create later" → skip + log; self-reference → last, note if it fails; ~2s between creations, 10s+retry on rate limit; unknown error → log/skip/continue.
