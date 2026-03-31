# Harness Guides

Documentation site for agent harness engineering, built with `MkDocs Material` and intended for free hosting on GitHub Pages.

## Local development

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
mkdocs serve
```

## Build

```bash
mkdocs build
```

## Deploy

Push to `main` and let the GitHub Actions workflow in `.github/workflows/docs.yml` publish the site to GitHub Pages.

## Important config

Update these values in `mkdocs.yml` when needed:

- `site_url`
- `repo_url`
- social links
- branding assets and colors
