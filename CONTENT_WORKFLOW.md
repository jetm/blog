# Content Workflow Guide

This guide documents the workflow for creating and managing blog content using Hugo with the PaperMod theme.

## A. Page Bundle Structure

Hugo supports **page bundles**, which are directories containing an `index.md` file along with co-located resources (images, data files, etc.). This keeps all content related to a post organized together.

### Directory Structure

```
content/posts/my-post/
├── index.md          # Post content and front matter
├── featured.jpg      # Cover image
├── diagram.png       # Inline image
└── data.json         # Any other resources
```

### Benefits

- **Organization**: Images and assets stay with their post
- **Portability**: Easy to move or archive posts with all their dependencies
- **Relative paths**: Reference images simply as `![Alt](image.jpg)`
- **Hugo processing**: Automatic responsive image handling

## B. Front Matter Schema Reference

All front matter uses YAML format (delimited by `---`).

| Field | Type | Required | Description | Example |
|-------|------|----------|-------------|---------|
| `title` | string | Yes | Post title displayed on the page | `"My First Post"` |
| `date` | date | Yes | Publication date (ISO 8601) | `2024-01-15T10:30:00Z` |
| `lastmod` | date | No | Last modification date | `2024-01-20T14:00:00Z` |
| `draft` | boolean | Yes | If true, post is hidden in production | `true` or `false` |
| `description` | string | Recommended | SEO meta description (150-160 chars) | `"A brief summary..."` |
| `slug` | string | No | Custom URL slug (overrides filename) | `"custom-url"` |
| `toc` | boolean | No | Show table of contents (default: true) | `true` |
| `readingTime` | boolean | No | Show estimated reading time | `true` |
| `images` | array | No | Open Graph images for social sharing | `["og-image.jpg"]` |
| `tags` | array | No | Post tags for taxonomy | `["hugo", "tutorial"]` |
| `categories` | array | No | Post categories for taxonomy | `["development"]` |
| `outputs` | array | No | Output formats | `["HTML"]` |

### Cover Image Fields

| Field | Type | Description | Example |
|-------|------|-------------|---------|
| `cover.image` | string | Path to cover image | `"featured.jpg"` |
| `cover.alt` | string | Alt text for accessibility | `"Screenshot of the app"` |
| `cover.caption` | string | Caption displayed below image | `"Figure 1: Architecture"` |
| `cover.relative` | boolean | Use relative path (for page bundles) | `true` |

### Config Overrides

These front matter fields override global settings in `config.yaml`:

- `toc` overrides `params.ShowToc`
- `readingTime` overrides `params.ShowReadingTime`
- `cover` settings override `params.cover` defaults

## C. Creating New Posts Workflow

### Step 1: Create Post Using Hugo Command

```bash
hugo new posts/<post-name>/index.md
```

Replace `<post-name>` with a URL-friendly slug:
- Use lowercase letters
- Replace spaces with hyphens
- Avoid special characters

**Example:**
```bash
hugo new posts/getting-started-with-hugo/index.md
```

This creates:
```
content/posts/getting-started-with-hugo/
└── index.md
```

### Step 2: Edit Front Matter

Open the generated `index.md` and update the front matter:

1. **Update `title`** if the auto-generated one needs adjustment
2. **Add `description`** - Write a compelling 150-160 character summary for SEO
3. **Add `tags`** - Relevant keywords for categorization
4. **Add `categories`** - Broader topic classification
5. **Configure display options** - Adjust `toc` and `readingTime` if needed
6. **Set `draft: false`** when ready to publish

### Step 3: Add Images to Page Bundle

Place images in the same directory as `index.md`:

```
content/posts/getting-started-with-hugo/
├── index.md
├── featured.jpg      # Cover image
├── screenshot-1.png  # Inline image
└── diagram.svg       # Another inline image
```

Reference images in your content with relative paths:
```markdown
![Screenshot of the dashboard](screenshot-1.png)
```

For cover images, add to front matter:
```yaml
cover:
  image: "featured.jpg"
  alt: "Hugo logo with PaperMod theme"
  caption: "Getting started with Hugo and PaperMod"
  relative: true
```

### Step 4: Write Content

Use standard Markdown syntax:

```markdown
## Introduction

Your introduction text here...

### Subsection

More content with a [link](https://example.com).

#### Code Example

```python
def hello():
    print("Hello, World!")
