# Copilot instructions for pupsourcing-website

## Build, test, and lint commands

- Install dependencies (matches CI):
  - `pip install -r requirements.txt`
- Run local docs server:
  - `mkdocs serve`
- Build static site (same command used in GitHub Actions):
  - `mkdocs build`
- Tests/lint:
  - No dedicated automated test suite or linter is configured in this repository.
  - No single-test command exists; use `mkdocs build` to validate changes.

## High-level architecture

- This repository is the documentation website for `pupsourcing/core`; product source code lives in the upstream `core` repository.
- Content source files are in `docs/*.md`.
- `mkdocs.yml` defines site metadata, top-level nav, and markdown extensions.
- The site theme is fully custom (`theme.name: null`, `custom_dir: custom_theme`):
  - `custom_theme/main.html` defines the full page shell, homepage layout, and docs navigation markup.
  - `custom_theme/assets/stylesheets/main.css` contains layout/theme/responsive/syntax styles.
  - `custom_theme/assets/javascripts/theme.js` handles dark/light mode and mobile menu behavior.
- `.github/workflows/deploy.yml` is the publish pipeline: install dependencies, run `mkdocs build`, upload `site/`, deploy to GitHub Pages.

## Key conventions in this repo

- Keep navigation in sync across configuration and template:
  1. `mkdocs.yml` (`nav`)
  2. `custom_theme/main.html` (hardcoded desktop and mobile docs links)
  If you add/rename docs pages, update both places.
- `custom_theme/main.html` uses inline `onclick` handlers that call global functions (`toggleTheme`, `toggleMobileMenu`) from `theme.js`; if function names change, update template handlers too.
- Markdown authoring assumes configured MkDocs extensions (`admonition`, `codehilite`, `toc.permalink`), and styles in `main.css` rely on that output (for admonitions and header permalink anchors).
- Use `requirements.txt` as the dependency source of truth, since CI installs from it.
