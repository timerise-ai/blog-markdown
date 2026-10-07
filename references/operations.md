# Operations

What breaks after launch, what an editor cannot see, and the script that catches
most of it before a build does.

## The content validator

Content defects in this module are **silent by design**: a bad `related` slug is
dropped, a missing `translationKey` degrades to the slug, a YAML failure falls
back to a weaker parser. That is correct at runtime (one bad file must not take
the site down) and useless to an author. The validator is where those become
visible.

Run it in CI and before publishing. Plain Node, no framework: copy it as
`scripts/validate-content.mjs`, not as TypeScript, so it runs before any build
without importing the app's modules. Its `LOCALES` and `slugify` are copies of
`lib/blog/locales.ts` and `getTagSlug`; change them together.

```js
// scripts/validate-content.mjs
import fs from "fs";
import path from "path";
import matter from "gray-matter";

const LOCALES = ["en", "pl", "de"];
const DEFAULT_LOCALE = "en";
const ROOT = "content/blog";
const REQUIRED = ["translationKey", "title", "date", "slug", "excerpt"];

const slugify = (t) =>
  t.toLowerCase().normalize("NFD").replace(/[\u0300-\u036f]/g, "").replace(/\s+/g, "-");

const errors = [];
const warnings = [];
const byLocale = {};

for (const locale of LOCALES) {
  const dir = path.join(ROOT, locale);
  if (!fs.existsSync(dir)) {
    warnings.push(`${locale}: no content directory`);
    byLocale[locale] = [];
    continue;
  }

  byLocale[locale] = fs
    .readdirSync(dir)
    .filter((name) => name.endsWith(".md"))
    .map((file) => {
      const source = fs.readFileSync(path.join(dir, file), "utf8");
      let data;
      try {
        data = matter(source).data;
      } catch (error) {
        // The app falls back to a line parser here, which cannot read values
        // spanning multiple lines, so a wrapped `related:` array becomes undefined.
        warnings.push(
          `${locale}/${file}: YAML failed, app will use the fallback parser ` +
            `(multi-line values will be lost): ${error.message.split("\n")[0]}`,
        );
        data = {};
      }
      for (const field of REQUIRED) {
        if (!data[field]) errors.push(`${locale}/${file}: missing "${field}"`);
      }
      if (data.slug && data.slug !== file.replace(/\.md$/, "")) {
        errors.push(`${locale}/${file}: slug "${data.slug}" does not match the filename`);
      }
      if (data.date && Number.isNaN(new Date(data.date).getTime())) {
        errors.push(`${locale}/${file}: unparseable date "${data.date}"`);
      }
      return { file, ...data };
    });
}

// related must resolve against the DEFAULT locale's slugs, in every locale.
const defaultSlugs = new Set(byLocale[DEFAULT_LOCALE].map((p) => p.slug));
for (const locale of LOCALES) {
  for (const post of byLocale[locale]) {
    for (const slug of post.related ?? []) {
      if (!defaultSlugs.has(slug)) {
        errors.push(
          `${locale}/${post.file}: related "${slug}" is not a ${DEFAULT_LOCALE} slug ` +
            `(relations are authored as ${DEFAULT_LOCALE} slugs)`,
        );
      }
    }
    if ((post.related ?? []).includes(post.slug)) {
      warnings.push(`${locale}/${post.file}: lists itself in related`);
    }
  }
}

// Translation coverage.
const keys = {};
for (const locale of LOCALES)
  for (const post of byLocale[locale]) (keys[post.translationKey] ??= []).push(locale);
for (const [key, locales] of Object.entries(keys)) {
  if (locales.length < LOCALES.length) {
    warnings.push(
      `translationKey "${key}": only ${locales.join(", ")}, missing ` +
        LOCALES.filter((l) => !locales.includes(l)).join(", "),
    );
  }
}

// Tag spelling drift: two labels that slug the same.
for (const locale of LOCALES) {
  const groups = new Map();
  for (const post of byLocale[locale])
    for (const tag of post.tags ?? []) {
      const slug = slugify(tag);
      groups.set(slug, (groups.get(slug) ?? new Set()).add(tag));
    }
  for (const [slug, variants] of groups) {
    if (variants.size > 1) {
      warnings.push(`${locale}: tag "${slug}" has variants {${[...variants].join(" | ")}}`);
    }
  }
}

for (const warning of warnings) console.warn(`warn  ${warning}`);
for (const error of errors) console.error(`error ${error}`);
console.log(`\n${errors.length} errors, ${warnings.length} warnings`);
process.exit(errors.length > 0 ? 1 : 0);
```