```

![Diagram showing the architecture](diagram.svg)
```

**Tips:**
- Use `##` and `###` headings for TOC generation
- Specify language in code blocks for syntax highlighting
- Use relative paths for co-located images

### Step 5: Preview Locally

Start the development server with drafts enabled:

```bash
hugo server -D
```

Access your site at: `http://localhost:1313/blog/`

The server auto-reloads on file changes.

### Step 6: Publish

1. Set `draft: false` in front matter
2. Update `lastmod` if making significant changes to existing content
3. Commit and push to trigger deployment:

```bash
git add .
git commit -m "Add post: Getting Started with Hugo"
git push
```

The GitHub Actions workflow (`.github/workflows/deploy.yml`) handles deployment automatically.

## D. Image Best Practices

### Formats

| Format | Use Case |
|--------|----------|
| JPEG | Photographs, complex images with gradients |
| PNG | Screenshots, diagrams, images requiring transparency |
| SVG | Logos, icons, simple vector graphics |
| WebP | Modern alternative with better compression |

### Naming Conventions

- Use descriptive, lowercase names
- Separate words with hyphens
- Include purpose in name when helpful

**Good:** `architecture-diagram.png`, `dashboard-screenshot.jpg`
**Avoid:** `IMG_001.JPG`, `Screenshot 2024-01-15.png`

### Cover Images

- **Minimum size:** 1200x630px for optimal social media display
- **Aspect ratio:** 1.91:1 (Open Graph standard)
- **File size:** Keep under 200KB when possible
- **Format:** JPEG for photos, PNG for graphics

### Accessibility

Always provide meaningful `alt` text:

```markdown
<!-- Good -->
![Bar chart showing monthly revenue growth from $10K to $50K](revenue-chart.png)

<!-- Avoid -->
![Chart](chart.png)
![](image.jpg)
```

## E. Examples

### Basic Post Creation

```bash
# Create the post
hugo new posts/my-first-post/index.md

# Start dev server
hugo server -D

# Edit content/posts/my-first-post/index.md
```

### Post with Cover Image

```yaml
---
title: "Building a REST API with Go"
date: 2024-01-15T10:00:00Z
lastmod: 2024-01-15T10:00:00Z
draft: false
description: "Learn how to build a production-ready REST API using Go and the standard library."
tags: ["go", "api", "backend"]
categories: ["tutorials"]
cover:
  image: "go-api-cover.jpg"
  alt: "Go gopher holding a REST API diagram"
  caption: "Building APIs with Go"
  relative: true
---

Your content here...
```

### Post with Multiple Images

```markdown
---
title: "Docker Compose for Development"
date: 2024-01-20T14:30:00Z
draft: false
description: "Set up a complete development environment with Docker Compose."
tags: ["docker", "devops"]
categories: ["tutorials"]
---

## Architecture Overview

Our setup uses three containers:

![Architecture diagram showing web, api, and database containers](architecture.png)

## Configuration

Here's our docker-compose.yml structure:

![Screenshot of VS Code with docker-compose.yml](vscode-compose.png)

## Running the Stack

After running `docker compose up`, you'll see:

![Terminal output showing successful container startup](terminal-output.png)
```

### Using Taxonomies

```yaml
tags:
  - "javascript"
  - "react"
  - "frontend"
  - "testing"

categories:
  - "tutorials"
  - "web-development"
```

Tags and categories create archive pages at `/tags/` and `/categories/`.

## F. Pre-Publish Checklist

Before setting `draft: false`, verify:

- [ ] **Front matter is valid YAML** - No syntax errors
- [ ] **Title is clear and descriptive**
- [ ] **Description is present** - 150-160 characters for SEO
- [ ] **Tags and categories are set** - Properly formatted arrays
- [ ] **All images have alt text** - For accessibility
- [ ] **Cover image is configured** (if desired)
- [ ] **Draft status is `false`** - Ready to publish
- [ ] **Content previewed locally** - `hugo server -D`
- [ ] **All links work correctly** - Internal and external
- [ ] **Code blocks have language specified** - For syntax highlighting
- [ ] **Headings follow hierarchy** - h2, h3, h4 in order
- [ ] **lastmod is updated** (if editing existing post)
