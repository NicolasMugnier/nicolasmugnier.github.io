---
name: cover-images
description: Generate and convert blog covers that survive crop.
---

# Cover images

Produce or replace `assets/img/<slug>.webp` for a post. Show the image before replacing a live file.

Also follow `.agents/rules/cover-crop.md` and `.agents/rules/no-em-dash.md`.

## When to Use

- New post needs a cover
- Replace a square/old-theme cover
- He asks for an alternative cover

Don't use for:

- Diagrams that belong in the article body (Mermaid, multi-panel charts)
- The in-body gift/screenshot assets (only `image:` in frontmatter)

## How to produce

1. Canvas **1280x720** PNG or JPG.
2. Motif in the **center band** (see cover-crop rule). One motif, not a two-row diagram.
3. Prefer geometry that matches the article (exponential = `2^n` bars, quicksort = unsorted partitions around a pivot, SOIS = four steps). If AI distorts the geometry, draw it programmatically.
4. Convert on the **host**, never Docker in this repo:

   ```bash
   ffmpeg -y -i <src.png|jpg> -c:v libwebp -quality 90 assets/img/<slug>.webp
   ```

5. Point frontmatter `image:` at `/assets/img/<slug>.webp`.
6. Show him the full frame **and** a ~3:1 center crop (what `.post-cover` keeps) before publishing.
7. Do not overwrite `assets/img/` until he says to use it.

Done when the webp exists, is 1280x720, and he approved it.

## Pitfalls

- Article page: `.post-cover img` is `width: 100%`, `max-height: 400px`, `object-fit: cover`. Wrapper up to 1600px. Top and bottom of a 16:9 die.
- Cards: `aspect-ratio: 16 / 10` (mild left/right crop).
- Hero: half width, `min-height: 300px` (banner crop again).
- Repo Docker `convert-image` / `cwebp` is banned.
- Labels in corners or two stacked panels become noise after crop.
- Do not change `_config.yml` `hero:` because a cover changed.
