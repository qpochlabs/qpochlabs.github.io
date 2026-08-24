# qpoch.com — Blog

Blog posts by Hari Prasad, served at [qpoch.com](https://qpoch.com). Same look and feel as [hari.qpoch.com](https://hari.qpoch.com) (design from [Jon Barron's website](https://jonbarron.info/)), built with Jekyll so posts can be written in Markdown.

## Writing a post

Add a Markdown file to `_posts/` named `YYYY-MM-DD-title.md`:

```markdown
---
title: "My post title"
---

Post body in Markdown. `## Subheadings` render in the site's 22px subheading style.
```

The home page list, post pages, RSS feed (`/feed.xml`) and sitemap are generated automatically.

## Running locally

```sh
bundle install
bundle exec jekyll serve
```

Then open http://localhost:4000.

## Deploying

Push to the `main` branch of the GitHub Pages repository. The `CNAME` file points the site at `qpoch.com`.
