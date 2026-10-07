# Tags

Free-text labels in frontmatter, URLs on the site. The whole file is about the
lossy step in the middle.

## Slugging

```ts
// lib/blog/tag-slug.ts
/** Tag label -> URL-safe slug. Diacritics are folded to ASCII. */
export function getTagSlug(tag: string): string {
  return tag
    .toLowerCase()
    .normalize("NFD")
    .replace(/[\u0300-\u036f]/g, "")
    .replace(/\s+/g, "-");
}
```

`NFD` + combining-mark strip is what makes `"Integracja"` and `"Rezerwacje ONLINE"`
produce clean ASCII slugs. Keep it: the alternative is percent-encoded URLs that
look broken when pasted anywhere.

Note what it does **not** do: it does not strip punctuation. `"CI/CD"` becomes
`ci/cd`, an extra path segment that will not match the `[slug]` route. If your
tags can contain `/`, `?`, `#`, `&` or `.`, extend it:

```ts
.replace(/[^\p{Letter}\p{Number}\s-]/gu, "")   // after the diacritic strip
.replace(/\s+/g, "-")
.replace(/-+/g, "-")
.replace(/^-|-$/g, "");
```

## The collision rule

Slugging is many-to-one. `"AI Agents"`, `"AI agents"` and `"ai agents"` all
produce `ai-agents`. In a repo where several people write frontmatter by hand,
that drift is a matter of time, not of chance.

The earlier implementation resolved a tag URL back to a label like this:

```ts
// The defect.
export function getTagFromSlug(slug: string, locale: string): string | null {
  const allTags = getAllTags(locale);            // sorted, distinct labels
  return allTags.find((tag) => getTagSlug(tag) === slug) || null;
}
// ...then filtered posts by exact label:
posts.filter((post) => post.tags?.includes(tag));
```

Two failures follow, and neither errors:

1. **Posts disappear from their own tag page.** `getTagFromSlug` returns the
   first label alphabetically, `"AI Agents"`. The filter then matches only that
   exact string, so every post tagged `"AI agents"` is absent from `/blog/tag/ai-agents`.
   There is no other page they appear on.
2. **`generateStaticParams` emits duplicate params.** Two labels, one slug. Next
   deduplicates, so the build is quiet, but which label won is decided by sort
   order.

The fix is to make **the slug the identity** and treat the label as display text.

```ts
// lib/blog/tags.ts
import { DEFAULT_LOCALE, type Locale } from "./locales.ts";
import { getAllPosts, type Post } from "./posts.ts";
import { getTagSlug } from "./tag-slug.ts";

export type TagSummary = {
  /** URL identity. Unique within a locale. */
  slug: string;
  /** Display label: the most-used spelling variant. */
  label: string;
  /** Posts carrying this tag. Invariant: equals getPostsByTagSlug(slug).length. */
  count: number;
  /** Every spelling that slugs to this tag. Length > 1 means drift to clean up. */
  variants: string[];
};

/** All tags in a locale, grouped by slug, ordered by label. */
export function getTags(locale: Locale = DEFAULT_LOCALE): TagSummary[] {
  const groups = new Map<string, { variants: Map<string, number>; posts: number }>();

  for (const post of getAllPosts(locale)) {
    // A post using two spellings of one tag must count once towards that tag,
    // but both spellings are still recorded so drift stays visible.
    const counted = new Set<string>();
    for (const tag of post.tags) {
      const slug = getTagSlug(tag);
      const group = groups.get(slug) ?? { variants: new Map<string, number>(), posts: 0 };
      group.variants.set(tag, (group.variants.get(tag) ?? 0) + 1);
      if (!counted.has(slug)) {
        group.posts += 1;
        counted.add(slug);
      }
      groups.set(slug, group);
    }
  }

  return [...groups.entries()]
    .map(([slug, group]) => {
      const sorted = [...group.variants.entries()].sort(
        // Most-used spelling wins; alphabetical breaks ties so the label is
        // stable across builds rather than dependent on file read order.
        ([labelA, countA], [labelB, countB]) =>
          countB - countA || labelA.localeCompare(labelB),
      );
      const first = sorted[0];
      return {
        slug,
        label: first ? first[0] : slug,
        // Number of posts carrying the tag; equals getPostsByTagSlug().length.
        count: group.posts,
        variants: sorted.map(([label]) => label),
      };
    })
    .sort((a, b) => a.label.localeCompare(b.label));
}

export function getTagBySlug(slug: string, locale: Locale): TagSummary | null {
  return getTags(locale).find((tag) => tag.slug === slug) ?? null;
}

/** Posts carrying a tag, matched by slug so every spelling variant is included. */
export function getPostsByTagSlug(slug: string, locale: Locale): Post[] {
  return getAllPosts(locale).filter((post) =>
    post.tags.some((tag) => getTagSlug(tag) === slug),
  );
}
```

`getPostsByTagSlug` matching on the slug is the actual repair: both spellings
land on one page, with the posts from both. `variants` then gives you a
one-line drift report; see [operations.md](operations.md).

**Tag slugs are per-locale.** Do not build a cross-locale tag map; nothing in the
content records that `"Booking"` and `"Rezerwacje"` are the same concept. See the
language-switcher rule in [i18n-and-routing.md](i18n-and-routing.md).

## Thin tag pages

The earlier implementation generated a tag page for every distinct tag: **about
two tags per post**, so a large share of them had a single post. Each is an
indexable page whose content is one card, exactly what thin-content heuristics
penalize, and every one of them is a near-empty URL in the sitemap, once per
locale.

The module ships the threshold, defaulted off, because the right value is an
editorial decision:

```ts
/** Tags eligible for their own page. Below the threshold a tag still renders as
 *  a label on a card, it just gets no page and no sitemap entry. */
export function getIndexableTags(locale: Locale, minPosts = 1): TagSummary[] {
  return getTags(locale).filter((tag) => tag.count >= minPosts);
}
```

Use `getIndexableTags` in `generateStaticParams` **and** in the sitemap, from one
constant. If those two disagree you get either 404s in the sitemap or orphan
pages. With `dynamicParams = false`, a tag below the threshold 404s at the
routing layer, which is the behaviour you want: a URL with nothing on it should
not exist.

Raising the threshold later removes URLs that may be indexed. Redirect them to
the blog index rather than letting them 404 if the tag ever ranked.

## Rendering tags

- On a card, render `post.tags`, the author's own spelling, and link with
  `tagPath(locale, getTagSlug(tag))`. Readers see what was written; the URL is
  canonical.
- On the tag page, render `tagSummary.label`.
- Never render `variants` to readers. It is a maintenance signal.

## Checklist

- [ ] `getTagSlug` handles diacritics, and punctuation if your tags contain any
- [ ] Tags grouped by slug; label chosen by frequency with an alphabetical tiebreak
- [ ] Posts matched by slug, not by exact label
- [ ] A post repeating a variant counts once, but every variant is recorded
- [ ] `generateStaticParams` and the sitemap use the same eligibility function
- [ ] Thin-tag threshold consciously chosen
- [ ] No cross-locale tag mapping anywhere
