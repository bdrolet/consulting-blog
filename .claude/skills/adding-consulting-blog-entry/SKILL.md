---
name: adding-consulting-blog-entry
description: Use when adding, creating, or writing a new post to the consulting blog (blog.drolet.ai) in this repo. Triggers on requests like "write a new post", "add a blog entry", "create a post about X", or "draft a new article".
---

New posts live in `src/content/blog/`. Drop a `.md` file there — Astro picks it up automatically via the glob loader and the dynamic route at `src/pages/[...slug].astro` renders it. The filename becomes the URL slug, at the site root.

## Required Frontmatter

```markdown
---
title: "Post Title"
pubDate: YYYY-MM-DD
description: "One-sentence summary shown in the post listing"
---
```

## Optional Frontmatter

```markdown
updatedDate: YYYY-MM-DD   # show a "last updated" date
tags: ["healthcare", "llm"]  # appear on tag index pages at /tag/<tag>
```

## Naming Convention

Use lowercase kebab-case: `my-post-title.md` → `/my-post-title/`

## Checklist

1. Create `src/content/blog/<slug>.md` with the required frontmatter
2. Set `pubDate` to today's date
3. Write content in standard Markdown below the closing `---`
4. Verify `title`, `description`, and `pubDate` are all present — the content schema in `src/content.config.ts` will reject the entry if any are missing
