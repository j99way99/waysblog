# Ways Blog

Work logs with Claude and reflections on systems, published at
<https://j99way99.github.io/waysblog>.

Built with [Academic Pages](https://github.com/academicpages/academicpages.github.io)
(a Jekyll template based on Minimal Mistakes) and hosted on GitHub Pages.

## Writing a post

Add a Markdown file to `_posts/` named `YYYY-MM-DD-title.md`:

```markdown
---
title: 'Post title'
date: YYYY-MM-DD
permalink: /posts/YYYY/MM/title/
tags:
  - claude
---

Post body in Markdown.
```

Commit and push to `main`; GitHub Pages rebuilds the site automatically.

## Local preview

```bash
bundle install
bundle exec jekyll serve
```
