# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

A **Claude Code skill package** — markdown only. There is no build, no lint, no test runner and no
`package.json` here; nothing in this repo executes. It teaches an agent how to build a file-based,
multilingual markdown blog (locale-partitioned posts, per-locale slugs joined by `translationKey`, tag pages,
related posts, cover art, RSS, sitemap, hreflang) inside a **host** Next.js App Router app.

Keep the two straight: commands and code in `references/` describe the app the agent will generate, not this
repository. The `node --experimental-strip-types --test` invocation in `references/testing.md` runs in that
generated app, against fixtures the agent creates there.

The skill was written by the engineer who has shipped this module; the earlier implementation it was audited
against was a multi-locale, statically generated marketing blog. `references/provenance.md` is the ledger of
that audit: twelve entries on what changed and how the templates verify it, what was kept deliberately, what
was added and what was left behind. That file is the rationale layer: read it before "simplifying" anything.

## Structure

- `SKILL.md` — entry point. The frontmatter `description` is the trigger surface; the body carries the
  architecture diagram, seven **critical facts**, four **hard rules**, the quick-start order, and the
  **reference directory table** mapping trigger keywords to files.
- `README.md`: the human-facing front door: install, the file table, and the four non-negotiables.
- `references/*.md` — one topic per file, loaded on demand. `adaptation.md` (the seam contract) and
  `content-model.md` (frontmatter schema) are the design entry points; `content-loader.md`,
  `i18n-and-routing.md`, `tags.md`, `pages-and-seo.md` and `rendering.md` carry the templates;
  `operations.md`, `testing.md` and `provenance.md` carry the validator, the suite and the audit.

## Editing conventions

- **Code blocks name their destination on the first line** as a comment — `// lib/blog/posts.ts`,
  `// app/[lang]/blog/[slug]/page.tsx`, `// scripts/validate-content.mjs`. Continuation blocks that extend a
  file already introduced omit it. Templates are written to compile under `strict` and
  `noUncheckedIndexedAccess`; keep imports complete and types explicit enough to hold that claim, which
  `provenance.md` makes in the reader's name.
- **Identifiers are shared across files.** `getAllPosts`, `getPostBySlug`, `getRelatedPosts`,
  `parseFrontmatter`, `LOCALES`, `DEFAULT_LOCALE`, `localizedPath` / `postPath` / `tagPath`, `getTagSlug`,
  `getIndexableTags`, `TagSummary`, `BLOG_CONTENT_DIR`, `clearPostCache`, `usedFallbackParser` and the
  frontmatter field names (`translationKey`, `related`, `excerpt`, `coverMotif`, `draft`) appear in several
  references. Rename in all of them or none.
- **Keep the three tables in sync** with `references/`: the reference directory in `SKILL.md`, the quick-start
  list in `SKILL.md`, and the file table in `README.md`. Links are relative — `[x.md](references/x.md)` from
  `SKILL.md`, `[x.md](x.md)` between references.
- **Do not remove the odd-looking parts.** The empty options object in `matter(fileContents, {})`, the
  line-by-line frontmatter fallback, the `.md` filter in `getPostSlugs`, the default-locale `related`
  fallback resolved through `translationKey`, grouping and matching tags by slug rather than label, the
  loader memoization, the explicit draft filter, `timeZone: "UTC"` in date formatting, and the language
  switcher returning the blog index on a tag URL: each is a ledger entry from the audit of the earlier
  implementation or a documented judgement call. Check `provenance.md` before touching one.
- **The numbers that remain are load-bearing.** The 71x read amplification, 20 tests, twelve ledger entries,
  seven fixtures. They were verified against the earlier implementation or this repository; do not restate
  them loosely and do not add new ones. Corpus figures of the earlier implementation (how many posts, tags,
  pages, locales) do not appear anywhere: describe the shape of a finding instead.
- **Mark additions as additions.** Anything designed in the skill and never run in the earlier implementation
  belongs in the "Added" section of `provenance.md`, stated as such. The skill's credibility is that it
  distinguishes the two.
- **Never present the load-bearing facts as optional.** `translationKey` (not the slug) as the cross-locale
  join, tag slug as the tag's identity, `matter(contents, {})`, and loader memoization are stated as
  non-negotiable in `SKILL.md` and `README.md`; keep them that way everywhere.
- Locale codes `en` / `pl` / `de` and the `blog` route segment are the earlier implementation's and are meant
  to be renamed by the host. The rename procedure and the canonical-to-host vocabulary table are in
  `adaptation.md`. Frontmatter field names are the authoring contract and are not renamed.
