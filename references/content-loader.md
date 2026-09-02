# Content loader

Reading markdown off disk, parsing it, and — critically — not doing that 71 times
per page. This is the module's only I/O boundary and the only place the content
source is named, which makes it the seam a CMS replaces.

## Files

```
lib/blog/locales.ts       # locale contract, imported everywhere
lib/blog/frontmatter.ts   # YAML + fallback parser
lib/blog/posts.ts         # Post type, loader, cache, queries
```

## The locale contract

One module. The earlier implementation declared `["en", "pl", "de"]` in five
files and only one of them was the config, so adding a locale meant finding the
other four.

```ts
// lib/blog/locales.ts
export const LOCALES = ["en", "pl", "de"] as const;
export type Locale = (typeof LOCALES)[number];

export const DEFAULT_LOCALE: Locale = "en";

export function isLocale(value: string | undefined | null): value is Locale {
  return !!value && (LOCALES as readonly string[]).includes(value);
}
```

## Frontmatter parsing

```ts
// lib/blog/frontmatter.ts
import matter from "gray-matter";

export type FrontmatterValue =
  | string
  | string[]
  | number
  | boolean
  | Record<string, unknown>[];
export type Frontmatter = Record<string, FrontmatterValue>;

export type ParsedFile = {
  frontMatter: Frontmatter;
  content: string;
  /** True when YAML failed and the line parser was used. Surfaced so the
   *  content validator can warn: the fallback cannot read multi-line values. */
  usedFallback: boolean;
};

// Anchored at the start so a `---` rule inside the body is never mistaken for a
// frontmatter fence, and CRLF-tolerant so files authored on Windows parse.
const BLOCK = /^\uFEFF?---\r?\n([\s\S]*?)\r?\n---\r?\n?([\s\S]*)$/;

/**
 * Fallback for files whose frontmatter is not valid YAML — most often a
 * double-quoted scalar containing raw quote characters, which typographic
 * quotes in a translated excerpt produce. Cannot read values spanning more
 * than one line; that limitation is why `usedFallback` is reported.
 */
function parseLineByLine(block: string): Frontmatter {
  const frontMatter: Frontmatter = {};

  for (const line of block.split("\n")) {
    const colonIndex = line.indexOf(":");
    if (colonIndex === -1) continue;

    const key = line.slice(0, colonIndex).trim();
    const value = line.slice(colonIndex + 1).trim();

    if (
      (value.startsWith("[") && value.endsWith("]")) ||
      (value.startsWith("{") && value.endsWith("}"))
    ) {
      try {
        frontMatter[key] = JSON.parse(value) as FrontmatterValue;
        continue;
      } catch {
        // Not JSON either — fall through and keep it as a string.
      }
    }

    if (value !== "" && !Number.isNaN(Number(value))) {
      frontMatter[key] = Number(value);
    } else {
      frontMatter[key] =
        value.startsWith('"') && value.endsWith('"') ? value.slice(1, -1) : value;
    }
  }

  return frontMatter;
}

// YAML turns an unquoted `date: 2026-02-09` into a Date; every consumer wants a
// string, so normalize it back — keeping the time only if the source had one.
function normalize(value: unknown): unknown {
  if (value instanceof Date) {
    const iso = value.toISOString();
    return iso.endsWith("T00:00:00.000Z") ? iso.slice(0, 10) : iso;
  }
  if (Array.isArray(value)) return value.map(normalize);
  return value;
}

export function parseFrontmatter(fileContents: string): ParsedFile {
  const match = BLOCK.exec(fileContents);
  if (!match) throw new Error("Invalid markdown format: no frontmatter block");

  const [, block = "", content = ""] = match;

  let data: Record<string, unknown>;
  let usedFallback = false;
  try {
    // The `{}` is load-bearing. gray-matter memoizes by content string, and it
    // writes the cache entry BEFORE parsing — so a file that throws a YAML
    // error leaves an empty `{ data: {} }` in the cache, and every later parse
    // of that file returns no frontmatter at all, with no error. Passing any
    // options object bypasses the cache. See "The poisoned cache" below.
    data = matter(fileContents, {}).data;
  } catch {
    data = parseLineByLine(block);
    usedFallback = true;
  }

  const frontMatter: Frontmatter = {};
  for (const [key, value] of Object.entries(data)) {
    frontMatter[key] = normalize(value) as FrontmatterValue;
  }

  return { frontMatter, content, usedFallback };
}
```

