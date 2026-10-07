# i18n and routing

Three locales, three slugs, one article. This file covers how a reader gets to
the right one: from a search engine, from the language switcher, and from a link
someone shared in the wrong language.

## URL shape

```
/blog                         default locale, no prefix
/blog/<en-slug>
/blog/tag/<en-tag-slug>
/pl/blog                      non-default locales are prefixed
/pl/blog/<pl-slug>            note: a different slug, not a translated path
/pl/blog/tag/<pl-tag-slug>
```

Stripping the prefix for the default locale is the usual SEO choice and the one
this module assumes. If your host prefixes every locale, change `localizedPath`
and nothing else.

```ts
// lib/blog/paths.ts
import { DEFAULT_LOCALE, type Locale } from "./locales.ts";

/** Seam: if the host has its own path helper (localized route slugs, aliases),
 *  delegate to it here rather than duplicating the rule. */
export function localizedPath(locale: Locale, path: string): string {
  return locale === DEFAULT_LOCALE ? path : `/${locale}${path}`;
}

export function blogIndexPath(locale: Locale): string {
  return localizedPath(locale, "/blog");
}

export function postPath(locale: Locale, slug: string): string {
  return localizedPath(locale, `/blog/${slug}`);
}

export function tagPath(locale: Locale, tagSlug: string): string {
  return localizedPath(locale, `/blog/tag/${tagSlug}`);
}
```

Every link in the module goes through these. Hand-built template strings are how
a locale prefix goes missing on one card in one component.

## Alternates and hreflang

```ts
// lib/blog/alternates.ts
import { blogIndexPath, postPath } from "./paths.ts";
import { LOCALES, type Locale } from "./locales.ts";
import { getAllPosts } from "./posts.ts";

/** Locale -> slug for every locale that has this article. */
export function getAlternateSlugs(translationKey: string): Partial<Record<Locale, string>> {
  const alternates: Partial<Record<Locale, string>> = {};
  for (const locale of LOCALES) {
    const post = getAllPosts(locale).find((p) => p.translationKey === translationKey);
    if (post) alternates[locale] = post.slug;
  }
  return alternates;
}

/**
 * hreflang map for a post. Locales without a translation point at that locale's
 * blog index rather than being omitted: a reader who switches language lands on
 * something in their language instead of a 404.
 */
export function getLanguageAlternates(translationKey: string): Record<string, string> {
  const alternateSlugs = getAlternateSlugs(translationKey);
  const languages: Record<string, string> = {};
  for (const locale of LOCALES) {
    const slug = alternateSlugs[locale];
    languages[locale] = slug ? postPath(locale, slug) : blogIndexPath(locale);
  }
  return languages;
}
```

**This function is why the loader must be memoized.** It touches every locale's
full index and runs once per page in `generateMetadata`.

The index fallback is a judgement call, not a rule. The alternative, omitting
untranslated locales, gives a stricter hreflang cluster but a dead end in the
switcher. Pick one and be consistent; do not point hreflang at a URL that 404s.

## Cross-locale slug resolution

The problem: someone shares `/blog/agent-experience-optimization` on social. A
German reader opens it; the host's locale detection prefixes it to
`/de/blog/agent-experience-optimization`. That German slug does not exist; the
German file is `agent-experience-optimierung.md`. Naively that is a 404 on a link
that was correct when it was posted.

```ts
// lib/blog/alternates.ts (continued)
/**
 * Given a slug that may belong to another locale, find the equivalent slug in
 * the target locale. Returns null when the slug is unknown everywhere, or when
 * the article has no translation in the target locale.
 */
export function resolveLocalizedSlug(slug: string, targetLocale: Locale): string | null {
  for (const locale of LOCALES) {
    if (locale === targetLocale) continue;
    const post = getAllPosts(locale).find((p) => p.slug === slug);
    if (post) return getAlternateSlugs(post.translationKey)[targetLocale] ?? null;
  }
  return null;
}
```

Call it only on the miss path, after `getPostBySlug` returns `null`, and issue
a **redirect**, not a rewrite, so the reader ends up on the canonical URL and the
link equity consolidates there. Iterate `LOCALES` in declaration order so the
result is deterministic when two locales happen to share a slug.

