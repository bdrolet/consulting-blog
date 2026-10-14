# blog.drolet.ai

The consulting blog for [drolet.ai](https://drolet.ai): getting AI from pilot to
production in healthcare. [Astro](https://astro.build), static output, Markdown
and MDX posts. The personal blog at blog.drolet.cloud is a separate repo with the
same theme.

## Deploying

There is nothing to run. The repo is a [Cloudflare Pages](https://pages.cloudflare.com)
project: **pushing to `main` builds and publishes the site**, usually in under a
minute. Anything committed to `main` goes live, so drafts stay in the gitignored
`pipeline/` folder until they're published.

| | |
|---|---|
| Live | https://blog.drolet.ai |
| Pages URL | https://consulting-blog.pages.dev |
| Build | `npm run build` → `dist`, Node 22.12.0 |

The Pages project itself is defined in Terraform, in a different repo —
`~/src/infra/cloudflare/main.tf`, `cloudflare_pages_project.consulting_blog`.
Change the build command, output directory or Node version there rather than in
the Cloudflare dashboard, or the next `terraform apply` will revert it.

`.claude/skills/deploying-consulting-blog/` has the deploy and build-status
workflow; `scripts/deploy-status.sh` there polls a running build to success or
failure.

## Writing

Drafts live in `pipeline/` (gitignored; see `pipeline/README.md`), driven by the
`managing-consulting-blog-pipeline` skill: capture an idea, draft it, mark it
ready, and `publish next` moves the oldest ready draft into `src/content/blog/`.

Published posts are Markdown or MDX in `src/content/blog/`, one file per post,
with the filename as the URL slug (`src/content/blog/foo.md` → `/foo/`).
Frontmatter is type-checked against the schema in `src/content.config.ts`:

```yaml
---
title: "Post title"
pubDate: 2026-10-14
description: "One or two sentences — used in the RSS feed and meta tags."
tags: ["healthcare", "llm"]
---
```

`src/consts.ts` holds the site title and description, which feed the RSS
channel and the default meta tags. The canonical URLs, the sitemap and the RSS
links all derive from `site` in `astro.config.mjs` — if that is wrong, every one
of them is wrong, silently.

## Commands

| Command | Action |
| :--- | :--- |
| `npm install` | Install dependencies |
| `npm run dev` | Dev server at `localhost:4321` |
| `npm run build` | Build to `./dist/` — the same command Pages runs |
| `npm run preview` | Serve the built site locally |

## Credit

The theme started from Astro's blog starter, which is based on
[Bear Blog](https://github.com/HermanMartinus/bearblog/).
