# blog-markdown

An [Agent Skill](https://agentskills.io) that teaches an agent to build a file-based, multilingual blog in a
**Next.js App Router** app: locale-partitioned markdown posts, per-locale slugs joined by a shared translation
key, tag pages, related posts, cover art, RSS, sitemap, hreflang, and a content validator for CI. Everything
is statically generated at build time. Content lives in the repository; there is no CMS.

The whole design turns on one idea: **a post's identity is its `translationKey`, not its slug.** Slugs are
localized for SEO and differ per language; the key is what makes three files one article, and it is what
powers hreflang, the language switcher, related posts and cross-locale redirects. This skill was written by
the engineer who built a production marketing site on this design. The audit of that earlier implementation
found twelve defects, every one of them silent, and the templates are built so none can recur without a build
failing or a test going red.

The filesystem is a seam, not a premise: reading happens behind one `loadLocale(locale)` function, so a CMS
fetch with the same signature replaces it without touching the routes, the tag pages or the feeds.

## Install

One command, via the [skills.sh](https://www.skills.sh) CLI, which installs the skill into every
skills-compatible agent it detects, including Claude Code, Codex CLI and Gemini CLI:

```bash
npx skills add timerise-ai/blog-markdown
```

Name the agents instead with `-a`, for example `npx skills add timerise-ai/blog-markdown -a claude-code -a codex`.

Or clone it yourself. Nothing here is Claude-specific: the skill is a plain [Agent
Skills](https://agentskills.io) folder, `SKILL.md` plus markdown references with no file that calls a model,
so cloning it into an agent's skills directory is all an install is. For Claude Code:

```bash
git clone https://github.com/timerise-ai/blog-markdown.git ~/.claude/skills/blog-markdown
```

To scope it to a single project instead, clone it into that project's `.claude/skills/` directory. For another
agent, clone into that agent's skills directory, or symlink the Claude Code copy so one `git pull` updates
every agent:

```bash
mkdir -p ~/.agents/skills
ln -s ~/.claude/skills/blog-markdown ~/.agents/skills/blog-markdown
```

Update the skill with `git pull` in its directory. The current release is **0.1.4**. See
[`CHANGELOG.md`](CHANGELOG.md). The [skills index](https://github.com/timerise-ai/skills) lists the other
Timerise Skills and how to install them all at once.

## Activation

The skill activates automatically when a task matches its description: adding a blog, changelog, news or
article section backed by markdown files rather than a CMS, making an existing markdown blog multilingual, or
adding tag pages, hreflang, feeds or related posts. Invoke it explicitly with `/blog-markdown` in Claude Code,
`$blog-markdown` in Codex CLI, or from `/skills` in Gemini CLI.

Each host matches a task against the description its own way, so invoke the skill explicitly on a first run
rather than assuming it fired. Only `SKILL.md` is read up front; the `references/` files load on demand, so
the skill stays cheap in context until a topic is actually needed.

## What's inside

| File | Contents |
|---|---|
| `SKILL.md` | Entry point: architecture, critical facts, hard rules, and the reference directory |
| `references/adaptation.md` | The seam contract with the host app:  content source, styling, i18n, routing, the rename |
| `references/content-model.md` | Frontmatter schema, drafts, translation keys, relations, excerpts |
| `references/content-loader.md` | Reading files into posts: gray-matter, the fallback parser, memoization, reading time |
| `references/i18n-and-routing.md` | Locales, alternates, hreflang, canonicals, language switcher, cross-locale redirects |
| `references/tags.md` | Tag slug identity, collisions, tag pages, thin-tag thresholds |
| `references/pages-and-seo.md` | The three routes, `generateStaticParams`, metadata, JSON-LD, sitemap, RSS |
| `references/rendering.md` | Markdown to React, code blocks, cover motifs and accents, the XSS boundary |
| `references/operations.md` | The content validation script, build cost, authoring workflow |
| `references/testing.md` | Seven fixtures and the 20-test content-layer suite |
| `references/provenance.md` | The twelve defects found in the earlier implementation and how the templates fix them |

## The four non-negotiables

These travel with the module and are never optional. Each one was live in the earlier implementation (see
`references/provenance.md`):

1. **`gray-matter` caches a failed parse as an empty result.** It writes its cache entry before parsing, so
   the second and every later parse of a file whose YAML throws returns `{}`, with no error and no
   frontmatter. Always call `matter(contents, {})`; any options object opts out of the cache.
2. **Relations are authored once, in the default locale.** A translated file that omits `related` must still
   resolve through `translationKey`, or the section quietly does not render.
3. **A tag's identity is its slug, not its label.** `"AI Agents"` and `"AI agents"` slug to the same URL;
   group and match by slug or one spelling variant's posts vanish from their own tag page.
4. **Loading is O(posts x locales) per page unless you memoize.** Measured on the earlier implementation: a
   71x file-read amplification at build time, all from resolving alternates during `generateMetadata`.

Everything else is the host app's: styling, naming, renderer, i18n system, cover artwork.

## Not this

| Not this | Use instead |
|---|---|
| A hosted CMS as the source of truth | That CMS's SDK; keep only `references/content-model.md` |
| A docs site or help center with categories, sidebar and search | The sibling [`help-center-markdown`](https://github.com/timerise-ai/help-center-markdown) skill, a different navigation model |
| A single-locale blog of under ten posts | Plain `fs` plus `gray-matter` inline; this skill is overhead there |
| Rendering user-submitted markdown | A sanitizing renderer; see the XSS note in `references/rendering.md` |

## Contributing

Issues and pull requests are welcome here. Pure markdown, with no build, lint or test step in this repository.
Code blocks name their destination on the first line, and every template is written to compile under `strict`
and `noUncheckedIndexedAccess`, so keep imports complete and types explicit. Claims in this skill are meant to
be verifiable: if you change a factual claim, say how you verified it, whether against the library, the docs,
or a reproduction.

Adding, removing or renaming a file in `references/` means updating the quick start and the reference
directory table in `SKILL.md`, the file table above, and any relative cross-links. The odd-looking parts of
the templates encode documented defects, and `references/provenance.md` is the ledger that must stay truthful:
read it before simplifying anything, and add an entry for anything you change. Commits follow Conventional
Commits and releases follow [STANDARD.md](https://github.com/timerise-ai/skills/blob/main/STANDARD.md) in the
index; `CLAUDE.md` carries the full editing conventions.
## Part of the Timerise Skills

This is one of the [Timerise Skills](https://github.com/timerise-ai/skills): modules for **Next.js App
Router** apps written by our own senior engineers from the modules they have shipped, not synthetic, each
published as its own repository and indexed there. They share one layout, so an agent that has read one knows
how to read the next: a `SKILL.md` entry point, `references/` loaded on demand, and a seam contract carrying
the module's non-negotiables.

## Author

Built and maintained by [Timerise](https://timerise.ai).

## License

MIT. See [LICENSE](LICENSE).