### Why the fallback exists

`gray-matter` delegates to `js-yaml`, which throws on this real content file:

```yaml
excerpt: "Dziś wkraczamy w erę agentów AI, którzy nie tylko „wiedzą", ale „działają"..."
```

```
YAMLException: end of the stream or a document separator is expected
  at line 6, column 158
```

The typographic quotes close the double-quoted scalar early. **Without the
fallback the entire build fails on one file**, and the error names a YAML column
rather than a post, so it reads as a tooling problem rather than a content one.

Do not "clean up" the fallback. Do surface `usedFallback` — a file on the
fallback path loses any value written across multiple lines, so a Prettier-wrapped
`related:` array in such a file silently becomes `undefined`.

### The poisoned cache

The fallback alone is not enough, and this is the subtlest defect in the module.
`gray-matter@4` caches parse results keyed by the content string, and it writes
the entry **before** parsing:

```js
// gray-matter/index.js — abridged
let file = toFile(input);
const cached = matter.cache[file.content];
if (!options) {
  if (cached) return Object.assign({}, cached);   // <- returns the empty entry
  // only cache if there are no options passed
  matter.cache[file.content] = file;              // <- written before parsing
}
return parseMatter(file, options);                // <- this is what throws
```

So for a file whose YAML fails:

| Call | Result |
|---|---|
| 1st | throws → your fallback runs → correct frontmatter |
| 2nd | **returns `{ data: {} }` from the cache — no throw, no data** |
| 3rd+ | same |

The second parse onward yields a post with an empty title, an empty date, no
tags, and `translationKey` collapsed to the slug. Nothing errors. Reproduced on
real content:

```
call 1: title="Idealny backend dla agentów AI…" tags=["GraphQL","API","AI"] date="2025-11-27T…"
call 2: title=undefined tags=undefined date=undefined
call 3: title=undefined tags=undefined date=undefined
```

The knock-on effects are exactly the ones that are hardest to trace back: the
post sorts last (empty date), renders an untitled card, disappears from its own
tag pages (no tags), and breaks its hreflang cluster (`translationKey` is now the
slug, so its translations no longer find it).

Passing `{}` opts out of gray-matter's cache entirely — the comment in its source
says as much. You lose nothing: `loadLocale`'s own memoization is strictly better,
because it caches the finished `Post`, not a re-parse.

**Memoizing the loader hides this bug rather than fixing it** — with one parse per
file per process the cache is never read a second time. Do both. The next person
who adds a second call site should not resurrect it.

## The loader

