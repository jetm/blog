# Custom Layouts

All customizations are in `layouts/`, overriding PaperMod theme defaults. Theme files in `themes/PaperMod/` are never edited directly.

## Image Rendering Pipeline

Markdown images (`![alt](image.png)`) are processed through a two-stage pipeline:

### Stage 1: Render Hook (`layouts/_default/_markup/render-image.html`)

Intercepts all markdown image syntax and:

1. Resolves relative paths to page bundle resources (or global resources)
2. Checks if the image format is processable (jpg, jpeg, png, gif, bmp, tif, webp with Hugo Extended)
3. **Processable + production**: delegates to `process-image.html` partial
4. **Non-processable or dev mode**: renders a plain `<img>` with lazy loading and async decoding

User-specified markdown attributes (e.g., `{width="300" class="photo"}`) are preserved and merged with base attributes. User attributes override defaults.

If a `title` is provided (`![alt](img "title")`), the image is wrapped in a `<figure>` with `<figcaption>`.

### Stage 2: Process Image Partial (`layouts/partials/process-image.html`)

Generates responsive images with srcset:

1. Reads configured formats and quality from `site.Params.imageProcessing`
2. For each format × each responsive size (where size <= original width), generates a resized variant
3. Builds a `srcset` attribute with all variants plus the original as fallback
4. Sets `sizes="(min-width: 768px) 720px, 100vw"`

**Only activates in production** (`hugo.IsProduction` or `params.env: production`). In dev mode, images are served unprocessed.

Parameters accepted (as dict):
- `image` (required): Hugo image resource
- `alt` (required): Alt text
- `title`: Optional, triggers figure/figcaption wrapping
- `loading`: `"lazy"` (default) or `"eager"`
- `class`: CSS class names
- `attributes`: Additional HTML attributes dict
- `sizes`: Override responsive sizes array
- `asFigure`: Force figure wrapping

## Syntax Highlighting (Prism.js)

Hugo's built-in code fence highlighting is disabled (`codeFences: false` in config.yaml). Prism.js handles all highlighting via CDN.

### CSS (`layouts/partials/extend_head.html`)

- **Theme**: Tomorrow theme (`prismjs-tomorrow-theme`)
- **Plugins**: Toolbar CSS, Line Numbers CSS
- **Fallback**: `<noscript>` block provides basic styling when JS is disabled

### JavaScript (`layouts/partials/extend_footer.html`)

Loaded with `defer` in dependency order:

1. **Core**: `prism.min.js`
2. **Languages**: JavaScript, Python, Go, Bash, YAML
3. **Plugins**: Toolbar → Copy to Clipboard → Line Numbers

**Initialization script** runs on `DOMContentLoaded`:
- Adds `line-numbers` class to all `<pre>` elements
- Calls `Prism.highlightAll()`

To add a new language: add another `<script defer>` tag for the Prism.js language component in `extend_footer.html`.

## Analytics (GoatCounter)

Configured in `layouts/partials/extend_footer.html`, after the Prism.js scripts.

- Only loads in production mode
- Reads `site.Params.goatcounter.code` for the account identifier
- Respects Do Not Track: sets `window.goatcounter.no_onload = true` when `navigator.doNotTrack === '1'`
- Loads the counter script from `gc.zgo.at/count.js`

## Search (Pagefind)

- **Search page template**: `layouts/_default/search.html`
- **Custom CSS**: `assets/css/extended/pagefind-custom.css` maps PaperMod theme CSS variables (--primary, --content, --theme, --border, --radius) to Pagefind UI variables
- **Search index**: built post-Hugo by `npx pagefind --site public` (see `docs/deployment-pipeline.md`)

## Head Override (`layouts/partials/head.html`)

Full `<head>` override (not an extension) that provides:
- Meta tags (robots, keywords, description, author, canonical URL)
- CSS bundle: concatenated and minified with fingerprinting
- OpenGraph, Twitter Card, and Schema.org JSON-LD metadata
- Favicon and icon links
- Pagefind CSS integration (replaces the disabled Fuse.js/FastSearch)

## Extension Points

Safe places to add new functionality without touching theme files:

| File | Purpose | Currently Used For |
|------|---------|-------------------|
| `layouts/partials/extend_head.html` | Add CSS/meta to `<head>` | Prism.js CSS, noscript fallback |
| `layouts/partials/extend_footer.html` | Add scripts before `</body>` | Prism.js JS, GoatCounter |
| `assets/css/extended/*.css` | Additional CSS (bundled by Hugo) | Pagefind UI theming |
