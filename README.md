# nimaara.com

Personal blog built with [Jekyll](https://jekyllrb.com/) and the [minima](https://github.com/jekyll/minima) theme, hosted on [GitHub Pages](https://pages.github.com/).

## Writing a New Post

1. Create a new file in `_posts/` with the naming format:
   ```
   YYYY-MM-DD-title-of-your-post.md
   ```

2. Add front matter at the top of the file:
   ```yaml
   ---
   layout: post
   title: "Your Post Title"
   date: YYYY-MM-DD
   tags: DotNet
   ---
   ```

3. Write your content in Markdown below the front matter.

4. Push to GitHub — the site will automatically rebuild and deploy.

## Running Locally

No Ruby required — the site runs in a Docker container:

```
docker compose up
```

The site will be available at `http://localhost:4000` with live reload: edit or add files under `_posts/` and the browser refreshes automatically.

Notes:

- Drafts in `_drafts/` (filename without a date prefix, e.g. `my-post.md`) and posts with future dates appear in the preview but stay hidden on the live site.
- Stop the server with `Ctrl+C`.
- Gems are cached in the `jekyll-gems` Docker volume; the first run takes a minute, later runs start in seconds.

## Project Structure

| Path | Purpose |
|------|---------|
| `_posts/` | Blog posts (Markdown) |
| `_layouts/` | Page templates (`home.html`, `post.html`) |
| `_includes/` | Reusable HTML partials (head, footer, nav, share links, analytics) |
| `_config.yml` | Site configuration (title, author, theme, plugins) |
| `docker-compose.yml` | Local preview server (Docker, no Ruby needed) |
| `css/` | Custom CSS overrides |
| `assets/` | Images and static files |
