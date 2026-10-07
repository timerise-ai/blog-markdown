# Pages and SEO

Three routes, plus the sitemap and feed that make them discoverable. All
statically generated; nothing here needs a request.

```
app/[lang]/blog/page.tsx              index
app/[lang]/blog/[slug]/page.tsx       post
app/[lang]/blog/tag/[slug]/page.tsx   tag
app/sitemap.ts                        sitemap.xml
app/feed.xml/route.ts                 RSS  (addition, see provenance.md)
```

Styling below is structural only. Replace the utility classes with the host's
tokens; keep the element structure, the ordering, and the `aria`/semantic tags.

## Static generation

```tsx
// app/[lang]/blog/[slug]/page.tsx
import type { Metadata } from "next";
import { notFound, redirect } from "next/navigation";

import { LOCALES, isLocale, type Locale } from "@/lib/blog/locales";
import { getAllPosts, getPostBySlug } from "@/lib/blog/posts";
import { getLanguageAlternates, resolveLocalizedSlug } from "@/lib/blog/alternates";
import { postPath } from "@/lib/blog/paths";

// Unknown slugs become routing-level 404s rather than rendered ones. This also
// works around a Next 16.2 bug where notFound() ignores segment-level
// not-found files (vercel/next.js#90837). Re-check before enabling dynamic
// params: drafts are only unreachable because they are not in the params list,
// so getPostBySlug must keep its own explicit draft guard.
export const dynamicParams = false;

export const revalidate = 86400;

export async function generateStaticParams() {
  return LOCALES.flatMap((lang) =>
    getAllPosts(lang, { includeDrafts: false }).map((post) => ({ lang, slug: post.slug })),
  );
}
```

Pass `includeDrafts: false` explicitly here even though it is the production
default. `generateStaticParams` runs at build time in every environment, and a
draft that gets a param gets a public URL.

## The post route

```tsx
type Props = { params: Promise<{ lang: string; slug: string }> };

export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const { lang, slug } = await params;
  if (!isLocale(lang)) return {};

  const post = getPostBySlug(slug, lang);
  if (!post) return { title: "Not found" };

  return {
    title: post.title,
    description: post.excerpt,
    openGraph: {
      title: post.title,
      description: post.excerpt,
      type: "article",
      publishedTime: post.date,
      authors: [post.author],
      images: post.coverImage ? [post.coverImage] : [],
    },
    twitter: {
      card: "summary_large_image",
      title: post.title,
      description: post.excerpt,
      images: post.coverImage ? [post.coverImage] : [],
    },
    alternates: {
      canonical: postPath(lang, post.slug),
      languages: getLanguageAlternates(post.translationKey),
    },
  };
}

export default async function BlogPostPage({ params }: Props) {
  const { lang, slug } = await params;
  if (!isLocale(lang)) notFound();

  const post = getPostBySlug(slug, lang);

  if (!post) {
    // The slug may belong to another locale: a link shared in one language and
    // opened in another. Send the reader to the canonical translated URL rather
    // than 404-ing a link that was correct when it was posted.
    const localized = resolveLocalizedSlug(slug, lang);
    if (localized && localized !== slug) redirect(postPath(lang, localized));
    notFound();
  }

  return (
    <article>
      <header>
        <h1>{post.title}</h1>
        <p>
          <time dateTime={post.date}>{formatDate(post.date, lang)}</time>
          {" | "}
          {t.by} {post.author}
          {" | "}
          {post.readingMinutes} {t.minRead}
        </p>
      </header>

      <PostCover post={post} variant="hero" />
      <MarkdownRenderer content={post.content} />
      <RelatedPosts post={post} locale={lang} />
    </article>
  );
}
```

`redirect()` throws, so TypeScript narrows `post` to non-null after the block
only if `notFound()` is the last statement; it is, and both are typed `never`.

### Dates

```ts
const DATE_LOCALES: Record<Locale, string> = { en: "en-US", pl: "pl-PL", de: "de-DE" };

export function formatDate(date: string, locale: Locale): string {
  return new Date(date).toLocaleDateString(DATE_LOCALES[locale], {
    year: "numeric",
    month: "long",
    day: "numeric",
    timeZone: "UTC",
  });
}
```

