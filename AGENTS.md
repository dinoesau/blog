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

## Post front matter extras

- `description` - shown in SEO meta and social previews; always set it on real posts.
- `image: cover.png` - adds a cover: OG/Twitter cards (`summary_large_image`), responsive article image, and homepage thumbnails.
- `series: ["Error Handling"]` - groups posts into a "Series" taxonomy shown in the sidebar widget. See `/series/error-handling/`.

## Cover images

Real posts need a `cover.png` in BOTH the en and es page bundles, with `image: cover.png` in both front matters.

The covers are branded 1200x630 PNGs (OG ratio 1.91:1) generated from a template SVG via `rsvg-convert` (macOS: `/opt/homebrew/bin/rsvg-convert`):

1. Write an SVG with the brand gradient `#1e3a8a` → `#0e7490`, the `EM` monogram in white `Arial, Helvetica, sans-serif` (centered), and a light accent bar `#7dd3fc`.
   Do NOT put the post title in the image: covers are reused as 250x150 tiles in "Related content" where embedded text becomes unreadable and duplicates the tile title.
   Use the same monogram design for both languages (no text means no language-specific SVG).
2. Render to PNG (scale the SVG to 1200x630 so the monogram stays crisp):
   ```bash
   rsvg-convert -w 1200 -h 630 -o content/en/post/<slug>/cover.png cover.svg
   rsvg-convert -w 1200 -h 630 -o content/es/post/<slug>/cover.png cover.svg
   ```
3. Set `image: cover.png` in both `index.md` front matters.

`svg` files are just the source template; only the PNGs are committed.

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