```ts
// lib/blog/posts.ts
import fs from "fs";
import path from "path";

import { parseFrontmatter } from "./frontmatter";
import { DEFAULT_LOCALE, LOCALES, type Locale } from "./locales";

// Overridable so the loader can be pointed at a fixture directory in tests.
// Without this seam nothing below is testable without a real content tree.
const postsDirectory = process.env.BLOG_CONTENT_DIR
  ? path.resolve(process.env.BLOG_CONTENT_DIR)
  : path.join(process.cwd(), "content/blog");

const DEFAULT_AUTHOR = "Editorial";
const WORDS_PER_MINUTE = 200;

export type Post = {
  slug: string;
  translationKey: string;
  title: string;
  date: string;
  excerpt: string;
  author: string;
  content: string;
  tags: string[];
  related: string[];
  readingMinutes: number;
  coverImage?: string;
  coverMotif?: string;
  coverAccent?: string;
  draft: boolean;
  usedFallbackParser: boolean;
};

function asString(value: unknown): string | undefined {
  return typeof value === "string" && value !== "" ? value : undefined;
}

function asStringArray(value: unknown): string[] {
  return Array.isArray(value) ? value.filter((v): v is string => typeof v === "string") : [];
}

export function readingTimeMinutes(content: string): number {
  const words = content.trim().split(/\s+/).filter(Boolean).length;
  return Math.max(1, Math.round(words / WORDS_PER_MINUTE));
}

/** Filenames only — `.md` filter included. See "The .DS_Store bug" below. */
export function getPostSlugs(locale: Locale): string[] {
  const localeDirectory = path.join(postsDirectory, locale);
  if (!fs.existsSync(localeDirectory)) return [];
  return fs
    .readdirSync(localeDirectory)
    .filter((name) => name.endsWith(".md"))
    .map((name) => name.slice(0, -3));
}

function loadPost(slug: string, locale: Locale): Post {
  const fullPath = path.join(postsDirectory, locale, `${slug}.md`);
  const { frontMatter, content, usedFallback } = parseFrontmatter(
    fs.readFileSync(fullPath, "utf8"),
  );

  return {
    slug,
    translationKey: asString(frontMatter.translationKey) ?? slug,
    title: asString(frontMatter.title) ?? "",
    date: asString(frontMatter.date) ?? "",
    excerpt: asString(frontMatter.excerpt) ?? "",
    author: asString(frontMatter.author) ?? DEFAULT_AUTHOR,
    content,
    tags: asStringArray(frontMatter.tags),
    related: asStringArray(frontMatter.related),
    readingMinutes: readingTimeMinutes(content),
    coverImage: asString(frontMatter.coverImage),
    coverMotif: asString(frontMatter.coverMotif),
    coverAccent: asString(frontMatter.coverAccent),
    draft:
      frontMatter.draft === true ||
      frontMatter.draft === "true" ||
      frontMatter.status === "draft",
    usedFallbackParser: usedFallback,
  };
}

// Content is immutable for the life of a production process, so read each
// locale once. In development the cache is bypassed so edits show on reload.
const CACHE_ENABLED = process.env.NODE_ENV === "production";
const cache = new Map<Locale, Post[]>();

/** Drop the memoized content. Needed by tests, and by on-demand revalidation if
 *  you later move the content behind a CMS. */
export function clearPostCache(): void {
  cache.clear();
}

/** Every post in a locale, drafts included, newest first. */
function loadLocale(locale: Locale): Post[] {
  const cached = CACHE_ENABLED ? cache.get(locale) : undefined;
  if (cached) return cached;

  const posts = getPostSlugs(locale)
    .map((slug) => loadPost(slug, locale))
    .sort((a, b) => b.date.localeCompare(a.date));

  if (CACHE_ENABLED) cache.set(locale, posts);
  return posts;
}

export function getAllPosts(
  locale: Locale = DEFAULT_LOCALE,
  { includeDrafts = process.env.NODE_ENV === "development" } = {},
): Post[] {
  const posts = loadLocale(locale);
  return includeDrafts ? posts : posts.filter((post) => !post.draft);
}

export function getPostBySlug(
  slug: string,
  locale: Locale,
  { includeDrafts = process.env.NODE_ENV === "development" } = {},
): Post | null {
  const post = loadLocale(locale).find((p) => p.slug === slug);
  if (!post) return null;
  if (post.draft && !includeDrafts) return null;
  return post;
}

export function getPostByTranslationKey(
  translationKey: string,
  locale: Locale,
): Post | null {
  return getAllPosts(locale).find((p) => p.translationKey === translationKey) ?? null;
}
```

### `getPostBySlug` returns `null`, it does not throw

