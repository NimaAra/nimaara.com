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

### Prerequisites
- [Ruby](https://www.ruby-lang.org/en/downloads/) >= 2.7.0
- [Bundler](https://bundler.io/) (`gem install bundler`)

### Setup & Run
```
bundle install
bundle exec jekyll serve
```

The site will be available at `http://localhost:4000`.

## Project Structure

| Path | Purpose |
|------|---------|
| `_posts/` | Blog posts (Markdown) |
| `_layouts/` | Page templates (`home.html`, `post.html`) |
| `_includes/` | Reusable HTML partials (head, footer, nav, share links, analytics) |
| `_config.yml` | Site configuration (title, author, theme, plugins) |
| `css/` | Custom CSS overrides |
| `assets/` | Images and static files |
