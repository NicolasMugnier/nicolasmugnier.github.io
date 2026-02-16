# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Jekyll-based static blog (personal technical blog in French) hosted on GitHub Pages at https://nicolasmugnier.github.io. Uses the Minima theme via `jekyll-remote-theme`.

## Commands

Local development uses Docker Compose (no local Ruby/Bundler required).

```bash
# Start development server in background (serves at http://localhost:4000, live reload on :35729, includes drafts)
docker compose up -d

# Restart after config changes
docker compose restart

# Stop the server
docker compose stop
```

No automated test suite exists. Testing is manual via the local dev server.

## Architecture

- **`_posts/`** — Published posts, organized by category subdirectories (e.g., `algorithm/`). Filename format: `YYYY-MM-DD-title.markdown`.
- **`_drafts/`** — Unpublished drafts, same structure as `_posts/`.
- **`_layouts/home.html`** — Custom homepage layout (card-based grid of articles).
- **`_includes/`** — Reusable HTML partials: `header.html` (site nav with logo), `custom-head.html` (Mermaid CDN import, custom CSS links).
- **`assets/css/`** — Custom styles: `cards.css` (post card grid), `lines.css` (code line numbers), `vim.css` (syntax highlighting).
- **`assets/img/`** — Blog images (prefer `.webp` format).

## Post Frontmatter Convention

All posts use this frontmatter structure:

```yaml
---
tags: [tag1, tag2]
author: Nicolas Mugnier
categories: category-name
description: "Short description"
image: /assets/img/image-name.webp
locale: fr_FR
---
```

## Key Details

- Content is written in **French**.
- Posts use **Mermaid** (v10, loaded via CDN in `custom-head.html`) for diagrams.
- Posts use **jemoji** for emoji rendering.
- **Disqus** comments are enabled (shortname: `nicolasmugnier-github-io`).
- The site uses `github-pages` gem (~> 232) for GitHub Pages compatibility — all plugins must be GitHub Pages-compatible.
