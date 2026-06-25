# Field Aliases

Alias tables for the matching step in Phase 2R. A field counts as an **alias match** (half weight) when an existing key/slug/field maps to a recommended field by name. Names are normalized before comparison: lowercase, strip separators (`-`, `_`, spaces, camelCase split), so `metaTitle`, `meta-title`, `meta_title`, and `Meta Title` are all equivalent.

Canonical names are listed first; aliases cover both markdown/code (camelCase / kebab) and CMS (kebab-slug) conventions.

---

## Core

| Recommended field | Canonical | Aliases |
|-------------------|-----------|---------|
| Title / Name | `title` | `name`, `heading`, `postTitle`, `articleTitle` |
| Slug | `slug` | `permalink`, `url`, `path`, (or filename-derived) |
| H1 override | `h1` | `h1Override`, `h1-override`, `pageHeading`, `displayTitle`, `pageTitle` |
| Meta title | `metaTitle` | `seoTitle`, `meta-seo-title`, `seo-title`, `metaSeoTitle`, `ogTitleSeo`, `titile` (common typo) |
| Meta description | `metaDescription` | `description`, `seoDescription`, `meta-seo-description`, `excerpt`, `summary`, `postSummary`, `metaDesc` |

> `description`/`excerpt`/`summary` are weak aliases for meta description — flag them: they're often the article intro, not a SERP-tuned meta. Count as alias, note "verify it's SERP copy, not body intro".

---

## Content

| Recommended field | Canonical | Aliases |
|-------------------|-----------|---------|
| Body | (markdown body) / `body` | `content`, `postBody`, `articleBody`, `richText`, `main-content` |
| Main image | `image` | `cover`, `coverImage`, `hero`, `heroImage`, `featuredImage`, `featured-image`, `thumbnail`, `main-visual`, `ogImageFallback` |
| Image alt | `imageAlt` | `coverAlt`, `heroAlt`, `image-alt`, `featured-image-alt`, `alt`, `altText` |
| Reading time | `readingTime` | `readTime`, `read-time`, `timeToRead`, `minutes`, `duration` |
| Updated date | `updated` | `lastModified`, `lastmod`, `last-updated`, `dateUpdated`, `modifiedDate`, `updatedAt`, `dateModified` |

> If only a `date` / `publishedDate` / `created` exists (publish date, no separate update date), count "Updated date" as a half-match via inference and recommend a dedicated `updated`/`lastmod` — it's what feeds `dateModified` in JSON-LD.

---

## Authority

| Recommended field | Canonical | Aliases |
|-------------------|-----------|---------|
| Author | `author` | `authors`, `authorRef`, `writtenBy`, `byline`, `by` |
| Category | `category` | `categories`, `categoryRef`, `topic`, `section` |
| Tags | `tags` | `tag`, `tagList`, `labels`, `keywords` (only if not already used as SEO keywords) |
| Pillar page | `pillar` | `pillarArticle`, `pillar-page`, `parentPost`, `cluster`, `clusterParent`, `topicHub` |

---

## Enhancement

| Recommended field | Canonical | Aliases |
|-------------------|-----------|---------|
| Primary keyword | `primaryKeyword` | `keyword`, `targetKeyword`, `focusKeyword`, `primary-keyword`, `focusKw` |
| Secondary keywords | `secondaryKeywords` | `secondaryKeyword`, `lsiKeywords`, `relatedKeywords`, `keywords` (if `tags` already covers labels) |
| Related posts | `related` | `relatedPosts`, `relatedArticles`, `seeAlso`, `related-posts` |
| FAQ | `faq` | `faqs`, `faqSection`, `questions`, `qa`, `faqItems` |

---

## Social

| Recommended field | Canonical | Aliases |
|-------------------|-----------|---------|
| OG title | `ogTitle` | `socialTitle`, `openGraphTitle`, `og-title`, `shareTitle` |
| OG description | `ogDescription` | `socialDescription`, `openGraphDescription`, `og-description`, `shareDescription` |
| OG image | `ogImage` | `socialImage`, `openGraphImage`, `og-image`, `shareImage`, `twitterImage` |

> A single `twitterImage`/`twitter:image` with no `ogImage` is a partial match — most platforms read OG first. Recommend adding `ogImage` as the canonical share image.

---

## Disambiguation rules

- `keywords` is ambiguous — it can mean SEO keywords (Enhancement) or display tags (Authority). Assign it to whichever recommended field is otherwise unmatched; if both are unmatched, prefer **Secondary keywords** and flag the ambiguity.
- A field already claimed by a higher-confidence match cannot be reused for a second recommended field.
- Type/shape still has to pass the Phase 2R.2 check — an alias name with the wrong shape is an alias match flagged ⚠️.
