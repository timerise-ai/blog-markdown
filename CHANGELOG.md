# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.6] - 2026-09-21

Wording release. The skill content is unchanged from 0.1.5.

### Added

- `SKILL.md` closes with a line linking the
  [Timerise Skills](https://github.com/timerise-ai/skills) index, so an agent that
  has the skill loaded can find the sibling skills for neighbouring modules without
  leaving the entry point.

### Changed

- `CLAUDE.md` records the closing line in the `SKILL.md` layout.

## [0.1.5] - 2026-09-02

Wording release. Templates and technical content are unchanged from 0.1.4.

### Changed
- The front door, `README.md`, `SKILL.md` and `CLAUDE.md`, describes the module by the
  properties the content-layer suite and the build verify; the audit record stays in
  `references/provenance.md`, linked rather than summarised.

## [0.1.4] - 2026-09-02

Wording release. Templates and technical content are unchanged from 0.1.3.

### Changed
- The corpus figures of the earlier implementation (posts, tags, pages, locales, file
  reads and their ratios) are gone from `SKILL.md`, `README.md`, `CLAUDE.md` and
  `references/`, replaced by the shape of each finding. The 71x read amplification and
  the skill's own test, fixture and defect counts stay.

## [0.1.3] - 2026-09-02

Wording release. The origin and audit statements across the skill follow section 2 of
the skill standard; templates and technical content are unchanged from 0.1.2. The
repository history starts at this release.

### Changed
- Origin and audit wording across `SKILL.md`, `CLAUDE.md`, `README.md` and `references/`
  now follows the skill standard: the reference point is the earlier implementation,
  stated in the standard's own words. The frontmatter
  example in `content-model.md` uses a placeholder author and neutral related slugs.

## [0.1.2] - 2026-09-02

Documentation-only release. The skill itself, `SKILL.md` and `references/`, is
unchanged from 0.1.1.

### Changed
- README: the install leads with `npx skills add timerise-ai/blog-markdown`, which installs the skill
  into every skills-compatible agent it detects, with the `-a` form for named agents; the
  Claude Code clone moves under a *Manual install* heading. Activation gets its own
  heading, and a *Not this* table points neighbouring problems to the right skill or tool.
- README: the skill's origin is reworded. It was written by the engineers who built the
  module it describes; the reference point for `provenance.md` is the earlier
  implementation rather than "the source"; the index is called Timerise Skills.
- README: every em-dash, arrow and en-dash in the prose is rewritten as a comma, colon,
  full stop or conjunction.

## [0.1.1] - 2026-09-01

Documentation-only release. The skill itself — `SKILL.md` and `references/` —
is unchanged from 0.1.0.

### Added
- README: one-command install via [skills.sh](https://www.skills.sh)
  (`npx skills add timerise-ai/blog-markdown`), the `~/.agents/skills` path used
  by Codex CLI and Gemini CLI, the symlink that keeps one clone in sync across
  hosts, and the per-host invocation and activation notes.

### Changed
- README reframed from "Claude Code skill" to
  [Agent Skill](https://agentskills.io): `SKILL.md` declares only `name` and
  `description` and no file calls a model, so any skills-compatible host reads it.
- README claims corrected against `SKILL.md` and `references/`: the CI content
  validator and build-time static generation are now in the pitch, the content
  source is stated as a seam behind `loadLocale(locale)`, and `references/testing.md`
  is credited with seven fixtures.

## [0.1.0] - 2026-08-30

Initial release of the `blog-markdown` skill, extracted from a production
marketing site: 3 locales x 23 posts, ~46 tags, 69 statically generated pages.

### Added
- `SKILL.md` entry point: the architecture diagram, seven critical facts, four
  hard rules, the quick-start order, and the reference directory table mapping
  trigger keywords to `references/`.
- `references/content-model.md` — frontmatter schema, drafts, `translationKey`,
  relations and excerpts.
- `references/content-loader.md` — reading files into posts: `gray-matter`, the
  line-by-line fallback parser, per-locale memoization, reading time.
- `references/i18n-and-routing.md` — locales, alternates, hreflang, canonicals,
  the language switcher and cross-locale redirects.
- `references/tags.md` — tag slug identity, collisions, tag pages, thin-tag
  thresholds.
- `references/pages-and-seo.md` — the three routes, `generateStaticParams`,
  metadata, JSON-LD, sitemap and RSS.
- `references/rendering.md` — markdown to React, code blocks, cover motifs and
  accents, the XSS boundary.
- `references/operations.md` — the content validation script, build cost and the
  authoring workflow.
- `references/testing.md` — seven fixtures and the 20-test content-layer suite.
- `references/adaptation.md` — the seam contract with the host app: content
  source, styling, i18n, routing, and the locale/segment rename procedure.
- `references/provenance.md` — the twelve defects the extraction audit found in
  the source app, what was kept deliberately, what was added and what was left
  behind.
- `README.md` — install, the reference file table, the four non-negotiables,
  contributing conventions, and the link to the
  [Timerise skills index](https://github.com/timerise-ai/skills).
- `CLAUDE.md` — editing conventions for this repository.
- `LICENSE` — MIT.
