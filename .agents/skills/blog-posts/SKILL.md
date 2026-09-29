---
name: blog-posts
description: Draft, edit, publish, or unpublish Jekyll posts.
---

# Blog posts

Work on articles in this Jekyll blog. Never create a new file in `_posts/` first. Never invent metrics or a live hero change.

Also follow `.agents/rules/` (em dash, frontmatter, mermaid, cover crop, content policy).

## When to Use

- Write, edit, outline, or publish a post
- Unpublish (move back to drafts)
- Fix rendering (Mermaid, titles, covers)

Don't use for:

- LinkedIn, GitHub README, or X
- New Web3 / Farcaster / BIP39 content
- CSS/theme work with no article change (edit SCSS directly)

## Frontmatter (copy this)

```yaml
---
layout: post
title: "Exact title, no em dash"
tags: [php, architecture]
author: Nicolas Mugnier
categories: architecture
description: "One concrete sentence for the card. Name the stack. No em dash."
image: /assets/img/slug.webp
locale: en_US
---
```

Do **not** add a body H1 that repeats `title:`. `post.html` already renders it.

Optional i18n (rare): `translation_key` plus a FR sibling. Default is English-only.

## Categories (new posts)

Allowed: `architecture`, `algorithm`, `docker`, `ai`.

On publish, folder must match `categories:`:

- `_posts/architecture/`
- `_posts/algorithm/`
- `_posts/docker/`
- `_posts/ai/`

If the topic does not fit, ask before adding a category.

## Voice

1. Problem at scale
2. Naive fix and why it fails
3. Pattern that worked
4. Short real code (PHP/TS/SQL), not pseudocode walls
5. Next decision, or what you would not repeat

Internal links: `{% post_url architecture/2026-02-22-scaling-background-jobs-and-caching-in-a-php-application %}` or a site-relative path. External: `{:target="_blank"}` on the Markdown link.

`---` between major parts is a Markdown hr, not an em dash.

## Procedure: draft

1. Confirm topic + category if ambiguous. Done when category is one of the four (or he named a new one).
2. Slug: lowercase, hyphens, no date yet. Write `_drafts/<slug>.markdown`. Done when the file starts with `---\nlayout: post`.
3. Cover: follow skill `cover-images`. Reuse an existing image only if he agrees. Done when `image:` points at a file that exists.
4. Search the draft for U+2014. Done when zero hits.
5. Show the draft. Do not publish. Optional: `docker compose up -d` and open `/` on :4000 (drafts are on).

Draft filename may omit the date. Incomplete drafts may use HTML comments as section stubs.

## Procedure: publish (only after explicit go)

1. Date = today in Europe/Paris: `_posts/<category>/YYYY-MM-DD-<slug>.markdown`.
2. Move the file there. Delete the `_drafts/` copy. Done when the post exists only under `_posts/`.
3. Leave `_config.yml` `hero:` unchanged unless he asked to feature this post (`hero` is the post **slug**).
4. Commit post + image only. Message: `blog(article): publish <slug>` with no em dash.
5. Push only if he asked. GitHub Pages will not show `_drafts/`.

## Procedure: unpublish (only after explicit go)

1. Move `_posts/.../YYYY-MM-DD-<slug>.markdown` to `_drafts/<slug>.markdown`.
2. Leave images in `assets/img/` unless he asked to delete them.
3. Do not change `hero:` unless that slug was the hero (then ask).

## Pitfalls

- `title:` missing: the layout falls back to a weak filename title.
- Subfolder under `_posts/` is not the Jekyll category. Frontmatter `categories:` is.
- `site.hero` matches `post.slug`, not the filename.
- Do not "fix" old Web3 / BIP39 / Farcaster posts as a side effect.
- Geometry in an intro must match the real query (example: offer **point** in talent **boxes**, `@>`, not box-box overlap unless that is what he said).

## Verification

Draft is done when:

- Frontmatter has `layout`, `title`, YAML `tags`, `author`, `categories`, `description`, `image`, `locale: en_US`
- No body H1 equal to `title:`
- Extension `.markdown`
- File is in `_drafts/` (until publish)
- Zero U+2014
- Image path exists
- Category is allowed
- No web3/farcaster tags or category on a **new** post
