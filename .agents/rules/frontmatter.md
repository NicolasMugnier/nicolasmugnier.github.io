# Frontmatter and titles

Every post (draft or published) needs:

```yaml
layout: post
title: "..."
tags: [yaml, list]
author: Nicolas Mugnier
categories: architecture
description: "..."
image: /assets/img/slug.webp
locale: en_US
```

- `tags:` is a YAML list only (`[git, php]`). Never a space-separated string.
- New posts: `locale: en_US`. Existing FR posts keep `fr_FR`.
- Extension **`.markdown`** for new files. Do not add new `.md` posts.
- **Do not** repeat `title:` as `# Heading` in the body. The layout prints the frontmatter title once.
- Missing `title:` makes the home/post layout show a weak filename title.
