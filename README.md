# blog.anyvoid.dev

Jekyll blog. Live: [https://blog.anyvoid.dev](https://blog.anyvoid.dev)

Backend notes: APIs, search, async jobs, architecture that holds in production. New posts are English. Older French posts stay.

GitHub Pages builds `main`. Comments: Giscus.

## Preview locally

Docker only. No local Ruby required.

```bash
docker compose up -d
```

- Site: http://localhost:4000 (drafts included)
- Live reload: :35729

Restart after `_config.yml` changes: `docker compose restart`.

## Layout

- `_posts/` published articles. Category folder matches frontmatter `categories:`
- `_drafts/` unpublished. GitHub Pages ignores them; local preview does not
- `assets/img/` covers and inline images (WebP)
- `_layouts/` home, post, topic
- `_sass/minima/custom-styles.scss` cards, hero, article cover crop
- `_config.yml` `hero:` is a post **slug**, not a filename

New post files use `.markdown`. Frontmatter needs `layout`, `title`, `tags` (YAML list), `author`, `categories`, `description`, `image`, `locale`.

Do not repeat `title:` as a body H1. The post layout already prints it.

## Covers

1280x720 WebP. The article page crops to a wide banner (`object-fit: cover`, `max-height: 400px`), so the motif belongs in the vertical center.

Convert on the host:

```bash
ffmpeg -y -i cover.png -c:v libwebp -quality 90 assets/img/slug.webp
```

Do not use the repo Docker `convert-image` service for covers.

## Diagrams

Mermaid v10 is loaded in `_includes/custom-head.html`. Use `<div class="mermaid">`, not a fenced `mermaid` code block.

## Agents

Conventions for coding agents: [AGENTS.md](./AGENTS.md), `.agents/rules/`, `.agents/skills/`.

`CLAUDE.md` is stale. AGENTS.md wins.

## License

MIT. See [LICENSE](./LICENSE).
