# Rendering

Markdown body to React, and the cover system. Both are places where the host's
design system takes over — the skill supplies structure, the host supplies looks.

## Markdown renderer

`react-markdown` with a component map. The map is the seam: every entry is a
place the host substitutes its own typography.

```tsx
// components/blog/MarkdownRenderer.tsx
import React from "react";
import ReactMarkdown, { type Components } from "react-markdown";
import remarkGfm from "remark-gfm";
import { highlight } from "sugar-high";

/** Builds a tag renderer with fixed classes. Replace the class strings with the
 *  host's typography tokens; keep the tag mapping. */
function styled(
  tag: keyof React.JSX.IntrinsicElements,
  className: string,
): React.FC<{ children?: React.ReactNode }> {
  function StyledComponent({ children }: { children?: React.ReactNode }) {
    return React.createElement(tag, { className }, children);
  }
  StyledComponent.displayName = `Styled(${tag})`;
  return StyledComponent;
}

const H2_CLASS = "text-xl font-bold mt-10 mb-4";

const markdownComponents: Components = {
  // The page owns the h1. If deep markdown carries one, demote it rather than
  // emitting a second h1 — two h1s on a post is an accessibility and SEO defect.
  h1: styled("h2", H2_CLASS),
  h2: styled("h2", H2_CLASS),
  h3: styled("h3", "text-lg font-bold mt-8 mb-3"),
  h4: styled("h4", "text-base font-bold mt-8 mb-3"),
  p: styled("p", "mb-6"),
  ul: styled("ul", "mb-6 pl-6 list-disc"),
  ol: styled("ol", "mb-6 pl-6 list-decimal"),
  li: styled("li", "mb-2"),
  blockquote: styled("blockquote", "border-l-4 pl-6 my-8 italic"),

  a: ({ href, children }) => {
    const external = !!href && /^https?:\/\//.test(href);
    return (
      <a
        href={href}
        // Outbound links get noopener/noreferrer; internal ones must not, or
        // they lose the referrer in your own analytics.
        {...(external ? { target: "_blank", rel: "noopener noreferrer" } : {})}
      >
        {children}
      </a>
    );
  },

  img: ({ src, alt }) => (
    <img
      src={typeof src === "string" ? src : undefined}
      alt={alt ?? ""}
      loading="lazy"
      className="my-8 h-auto max-w-full rounded-2xl"
    />
  ),

  // Tables need their own scroll container. Without it a wide table makes the
  // whole page scroll horizontally on mobile.
  table: ({ children }) => (
    <div className="my-8 overflow-x-auto">
      <table className="w-full border-collapse">{children}</table>
    </div>
  ),
  th: styled("th", "px-5 py-4 text-left font-semibold"),
  td: styled("td", "px-5 py-4 align-top"),

  // `pre` unwraps: the `code` renderer below owns the whole block, so wrapping
  // it in a second <pre> would nest two block elements and break the layout.
  pre: ({ children }) => <>{children}</>,

  code: ({ className, children, ...props }) => {
    const languageMatch = /language-(\w+)/.exec(className ?? "");
    const language = languageMatch?.[1];
    const codeString = String(children).replace(/\n$/, "");
    // A fenced block with no language still spans lines — treat it as a block,
    // or multi-line code renders as an inline pill.
    const isBlock = Boolean(language) || codeString.includes("\n");

    if (!isBlock) {
      return (
        <code className="rounded px-1.5 py-0.5 font-mono text-[0.9em]" {...props}>
          {children}
        </code>
      );
    }

    if (language) {
      return (
        <div className="relative mb-8 overflow-hidden rounded-2xl">
          <div className="absolute top-4 right-6 text-[10px] font-bold uppercase">
            {language}
          </div>
          <div className="overflow-x-auto p-6 text-sm">
            {/* sugar-high escapes its input before wrapping tokens in spans, so
                this is safe for repo-authored content. See the XSS note. */}
            <code dangerouslySetInnerHTML={{ __html: highlight(codeString) }} />
          </div>
        </div>
      );
    }

    return (
      <div className="mb-8 overflow-hidden rounded-2xl">
        <div className="overflow-x-auto p-6 text-sm">
          <pre className="m-0 font-mono whitespace-pre">{codeString}</pre>
        </div>
      </div>
    );
  },
};

export default function MarkdownRenderer({ content }: { content: string }) {
  return (
    <div className="text-lg leading-[1.8]">
      <ReactMarkdown remarkPlugins={[remarkGfm]} components={markdownComponents}>
        {content}
      </ReactMarkdown>
    </div>
  );
}
```

