# Publishing Guide

Harness Guides is designed for `Material for MkDocs` and free hosting on GitHub Pages.

## Repository shape

```text
harness-guides/
├── docs/
├── mkdocs.yml
└── .github/workflows/docs.yml
```

## Recommended setup

1. Create the GitHub repository.
2. Push this scaffold.
3. Enable GitHub Pages with GitHub Actions as the source.
4. Set your real `site_url`, `repo_url`, and `repo_name` in `mkdocs.yml`.
5. Replace placeholder branding assets in `docs/assets/`.

## Local development

Install:

```bash
pip install mkdocs-material
```

Run:

```bash
mkdocs serve
```

Build:

```bash
mkdocs build
```

## What to customize next

- chapter split-outs from the full blueprint
- diagrams and architecture visuals
- social preview image
- custom domain
- blog or release notes section
