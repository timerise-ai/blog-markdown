# Content model

The frontmatter contract, and the three fields that carry all the cross-locale
behaviour. Everything in the rest of this skill is a lookup over this shape.

## Directory layout

```
content/blog/
  en/agent-experience-optimization.md
  pl/optymalizacja-doswiadczen-agentow.md
  de/agent-experience-optimierung.md
```

One directory per locale. **The filename (minus `.md`) must equal the `slug`
field**: the loader addresses files by slug, so a mismatch produces a post that
lists but 404s. The validation script in [operations.md](operations.md) checks it.

Slugs are localized on purpose: a Polish reader and Google both want a Polish
URL. That is precisely why nothing may key off the slug across locales.

## Frontmatter schema

| Field | Type | Required | Purpose |
|---|---|---|---|
| `translationKey` | string | **yes** | Cross-locale identity. Same value in all locales. Defaults to the slug if absent, which silently breaks alternates; always set it. |
| `title` | string | yes | |
| `date` | string | yes | ISO. Quote it (see "The date trap" below). |
| `slug` | string | yes | Must match the filename. |
| `excerpt` | string | yes | Card copy and meta description. |
| `author` | string | no | Falls back to a configured default. |
| `tags` | string[] | no | Free text. Slugged for URLs; see [tags.md](tags.md). |
| `related` | string[] | no | **Default-locale slugs**, not local ones. See below. |
| `coverImage` | string | no | Public path to a raster cover. |
| `coverMotif` | string | no | Key into the generated-cover registry. Takes precedence over `coverImage` when both are set. |
| `coverAccent` | string | no | Palette key; omitted means "derive from the first tag". |
| `draft` | boolean | no | `true` or `"true"` hides the post outside development. |
| `status` | string | no | Legacy alias: `"draft"` behaves as `draft: true`. |

Everything not listed is ignored by the loader, so extra editorial fields are
harmless.

### A complete example

```yaml
---
translationKey: "agent-experience-optimization"
title: "Agent Experience Optimization: How AI Agents Find Your Services"
date: "2026-04-28T10:00:00.000Z"
slug: "agent-experience-optimization"
excerpt: "AEO is the new SEO. When your customer is an agent, discoverability changes."
author: "Jane Doe"
coverImage: "/images/blog/agent-experience-optimization.png"
coverMotif: agent-discovery
coverAccent: blue
tags: ["AEO", "AI", "Discovery", "Strategy"]
related:
  ["writing-for-translation", "choosing-a-static-site-generator"]
---

Body starts here.
```

## The three load-bearing fields

### `translationKey`: identity

The join column. Three files share one key and are therefore one article.
It drives:

- **hreflang / `alternates.languages`**: the other locales' URLs
- **the language switcher**: which post to switch *to*, not just which locale
- **cross-locale redirects**: an English link opened under `/de` resolves to the
  German slug instead of 404-ing
- **related-post resolution across locales**

Rules:

- Set it explicitly in every file. The loader's fallback (`slug`) makes a missing
  key look fine in one locale and break silently in the other two.
- Never change it after publication; it is the article's permanent identity.
- Keep it stable and English even when the slug is not. It is an internal key,
  not a URL.

### `related`: relations authored once

`related` holds **default-locale slugs**, in every locale's file. A translated
post may omit the field entirely.

Resolution, in order:

1. If this post has `related`, use it.
2. Otherwise fall back to the **default-locale sibling's** `related`, found by
   `translationKey`.
3. Map each default-locale slug to a post, then to this locale's translation via
   `translationKey`. Drop anything with no translation.

Step 2 is the fix that matters. In the earlier implementation it was absent, so
a translator who did not copy the `related` block got no related section: **most
translated posts shipped with an empty "Further reading"** while every
default-locale post had one. Nothing errored; the section simply did not render.

Two smaller guards belong in the same resolver: **exclude the current post**
(a post listing itself renders itself) and **de-duplicate by `translationKey`**
(two default-locale slugs can resolve to the same translation).

Writing local slugs into a translated file's `related` is the common authoring
mistake. They resolve against the default-locale index, find nothing, and are
dropped without a warning. The validation script catches it.

### `draft`: visibility

```ts
draft:
  frontMatter.draft === true ||
  frontMatter.draft === "true" ||
  frontMatter.status === "draft",
```

Both spellings are accepted because content files outlive schema decisions. The
filter belongs in the loader:

```ts
.filter((post) => includeDrafts || !post.draft)
```

with `includeDrafts` defaulting to `process.env.NODE_ENV === "development"`.

**Make the guard explicit at the page level too.** In the earlier implementation,
drafts stayed out of production only because `dynamicParams = false` meant an
unpublished slug was never a valid route. That is an accident of a routing flag,
not an access rule: enable dynamic params for any reason and every draft becomes
publicly reachable at its real URL. `getPostBySlug` therefore takes an explicit
`includeDrafts` argument and the post route passes the environment-derived value.

## The date trap

YAML parses an unquoted `date: 2026-02-09` into a JavaScript `Date`, and a
quoted `date: "2026-02-09"` into a string. Consumers want a string, so normalize
on the way out of the parser:

```ts
function normalize(value: unknown): unknown {
  if (value instanceof Date) {
    const iso = value.toISOString();
    // Keep the time only when the source actually carried one.
    return iso.endsWith("T00:00:00.000Z") ? iso.slice(0, 10) : iso;
  }
  if (Array.isArray(value)) return value.map(normalize);
  return value;
}
```

Sorting is then `b.date.localeCompare(a.date)`, which is correct for ISO strings
of either length and does not construct a `Date` object per post per sort. Posts
with an empty date sort last, which is the behaviour you want for a malformed file.

## Adding a locale

1. Add the code to `LOCALES` (one place; see
   [i18n-and-routing.md](i18n-and-routing.md)).
2. Create `content/blog/<code>/`. An **empty or missing directory is legal**:
   the loader returns `[]`, so the locale can ship before its translations.
3. Add the string keys for that locale.

No code change. If adding a locale requires editing a hardcoded
`["en", "pl", "de"]`, fix that first: the earlier implementation had that array
in five places, only one of which was the config module.

## Checklist for a new post

- [ ] Filename equals `slug`, ends in `.md`
- [ ] `translationKey` present and identical across locales
- [ ] `date` quoted, ISO
- [ ] `excerpt` written for a search result, not truncated body text
- [ ] `related` uses default-locale slugs, or is omitted in a translation
- [ ] Tags reuse existing spellings; check `getTags()` output first
- [ ] `bun run validate:content` (or the equivalent) passes