The earlier implementation threw `ENOENT` from `fs.readFileSync` and every
caller wrapped it in `try/catch`. Two consequences: a genuine disk error was
indistinguishable from a missing post, and `getRelatedPosts` used
`catch { return null }` to swallow misconfigured relations, so bad `related`
entries never surfaced anywhere. A nullable return makes "not found" a value and
lets real I/O errors propagate.

### The `.DS_Store` bug

The earlier implementation's `getPostSlugs` returned `fs.readdirSync(dir)` unfiltered and
`getPostBySlug` appended `.md` after stripping a trailing `.md`. On any macOS
checkout where Finder has opened the folder, that is `readFileSync(".DS_Store.md")`
→ `ENOENT` → **the whole build fails**, with an error naming a file the author
never created. One `.filter()` prevents it. The same applies to `.mdx`, editor
swap files, and `README.md` if you keep notes beside the content.

## Related posts, with the cross-locale fallback

```ts
/**
 * Relations are authored once, on the default-locale post, as default-locale
 * slugs. A translated file may omit `related` and still resolve correctly.
 */
export function getRelatedPosts(post: Post, locale: Locale): Post[] {
  const defaultPosts = getAllPosts(DEFAULT_LOCALE);

  const sourceSlugs = post.related.length
    ? post.related
    : (defaultPosts.find((p) => p.translationKey === post.translationKey)?.related ?? []);

  const localePosts = locale === DEFAULT_LOCALE ? defaultPosts : getAllPosts(locale);

  // Seeded with this post so an article can never list itself, and used to
  // de-duplicate when two source slugs share a translation.
  const seen = new Set<string>([post.translationKey]);
  const related: Post[] = [];

  for (const slug of sourceSlugs) {
    const source = defaultPosts.find((p) => p.slug === slug);
    if (!source || seen.has(source.translationKey)) continue;
    seen.add(source.translationKey);

    const translated = localePosts.find((p) => p.translationKey === source.translationKey);
    if (translated) related.push(translated);
  }

  return related;
}
```

## Why memoization is not optional

Measured on the earlier implementation, the build read every content file about
71 times. `generateStaticParams` accounts for one read per file per locale;
`generateMetadata`, run once per page, accounts for the rest, because it
re-reads every locale to resolve alternates.

The cause is `getAlternateSlugs`, which calls `getAllPosts` once per locale and
is called once per page. Uncached, that is quadratic in post count: doubling the
posts roughly quadruples the reads, each one a `readFileSync` plus a full
`gray-matter` parse. The `Map` above makes it one read per file, flat.

Two things make the cache safe:

- **Content cannot change while a production process lives** — it is deployed
  with the build. There is nothing to invalidate.
- **Development bypasses it**, so editing a file and reloading works.

If you must cache in development too, key the entry on the directory's `mtime`.
Do not add a TTL: a TTL makes a build non-deterministic for no benefit.

### If your content source is not the filesystem

Everything above `loadLocale` is source-agnostic. To back this with a CMS,
replace `loadLocale` with one that fetches a locale's posts and maps them onto
`Post`; keep the memoization (per request, via React `cache`, rather than per
process) and keep `translationKey` as the identity field. The rest of the
skill — tags, alternates, relations, routes — is unchanged.

## Checklist

- [ ] `LOCALES` declared once and imported; no inline locale arrays anywhere
- [ ] `getPostSlugs` filters `.md`
- [ ] Missing locale directory returns `[]`, not a throw
- [ ] `parseFrontmatter` anchored, CRLF-tolerant, with the line-parser fallback
- [ ] `matter(contents, {})` — gray-matter's own cache opted out of
- [ ] `usedFallback` surfaced to the validator
- [ ] Dates normalized to strings; sorting is `localeCompare`, not `new Date`
- [ ] Loader memoized in production, bypassed in development
- [ ] `getPostBySlug` returns `null` and takes an explicit `includeDrafts`
- [ ] `getRelatedPosts` falls back to the default locale, excludes self, dedupes
- [ ] Content root overridable, so the loader is testable against fixtures
