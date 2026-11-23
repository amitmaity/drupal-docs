# Drupal 11 Headless & Custom Coding Guide

A comprehensive developer guide for Drupal 11, focused on headless/decoupled architecture, custom module development, and Symfony components.

**Live site:** [https://amitmaity.github.io/drupal-docs/](https://amitmaity.github.io/drupal-docs/)

## Local development

Use Python 3.10 or newer (CI uses 3.13). The macOS system `python3` (3.9) is end-of-life; install a current Python via Homebrew, pyenv, or uv.

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
mkdocs serve
```

Open [http://127.0.0.1:8000/drupal-docs/](http://127.0.0.1:8000/drupal-docs/) in your browser.

## Build

```bash
mkdocs build --strict
```

Output is written to `site/`.

## Deploy to GitHub Pages

1. Push this repository to GitHub as `drupal-docs`.
2. In **Settings → Pages**, set **Source** to **GitHub Actions**.
3. Push to `main` — the workflow in `.github/workflows/deploy.yml` builds and deploys automatically.

## Structure

| Directory | Content |
|-----------|---------|
| `docs/intro/` | What changed in D11, architecture, why headless |
| `docs/getting-started/` | Installation, Composer workflows |
| `docs/internals/` | Bootstrap, request lifecycle, core systems |
| `docs/dependency-injection/` | DI patterns and service container |
| `docs/symfony/` | All Symfony components Drupal uses |
| `docs/custom-coding/` | Custom modules, forms, plugins, entities |
| `docs/headless/` | REST, JSON:API, auth, Next.js |
| `docs/caching/` | Render cache, tags, contexts |
| `docs/deployment/` | GitHub Pages and Drupal hosting |
