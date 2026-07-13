# Source repository for blog.lespaulstudioplus.info

Content source for the self-hosted blog: a Spring Boot app polls this repository and
renders the markdown itself (there is no Hugo or other static-site generator here). Only
two directories matter:

- `content/` — posts and the about page, as markdown with frontmatter
- `data/` — site configuration (`menu.yaml`, `deprecation.yaml`)

## Layout

```
content/
  about.md                 # about page (Japanese)
  about.en.md              # about page (English)
  posts/
    1/                     # one folder per post; the folder name is the slug
      ja.md                # Japanese version
      en.md                # English version
      CFF_Logo_vertical_RGB.png   # images live in the post folder
      2020-09-23T14-53-19.png
    wf-demanabu-dt/        # slugs can be words, not just numbers
      ja.md
      en.md
    ...
data/
  menu.yaml                # top navigation
  deprecation.yaml         # posts/tags that are no longer relevant
```

Each post is a folder under `content/posts/`. The **folder name is the slug** and the
**file name is the language** (`ja.md`, `en.md`) — so `posts/1/en.md` is post `1` in
English. A post may exist in one language only. The flat about page uses a filename suffix
instead (`about.md` = Japanese, `about.en.md` = English).

## Writing a post

Frontmatter (YAML) plus markdown body:

```markdown
---
title: "記事のタイトル"
date: 2025-01-10T12:00:00+09:00
categories: ["Tanzu Platform"]
tags: ["Tanzu Platform", "Kubernetes"]
thumbnail: "cover.png"      # optional; a file in this post folder
draft: true                 # optional; drafts are not published
---

Intro shown in listings.<!--more-->

The rest of the article.

![](diagram.png)
```

- **Images** go in the post folder and are referenced **relatively** (`![](diagram.png)`,
  `thumbnail: "cover.png"`). The app serves them at `/media/posts/<slug>/<file>`.
- **Links to other posts** are relative to the posts directory (`[see](../35)`), which the
  app resolves to `/posts/35`.
- **`<!--more-->`** marks the end of the listing excerpt.
- **`draft: true`** keeps a post out of the published site.
- `date` drives ordering (newest first); posts without a date are skipped.

## `data/menu.yaml`

Top navigation. One level of submenus is supported:

```yaml
- name: Home
  link: /
- name: Series
  submenu:
    - name: "Wavefrontで学ぶ分散トレーシング"
      link: /posts/wf-demanabu-dt
- name: About
  link: about/
```

## `data/deprecation.yaml`

Flags posts whose product or feature is no longer available; the site shows a "kept for
reference" banner on those posts and a badge in listings. A post is flagged when its slug
is listed under `posts`, or when any of its tags **or categories** matches a value under
`tag`:

```yaml
posts:      # post slugs (folder names)
  - 33
  - 34
tag:        # tag or category names
  - Tanzu Observability
  - Tanzu Platform for Maniacs
```

## Sync behaviour

The app polls this repository and imports changes incrementally, so editing a single file
updates only that post. It also posts a `blog/sync` commit status back here (`pending`
then `success`/`failure`) so each pushed commit shows whether the blog picked it up.