### The XSS boundary

`react-markdown` does **not** render raw HTML unless you add `rehype-raw`. That
default is the whole safety story here: a `<script>` in a post body is printed as
text. The only `dangerouslySetInnerHTML` is around `sugar-high` output, which
escapes before tokenizing.

Both facts hold **only for content you control**. This module assumes posts are
repo-authored and reviewed in a pull request. If a body can ever come from a
user, a comment, or an unreviewed CMS field:

- add `rehype-sanitize` to `rehypePlugins`, and
- replace the `highlight()` call with a renderer that does not take HTML.

Say which regime you are in near the renderer. The next reader will otherwise add
`rehype-raw` to get an embed working and quietly remove the boundary.

### Server or client

`MarkdownRenderer` has no state and no handlers — keep it a **server component**.
Rendering markdown on the client ships the parser, the GFM plugin and the
highlighter to every reader for output that never changes.

## Covers

Two cover sources, one precedence rule: a generated motif wins over a raster
image when a post declares both, because the motif scales and the image does not.

```tsx
export function hasCover(post: Post): boolean {
  return isCoverMotif(post.coverMotif) || !!post.coverImage;
}
```

### The accent palette

```ts
// lib/blog/cover-palette.ts
export type CoverAccent = "blue" | "slate" | "amber" | "emerald";

export const coverAccents: Record<CoverAccent, string> = {
  blue: "#0056b3",
  slate: "#475569",
  amber: "#d97706",
  emerald: "#059669",
};

const ORDER: readonly CoverAccent[] = ["blue", "slate", "amber", "emerald"];

export function resolveAccent(name?: string): string {
  return name && name in coverAccents ? coverAccents[name as CoverAccent] : coverAccents.blue;
}

/**
 * Deterministic accent from a tag. Two posts with the same first tag always get
 * the same colour, and it never changes between builds — a random or
 * index-based choice would reshuffle the whole index whenever a post is added.
 */
export function getAccentForTag(tag?: string): string {
  if (!tag) return coverAccents.blue;
  const key = tag.toLowerCase().normalize("NFD").replace(/[\u0300-\u036f]/g, "");
  let hash = 0;
  for (let i = 0; i < key.length; i++) hash = (hash * 31 + key.charCodeAt(i)) >>> 0;
  return coverAccents[ORDER[hash % ORDER.length] ?? "blue"];
}
```

Replace the four hex values with the host's palette. Keep the hash: the property
that matters is stability, not the specific colours.

### The motif registry

Covers are inline SVG keyed by a frontmatter string. The registry is the pattern
worth copying; the artwork is not: **ship your own motifs**, the earlier
implementation's are drawn for its brand.

