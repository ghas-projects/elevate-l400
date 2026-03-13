# GitHub Advanced Security L400

Course material for the GitHub Advanced Security (GHAS) L400 Elevate training, built with [mdBook](https://rust-lang.github.io/mdBook/).

## Topics

- **Supply Chain Security** — Dependency graph, dependency submission, SBOMs & attestation, Dependabot alerts & updates, dependency review
- **Secret Scanning** — Push protection, custom patterns
- **Code Scanning** — CodeQL CLI, CodeQL fundamentals, models-as-data, custom query packs

## Local Development

```bash
# Install mdBook (https://rust-lang.github.io/mdBook/guide/installation.html)
cargo install mdbook

# Build the book
mdbook build

# Serve locally with live reload
mdbook serve --open
```

## Deployment

The book is automatically published to GitHub Pages on push to `main` via the workflow in `.github/workflows/deploy-pages.yml`.
