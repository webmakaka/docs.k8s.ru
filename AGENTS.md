# docs.k8s.ru

Russian-language Jekyll static site about Kubernetes / DevOps / GitOps.

## Dev server

```bash
docker-compose up
```
Serves at `http://localhost:80`. Uses `jekyll/jekyll:3` image, watches for changes (`--watch --incremental`).

## Content

All pages are Markdown in `website/`, organized by category (`01-tools/`, `02-basics/`, `05-devops/`).  
Each page needs YAML front matter with `layout: page` and `permalink: ...`.  
Home page is `website/00-index.md` with `permalink: /`.

## Build

CI (GitHub Actions) builds the Docker image on push/PR to `main`:

```bash
docker build ./ -f ./Dockerfile -t marley/docs.k8s.ru:latest
```

The multi-stage `Dockerfile`: (1) `bundle exec jekyll build` → `_site/`, (2) nginx serves `_site/` on port 80.  
html-proofer runs during Docker build (`bundle exec htmlproofer ./_site ...`).

## Formatting

Prettier with `singleQuote: true`, `bracketSpacing: true`. Run on Markdown/HTML files as needed.

## Git

- `_site/`, `Gemfile.lock`, `.sass-cache`, `.jekyll-cache`, `.jekyll-metadata` are gitignored
- Mirrored to GitLab (`webmakaka/docs.k8s.ru`) via GitHub Action
