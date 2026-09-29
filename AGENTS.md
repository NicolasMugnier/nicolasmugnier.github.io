# AGENTS.md

Instructions for any coding agent working in this repository.

This is Nicolas Mugnier's Jekyll blog. Live site: https://blog.anyvoid.dev
GitHub Pages from `main`. Theme: Minima (remote) plus custom layouts and SCSS.

`CLAUDE.md` is stale (it still says French-only content and Disqus). **This file and `.agents/` win.**

## Where to look

- Always-on constraints: `.agents/rules/`
- Task workflows: `.agents/skills/`
  - `blog-posts` when drafting, editing, publishing, or unpublishing an article
  - `cover-images` when generating or replacing a post cover

## Repo map

- `_posts/` published articles. Category folder must match frontmatter `categories:`
- `_drafts/` unpublished. GitHub Pages does not build these. Local `docker compose` does.
- `_layouts/post.html` article page. Cover is `.post-cover img`
- `_layouts/home.html` hero + cards
- `_includes/custom-head.html` Mermaid v10, KaTeX, Giscus theme hook
- `_sass/minima/custom-styles.scss` cover crop, cards, hero
- `assets/img/` covers and inline images (WebP)
- `_config.yml` `hero:` is a post **slug** (example: `mc-deploy-check`). Do not change it unless asked.

## Local preview

```bash
docker compose up -d
```

Serves http://localhost:4000 including drafts. Restart after `_config.yml` changes.

There is no test suite. Verify in the browser.

## Non-negotiables (detail in rules)

- New posts: English, `locale: en_US`, extension `.markdown`
- Never Unicode em dash U+2014
- Never duplicate `title:` as a body H1
- Mermaid: `<div class="mermaid">`, never a fenced `mermaid` block
- Covers: 1280x720, motif in the vertical center band (article page crops to a wide banner)
- Convert covers with host `ffmpeg` to WebP. Never the repo Docker `convert-image` / `cwebp` service
- Draft in `_drafts/` first. Publish only when asked, by moving to `_posts/<category>/YYYY-MM-DD-slug.markdown`
- No new Web3 / Farcaster / BIP39 posts. Leave existing ones and their images alone unless asked
- Do not invent metrics, quotes, or architecture. If the domain polarity is unclear, ask
- Comments are Giscus, not Disqus
- LinkedIn (his): https://www.linkedin.com/in/nicolas-mugnier/ (homonym without the hyphen is someone else)

## Voice

Staff backend, not a 2018 tutorial blog. First person when it is his work. Factual. No motivation theater.

Prefer: search, matching, jobs/queues, cache, observability, serverless, PHP/Symfony, Node/TS, PostgreSQL, AWS, Docker.
