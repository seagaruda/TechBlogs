# SeaGaruda Tech Blog

Source of truth for my technical writing: long-form articles on cloud
architecture, database engineering, OT/IT security, and industrial
digitalization.

The repository **is** the website. Content is plain Markdown; a Hugo Extended
build with the [OINK](https://oink.pgsty.com/) theme produces the published
site — responsive HTML, RSS, per-section print views, per-page Markdown, and
`llms.txt` / `navigation.json` indexes for AI agents. No Node.js, npm, PostCSS,
or bundler is involved.

- Primary site (Cloudflare Pages): <https://blog.seagaruda.com/>
  (project `seagaruda-blog`, also reachable at
  <https://seagaruda-blog.pages.dev/>)
- Mirror (GitHub Pages): <https://seagaruda.github.io/TechBlogs/>
- Chinese edition: `/zh/` on any host
- Agent index: `/llms.txt`, `/navigation.json`, `/blog/llms-full.txt`
- This repository also feeds `posts.json` to the Vercel blog index
  (see [posts.json pipeline](#postsjson-pipeline)).

## Structure

```text
.
├── hugo.yaml                     site configuration (languages, taxonomies, UI)
├── go.mod / go.sum               pinned OINK Hugo Module version
├── content/
│   ├── _index.md / _index.zh.md  home page front matter
│   ├── about.md / about.zh.md    about page
│   └── blog/
│       ├── _index.md / .zh.md    blog shell (type: blog + cascade)
│       └── post/
│           ├── _index.md / .zh.md  article index
│           ├── <slug>.md           English article
│           └── <slug>.zh.md        its Chinese peer
├── data/home/en.yaml, zh.yaml    home page landing sections
├── assets/logo.png               brand mark used by the site shell
└── .github/workflows/
    ├── deploy-pages.yaml         Hugo build → GitHub Pages
    └── sync-posts.yml            content → posts.json → blog-index
```

## Local toolchain

| Tool | Version | Why |
| --- | --- | --- |
| Hugo Extended | ≥ 0.168.0 | the only build tool |
| Go | ≥ 1.27.3 | resolves the OINK Hugo Module (`go.mod`) |
| Git | any | module resolution and `enableGitInfo` |

```bash
# one-time: resolve the pinned theme
go mod download github.com/pgsty/oink

# write
hugo server              # http://localhost:1313/

# verify a production build before pushing
hugo --cleanDestinationDir --gc --minify --environment production \
  --printPathWarnings
```

`public/` and `resources/` are ignored. The LibSass deprecation warning from
Hugo's built-in Sass compiler is expected — keep warnings visible, and do not
enable `--panicOnWarning` / `HUGO_PANICONWARNING`.

## Writing an article

1. Pick a slug: lowercase, hyphenated, no dates in the filename.
2. Create `content/blog/post/<slug>.md` (Chinese peer: `<slug>.zh.md`).
3. Front matter — every key below is used by the site:

```yaml
---
title: "Full article title"
date: 2026-08-11              # required; drives list order and RSS
description: "One or two sentences shown above the body and in search."
author: SeaGaruda
categories: ["Cloud Architecture"]   # one category per article
tags: ["Cloud", "Disaster Recovery", "VPC"]
slug: "cloud-dr-networking"
---
```

Rules that keep the site consistent:

- **No H1 in the body.** The title comes from `title`; a duplicated `# Heading`
  renders twice.
- **No byline in the body.** `author` + `date` are rendered by the shell.
- **Bilingual pairs share** `date`, `author`, `slug` and the filename stem;
  translate `title`, `description`, and `tags`.
- **Categories** stay within the five editorial buckets so the home page and
  taxonomy pages keep their shape:

  | Category | Scope |
  | --- | --- |
  | Cloud Migration | moving workloads off bare metal; what breaks afterwards |
  | Cloud Architecture | load balancing, distributed design, DR networking |
  | Cloud Database | managed RDS, time-series, engine selection |
  | OT Security | Purdue model, ICS/OT boundaries on public cloud |
  | Digital Transformation | process, data hygiene, why systems fail to stick |

- **Tags** are free-form but reused: `Cloud`, `Database`, `Migration`,
  `Architecture`, `Huawei Cloud`, `AMI`, `DevOps`, `OT Security`. Reuse before
  inventing.
- Code fences get a language; tables get a header row. Everything else is
  ordinary Markdown — no shortcode is required for a normal article.

## Publishing

### GitHub Pages (mirror, already live)

`.github/workflows/deploy-pages.yaml` builds with the pinned toolchain on every
push to `main` and publishes to GitHub Pages (Settings → Pages → Source:
**GitHub Actions**). `baseURL` is derived from the Pages URL at build time, so
the project subpath stays correct.

### Cloudflare Pages (primary — live)

Live project: **`seagaruda-blog`** → <https://seagaruda-blog.pages.dev/>,
served at the custom domain **<https://blog.seagaruda.com/>**.

`.github/workflows/cloudflare-pages.yaml` runs the same production build and
uploads it with **Direct Upload** (`wrangler pages deploy`). Direct Upload is
used on purpose: Cloudflare's own build image is not guaranteed to carry Hugo
Extended *and* Go, which the OINK Hugo Module needs.

One-time setup — Settings → Secrets and variables → Actions:

| Kind | Name | Value |
| --- | --- | --- |
| secret | `CLOUDFLARE_API_TOKEN` | API token with **Account → Cloudflare Pages → Edit** |
| secret | `CLOUDFLARE_ACCOUNT_ID` | `3b9eee490d2e9e20a8378a85677d8774` |
| variable | `CLOUDFLARE_PAGES_ENABLED` | `true` (gates pushes; leave unset to skip) |
| variable | `CLOUDFLARE_PROJECT_NAME` | optional, default `seagaruda-blog` |
| variable | `CLOUDFLARE_SITE_URL` | optional, default `https://blog.seagaruda.com/` |

Until `CLOUDFLARE_PAGES_ENABLED=true` is set, pushes skip the Cloudflare job
and only the GitHub Pages mirror is updated. A manual **Run workflow** also
deploys once the two secrets exist.

#### Custom domain

`blog.seagaruda.com` is attached to the Pages project. Because wrangler's OAuth
login has no DNS scope, the record is created by hand once:

| Type | Name | Content | Proxy |
| --- | --- | --- | --- |
| CNAME | `blog` | `seagaruda-blog.pages.dev` | Proxied (orange cloud) |

Without that record the domain stays in `pending` and the certificate is never
issued — a `pending` Pages domain plus a missing CNAME is the usual cause.

#### Manual deploy (no Actions quota, uses local wrangler)

```bash
hugo --cleanDestinationDir --gc --minify --environment production \
  --baseURL "https://blog.seagaruda.com/"
wrangler pages deploy public --project-name seagaruda-blog --branch main
```

Both hosts serve the same content; only the absolute URLs inside RSS,
`llms.txt`, and the sitemap differ, because `baseURL` is passed per deploy.

## posts.json pipeline

`.github/workflows/sync-posts.yml` runs on every Markdown change, regenerates
`posts.json` from `content/blog/post/*.md`, and commits it to
`seagaruda/blog-index` (branch `herbert`, path `public/posts.json`). That is
what the Vercel site fetches for its featured-post cards.

The schema is stable for that consumer:

```json
{
  "generated_at": "2026-11-06T…Z",
  "posts": [
    {
      "title": "…", "date": "2026-08-11", "summary": "…",
      "slug": "…", "lang": "en", "author": "SeaGaruda",
      "categories": ["…"], "tags": ["…"],
      "url": "https://seagaruda.github.io/TechBlogs/blog/post/…/",
      "raw_url": "https://raw.githubusercontent.com/…/content/blog/post/….md",
      "github_url": "https://github.com/seagaruda/TechBlogs/blob/main/…",
      "filename": "content/blog/post/….md"
    }
  ]
}
```

## License

Article text © SeaGaruda. The theme (OINK) is Apache-2.0 and derived from
Google Docsy; see the theme repository for upstream attribution.