## The language switcher

Switching language on a blog URL is three cases, and only the first is obvious:

| Current URL | Switch to `de` | Why |
|---|---|---|
| `/blog` | `/de/blog` | plain path mapping |
| `/blog/<en-slug>` | `/de/blog/<de-slug>` | must go through `translationKey` |
| `/blog/tag/<slug>` | `/de/blog` | **tags have no cross-locale identity** |

The third row is the one that bites. Tags are free-text strings authored per
locale: `"Rezerwacje"` is not a translation of `"Booking"` recorded anywhere in
the content. There is no correct target tag URL, so falling back to the blog
index is the honest answer. Building `/de/blog/tag/booking` from the English slug
produces a 404 for a page that never existed.

```ts
export function switchLocaleForBlogPath(
  pathname: string,
  currentLocale: Locale,
  targetLocale: Locale,
  post?: { translationKey: string },
): string {
  // Tag pages: no cross-locale identity exists. Fall back to the index.
  if (pathname.includes("/blog/tag/")) return blogIndexPath(targetLocale);

  if (post) {
    const slug = getAlternateSlugs(post.translationKey)[targetLocale];
    return slug ? postPath(targetLocale, slug) : blogIndexPath(targetLocale);
  }

  return blogIndexPath(targetLocale);
}
```

Passing the post in is deliberate. Deriving it from the pathname means a second
lookup and a second place that can disagree about which post you are on; the page
already has it.

## Strings

**Every string is a key.** The earlier implementation got this right everywhere
except the tag page, which carried an inline map:

```ts
// Do not do this.
const headingText = {
  en: `Articles tagged with "${tag}"`,
  pl: `Artykuły oznaczone "${tag}"`,
  de: `Artikel mit Tag "${tag}"`,
}[lang] || `Articles tagged with "${tag}"`;
```

Add a fourth locale and it silently serves English, with no missing-key error and
nothing for a translator to find. Its meta description had the same problem, and
worse: it hardcoded English marketing copy into the SEO description of every
locale's tag pages.

The key set this module needs, in the default locale:

```json
{
  "blog": {
    "badge": "Discover",
    "title": "Our Blog",
    "description": "Latest articles and updates.",
    "readArticle": "Read article",
    "minRead": "min read",
    "by": "by",
    "relatedPosts": "Further reading",
    "no_posts": "No articles yet. Check back soon.",
    "no_posts_for_tag": "No articles found for this tag.",
    "tag_heading": "Articles tagged with \"{tag}\"",
    "tag_count_one": "{count} article found",
    "tag_count_other": "{count} articles found",
    "tag_meta_description": "Read {count} articles tagged {tag}."
  }
}
```

`tag_count_one` / `tag_count_other` exist because pluralization is per-locale.
`${count} article${count !== 1 ? "s" : ""}` is an English rule compiled into
code; Polish alone needs three forms. If the host has `Intl.PluralRules` or an
i18n library with plural support, use it and drop the two keys.

Interpolate with a helper, not a template literal, so the key stays translatable:

```ts
export function interpolate(template: string, values: Record<string, string | number>): string {
  return template.replace(/\{(\w+)\}/g, (match, key: string) =>
    key in values ? String(values[key]) : match,
  );
}
```

## Server and client string access

Server components read the locale's dictionary directly from disk or an import;
client components (a card with a hover animation, the switcher) need it through
context. Both are host concerns; the module only requires that a string never
appears as a literal.

The one module-specific rule: **`readingMinutes` is computed on the server and
passed as a prop.** A client card must not receive the post body just to count
its words.

## Checklist

- [ ] All URLs built through `localizedPath` / `postPath` / `tagPath`
- [ ] Alternates resolved through `translationKey`, never a slug match
- [ ] hreflang covers every locale, and no entry 404s
- [ ] Wrong-locale slugs redirect (308/permanent), not rewrite
- [ ] Language switcher falls back to the index on tag URLs
- [ ] No inline `{ en, pl, de }` string maps anywhere
- [ ] Plural forms handled per-locale, not with an English ternary
- [ ] Meta descriptions localized, not English boilerplate under every locale
