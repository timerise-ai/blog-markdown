# Provenance

Written by the engineer who has shipped this module. The earlier implementation
it was audited against was the blog module of a Next.js 16 marketing site:
multi-locale, statically generated, with its sitemap and SEO surface. A small
content layer plus routes; its brand-specific SVG cover motifs were deliberately
left behind.

**Fidelity: hardened.** The templates are the earlier implementation with its
defects fixed and its gaps filled. Every deviation is listed below. Nothing is
transcribed unverified: each claim was checked against it, measured, or
reproduced. Every code template compiles under `strict` and
`--noUncheckedIndexedAccess`; the content layer has 20 passing tests
([testing.md](testing.md)). Route handlers, the renderer and the feed are
type-checked but not behaviour-tested — stated as such rather than implied.

## Fixed in the templates

### 1. gray-matter's cache turns one bad file into a blank post, forever

The highest-severity finding, live in the earlier implementation. `gray-matter@4`
memoizes by content string and writes its cache entry **before** parsing, so a
file whose YAML throws leaves an empty `{ data: {} }` behind. The module's
fallback parser runs on the first parse and produces correct frontmatter; the
second and every later parse of that file returns `{}` — **no error, no
frontmatter** — and the fallback never runs again.

Reproduced against the real content file that fails YAML:

```
call 1: title="(the post's title)" tags=["GraphQL","API","AI"] date="2025-11-27T…"
call 2: title=undefined tags=undefined date=undefined
call 3: title=undefined tags=undefined date=undefined
```

Because the earlier implementation called `getAllPosts` dozens of times per
build, the first parse was consumed by `generateStaticParams` and every rendered
page got the empty version: an untitled card, an empty excerpt, an empty date sorting the post
last, no tags (so it vanished from its own tag pages), and a `translationKey`
collapsed to the slug, which broke its hreflang cluster.

**Shipped:** `matter(fileContents, {})`. Any options object opts out of the
cache, per gray-matter's own source comment. Loader memoization would also mask
this, which is exactly why both are shipped — masking is not fixing.
[content-loader.md](content-loader.md), regression test in
[testing.md](testing.md)

### 2. Translated posts silently lost their related-posts section

`getRelatedPosts` read the current file's own `related` array. Relations are
authored as default-locale slugs, and translators generally did not copy the
block, so the section simply did not render. Measured on the earlier
implementation's content: **every default-locale post had related posts; most
translated posts had none.** No error, no empty state: the section was
conditional on `post.related?.length`, so it vanished.

**Shipped:** `getRelatedPosts` falls back to the default-locale sibling's
`related` via `translationKey`.
[content-loader.md](content-loader.md), [content-model.md](content-model.md)

### 3. 71x file-read amplification at build time

No memoization anywhere in the content layer. `getAlternateSlugs` calls
`getAllPosts` once per locale and runs once per page in `generateMetadata`.
Measured: **every content file read about 71 times per build**, each read
followed by a full `gray-matter` parse. The cost is quadratic in post count.

**Shipped:** a per-locale `Map` cache, enabled in production only, bypassed in
development so edits still hot-reload. Each file is read once.
[content-loader.md](content-loader.md)

### 4. A stray dotfile fails the entire build

`getPostSlugs` returned every directory entry unfiltered; `getPostBySlug`
appended `.md`. On any macOS checkout where Finder has opened the content folder,
that is `readFileSync(".DS_Store.md")` → `ENOENT` → build failure, reported
against a file the author never created.

**Shipped:** `.filter(name => name.endsWith(".md"))`.
[content-loader.md](content-loader.md)

### 5. Tag pages lose posts to spelling drift