Errors fail the build; warnings do not. The split matters: a missing German
translation is a normal state of the world, a `related` slug pointing nowhere is
a mistake.

Wire it in:

```json
{ "scripts": { "validate:content": "node scripts/validate-content.mjs" } }
```

Run it in CI on every content change, and in `prebuild` if your CI is slow to
give feedback.

## Build cost

The dominant cost is not markdown parsing, it is repeated loading. Measured on
the earlier implementation before memoization, the build read every content
file about 71 times: once per post per page per locale, because alternate-slug
resolution re-reads every locale on every page. That is quadratic in post
count, so doubling the posts roughly quadruples the reads. Memoized, the build
reads each file once.

If a build slows down as content grows, check for an uncached `getAllPosts` in a
per-page function before anything else. `generateMetadata` is the usual site,
because alternate-slug resolution has to touch every locale.

Sanity check on any content-heavy build:

```bash
# Reads of content files during a build should be ~= the number of source files.
NODE_OPTIONS=--stack-trace-limit=50 npm run build 2>&1 | tail -30
```

## What an editor cannot see

The honest gap list. Everything here was absent from the earlier implementation;
each is a documented extension, not shipped code.

| Gap | Consequence | Cheapest fix |
|---|---|---|
| No draft preview | An unpublished post can only be viewed by running the site locally | A preview route keyed on a signed token that passes `includeDrafts: true` |
| No publish date in the future | A post is live the moment it merges | Filter `post.date > now` in production; schedule a rebuild |
| No content dashboard | Coverage gaps are invisible until someone notices | The validator's warnings, printed in CI's summary |
| No "where is this linked from" | Renaming a slug silently breaks every `related` pointing at it | Reverse index over `related` in the validator |
| No word/reading-time targets | Length drifts | `readingMinutes` is already computed; surface it in the validator |
| No redirect map | Renaming a published slug 404s an indexed URL | A `redirects` entry per rename; make it part of the rename ritual |

The last one is worth a rule: **a published slug is an API.** Renaming it is a
breaking change that needs a permanent redirect, and nothing in this module
enforces that.

## Authoring workflow

1. Copy the newest post in the default locale as a template.
2. Change `translationKey`, `slug`, `title`, `date`, `excerpt`; rename the file to
   match the slug.
3. Reuse tag spellings: run the validator and check the drift warnings.
4. Add `related` (default-locale slugs). Translations may omit it.
5. Add the same `translationKey` to each translation.
6. `npm run validate:content`, then build and open the post in every locale.

## Failure modes and fixes

| Symptom | Cause | Fix |
|---|---|---|
| Build fails with `ENOENT ... .DS_Store.md` | `getPostSlugs` did not filter `.md` | `.filter(name => name.endsWith(".md"))` |
| Build fails with `YAMLException: end of the stream` | Typographic quotes inside a double-quoted scalar | The fallback parser; then fix the file |
| A translated post's `related` is silently `undefined` | That file is on the fallback parser and its array is wrapped across lines | Put the array on one line, or fix the quoting |
| "Further reading" empty in translations only | `related` authored only in the default locale, with no fallback | The `translationKey` fallback in `getRelatedPosts` |
| Posts missing from their own tag page | Tag matched by exact label after a lossy slug round trip | Match by slug |
| A date renders one day early | `toLocaleDateString` without `timeZone: "UTC"` | Set it |
| Sitemap rejected as invalid | `Invalid Date` from one malformed `date` | `parseDate` with a `Number.isNaN` guard |
| Every tag page's `lastmod` equals the deploy time | `lastModified: now` | Derive from the newest post carrying the tag |

## Checklist

- [ ] Validator wired into CI; errors fail, warnings do not
- [ ] Loader memoization verified by build-time read count
- [ ] Slug renames paired with permanent redirects
- [ ] Draft preview decided: local-only, or a token route
- [ ] Tag drift warnings triaged rather than accumulated
