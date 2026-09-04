# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Static marketing site for **Restaurant Galia** (Paris 14e), deployed to GitHub Pages under the CNAME `www.restaurantgalia.com`. Content is in French. Jekyll renders Markdown pages; Gulp bundles CSS/JS/images from `_app/` sources.

## Common commands

Local dev requires both the Ruby (Jekyll) and Node (Gulp) toolchains. The `README` at the repo root documents the original Docker-based workflow (Node 8 + Gulp 3). On the host:

```bash
bundle install          # install jekyll + github-pages gem
npm install             # install gulp toolchain (Node 8 era — use nvm if needed)
npm start               # == `gulp serve`: clean → build → jekyll build → browser-sync
gulp build              # one-shot build without dev server
gulp build:styles       # rebuild just main.css
gulp build:scripts      # rebuild main.js, gallery.js, reservation.js
gulp build:images       # regenerate responsive image variants
gulp build:jekyll       # `bundle exec jekyll build` with dev config overlay
gulp clean              # wipe generated assets + _site/
```

There is no test suite or linter configured.

## Architecture

This repo is unusual for a Jekyll site: **the Gulp pipeline writes compiled assets back into the Jekyll source tree (repo root), and those compiled files are committed to git.** Jekyll then copies them into `_site/`. Do not hand-edit the root-level compiled assets — edit the sources under `_app/` and rebuild.

### Source vs. generated (both live in the repo)

| Compiled file at repo root | Source(s) under `_app/` |
| --- | --- |
| `main.css` | `_app/styles/**/*.css` (concatenated + minified) |
| `main.js` | `_app/scripts/main/*.js` (jquery, bootstrap, jasny, loadCSS — concat + uglify) |
| `gallery.js` | `_app/scripts/gallery/*.js` |
| `reservation.js` | `_app/scripts/reservation/*.js` |
| `images/**` | `_app/images/**` (resized into responsive variants by `gulp-responsive`; see the size map in `gulpfile.js`) |

Ordering matters for `main.js`: the glob order in `gulpfile.js` (jquery → bootstrap → jasny → loadCSS) is what makes jQuery plugins resolve. Preserve that if adding scripts.

### Jekyll layer

- Pages are Markdown files at the repo root (`index.md`, `chef.md`, `galia.md`, `gallery.md`, `news.md`, `reservation.md`). Each uses `layout: default` and may set a `script:` front-matter var — `_layouts/default.md` injects that as an async `<script>` tag. That's how per-page JS bundles (`gallery.js`, `reservation.js`) get loaded.
- `_includes/{head,header,footer}.html` are shared partials.
- `_config.yml` is production config; `_app/localhost_config.yml` is layered on top in dev (`--config _config.yml,_app/localhost_config.yml`) to blank out `url:` and set `development: true`.
- Deployed via `github-pages` gem (see `Gemfile`) — plugin allowlist is limited to what GitHub Pages permits (`jekyll-sitemap`, `jekyll-seo-tag`).
- `_config.yml`'s `exclude:` list keeps build tooling (`gulpfile.js`, `_app/`, `node_modules`, `Dockerfile`, etc.) out of `_site/`. Add new tooling files there if you don't want them published.

### Reservation form

`reservation.md` posts to a Formspree endpoint (`https://formspree.io/f/mknakglb`). Client-side validation and the datepicker/clockpicker init live in `_app/scripts/reservation/script.js`. Closed dates (holidays, weekly closures) are hardcoded in the `beforeShowDay` callback there — that's the file to edit when opening hours change (the recent commits `Cambio en las reservas` / `Cambio de horarios` are exactly this).

### Gulp specifics

- `_app/gulp/paths.js` centralizes all input/output paths. `paths.jekyllDir` is `''` (repo root) — that's why compiled assets land at the root.
- `_app/gulp/config.js` exposes a `--drafts` flag that adds `--drafts` to `jekyll build` and watches `_drafts/`.
- `gulp serve` watches sources and reloads via browser-sync; it serves from `_site/`, not the repo root.
- The stack is pinned to Gulp 3.x / Node 8 era. `runSequence` and task arrays (`gulp.task('x', ['y'], ...)`) are Gulp 3 syntax — don't "modernize" to Gulp 4 without also updating every task definition.