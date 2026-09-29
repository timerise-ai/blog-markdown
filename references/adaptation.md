# Adaptation contract

Everywhere this module touches its host. Fill the right-hand column in before
writing code; a seam you cannot name is a seam you will hardcode.

| Seam | The skill ships | The host supplies |
|---|---|---|
| **Content source** | `fs` + frontmatter reader over `content/blog/<locale>/`, behind `loadLocale(locale)` | its content root, or a CMS fetch with the same signature |
| **Locale set** | `LOCALES` / `DEFAULT_LOCALE` / `isLocale` in one module | its locale codes, and whether the default is prefixed |
| **Path building** | `localizedPath` / `postPath` / `tagPath` | its router, proxy rewrites, localized route slugs |
| **Strings** | keys only, plus a default-locale key block | its i18n loader, server and client |
| **Plurals** | two count keys, or `Intl.PluralRules` | its plural mechanism |
| **Date formatting** | `Record<Locale, string>` of BCP-47 tags, `timeZone: "UTC"` | its format preferences |
| **Layout chrome** | a page body; no header, footer or shell | its `Header` / `Footer` / route-group layout |
| **UI primitives** | element structure, ordering, `aria` | its Link, Image, Card, Badge |
| **Styling** | layout intent; placeholder utility classes marked as such | its design tokens, dark mode, typography scale |
| **Cover art** | registry, type guard, deterministic accent formula, 2 example motifs | its own motifs and palette |
| **Animation** | none required | framer-motion, or nothing |
| **Image component** | `<img loading="lazy">` in markdown | `next/image` for covers, if it uses one |
| **Site origin** | one `SITE_URL` constant | its production domain |
| **Validation** | a standalone Node script, no framework | its script runner and CI step |

Seams this module does **not** have, unlike most portable modules: no auth
guard, no tenant scope, no database, no object storage, no background jobs. Every
read is a file read at build time. If you find yourself adding one of those, you
have left this module's boundary: that is a CMS or a comments system, not this.

## Host probe

```bash
cat package.json | grep -A60 '"dependencies"'   # next, react, gray-matter?
                                                # react-markdown? an i18n lib?
ls content/ src/content/ 2>/dev/null            # existing content root
cat tsconfig.json | grep -A5 '"paths"'          # @/ or ~/ or relative
ls src/app app 2>/dev/null                      # App Router? a [lang] segment?
cat CLAUDE.md AGENTS.md 2>/dev/null | head -60  # house rules: follow over this
```

Then read the closest existing feature slice end to end, such as a docs section, a
changelog or a case-studies list, and copy its shape. Consistency inside one
codebase beats correctness imported from another.

## Naming

The canonical vocabulary is `Post`, `tag`, `translationKey`, `related`,
`excerpt`, `coverMotif`. Rename to the host's words as one decision before
generating, not file by file:

| Canonical | Common host words |
|---|---|
| `Post` | `Article`, `Entry`, `Story`, `Release`, `Note` |
| `blog` (route) | `articles`, `news`, `changelog`, `insights` |
| `tag` | `topic`, `category`, `label` |
| `translationKey` | `i18nKey`, `contentId`, `articleKey` |

`translationKey` is worth keeping if the host has no better word: it says what it
is for. Do **not** rename `slug`, `excerpt` or `frontmatter`: those are the
ecosystem's terms.

## Order of work

1. `locales.ts`, `frontmatter.ts`, `posts.ts`: types and loader
2. One real content file per locale, so the loader has something to read
3. `tags.ts`, `alternates.ts`, `paths.ts`
4. Routes and metadata
5. Renderer and cards: the host's primitives, last and cheapest to redo
6. Sitemap, feed, validation script

Type-check after each step. A rename fixed at step 1 is one edit; at step 5 it is
twenty.

## Checklist

- [ ] No dependency added that the host lacks, without asking
- [ ] `CLAUDE.md` / `AGENTS.md` read and followed over this skill
- [ ] Rename decided once and applied everywhere
- [ ] Host UI primitives used; placeholder classes all replaced
- [ ] Strings in the host's i18n system, in every locale file
- [ ] Content root matches the host's existing convention
- [ ] Host's lint, type-check, tests and build pass
