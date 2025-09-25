# Sometimes Code Blog

A personal blog built with Hugo, designed for longevity and simplicity.

## Quick Start

1. **Install Hugo**: `brew install hugo`
2. **Clone with submodules**: `git clone --recurse-submodules <repo-url>`
3. **Start development server**: `hugo server --buildDrafts`
4. **View at**: http://localhost:1313

## Writing Content

### New Post
```bash
hugo new content posts/my-new-post.md
```

### Categories
- Use `categories = ['programming']` for technical posts
- Use `categories = ['family']` for personal/family posts

### Publishing
- Set `draft = false` in the frontmatter to publish

## Project Structure

```
├── content/
│   ├── posts/          # Blog posts
│   └── resume.md       # Resume page
├── themes/ananke/      # Hugo theme (git submodule)
├── hugo.toml           # Site configuration
└── .github/workflows/  # Deployment automation
```

## Deployment

### Digital Ocean Setup

1. **Server Requirements**:
   - Ubuntu 22.04+ droplet
   - Nginx installed
   - SSL certificate (use certbot)

2. **GitHub Secrets** (for automatic deployment):
   - `DO_HOST`: Your server IP or domain
   - `DO_USERNAME`: SSH username (usually `root` or `ubuntu`)
   - `DO_SSH_KEY`: Private SSH key for server access

3. **Server Setup**:
   ```bash
   # Create web directory
   sudo mkdir -p /var/www/sometimescode.com

   # Copy nginx config
   sudo cp nginx-example.conf /etc/nginx/sites-available/sometimescode.com
   sudo ln -s /etc/nginx/sites-available/sometimescode.com /etc/nginx/sites-enabled/

   # Get SSL certificate
   sudo certbot --nginx -d sometimescode.com -d www.sometimescode.com

   # Restart nginx
   sudo systemctl restart nginx
   ```

### Manual Deployment

```bash
# Build site
hugo --minify

# Upload to server
scp -r public/* user@server:/var/www/sometimescode.com/
```

## Development

- **Theme**: [Ananke](https://github.com/theNewDynamic/gohugo-theme-ananke) with custom dark mode
- **Hugo Version**: 0.150.0+
- **Content Format**: Markdown with YAML frontmatter
- **Dark Mode**: Custom CSS and JavaScript implementation with toggle button

## Philosophy

This blog is built for longevity:
- **Markdown files**: Human-readable, future-proof content
- **Static generation**: No database dependencies
- **Simple theme**: Minimal dependencies, maximum compatibility
- **Git history**: Complete version control of all content

The goal is for this blog to be readable and maintainable for decades to come.