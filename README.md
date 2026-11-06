# SeaGaruda Tech Blog

Source of truth for my technical writing: long-form articles on cloud
architecture, database engineering, OT/IT security, and industrial
digitalization.

The repository **is** the website. Content is plain Markdown; a Hugo Extended
build with the [OINK](https://oink.pgsty.com/) theme produces the published
site — responsive HTML, RSS, per-section print views, per-page Markdown, and
`llms.txt` / `navigation.json` indexes for AI agents. No Node.js, npm, PostCSS,
or bundler is involved.

- Published site: <https://seagaruda.github.io/TechBlogs/>
- Chinese edition: <https://seagaruda.github.io/TechBlogs/zh/>
- Agent index: <https://seagaruda.github.io/TechBlogs/llms.txt>
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

`.github/workflows/deploy-pages.yaml` builds with the pinned toolchain on every
push to `main` and publishes to GitHub Pages (Settings → Pages → Source:
**GitHub Actions**). `baseURL` is derived from the Pages URL at build time, so
the project subpath stays correct.

To publish under a custom domain, point the domain at Pages and change
`baseURL` in `hugo.yaml` to the canonical address.

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
