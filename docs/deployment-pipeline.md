# Deployment Pipeline

The site is deployed to GitHub Pages via a GitHub Actions workflow at `.github/workflows/deploy.yml`.

## Triggers

- **Push to `main`** - automatic build and deploy
- **Manual dispatch** (`workflow_dispatch`) - trigger from GitHub Actions UI with optional Hugo version override

## Build Steps

1. Install Hugo Extended (`hugo_extended_<version>_linux-amd64.deb`)
2. Setup Node.js 18
3. Checkout repo with submodules (recursive, full history)
4. Configure GitHub Pages
5. Install npm dependencies (`npm ci`)
6. Build with Hugo (`hugo --minify --environment production`)
7. Build Pagefind search index (`npx pagefind --site public`)
8. Upload `public/` as Pages artifact
9. Deploy to GitHub Pages (`actions/deploy-pages@v4`)

## Hugo Version

Default: **0.154.2** (Hugo Extended edition).

Override via manual dispatch by specifying a `hugoVersion` input in the workflow UI.

## Environment Variables

| Variable | Value | Purpose |
|----------|-------|---------|
| `HUGO_ENVIRONMENT` | `production` | Enables production-only features (responsive images, analytics) |
| `TZ` | `America/Los_Angeles` | Timezone for date rendering |
| `HUGO_CACHEDIR` | `/tmp/hugo_cache` | Cache location for Hugo modules |

## Caching

Two caches speed up builds:

- **Hugo modules** - keyed on `go.sum` hash, stored at `/tmp/hugo_cache`
- **npm packages** - keyed on `package-lock.json` hash, stored at `~/.npm`

## Concurrency

Only one deployment runs at a time (group: `pages`). New pushes cancel in-progress builds.

## Permissions

The workflow uses minimal GITHUB_TOKEN permissions: `contents: read`, `pages: write`, `id-token: write`.