**`timeZone: "UTC"` is required, not cosmetic.** Without it a date-only
`"2026-02-09"` is parsed as UTC midnight and rendered in the server's zone, so a
build machine west of Greenwich prints the 8th. It renders one day early, only in
some deployments, and only for some posts, the worst kind of bug to chase.

The map is keyed by `Locale`, so adding a locale is a type error rather than a
silent fallback to `en-US`. The earlier implementation used
`lang === "pl" ? "pl-PL" : "en-US"`, which formatted German posts in American
English.

## JSON-LD

```tsx
const jsonLd = {
  "@context": "https://schema.org",
  "@type": "BlogPosting",
  headline: post.title,
  description: post.excerpt,
  datePublished: post.date,
  inLanguage: lang,
  author: { "@type": "Person", name: post.author },
  image: post.coverImage ? `${SITE_URL}${post.coverImage}` : undefined,
  mainEntityOfPage: `${SITE_URL}${postPath(lang, post.slug)}`,
};

<script
  type="application/ld+json"
  // Post content is trusted (repo-authored). If it ever is not, escape `<` in
  // the serialized JSON: a `</script>` inside a title closes the tag.
  dangerouslySetInnerHTML={{ __html: JSON.stringify(jsonLd) }}
/>;
```

`inLanguage` and `mainEntityOfPage` are additions; the earlier implementation
omitted both. Both matter for a multilingual site: without `inLanguage`, three
translations look like three articles about the same thing.

`SITE_URL` is a seam: one exported constant, used by JSON-LD, the sitemap and the
feed. Never inline the domain. It ships as the placeholder `https://example.com`;
reading it from an environment variable with that fallback is fine, as long as
the build does not require it. Either way, tell the operator to set it to the
production domain before deploying: every absolute URL in the feed, the sitemap
and the JSON-LD is built from it.

## The tag route

```tsx
export const dynamicParams = false;

export async function generateStaticParams() {
  return LOCALES.flatMap((lang) =>
    getIndexableTags(lang, MIN_POSTS_PER_TAG).map((tag) => ({ lang, slug: tag.slug })),
  );
}

export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const { lang, slug } = await params;
  if (!isLocale(lang)) return {};

  const tag = getTagBySlug(slug, lang);
  if (!tag) return { title: t.no_posts_for_tag };

  return {
    title: interpolate(t.tag_heading, { tag: tag.label }),
    // Localized, interpolated, not English boilerplate under every locale.
    description: interpolate(t.tag_meta_description, { tag: tag.label, count: tag.count }),
    alternates: { canonical: tagPath(lang, tag.slug) },
  };
}
```

**No `languages` on a tag page.** Tags have no cross-locale identity, so there is
no honest hreflang cluster to declare. Canonical only.

## Sitemap

```ts
// app/sitemap.ts
import type { MetadataRoute } from "next";

import { LOCALES, type Locale } from "@/lib/blog/locales";
import { localizedPath } from "@/lib/blog/paths";
import { getAllPosts } from "@/lib/blog/posts";
import { getTagSlug } from "@/lib/blog/tag-slug";
import { getIndexableTags } from "@/lib/blog/tags";

const SITE_URL = "https://example.com";
const MIN_POSTS_PER_TAG = 1;

function url(locale: Locale, path = "/"): string {
  return `${SITE_URL}${localizedPath(locale, path)}`;
}

function parseDate(value: string | undefined, fallback: Date): Date {
  if (!value) return fallback;
  const parsed = new Date(value);
  return Number.isNaN(parsed.getTime()) ? fallback : parsed;
}

export default function sitemap(): MetadataRoute.Sitemap {
  const now = new Date();
  const byLocale = LOCALES.map((locale) => ({ locale, posts: getAllPosts(locale) }));

  const indexes = LOCALES.map((locale) => ({
    url: url(locale, "/blog"),
    lastModified: now,
    changeFrequency: "weekly" as const,
    priority: 0.8,
  }));

  const posts = byLocale.flatMap(({ locale, posts }) =>
    posts.map((post) => ({
      url: url(locale, `/blog/${post.slug}`),
      lastModified: parseDate(post.date, now),
      changeFrequency: "monthly" as const,
      priority: 0.7,
    })),
  );

  // A tag page's freshness is its newest post, not the build time; otherwise
  // every tag page claims to have changed on every deploy and lastModified
  // stops carrying information.
  const tags = byLocale.flatMap(({ locale, posts }) =>
    getIndexableTags(locale, MIN_POSTS_PER_TAG).map((tag) => {
      const newest = posts
        .filter((post) => post.tags.some((t) => getTagSlug(t) === tag.slug))
        .reduce((acc, post) => Math.max(acc, parseDate(post.date, now).getTime()), 0);
      return {
        url: url(locale, `/blog/tag/${tag.slug}`),
        lastModified: newest > 0 ? new Date(newest) : now,
        changeFrequency: "weekly" as const,
        priority: 0.6,
      };
    }),
  );

  return [...indexes, ...posts, ...tags];
}
```

