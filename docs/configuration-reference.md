# Configuration Reference

Detailed reference for `config.yaml` settings beyond what's covered in `site-structure.md`.

## Image Processing

Hugo's built-in image processing is configured at two levels:

### Global (`imaging`)

```yaml
imaging:
  quality: 75
  resampleFilter: Lanczos
  anchor: Smart
```

- **quality**: JPEG/WebP compression quality (1-100)
- **resampleFilter**: Resampling algorithm for resizing (Lanczos = high quality, slower)
- **anchor**: Crop anchor point (Smart = content-aware cropping)

### Custom Responsive Pipeline (`params.imageProcessing`)

```yaml
params:
  imageProcessing:
    quality: 75
    formats:
      - webp
    lazyLoading: true
    responsiveSizes:
      - 360
      - 480
      - 720
      - 1080
      - 1500
```

These params are consumed by the custom `process-image.html` partial (see `docs/custom-layouts.md`):

- **formats**: Output formats for srcset generation (currently WebP only)
- **lazyLoading**: Adds `loading="lazy"` to images
- **responsiveSizes**: Breakpoint widths (px) for srcset variants. Only sizes <= the original image width are generated.

Responsive processing only activates in production (`HUGO_ENVIRONMENT=production` or `params.env: production`).

## Syntax Highlighting

```yaml
markup:
  highlight:
    codeFences: false
    guessSyntax: true
    lineNos: false
    noClasses: true
    style: monokai
```

Hugo's built-in code fence highlighting is **disabled** (`codeFences: false`). Instead, Prism.js handles all syntax highlighting via CDN resources loaded in `extend_head.html` and `extend_footer.html`. See `docs/custom-layouts.md` for details.

The `style: monokai` and other settings are retained as fallback but have no effect while `codeFences` is disabled.

## Analytics (GoatCounter)

```yaml
params:
  goatcounter:
    code: "jetm"
```

GoatCounter is a privacy-friendly analytics service. The tracking script is injected by `extend_footer.html` only in production. It respects the browser's Do Not Track (`navigator.doNotTrack`) header - tracking is disabled when DNT is set.

Dashboard: `https://jetm.goatcounter.com`

## Output Formats

```yaml
outputs:
  home:
    - HTML
    - RSS
    - JSON
```

The homepage generates three output formats:
- **HTML**: Standard web pages
- **RSS**: Feed at `/index.xml` (limited to 20 items via `rss.limit`)
- **JSON**: Used by search/index features

## RSS

```yaml
rss:
  limit: 20
```

Limits the RSS feed to the 20 most recent items.

## Privacy

```yaml
privacy:
  disqus:
    disable: true
  googleAnalytics:
    disable: true
```

Disqus comments and Google Analytics are explicitly disabled. GoatCounter is used for analytics instead.

## Build Flags

```yaml
buildDrafts: false
buildFuture: false
buildExpired: false
```

Production builds exclude drafts, future-dated, and expired content. Use `hugo server -D` locally to preview drafts.

## Pagination

```yaml
pagination:
  pagerSize: 10
```

Post listing pages show 10 posts per page.
