# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Jekyll blog using the **Chirpy theme** (v7.3), published via GitHub Pages. The site is in Chinese (`lang: zh`) and covers AI + reverse engineering topics.

## Build & Dev Commands

```bash
# Local dev server with live reload (localhost:4000)
bash tools/run.sh

# Production mode dev server
bash tools/run.sh -p

# Build and test (production build + htmlproofer validation)
bash tools/test.sh

# Direct Jekyll commands
bundle exec jekyll s -l                    # dev server
JEKYLL_ENV=production bundle exec jekyll b  # production build
```

## Writing Blog Posts

**File naming:** `_posts/YYYY-MM-DD-kebab-case-title.md`

**Required front matter:**
```yaml
---
title: 文章标题
date: YYYY-MM-DD HH:MM:SS +0800
categories: [Category1, Category2]
tags: [tag1, tag2, tag3]
---
```

Post defaults (from `_config.yml`): layout `post`, comments enabled, TOC enabled, permalink `/posts/:title/`.

**Date caution:** Do not set `date` in the future relative to the build time — Jekyll will skip future-dated posts unless configured otherwise. This has caused issues before (commit `0f8e5a2`).

**时区注意:** GitHub Actions 构建环境使用 UTC 时区，而文章日期使用 `+0800`（北京时间）。如果文章日期设为当天（如 `2026-03-24 00:00:00 +0800`），在北京时间 08:00 之前触发的构建会将其视为未来文章而跳过。建议将当天发布的文章时间设为前一天或明确设置一个较早的时间点。

## Key Architecture

- **Theme:** `jekyll-theme-chirpy` gem — layouts/includes/assets live in the gem, not in this repo. Override by placing files in the same relative path locally.
- **Custom plugin:** `_plugins/posts-lastmod-hook.rb` — auto-sets `last_modified_at` from git history (requires full git clone, not shallow). The CI workflow fetches with `fetch-depth: 0` for this reason.
- **Tabs:** `_tabs/` contains the sidebar navigation pages (about, archives, categories, tags).
- **CI/CD:** `.github/workflows/pages-deploy.yml` — builds with production env, runs htmlproofer, deploys to GitHub Pages.

## Conventions

- Blog posts are written in Chinese
- Post filenames use English kebab-case slugs
- Commit messages: Chinese descriptions with conventional prefixes (`feat:`, `fix:`, etc.)
