---
name: deploying-consulting-blog
description: Use when the user wants to deploy, publish, or release the consulting blog (blog.drolet.ai), push a new post live, check whether a blog deploy finished, or debug a failed blog build on Cloudflare Pages.
compatibility: Requires git, curl and python3. Reads the Cloudflare API token from ~/src/infra/cloudflare/terraform.tfvars for build status; the live-site check needs no credentials.
---

# Deploying the Consulting Blog

The blog is a **Cloudflare Pages** project. There is no image to build and
nothing to roll out: pushing to `main` on `github.com/bdrolet/consulting-blog`
triggers a Pages build, and the build publishes the site.

| | |
|---|---|
| **Repo** | `bdrolet/consulting-blog`, production branch `main` |
| **Pages project** | `consulting-blog` |
| **Build** | `npm run build` → `dist`, `NODE_VERSION=22.12.0` |
| **Live** | https://blog.drolet.ai |
| **Pages URL** | https://consulting-blog-2ik.pages.dev |

The project is defined in Terraform, not in this repo:
`~/src/infra/cloudflare/main.tf` (`cloudflare_pages_project.consulting_blog`).
Change the build command, output directory or Node version there, not in the
Cloudflare dashboard.

## Step 1 — Commit and push

```bash
git status --short          # confirm what is going out
git push origin main
```

Anything committed to `main` goes live. Drafts must stay uncommitted or out of
`src/content/blog/`.

## Step 2 — Watch the build

```bash
PAGES_PROJECT=consulting-blog bash .claude/skills/deploying-consulting-blog/scripts/deploy-status.sh          # HEAD
PAGES_PROJECT=consulting-blog bash .claude/skills/deploying-consulting-blog/scripts/deploy-status.sh <sha>    # a specific commit
```

It waits for the deployment built **from that commit** and polls it to a
terminal stage. Matching on the commit matters: Cloudflare takes a few seconds
to queue a build after a push, so "the most recent deployment" immediately after
`git push` is still the previous one, and a check that trusts it reports success
for the wrong build.

Exit codes: `0` on `deploy/success`, `1` on a failed build or a commit that
never produced one, `2` when there are no credentials to check with.

Without a Cloudflare token it falls back to reporting that it cannot read build
status — in that case skip to step 3, which needs no credentials, or read the
build log in the Cloudflare dashboard under **Workers & Pages → consulting-blog**.

## Step 3 — Verify the live site

```bash
curl -sI https://blog.drolet.ai/ | head -1
curl -s https://blog.drolet.ai/ | grep -o '<title>[^<]*</title>'
```

Expect `HTTP/2 200`. To confirm a specific post is live, request its URL
directly and check for a `200` rather than trusting the index page, which may
still be cached at the edge.

## Troubleshooting

| Symptom | Cause |
|---|---|
| Build fails on an Astro or syntax error | Reproduce locally with `npm run build`; the Pages build runs the same command |
| Build fails with an unsupported Node version | `NODE_VERSION` is pinned in `cloudflare_pages_project.consulting_blog`; Astro 6 needs ≥ 22.12 |
| Build succeeds but the site is unchanged | Cloudflare edge cache — retry with a cache-busting query string, or purge the cache for the zone |
| No deployment ever appears for a pushed commit | Cloudflare's GitHub app can't see the repo — add `consulting-blog` under github.com/settings/installations → Cloudflare Pages → Repository access |
| `deploy-status.sh` reports no token | The token lives in `~/src/infra/cloudflare/terraform.tfvars`, which exists only on the machine that runs the infra Terraform |
| 522 from `blog.drolet.ai` | The Pages custom domain is not `active`; check the domain's status in the infra Terraform state or the dashboard |
