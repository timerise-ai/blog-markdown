# Testing

The content layer is pure over a directory of files, which makes it unusually
testable, provided the content root is a seam. These tests are the proof of the
claims this skill makes; they are not illustrative, they all pass.

Written for `node --test`, so they need no dependency beyond `gray-matter`,
which is installed from the package registry like any other dependency; the
registry is not an external service. Swap `node:test` / `node:assert` for
`vitest` if the host already has it: the bodies are unchanged.

The file lives at `lib/blog/blog.test.ts`, beside the modules it imports. Node's
type stripping resolves no extensionless path, so the `lib/blog` templates
import each other with explicit `.ts` extensions; set
`"allowImportingTsExtensions": true` in the host's `tsconfig.json` (Next.js
already sets the `noEmit` it requires). Wire this command to `npm test` as
written:

```bash
BLOG_CONTENT_DIR=./test/fixtures/blog node --experimental-strip-types \
  --test lib/blog/blog.test.ts
```

Copy the file and the fixtures verbatim and expect 20 passing tests. Rewriting
the imports, converting the runner or appending tests to this file breaks the
count the skill states; a test of your own goes in a file of its own.

## Fixtures

Seven files, each present to trigger one behaviour. Under `test/fixtures/blog/`.

| File | Exists to test |
|---|---|
| `en/alpha.md` | related: a duplicate, a draft target, a self-reference, a dead slug |
| `en/beta.md` | an **unquoted** date; two spellings of one tag |
| `en/gamma.md` | `draft: true` |
| `en/.DS_Store` | the non-markdown filter |
| `pl/alfa.md` | a translation with **no** `related`, the fallback path |
| `pl/beta-pl.md` | the target that fallback must resolve to |
| `pl/cytaty.md` | typographic quotes that break YAML |
| `de/` (absent) | a missing locale directory |

```yaml
# en/alpha.md
---
translationKey: "alpha"
title: "Alpha"
date: "2026-03-01"
slug: "alpha"
excerpt: "First"
tags: ["AI Agents", "Booking"]
related: ["beta", "beta", "gamma", "alpha", "does-not-exist"]
---
one two three four five
```

```yaml
# en/beta.md: note the unquoted date, and the two tag spellings
---
translationKey: "beta"
title: "Beta"
date: 2026-02-09
slug: "beta"
excerpt: "Second"
tags: ["AI agents", "ai agents"]
---
body
```

```yaml
# en/gamma.md
---
translationKey: "gamma"
title: "Gamma"
date: "2026-01-05"
slug: "gamma"
excerpt: "Third"
tags: ["Booking", "booking"]
draft: true
---
body
```

```yaml
# pl/alfa.md: deliberately has no `related`
---
translationKey: "alpha"
title: "Alfa"
date: "2026-03-01"
slug: "alfa"
excerpt: "Pierwszy"
tags: ["Rezerwacje ONLINE"]
---
body
```

```yaml
# pl/beta-pl.md
---
translationKey: "beta"
title: "Beta PL"
date: "2026-02-09"
slug: "beta-pl"
excerpt: "Drugi"
---
body
```

`pl/cytaty.md` is the important one. Its excerpt must contain raw typographic
quotes **inside** a double-quoted scalar; that is what makes YAML fail:

```yaml
---
translationKey: "quotes"
title: "Quotes"
date: "2026-04-01"
slug: "cytaty"
excerpt: "Agenci, którzy nie tylko „wiedzą", ale „działają"..."
tags: ["AI"]
---
body
```

If your editor auto-corrects those quotes the fixture stops testing anything.
Assert on `usedFallback === true` so a silently-fixed fixture fails loudly.

## The tests

