# Site Structure

## Directory Layout

```text
blog/
├── archetypes/
│   └── default.md              # Front matter template for hugo new
├── assets/                     # Processed by Hugo Pipes (SCSS, JS)
├── config.yaml                 # Site configuration
├── content/
│   ├── about/                  # /about/ page
│   ├── posts/                  # Blog posts (page bundles)
│   │   └── <post-name>/
│   │       ├── index.md        # Post content + front matter
│   │       └── *.{jpg,png,svg} # Co-located images
│   ├── search.md               # /search/ page (Pagefind)
│   └── _index.md               # Homepage content
├── layouts/
│   ├── _default/
│   │   ├── _markup/            # Render hook overrides
│   │   └── search.html         # Search page template
│   └── partials/
│       ├── extend_footer.html  # Footer additions (GoatCounter analytics)
│       ├── extend_head.html    # Head additions
│       ├── head.html           # Full head override
│       └── process-image.html  # Responsive image processing
├── docs/
│   ├── configuration-reference.md  # Detailed config.yaml reference
│   ├── content-workflow.md         # Content creation guide
│   ├── custom-layouts.md           # Layout overrides and render hooks
│   ├── deployment-pipeline.md      # GitHub Actions CI/CD workflow
│   ├── hugo-commands.md            # Dev server, build, and deploy commands
│   └── site-structure.md           # This file
├── public/                     # Build output (gitignored)
├── themes/PaperMod/            # PaperMod theme
└── package.json                # npm deps (pagefind)
```

## Key config.yaml Settings

| Setting | Value | Purpose |
|---------|-------|---------|
| `baseURL` | `https://jetm.github.io/blog/` | GitHub Pages base URL |
| `theme` | `PaperMod` | Hugo theme |
| `params.defaultTheme` | `auto` | Light/dark mode follows system |
| `params.ShowToc` | `true` | Table of contents on posts |
| `params.ShowReadingTime` | `true` | Estimated reading time |
| `params.ShowShareButtons` | `true` | LinkedIn + Twitter share |
| `params.goatcounter.code` | `jetm` | Analytics tracking |

## Theme Customization Points

PaperMod customization is done through layout overrides in `layouts/`, not by editing theme files:

- **`layouts/partials/head.html`** - full `<head>` override (Pagefind CSS, preloads)
- **`layouts/partials/extend_head.html`** - additional head content
- **`layouts/partials/extend_footer.html`** - footer additions (GoatCounter script)
- **`layouts/partials/process-image.html`** - responsive image partial
- **`layouts/_default/_markup/`** - markdown render hook overrides

## Search

Search uses [Pagefind](https://pagefind.app/) (static search library):
- Index built post-Hugo via `npm run build:search` (`pagefind --site public`)
- Search page at `/search/` using `layouts/_default/search.html`
- Declared in `package.json` as a dev dependency
