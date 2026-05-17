# blog.higcp.com

Source for [blog.higcp.com](https://blog.higcp.com), the personal tech blog of
[Chris Yang](https://github.com/yangwhale).

## Stack

- **Jekyll** (via GitHub Pages built-in)
- **minima** theme + custom GCP Console-style SCSS overrides
  (`_sass/gcp-overrides.scss`)
- **GitHub Pages** for hosting + Let's Encrypt HTTPS
- **Cloud DNS** (chris-pgp-host project, `higcp-com` zone) for the
  `blog.higcp.com` CNAME → `yangwhale.github.io`

## Writing a new post

1. Create `_posts/YYYY-MM-DD-title-with-dashes.md`
2. Add front matter:
   ```yaml
   ---
   layout: post
   title: "Your title"
   date: YYYY-MM-DD HH:MM:SS +0800
   categories: [tpu, vllm]
   ---
   ```
3. Write markdown body
4. `git add . && git commit -m "post: ..." && git push`
5. GitHub Pages rebuilds in ~30s, live at `https://blog.higcp.com/`

## Local preview

```bash
bundle install
bundle exec jekyll serve
# open http://localhost:4000
```

## Design system

GCP Console / cloud.google.com inspired:

- Background: `#FFFFFF` (white) with `#F8F9FA` (light gray) panels
- Text: `#202124` (almost black), `#5F6368` (secondary)
- Accent: `#1A73E8` (Google Blue)
- Borders: `#E8EAED` (light gray, 1px)
- Typography: Google Sans (body) / Roboto Mono (code)
- No gradients, no emoji decoration, no AI-style purple/cyan glow

See `_sass/gcp-overrides.scss` for the full palette.
