# Deploying to GitHub Pages

This guide is deployed as a static site at **[amitmaity.github.io/drupal-docs/](https://amitmaity.github.io/drupal-docs/)** using Material for MkDocs and GitHub Actions.

## Prerequisites

- GitHub account: [github.com/amitmaity](https://github.com/amitmaity)
- Repository named `drupal-docs`
- Python 3.11+ locally for testing

## One-time GitHub setup

### 1. Create the repository

```bash
cd drupal-docs
git init
git add .
git commit -m "Initial commit: Drupal 11 Headless Guide"
git branch -M main
git remote add origin git@github.com:amitmaity/drupal-docs.git
git push -u origin main
```

### 2. Enable GitHub Pages

1. Go to **github.com/amitmaity/drupal-docs → Settings → Pages**
2. Under **Build and deployment**, set **Source** to **GitHub Actions**
3. Save — no branch selection needed

### 3. Verify deployment

After pushing to `main`, the workflow runs automatically:

1. **Actions** tab → "Deploy to GitHub Pages" workflow
2. Build job: installs MkDocs, runs `mkdocs build --strict`
3. Deploy job: publishes `site/` artifact to GitHub Pages
4. Site live at `https://amitmaity.github.io/drupal-docs/`

## Workflow file

The deployment workflow at `.github/workflows/deploy.yml`:

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
          cache: pip
          cache-dependency-path: requirements.txt
      - run: pip install -r requirements.txt
      - run: mkdocs build --strict
      - uses: actions/upload-pages-artifact@v3
        with:
          path: ./site

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - uses: actions/deploy-pages@v4
        id: deployment
```

!!! note "Modern Pages deployment"
    This uses artifact-based deployment (`upload-pages-artifact` + `deploy-pages`), not the legacy `gh-pages` branch method.

## Local development

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# Live reload dev server
mkdocs serve
# Open http://127.0.0.1:8000/drupal-docs/

# Production build
mkdocs build --strict
# Output in site/
```

## MkDocs configuration

Key settings in `mkdocs.yml`:

```yaml
site_name: Drupal 11 Headless & Custom Coding Guide
site_url: https://amitmaity.github.io/drupal-docs/
repo_url: https://github.com/amitmaity/drupal-docs
repo_name: amitmaity/drupal-docs
```

The `site_url` with `/drupal-docs/` path is required for project sites (not user sites).

## Custom domain (optional)

To use a custom domain like `docs.example.com`:

1. Enter `docs.example.com` under **Settings → Pages → Custom domain**. With GitHub Actions deployment, no `CNAME` file is needed (any `CNAME` file is ignored)
2. Configure DNS: CNAME record pointing to `amitmaity.github.io`
3. Change `site_url` in `mkdocs.yml` to `https://docs.example.com/` (the site is then served from the domain root, not `/drupal-docs/`)

## Troubleshooting

!!! warning "404 on all pages"
    Verify `site_url` in `mkdocs.yml` includes the `/drupal-docs/` suffix for project repos.

!!! warning "Workflow permission denied"
    Ensure repository **Settings → Actions → General → Workflow permissions** allows read/write for the `GITHUB_TOKEN`, or that the workflow has explicit `pages: write` and `id-token: write` permissions.

!!! warning "Build fails on strict mode"
    Run `mkdocs build --strict` locally to catch broken links and missing nav entries before pushing.

## Manual deployment trigger

```bash
# Trigger deployment without a code push
gh workflow run deploy.yml
```

Or use the **Run workflow** button in the Actions tab.
