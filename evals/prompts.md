---
prompts:
  - prompt: "Add a blog to this Next.js app from markdown files in content/blog: post pages, tag pages, related posts, an RSS feed and sitemap entries, all statically generated."
    stack: Repository files
  - prompt: Make our markdown blog bilingual, English and Polish, with a different slug per language joined by one translation key, and hreflang between the two.
    stack: Repository files
  - prompt: Our blog build got slow as the posts grew. Find where the loader rereads the markdown files and fix it.
---

# Prompts

What an operator types after installing this skill, in their own words. An agent eval installs the skill
into an empty Next.js app, gives the agent one of these prompts and no further help, then type-checks, builds
and tests the result; the first prompt runs before every release. The results are the other files in this
folder. Section 10 of [STANDARD.md](https://github.com/timerise-ai/skills/blob/main/STANDARD.md) says how a
run is made. The prompts and the newest runs are on
[the skill's page](https://timerise.ai/skills/blog-markdown) on timerise.ai.