```ts
// lib/blog/blog.test.ts
import { test } from "node:test";
import assert from "node:assert/strict";
import fs from "fs";
import path from "path";

import { parseFrontmatter } from "./frontmatter.ts";
import {
  getAllPosts, getPostBySlug, getPostSlugs, getRelatedPosts, readingTimeMinutes,
} from "./posts.ts";
import { getTags, getPostsByTagSlug, getTagBySlug } from "./tags.ts";
import { getTagSlug } from "./tag-slug.ts";

const post = (slug: string, locale: "en" | "pl") => {
  const p = getPostBySlug(slug, locale, { includeDrafts: true });
  assert.ok(p, `fixture ${locale}/${slug} missing`);
  return p;
};

// --- loader -----------------------------------------------------------------

test("getPostSlugs ignores non-markdown entries (.DS_Store must not crash)", () => {
  assert.ok(fs.existsSync(path.join(process.env.BLOG_CONTENT_DIR!, "en/.DS_Store")));
  assert.deepEqual(getPostSlugs("en").sort(), ["alpha", "beta", "gamma"]);
  assert.doesNotThrow(() => getAllPosts("en", { includeDrafts: true }));
});

test("a missing locale directory yields no posts instead of throwing", () => {
  assert.deepEqual(getAllPosts("de" as "en"), []);
});

test("posts sort newest first", () => {
  assert.deepEqual(
    getAllPosts("en", { includeDrafts: true }).map((p) => p.slug),
    ["alpha", "beta", "gamma"],
  );
});

test("an unquoted YAML date is normalized back to a date-only string", () => {
  assert.equal(post("beta", "en").date, "2026-02-09");
  assert.equal(typeof post("beta", "en").date, "string");
});

test("drafts are hidden unless explicitly included", () => {
  assert.equal(getPostBySlug("gamma", "en", { includeDrafts: false }), null);
  assert.equal(post("gamma", "en").draft, true);
  assert.ok(!getAllPosts("en", { includeDrafts: false }).some((p) => p.slug === "gamma"));
});

test("getPostBySlug returns null for an unknown slug rather than throwing", () => {
  assert.equal(getPostBySlug("nope", "en", { includeDrafts: true }), null);
});

test("reading time is derived from the body and never zero", () => {
  assert.equal(readingTimeMinutes("one two three"), 1);
  assert.equal(readingTimeMinutes(""), 1);
  assert.equal(readingTimeMinutes("word ".repeat(600)), 3);
  assert.ok(post("alpha", "en").readingMinutes >= 1);
});

// --- frontmatter ------------------------------------------------------------

test("a file gray-matter cannot parse still loads, and reports the fallback", () => {
  const raw = fs.readFileSync(
    path.join(process.env.BLOG_CONTENT_DIR!, "pl/cytaty.md"), "utf8");
  const parsed = parseFrontmatter(raw);
  assert.equal(parsed.usedFallback, true, "expected the line parser to be used");
  assert.equal(parsed.frontMatter.title, "Quotes");
  assert.equal(parsed.frontMatter.slug, "cytaty");
  assert.equal(post("cytaty", "pl").usedFallbackParser, true);
});

test("a failed YAML parse is not poisoned by gray-matter's cache on re-parse", () => {
  // gray-matter writes its cache entry before parsing, so a file that throws
  // leaves an empty result behind. Every parse must return the same data.
  const raw = fs.readFileSync(
    path.join(process.env.BLOG_CONTENT_DIR!, "pl/cytaty.md"), "utf8");
  const first = parseFrontmatter(raw);
  const second = parseFrontmatter(raw);
  const third = parseFrontmatter(raw);
  assert.deepEqual(second.frontMatter, first.frontMatter);
  assert.deepEqual(third.frontMatter, first.frontMatter);
  assert.equal(second.frontMatter.title, "Quotes");
  assert.equal(second.usedFallback, true);
});

test("the frontmatter block is anchored: a body rule is not a fence", () => {
  assert.throws(() => parseFrontmatter("Intro paragraph\n\n---\n\nA rule, not frontmatter.\n"));
});

test("CRLF frontmatter parses", () => {
  const { frontMatter, content } = parseFrontmatter(
    '---\r\ntitle: "CRLF"\r\nslug: "crlf"\r\n---\r\nbody\r\n');
  assert.equal(frontMatter.title, "CRLF");
  assert.match(content, /body/);
});

// --- related posts ----------------------------------------------------------

test("related resolves, drops unknown slugs, excludes self, and dedupes", () => {
  // alpha.related = [beta, beta, gamma(draft), alpha(self), does-not-exist]
  assert.deepEqual(getRelatedPosts(post("alpha", "en"), "en").map((p) => p.slug), ["beta"]);
});

test("a translation with no related falls back to the default locale's relations", () => {
  const alfa = post("alfa", "pl");
  assert.deepEqual(alfa.related, [], "fixture must have no related of its own");
  // Resolved through translationKey: EN alpha -> beta -> the PL translation.
  assert.deepEqual(getRelatedPosts(alfa, "pl").map((p) => p.slug), ["beta-pl"]);
});

test("a post never lists itself", () => {
  for (const locale of ["en", "pl"] as const)
    for (const p of getAllPosts(locale, { includeDrafts: true }))
      assert.ok(!getRelatedPosts(p, locale).some((r) => r.slug === p.slug), p.slug);
});

// --- tags -------------------------------------------------------------------

test("tags are grouped by slug across spelling variants", () => {
  const tag = getTagBySlug("ai-agents", "en");
  assert.ok(tag);
  assert.deepEqual([...tag.variants].sort(), ["AI Agents", "AI agents", "ai agents"]);
});

test("posts are matched by tag slug, so no spelling variant loses its posts", () => {
  // alpha says "AI Agents"; beta says "AI agents"/"ai agents". Both must appear.
  assert.deepEqual(
    getPostsByTagSlug("ai-agents", "en").map((p) => p.slug).sort(),
    ["alpha", "beta"],
  );
});

test("count equals the number of posts, counting a post once per tag slug", () => {
  for (const locale of ["en", "pl"] as const)
    for (const tag of getTags(locale))
      assert.equal(tag.count, getPostsByTagSlug(tag.slug, locale).length, tag.slug);
});

test("the display label is stable across calls and is one of the variants", () => {
  const a = getTags("en");
  const b = getTags("en");
  assert.deepEqual(a.map((t) => `${t.slug}:${t.label}`), b.map((t) => `${t.slug}:${t.label}`));
  for (const tag of a) assert.ok(tag.variants.includes(tag.label));
});

test("tag slugs fold diacritics and whitespace", () => {
  assert.equal(getTagSlug("Rezerwacje ONLINE"), "rezerwacje-online");
  assert.equal(getTagSlug("Integração  Ágil"), "integracao-agil"); // \s+ collapses runs
  assert.equal(getTagBySlug("rezerwacje-online", "pl")?.label, "Rezerwacje ONLINE");
});

// --- inputs are not mutated -------------------------------------------------

test("queries do not mutate the cached posts", () => {
  const before = JSON.stringify(getAllPosts("en", { includeDrafts: true }));
  getRelatedPosts(post("alpha", "en"), "en");
  getTags("en");
  getPostsByTagSlug("booking", "en");
  assert.equal(JSON.stringify(getAllPosts("en", { includeDrafts: true })), before);
});
```