```tsx
// components/blog/BlogCover.tsx
import type { CSSProperties } from "react";

import { getAccentForTag, resolveAccent } from "@/lib/blog/cover-palette";
import { GridMotif } from "./motifs/grid";
import { WaveMotif } from "./motifs/wave";

const motifs = {
  grid: GridMotif,
  wave: WaveMotif,
} as const;

export type CoverMotif = keyof typeof motifs;

/** Type guard, so an unknown frontmatter value falls back to the raster image
 *  instead of indexing the registry with undefined and crashing the page. */
export function isCoverMotif(value: string | undefined): value is CoverMotif {
  return !!value && value in motifs;
}

export default function BlogCover({
  motif,
  accent,
  tags,
  title,
  variant = "card",
}: {
  motif: CoverMotif;
  accent?: string;
  tags?: string[];
  title: string;
  variant?: "card" | "hero";
}) {
  const Motif = motifs[motif];
  const accentColor = accent ? resolveAccent(accent) : getAccentForTag(tags?.[0]);

  // The accent reaches the SVG as a CSS variable, so a motif is a plain static
  // component with no props — adding one is a file, not a signature change.
  const style = { "--cover-accent": accentColor } as CSSProperties;

  return (
    <div
      className="relative h-full w-full overflow-hidden"
      style={style}
      // The cover carries no information the title does not, but it is not
      // decorative either — role+label keeps it announced once, not twice.
      role="img"
      aria-label={title}
    >
      <svg
        viewBox="0 0 1600 1000"
        preserveAspectRatio="xMidYMid slice"
        className="absolute inset-0 h-full w-full"
        aria-hidden="true"
      >
        <Motif />
      </svg>
    </div>
  );
}
```

A motif is a bare SVG fragment referencing the variable:

```tsx
// components/blog/motifs/grid.tsx
export function GridMotif() {
  return (
    <g stroke="var(--cover-accent)" strokeWidth={2} opacity={0.35} fill="none">
      {Array.from({ length: 9 }, (_, i) => (
        <line key={`v${i}`} x1={i * 200} y1={0} x2={i * 200} y2={1000} />
      ))}
      {Array.from({ length: 6 }, (_, i) => (
        <line key={`h${i}`} x1={0} y1={i * 200} x2={1600} y2={200 * i} />
      ))}
    </g>
  );
}
```

```tsx
// components/blog/motifs/wave.tsx
export function WaveMotif() {
  return (
    <g stroke="var(--cover-accent)" strokeWidth={3} fill="none">
      {Array.from({ length: 7 }, (_, i) => (
        <path
          key={i}
          d={`M0 ${200 + i * 100} C 400 ${120 + i * 100}, 800 ${280 + i * 100}, 1600 ${200 + i * 100}`}
          opacity={0.15 + i * 0.06}
        />
      ))}
    </g>
  );
}
```

Fixed `viewBox` + `preserveAspectRatio="xMidYMid slice"` is what lets one motif
serve both a 16:10 card and a 16:9 hero without a second drawing.

Why generated covers at all: the alternative is commissioning an image per post,
and the posts without one look unfinished next to the posts with one. A motif
gives every post a cover on the day it is written.

## Cards

The card is a client component only if it animates. Keep the animation optional:
the module must render without a motion library.

Rules that are not cosmetic:

- **`readingMinutes` arrives as a prop.** Never send the body to a client
  component to count words. The earlier implementation sidestepped this by
  printing a hardcoded `5 min read` on every card, for every post, in every
  language.
- **The cover link needs `aria-label={title}`** — it wraps an image, so without
  it screen readers announce an unlabelled link, and it duplicates the title
  link below it.
- **Excerpt clamping is CSS** (`line-clamp-3`), not a JS truncation. Truncating
  in JS cuts mid-word and differs between server and client render.
- **Tags render the author's spelling; the link uses the slug.** See
  [tags.md](tags.md).

## Checklist

- [ ] `MarkdownRenderer` is a server component
- [ ] `h1` demoted to `h2`; the page owns the only `h1`
- [ ] Tables wrapped in an `overflow-x-auto` container
- [ ] Multi-line fenced blocks without a language render as blocks
- [ ] External links get `rel="noopener noreferrer"`; internal ones do not
- [ ] Trust regime stated near the renderer; `rehype-sanitize` if untrusted
- [ ] Motif registry guarded by `isCoverMotif`
- [ ] Accent derived deterministically; own palette substituted
- [ ] `readingMinutes` computed server-side and passed as a prop
- [ ] Host tokens substituted for all placeholder classes
