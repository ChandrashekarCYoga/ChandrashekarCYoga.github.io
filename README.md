# Modern Embedded C++ — GitHub Pages Starter

A clean, measurement-driven technical blog for embedded systems and Modern C++.

## 1. Personalize it

Edit `_config.yml`:

- `title`
- `description`
- `url`
- `author.name`
- `author.github`
- `author.linkedin`

Edit `about.md` and replace `YOUR NAME`.

## 2. Create the GitHub repository

For a personal site, name the repository:

```text
YOUR_GITHUB_USERNAME.github.io
```

For a project site, any repository name works, but update `baseurl` in `_config.yml`.

## 3. Push the site

```bash
git init
git add .
git commit -m "Create Modern Embedded C++ site"
git branch -M main
git remote add origin git@github.com:YOUR_GITHUB_USERNAME/YOUR_GITHUB_USERNAME.github.io.git
git push -u origin main
```

## 4. Enable GitHub Pages

In the repository:

1. Open **Settings → Pages**
2. Under **Build and deployment**, choose **GitHub Actions**
3. Push to `main` and watch the `Deploy Jekyll site to Pages` workflow

## 5. Test locally

Install Ruby and Bundler, then:

```bash
bundle install
bundle exec jekyll serve
```

Open `http://localhost:4000`.

## 6. Add a custom domain

Do this only after the default `github.io` site is working.

1. Buy or use a domain you control.
2. In **Settings → Pages → Custom domain**, enter the domain.
3. Configure DNS with your registrar.
4. Verify the domain in GitHub account settings.
5. Enable **Enforce HTTPS** after DNS validation succeeds.

For a subdomain such as `blog.example.com`, create a DNS `CNAME` pointing to:

```text
YOUR_GITHUB_USERNAME.github.io
```

For an apex domain such as `example.com`, use the current GitHub Pages DNS records
shown in the official GitHub documentation rather than copying old IP addresses from blogs.

## 7. Publish an article

Create:

```text
_posts/YYYY-MM-DD-title.md
```

Start with:

```yaml
---
title: "Article title"
description: "One-sentence result or engineering question."
reading_time: "8 min read"
---
```

A strong article should contain:

- The system context
- The constraint
- Two or three viable options
- Generated code or target measurements
- Trade-offs and failure modes
- Recommendation and reproducibility notes