`parseDate` guarding `Number.isNaN` matters: one malformed `date` in one file
otherwise puts `Invalid Date` into the XML and invalidates the whole sitemap.

## RSS feed

**Addition**: the earlier implementation had no feed. One route per locale's index.

```ts
// app/feed.xml/route.ts   (and app/[lang]/feed.xml/route.ts)
const MAX_ITEMS = 20;

function escapeXml(value: string): string {
  return value
    .replace(/&/g, "&amp;")
    .replace(/</g, "&lt;")
    .replace(/>/g, "&gt;")
    .replace(/"/g, "&quot;");
}

export function GET(): Response {
  const locale = DEFAULT_LOCALE;
  const posts = getAllPosts(locale).slice(0, MAX_ITEMS);

  const items = posts
    .map(
      (post) => `    <item>
      <title>${escapeXml(post.title)}</title>
      <link>${SITE_URL}${postPath(locale, post.slug)}</link>
      <guid isPermaLink="true">${SITE_URL}${postPath(locale, post.slug)}</guid>
      <description>${escapeXml(post.excerpt)}</description>
      <pubDate>${new Date(post.date).toUTCString()}</pubDate>
    </item>`,
    )
    .join("\n");

  const xml = `<?xml version="1.0" encoding="UTF-8"?>
<rss version="2.0" xmlns:atom="http://www.w3.org/2005/Atom">
  <channel>
    <title>${escapeXml(SITE_TITLE)}</title>
    <link>${SITE_URL}${blogIndexPath(locale)}</link>
    <description>${escapeXml(SITE_DESCRIPTION)}</description>
    <language>${locale}</language>
    <atom:link href="${SITE_URL}/feed.xml" rel="self" type="application/rss+xml" />
${items}
  </channel>
</rss>`;

  return new Response(xml, {
    headers: {
      "Content-Type": "application/rss+xml; charset=utf-8",
      "Cache-Control": "public, max-age=3600",
    },
  });
}
```

`escapeXml` is not optional. An ampersand in a title produces XML that no reader
will parse, and the failure is invisible until someone subscribes.

Link it from the layout so readers and crawlers find it:

```tsx
<link rel="alternate" type="application/rss+xml" href={`${SITE_URL}/feed.xml`} />
```

## Revalidation

| Route | `revalidate` | Why |
|---|---|---|
| index | 3600 | cheap, and picks up a fixed typo within the hour |
| post | 86400 | posts change rarely |
| tag | 3600 | tracks the index |

With content deployed alongside the app, every publish is a rebuild and
`revalidate` is a safety net, not the delivery mechanism. If you move the content
to a CMS, these become the actual publish latency: lower them and add on-demand
revalidation by tag.

## Checklist

- [ ] `generateStaticParams` passes `includeDrafts: false` explicitly
- [ ] `dynamicParams = false`, with the draft implication understood
- [ ] `lang` validated with `isLocale` before use
- [ ] Wrong-locale slug redirects before `notFound()`
- [ ] Dates formatted with `timeZone: "UTC"` and a per-locale format
- [ ] JSON-LD carries `inLanguage` and `mainEntityOfPage`
- [ ] Tag pages: canonical only, no hreflang
- [ ] Sitemap and `generateStaticParams` share one tag-eligibility function
- [ ] `parseDate` guards `Invalid Date` in the sitemap
- [ ] Feed values XML-escaped; feed linked from the layout
- [ ] `SITE_URL` defined once
