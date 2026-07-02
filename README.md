# jonzarecki.github.io

Personal blog, built with Jekyll and hosted on GitHub Pages.

## Local development

```bash
bundle install
bundle exec jekyll serve
```

Then open `http://localhost:4000`.

## Writing a new post

Add a Markdown file to `_posts/` named `YYYY-MM-DD-title.md` with front matter:

```yaml
---
title: "Post title"
date: YYYY-MM-DD
tags: [tag1, tag2]
excerpt: "One or two sentence summary."
---
```
