---
# Basic metadata
title: "{{ replace .File.ContentBaseName "-" " " | title }}"
date: {{ .Date }}
lastmod: {{ .Date }}
draft: true

# SEO and social media - description should be 150-160 characters
description: ""

# Override URL slug if different from filename (leave empty to use filename)
slug: ""

# Display options
toc: true
readingTime: true

# Open Graph/social media images (array of image paths)
images: []

# Taxonomies - example: ["tag1", "tag2"]
tags: []
categories: []

# Output formats
outputs:
  - "HTML"

# Cover image (uncomment and configure if needed)
# cover:
#   image: "featured.jpg"  # Path relative to page bundle
#   alt: "Description of the image for accessibility"
#   caption: "Optional caption displayed below the image"
#   relative: true  # Set to true for page bundle images
---

