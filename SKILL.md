---
name: blog-markdown
description: >
  Build a file-based, multilingual blog from markdown content: locale-partitioned
  posts, per-locale slugs joined by a shared translation key, tag pages, related
  posts, cover art, RSS, sitemap and hreflang. Use when: (1) adding a blog,
  changelog, news or article section backed by markdown files rather than a CMS,
  (2) making an existing markdown blog multilingual, or adding tag pages,
  hreflang, feeds or related posts, (3) the content lives in a repo folder and
  must be statically generated, (4) the user mentions: markdown blog, MDX blog,
  frontmatter, gray-matter, content/blog, translationKey, tag pages, hreflang,
  related posts, generateStaticParams for posts, blog RSS feed. Carries the
  cross-locale slug model, the tag-slug collision rule, the frontmatter parser
  that survives smart quotes, and the loader memoization that removes a measured
  71x build-time read amplification. Next.js App Router oriented; the content
  source is a seam, so a CMS can replace the filesystem. Not a CMS integration
  skill and not a docs/help-center skill.
---

# Markdown Blog (multilingual, tagged)

A blog where the content is files, not rows. The whole design turns on one idea:
**a post's identity is its `translationKey`, not its slug.** Slugs are localized
for SEO and differ per language; the key is what makes three files one article,
and it is what powers hreflang, the language switcher, related posts, and
cross-locale redirects. Get that one field right and everything else is lookups.

Written by the engineer who has shipped this module. The earlier implementation
it was audited against ran 3 locales x 23 posts, ~46 tags, statically generated,
plus its sitemap and SEO surface.

## When to use

- Content authored as markdown in the repo, deployed with the app
- More than one language, with different slugs per language
- Tag pages, related posts, feeds, or hreflang are in scope
- You want the whole thing statically generated at build time

## When NOT to use

| Instead of this | Use |
|---|---|
| A hosted CMS is the source of truth | that CMS's SDK; keep only `references/content-model.md` |
| A docs site / help center (categories, sidebar, search) | a help-center skill — different navigation model |
| A single-locale blog of <10 posts | plain `fs` + `gray-matter` inline; this is overhead |
| Rendering user-submitted markdown | a sanitizing renderer; see the XSS note in `references/rendering.md` |

## Architecture

```
content/blog/<locale>/<localized-slug>.md   # frontmatter + body
        |
        v
 parseFrontmatter  --YAML, falls back to a line parser on malformed quotes
        |
        v
   getAllPosts(locale)  <-- memoized per locale in production
        |
        +--> translationKey ----> alternates / hreflang / language switch
        +--> tags -> tag SLUG ---> tag pages (slug is the identity, not the label)
        +--> related (EN slugs) -> related posts, with an EN fallback
        |
        v
  /blog   /blog/[slug]   /blog/tag/[slug]   sitemap.xml   feed.xml
```

## Critical facts

1. **Relations are authored once, in the default locale, as default-locale
   slugs.** A translated file may omit `related` entirely and still get the right
   related posts, resolved through `translationKey`. In the earlier
   implementation this fallback was missing and **17 of 23 translated posts silently showed no
   related section**.
2. **A tag's identity is its slug, not its label.** `"AI Agents"` and
   `"AI agents"` slug to the same URL. Group by slug and match posts by slug, or
   one spelling variant's posts vanish from its own tag page.
3. **`gray-matter` throws on a double-quoted YAML scalar containing raw quote
   characters** — typographic quotes in a translated excerpt do it. Without a
   fallback parser the build dies on one content file. Verified live.
4. **`gray-matter` caches a failed parse as an empty result.** It writes its
   cache entry before parsing, so the second and every later parse of a file
   that threw returns `{}` — no error, no frontmatter. Always call
   `matter(contents, {})`; any options object opts out of the cache.
5. **Loading is O(posts x locales) per page unless you memoize.** Measured on the
   earlier implementation: 4,899 file reads to generate 69 pages, a 71x
   amplification, all from resolving alternates during `generateMetadata`.
6. **`getPostSlugs` must filter `.md`.** A stray `.DS_Store` makes the loader
   read `.DS_Store.md`, throw `ENOENT`, and fail the entire build.
7. **Tag pages have no cross-locale identity.** Tags are per-locale free text, so
   the language switcher must fall back to the blog index on a tag URL rather
   than build a URL that 404s.

## Hard rules

> **Never key cross-locale lookups on the slug.** Slugs are localized on purpose.
> `translationKey` is the join column; a slug match across locales is a
> coincidence you will ship a bug on.

> **Never let `getPostBySlug` decide draft visibility implicitly.** In the earlier
> implementation drafts were kept out of production only as a side effect of
> `dynamicParams = false`. Flip that flag for any reason and every draft is live.

> **Never render a "5 min read" constant.** The earlier implementation printed a
> hardcoded 5 on every card. Compute it from the body; it costs one `split`.

> **Never derive a tag page's URL from the label at read time and the label from
> the URL at render time without a collision check.** That round trip is only
> lossless while every label slugs uniquely.

## Quick start

1. Set the locale contract and the frontmatter schema —
   [content-model.md](references/content-model.md)
2. Build the loader (memoized, `.md`-filtered, fallback parser) —
   [content-loader.md](references/content-loader.md)
3. Wire locales, alternates and cross-locale redirects —
   [i18n-and-routing.md](references/i18n-and-routing.md)
4. Add tag grouping and tag pages — [tags.md](references/tags.md)
5. Build the three routes, metadata and JSON-LD —
   [pages-and-seo.md](references/pages-and-seo.md)
6. Render bodies and covers — [rendering.md](references/rendering.md)
7. Add the validation script and read the operator gaps —
   [operations.md](references/operations.md)
8. Run the fixtures and tests — [testing.md](references/testing.md)

Porting this into an existing app? Fill in the seam table in
[adaptation.md](references/adaptation.md) first — it takes ten minutes and saves
a rename.

## Reference directory

| Scenario | Trigger keywords | Reference |
|---|---|---|
| Frontmatter fields, drafts, translation keys, relations | frontmatter, schema, translationKey, draft, related, excerpt | [content-model.md](references/content-model.md) |
| Reading files, caching, YAML failures, reading time | gray-matter, fs, cache, memoize, ENOENT, YAMLException, parse | [content-loader.md](references/content-loader.md) |
| Locales, hreflang, alternates, wrong-locale links | i18n, locale, hreflang, alternates, canonical, language switcher, redirect | [i18n-and-routing.md](references/i18n-and-routing.md) |
| Tag URLs, collisions, thin tag pages | tag, slug, diacritics, collision, tag page, taxonomy | [tags.md](references/tags.md) |
| The routes, metadata, JSON-LD, sitemap, RSS | page.tsx, generateStaticParams, generateMetadata, dynamicParams, sitemap, RSS, feed, BlogPosting | [pages-and-seo.md](references/pages-and-seo.md) |
| Markdown to React, code blocks, cover images | react-markdown, remark-gfm, syntax highlighting, cover, motif, accent, XSS | [rendering.md](references/rendering.md) |
| Validating content, build cost, editor workflow | validation, CI check, build time, drafts, authoring, operator | [operations.md](references/operations.md) |
| Fixtures and the passing test suite | test, fixture, vitest, node:test, regression, assert | [testing.md](references/testing.md) |
| Fitting it into a host app: seams, renames, probe | adapt, port, host, seam, rename, CMS, integrate | [adaptation.md](references/adaptation.md) |
| What was fixed, kept, and added vs. the earlier implementation | provenance, deviations, fidelity, defects | [provenance.md](references/provenance.md) |