`getTagFromSlug` resolved a URL slug to the first matching label alphabetically,
then posts were filtered by that exact label. Two spellings of one tag
(`"AI Agents"` / `"AI agents"`) slug identically, so one variant's posts were
absent from the only page they belonged on — and `generateStaticParams` emitted
duplicate params. Not triggered in the earlier implementation (checked: every
locale's tag slugs, zero collisions on audit day), but the mechanism is live
and hand-authored frontmatter drifts.

**Shipped:** tags grouped by slug, label chosen by frequency with an alphabetical
tiebreak, posts matched by slug. [tags.md](tags.md)

### 6. Every card claimed "5 min read"

A literal `5` in the card component, on every post, in every language.

**Shipped:** `readingMinutes` computed from the body at load time and passed as a
prop, so no client component receives the body.
[content-loader.md](content-loader.md), [rendering.md](rendering.md)

### 7. Draft visibility depended on a routing flag

`getPostBySlug` did not filter drafts. Drafts stayed out of production only
because `dynamicParams = false` meant an unpublished slug was never a valid
route. Enabling dynamic params for any reason — a legitimate change nobody would
connect to drafts — would make every draft publicly reachable.

**Shipped:** an explicit `includeDrafts` option on `getPostBySlug` and
`getAllPosts`, defaulted from `NODE_ENV`, and `includeDrafts: false` passed
explicitly in `generateStaticParams`.
[content-model.md](content-model.md), [pages-and-seo.md](pages-and-seo.md)

### 8. Locale list declared five times

`["en", "pl", "de"]` appeared in the loader (twice), both blog route files, and
the path helper — while a `LOCALES` config module existed and was used only by
the sitemap. Adding a locale meant finding four other copies.

**Shipped:** one `locales.ts` with `LOCALES`, `DEFAULT_LOCALE` and an `isLocale`
type guard, imported everywhere. [content-loader.md](content-loader.md)

### 9. Untranslatable strings on the tag page

The tag page carried an inline `{ en, pl, de }` map with an English `||`
fallback — so a fourth locale would serve English with no missing-key signal —
and its meta description hardcoded English marketing copy into every locale's tag
pages. Pluralization used `article${n !== 1 ? "s" : ""}`, an English rule in code.

**Shipped:** i18n keys with `{tag}` / `{count}` interpolation and per-locale
plural keys. [i18n-and-routing.md](i18n-and-routing.md)

### 10. Dates formatted as American English for German readers

`lang === "pl" ? "pl-PL" : "en-US"`. Also, the post page passed
`timeZone: "UTC"` and the card did not, so the same date could render one day
apart on the index and the article.

**Shipped:** a `Record<Locale, string>` of BCP-47 tags — adding a locale is a
type error, not a silent fallback — with `timeZone: "UTC"` everywhere.
[pages-and-seo.md](pages-and-seo.md)

### 11. `getPostBySlug` threw, and callers swallowed it

Every caller wrapped it in `try/catch`, which made a real disk error
indistinguishable from a missing post and let `getRelatedPosts` discard
misconfigured relations with `catch { return null }`.

**Shipped:** returns `Post | null`; genuine I/O errors propagate.
[content-loader.md](content-loader.md)

### 12. Frontmatter regex unanchored and CRLF-intolerant

`/---\n([\s\S]*?)\n---\n([\s\S]*)/` matched a `---` horizontal rule in the body
of a file with no frontmatter, and failed outright on CRLF line endings.

**Shipped:** anchored at start, optional BOM, `\r?\n` throughout.
[content-loader.md](content-loader.md)

## Kept deliberately

- **The line-by-line frontmatter fallback.** It looks like dead legacy code. It
  is the only reason one real content file builds: `gray-matter` throws
  `YAMLException` on a double-quoted excerpt containing typographic quotes.
  Verified by running the parser over the earlier implementation's content.
  Removing it fails the build on that file. What changed is that the fallback
  now *reports* itself, so the validator can warn that multi-line values were
  lost.
- **`translationKey` with per-locale slugs.** More moving parts than one shared
  slug, and the right trade: localized URLs are worth real SEO, and the key keeps
  hreflang and the switcher correct.
- **`related` authored as default-locale slugs.** A link is written once, in one
  place, rather than three times in three vocabularies.
- **Tag pages excluded from the language switcher.** Tags are per-locale free
  text with no recorded cross-locale identity, so there is no correct target.
  Falling back to the blog index is honest; constructing a target URL is a 404.
- **`dynamicParams = false`.** Documented in the earlier implementation as a
  workaround for Next 16.2 ignoring segment-level `not-found` on `notFound()`
  (vercel/next.js#90837). The comment is kept with the code.
- **Drafts visible in development.** Correct for a repo-authored blog.
- **Sorting by `date.localeCompare`.** Correct for ISO strings of either length,
  and avoids constructing a `Date` per comparison.
- **hreflang falling back to the locale's blog index** for untranslated locales.
  A stricter cluster would omit them; that gives the switcher a dead end. Called
  out as a judgement call rather than presented as the only option.

## Added

Additions, designed in the skill and never run in the earlier implementation:

- **RSS feed** with XML escaping and a self-referencing `atom:link`. The earlier
  implementation had no feed. [pages-and-seo.md](pages-and-seo.md)
- **`inLanguage` and `mainEntityOfPage` in the JSON-LD.** Without `inLanguage`,
  three translations look like three articles about the same thing.
- **The content validation script.** [operations.md](operations.md)
- **`getIndexableTags(locale, minPosts)`,** defaulted to 1 so behaviour is
  unchanged. The earlier implementation generated a page for every tag, about
  two tags per post, so many pages held a single card. The threshold is
  offered, not imposed. [tags.md](tags.md)
- **`variants` on `TagSummary`** as a spelling-drift signal.
- **`usedFallbackParser` on `Post`,** so a weak parse is visible.
- **External-link `rel="noopener noreferrer"`** in the markdown renderer.
- **A testability seam** (`BLOG_CONTENT_DIR`) and `clearPostCache()`. Nothing in
  the earlier implementation could be tested without a real content tree.
- **A 20-test suite** covering every claim above, plus fixtures that reproduce
  the YAML failure. [testing.md](testing.md)
- **The XSS trust boundary, written down.** The earlier implementation was safe,
  with no `rehype-raw`, but nothing said so, and the next person adding an embed
  would have removed the boundary without knowing it existed.
  [rendering.md](rendering.md)

## Left behind

- **The SVG cover motifs.** Brand artwork, not capability. The registry, the type
  guard, the deterministic accent hash and two example motifs are shipped; draw
  your own. [rendering.md](rendering.md)
- **The host's design tokens, `Header`/`Footer`, motion provider, chat widget and
  contact CTA.** Host chrome. The CTA in particular was a blog-adjacent marketing
  component with no blog logic in it.

## If you are fixing the earlier implementation in place

Fix order, most damaging first:

1. **`matter(contents, {})`** (#1) — one live post is currently rendering with no
   title, no tags and a broken hreflang cluster. One argument.
2. **Related-posts fallback** (#2) — user-visible on two thirds of translated
   posts, right now.
3. **`.md` filter** (#4) — one character away from a failed build on any macOS
   checkout.
4. **Memoization** (#3) — build time, and it gets worse as content grows.
5. **Draft guard** (#7) — latent, but the failure mode is publishing something
   unpublished.
6. **Tag grouping by slug** (#5) — latent today; triggered by one inconsistent
   tag spelling.
7. **Reading time** (#6) — small, visible, trivially fixed.
8. **Locale config, tag-page strings, date formatting** (#8, #9, #10) — quality
   and maintenance; do them together.
