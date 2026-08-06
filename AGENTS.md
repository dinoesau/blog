# AGENTS.md

## Project

Personal blog of Esau Martinez built with [Hugo](https://gohugo.io/) (extended) and the [Stack v4](https://stack.cai.im/) theme.
Deployed to GitHub Pages via `.github/workflows/deploy.yml`; domain `esau.com.mx`.
The site is bilingual: English (default, served at the root) and Spanish (served under `/es/`).

## Bilingual content (en + es)

Every post exists in two versions:

- `content/en/post/<slug>/index.md` - English
- `content/es/post/<slug>/index.md` - Spanish translation

Both files must use the same directory name (`<slug>`) so Hugo links them as translations.
Keep front matter (date, categories, tags) identical across the two versions.

### Language config gotchas

- `config/_default/languages.toml` defines non-overlapping `contentDir` per language (`content/en` and `content/es`).
  Do not move content back to `content/` root; Hugo rejects overlapping language content dirs.
- Main menus are defined per-language in `languages.toml`, NOT in page front matter.
  Front matter `menu:` entries merge across languages and pollute the other language's menu.
- Create both `content/en/post/<slug>/index.md` and `content/es/post/<slug>/index.md` for every new post.

## Internal cross-post links

Use the `relref` shortcode, never relative paths:

```markdown
[Stop Validating Everywhere]({{< relref "/post/typescript-error-handling-architecture" >}})
```

Relative paths like `../foo/` resolve against the rendered permalink (`/p/:slug/`) and 404.
`relref` paths are relative to the language content root and resolve to the same-language page.

## Keep labels and categories in sync

`labels.md` and `categories.md` at the repo root track the tags and categories used across posts.
Update them whenever a post introduces a new tag or category.

## Commands

- `hugo server` - dev server with live reload
- `hugo` - build into `public/` (the verification step; there are no tests or linters)
- `hugo new content/en/post/<slug>/index.md` - scaffold an English post

## Deploy

Push to `main`/`master` triggers the GitHub Actions deploy.
`update-theme.yml` runs daily and bumps the theme via `hugo mod get -u`.

Theme config documentation: https://stack.cai.im/config/
