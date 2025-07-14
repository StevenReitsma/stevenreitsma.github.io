# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a Jekyll-based personal blog and portfolio website (reitsma.io) hosted on GitHub Pages. The site showcases Steven Reitsma's work in data engineering, infrastructure, and machine learning.

## Development Commands

### Jekyll Development
```bash
# Install dependencies (requires Ruby and bundler)
bundle install

# Run development server
bundle exec jekyll serve

# Build the site
bundle exec jekyll build

# Build for production
JEKYLL_ENV=production bundle exec jekyll build
```

### GitHub Pages Deployment
The site is automatically deployed to GitHub Pages when changes are pushed to the master branch. No manual deployment commands are needed.

## Architecture and Structure

### Core Jekyll Structure
- `_config.yml` - Main Jekyll configuration
- `_data/settings.yml` - Site settings and author information
- `_includes/` - Reusable HTML components (header, footer, etc.)
- `_layouts/` - Page templates (default, post, projects, etc.)
- `_sass/` - Sass stylesheets organized by components
- `_pages/` - Static pages (about, blog, projects)
- `_posts/` - Blog posts in Markdown format
- `_projects/` - Project descriptions and portfolio items

### Content Organization
- Blog posts follow naming convention: `YYYY-MM-DD-title.markdown`
- Projects are stored as individual Markdown files in `_projects/`
- Images are stored in `images/` directory
- JavaScript files are in `js/` directory

### Key Features
- Blog with pagination (5 posts per page)
- Project portfolio with custom collection
- SEO optimization with jekyll-seo-tag
- Google Analytics integration
- Responsive design with custom Sass architecture
- Search functionality
- Tag-based post categorization

### Content Types
- **Blog Posts**: Technical articles about Kubernetes, ML, security, etc.
- **Projects**: Professional work and personal projects with descriptions
- **About Page**: Personal information and bio
- **Tags**: Organized by technology (Kubernetes, Istio, Security, etc.)

### Styling Architecture
Sass files are organized in a structured approach:
- `0-settings/` - Variables, mixins, helpers
- `1-tools/` - Grid, normalization, syntax highlighting
- `2-base/` - Base styles
- `3-modules/` - Component-specific styles
- `4-layouts/` - Page layout styles