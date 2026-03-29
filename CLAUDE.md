# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Install dependencies
bundle install

# Local development server (hot-reload on port 4000)
bundle exec jekyll serve

# Or using Docker
docker compose up

# Build static site
bundle exec jekyll build
```

## Architecture

This is a Jekyll-based personal blog deployed to GitHub Pages at `https://janeski.tech`. Pushing to `master` triggers automatic deployment — there is no CI/CD pipeline.

**Theme:** Custom theme derived from [jekyll-uno](https://github.com/joshgerdes/jekyll-uno) (despite `_config.yml` referencing `jekyll-theme-slate`, the actual layout/styling lives in `_layouts/`, `_includes/`, and `_sass/`).

**Key config:** `_config.yml` — pagination (10/page), permalink format `/:year/:title/`, kramdown with GFM, rouge syntax highlighting.

## Blog Posts

Posts go in `_posts/` with filename format `YYYY-MM-DD-slug.markdown`.

Required frontmatter fields:
```yaml
title: ""
date: YYYY-MM-DD HH:MM:SS +0000
description: ""      # SEO meta description
categories: []       # e.g., [cloud, dotnet]
tags: []             # e.g., [azure, aspire, dotnet]
```

Optional frontmatter:
```yaml
image: /images/filename.jpg          # featured image
image_credit: '<a href="...">...</a>' # attribution HTML
comments: false                       # disables Disqus (on by default)
canonical: "https://..."             # shows "originally published at" banner
```

Read time is calculated automatically from word count (words/200).