## What each group proves

| Group | The claim it proves |
|---|---|
| loader | `.md` filtering, missing directories, sort order, date normalization, draft filtering, null-not-throw, reading time |
| frontmatter | the fallback runs, the cache is not poisoned, the block is anchored, CRLF parses |
| related | dead slugs dropped, self excluded, duplicates collapsed, **and the cross-locale fallback** |
| tags | variants grouped, no variant loses its posts, `count` matches `getPostsByTagSlug().length`, label stable |
| purity | queries do not mutate the memoized posts |

The two that are worth the whole file:

- **"a translation with no related falls back to the default locale's relations"**
  is the regression test for the defect that left most translated posts
  without a related section.
- **"a failed YAML parse is not poisoned by gray-matter's cache on re-parse"**
  fails against the obvious implementation. It is the only thing standing between
  you and a post that renders correctly once per process and blank thereafter.

## Not covered here

State it rather than implying coverage:

- **Rendering.** `MarkdownRenderer` and `BlogCover` are type-checked, not
  behaviour-tested. Cover them with the host's component runner if it has one.
- **Route handlers.** `generateStaticParams`, `generateMetadata` and the sitemap
  are exercised only by a real build.
- **The RSS feed.** Validate the output once against a feed validator; the
  escaping is the only logic in it.

## Checklist

- [ ] Content root is a seam (`BLOG_CONTENT_DIR`), or the tests cannot run
- [ ] Fixtures include a file that genuinely breaks YAML
- [ ] The cross-locale related fallback has a test
- [ ] The gray-matter cache has a re-parse test
- [ ] `count` / `getPostsByTagSlug().length` invariant asserted for every tag
- [ ] Uncovered areas named, not implied
