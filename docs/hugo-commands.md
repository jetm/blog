# Hugo Commands

## Development

```bash
# Start dev server with drafts visible
hugo server -D

# Dev server accessible at http://localhost:1313/blog/
# Auto-reloads on file changes
```

## Content Creation

```bash
# Create a new post (page bundle with index.md)
hugo new posts/<post-name>/index.md

# Example
hugo new posts/getting-started-with-hugo/index.md
# Creates content/posts/getting-started-with-hugo/index.md
```

## Build

```bash
# Production build (outputs to public/)
hugo

# Build search index (requires npm dependencies)
npm run build:search
# Runs: pagefind --site public
```

## Deployment

Deployment is automated via `.github/workflows/deploy.yml` - push to the default branch triggers build and deploy to GitHub Pages.
