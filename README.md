# idris10x Blog

A Jekyll-powered blog hosted on GitHub Pages.

## Quick Start

1. **Create a new repository** on GitHub named `idris10x-blog`
2. **Push this code** to the repository:
   ```bash
   git init
   git add .
   git commit -m "Initial blog setup"
   git remote add origin https://github.com/YOUR_USERNAME/idris10x-blog.git
   git push -u origin main
   ```

3. **Enable GitHub Pages**:
   - Go to your repository on GitHub
   - Settings → Pages
   - Under "Build and deployment", select:
     - Source: **Deploy from a branch**
     - Branch: **gh-pages** (or wait for first build with Actions)
   - Or use the "GitHub Actions" option for automatic builds

4. **Your blog will be live at**: `https://YOUR_USERNAME.github.io/idris10x-blog`

## Writing Blog Posts

Create new posts in `_posts/` directory with the format:
```
YYYY-MM-DD-your-post-title.md
```

Add Jekyll front matter at the top:

```markdown
---
layout: post
title: "Your Post Title"
date: 2026-03-14
description: "A short description for the post"
---

Your content here...
```

## Local Development

To preview locally:

```bash
# Install dependencies
gem install jekyll bundler
bundle install

# Serve locally
bundle exec jekyll serve

# Build for production
bundle exec jekyll build
```

## Customization

- Edit `_config.yml` to change site title, description, URL
- Modify `assets/css/style.css` for custom styling
- Edit `index.html` to customize the homepage
