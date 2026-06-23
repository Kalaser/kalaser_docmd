# Create Repo with docmd · GitHub Templates

[![Use this template](https://img.shields.io/badge/template-launch_repo_with_docmd-blue?style=flat-square&logo=github)](https://github.com/docmd-io/docmd-template/generate)

The fastest way to start a documentation site with [docmd](https://github.com/docmd-io/docmd) — pre-configured and ready to deploy.

## Get started

1. Click **[Use this template](https://github.com/docmd-io/docmd-template/generate)** and create your repository
2. Update `docmd.config.json` with your site title and URL
3. Push to `main` — your site deploys automatically

> Your site will be live at `https://<username>.github.io/<repo>/`

## What's included

```
.github/workflows/docs.yml   # Auto-deploy on push to main
docmd.config.json            # Site title, URL, and output directory
docs/index.md                # Your first page
package.json                 # Local dev scripts
```

## Local development

```bash
npm install
npm run dev      # Live preview at localhost:3000
npm run build    # Build to site/
```

## First-time setup

GitHub Pages must be set to deploy from **GitHub Actions** (not a branch).

Go to **Settings → Pages → Source → GitHub Actions**.

This only needs to be configured once per repository.

#### [Website](https://docmd.io) • [Documentation](https://docs.docmd.io) • [MIT License](LICENSE)
