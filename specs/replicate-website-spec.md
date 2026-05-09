# Static Website + AI-Driven Blog Publishing System — Replication Spec

> Drop this file into a fresh repo as `DEVELOPMENT_SPEC.md`. Claude Code or Codex CLI will read it as an agent-executable build plan.

---

## 1. Goal

When done, you will have a production-ready company website with the following properties: a code-first static site (HTML + Tailwind CSS + vanilla JS) served from any SCP-capable static host, a GitHub Actions pipeline that automatically builds a full-text search index, deploys the site, and runs a smoke check on every push to `main`, a Claude Code custom skill (`/my-blog`) that transforms a one-line topic into a fully published blog post with hero image in 15–25 minutes with five human-in-the-loop checkpoints, and a brand-guidelines skill that is auto-loaded by all content-creation skills so the model never produces off-brand output. No CMS, no database, no server-side runtime — the repo is the single source of truth.

---

## 2. Prerequisites Checklist

Complete every item before running any phase command.

### Accounts

- [ ] GitHub account with a new empty repository created
- [ ] Static hosting account that accepts SSH/SCP (Hostinger Business, DigitalOcean Droplet, any VPS) — record host, port, username, and remote path
- [ ] Claude Code CLI installed and authenticated (`claude --version` outputs a version string)
- [ ] GitHub CLI authenticated (`gh auth status` shows your username)
- [ ] **(Optional)** Image generation provider — see "Image Provider Setup" below. The blog skill works WITHOUT a generative provider via `--no-image`; AI image generation is a quality-of-life upgrade, not a hard requirement.

### CLI Tools

- [ ] `git` — version 2.x or later
- [ ] `gh` — GitHub CLI, version 2.x or later
- [ ] `ssh` and `scp` — standard OpenSSH client
- [ ] `node` — version 20 or later (`node --version`)
- [ ] `python3` — version 3.10 or later (`python3 --version`)
- [ ] `pip install Pillow` — required for hero image processing
- [ ] `curl` — for smoke checks

### Environment Variables

Set the following as GitHub Actions repository secrets (Settings → Secrets and variables → Actions). All five are read by the deploy workflow:

| Secret | Description |
|--------|-------------|
| `DEPLOY_SSH_HOST` | Static host IP or hostname |
| `DEPLOY_SSH_PORT` | SSH port (commonly 22 for VPS; some shared hosts use a non-standard port — check your host's panel) |
| `DEPLOY_SSH_USER` | SSH username on the host |
| `DEPLOY_PATH` | Remote path to web root, e.g. `public_html` or `www` |
| `DEPLOY_SSH_KEY_B64` | Base64-encoded private SSH key (single-line, no wrapping) |

### SSH Key Setup

Generate a deploy-only SSH key, install the public half on your host, and base64-encode the private half for GitHub.

```bash
# 1. Generate the key (do NOT use a passphrase — CI cannot type one)
ssh-keygen -t ed25519 -C "deploy@<DOMAIN>" -N "" -f ~/.ssh/deploy_key_<COMPANY_NAME>

# 2. Install the public key on the host (paste into the host's authorized_keys panel,
#    or use ssh-copy-id if you already have another auth method working)
cat ~/.ssh/deploy_key_<COMPANY_NAME>.pub
# ssh-copy-id -p <PORT> -i ~/.ssh/deploy_key_<COMPANY_NAME>.pub <USER>@<HOST>

# 3. Base64-encode the PRIVATE key as a single line for GitHub Secrets
#    POSIX-portable form (works on macOS and Linux):
base64 < ~/.ssh/deploy_key_<COMPANY_NAME> | tr -d '\n' > /tmp/key_b64.txt

# 4. Copy /tmp/key_b64.txt contents into GitHub Secrets as DEPLOY_SSH_KEY_B64
#    macOS:  pbcopy < /tmp/key_b64.txt
#    Linux:  xclip -selection clipboard < /tmp/key_b64.txt   (if xclip installed)
#    or just open the file and copy its single-line contents
```

> **NEVER store the unencoded private key inside the repo.** Keep `~/.ssh/deploy_key_<COMPANY_NAME>` outside the working tree. If you need the key locally for manual SCP, reference it with `-i ~/.ssh/deploy_key_<COMPANY_NAME>`, never copy it into `tmp/`.

### Image Provider Setup (Optional)

The blog skill's Stage 4 (hero image generation) supports three execution modes — auto / manual / skip. Configure at most one provider for `auto` mode; otherwise the skill falls back to `manual` mode (you drop a file at `tmp/hero-input.png` when prompted).

Pick ONE of the following provider options, or skip entirely and rely on `--no-image` / `--manual-image` flags every time:

**Option A — Gemini via MCP (recommended if you're already using Claude Code with Gemini MCP)**

1. Install the Gemini MCP server (e.g., `gemini-mcp` package or your preferred wrapper)
2. Register it in your Claude Code MCP config so the `gemini-generate-image` tool is discoverable via ToolSearch
3. The skill auto-detects this tool and uses it

**Option B — OpenAI Images API**

1. Set environment variable `OPENAI_API_KEY=sk-...`
2. The skill calls `POST https://api.openai.com/v1/images/generations` with model `gpt-image-1` (or `dall-e-3` if your account does not have `gpt-image-1` access)

**Option C — No provider, manual upload only**

1. Skip provider configuration
2. Every `/my-blog` invocation enters `manual` mode at Stage 4 — you drop a 16:9 image at `tmp/hero-input.png` (or `.jpg`) when prompted

**Option D — No images at all**

1. Always invoke with `--no-image`. Posts use a site-wide default OG image at `/assets/social-default.png` instead of per-post heroes

---

## 3. Repository Skeleton

The following tree represents the complete target structure. Create every directory and file listed here before starting Phase 2.

```
<repo-root>/
├── .claude/
│   ├── settings.json
│   └── skills/
│       ├── my-blog/
│       │   ├── SKILL.md
│       │   └── templates/
│       │       ├── post-template.html
│       │       └── card-template.html
│       └── my-brand-guidelines/
│           └── SKILL.md
├── .github/
│   └── workflows/
│       └── deploy.yml
├── scripts/
│   └── process_hero_image.py
├── site/
│   ├── index.html
│   ├── .htaccess
│   ├── robots.txt
│   ├── sitemap.xml
│   ├── 404.html
│   ├── assets/
│   │   ├── logo.png
│   │   └── logo-reverse.png
│   ├── contact/
│   │   └── index.html
│   └── blog/
│       ├── index.html
│       └── post-template.html
├── docs/
│   └── plans/
├── tmp/
│   └── .gitkeep
├── CLAUDE.md
└── README.md
```

---

## 4. Phase 1: Repo Bootstrap

### 4.1 Initialize Git Repository

```bash
git init
git remote add origin https://github.com/<YOUR_ORG>/<YOUR_REPO>.git
```

### 4.2 Create Directory Structure

```bash
mkdir -p .claude/skills/my-blog/templates
mkdir -p .claude/skills/my-brand-guidelines
mkdir -p .github/workflows
mkdir -p scripts
mkdir -p site/assets
mkdir -p site/blog
mkdir -p site/contact
mkdir -p docs/plans
mkdir -p tmp
touch tmp/.gitkeep
```

### 4.3 Create .gitignore

Create `.gitignore` with the following content. The secret-related entries are mandatory — do not remove them.

```
# Build output
node_modules/
_pagefind/
site/_pagefind/

# Secrets — NEVER commit these
.env
.env.*
*.pem
*.key
*_deploy_key
*_deploy_key.pub
.mcp.json

# Temp directory — ignore everything except .gitkeep
tmp/*
!tmp/.gitkeep

# Local references / scratch
references/
```

The `tmp/*` + `!tmp/.gitkeep` pattern keeps the directory in source control as an empty placeholder while ignoring everything ever written into it. This protects you if any tool writes a private key, build artifact, or scratch file under `tmp/` and you later run `git add .`.

### 4.4 Create CLAUDE.md

Create `CLAUDE.md` with the following content. Replace every `<PLACEHOLDER>` with your actual values.

```markdown
# CLAUDE.md

## Project Overview

This is the website repository for **<COMPANY_NAME>** — a [brief description].
Website: <DOMAIN>
Hosting: SSH/SCP to <SSH_HOST>:<SSH_PORT>

Static site: HTML + Tailwind CSS + vanilla JS. No frontend build pipeline.
Pagefind index is generated in CI — no need to run locally before deploy.

## Repository Structure

- `site/` — Website source. Deployed to `<DEPLOY_PATH>/`
- `docs/plans/` — Design docs and implementation plans
- `.claude/skills/` — Claude Code custom skills (versioned in git)

## Deployment

**Automated:** Push to `main` with changes in `site/` → GitHub Actions builds
Pagefind index → SCPs to host → live.

**Manual SCP fallback** (uses your local SSH key, NEVER stored in the repo):
```bash
scp -P <SSH_PORT> -i ~/.ssh/deploy_key_<COMPANY_NAME> -r site/. \
    <SSH_USER>@<SSH_HOST>:<DEPLOY_PATH>/
```

The `-i` flag points at the key OUTSIDE the working tree. Do not copy private keys into `tmp/` or any path inside this repo, even if `tmp/` is gitignored.

## Brand System

| Token | Hex | Usage |
|-------|-----|-------|
| Primary | <BRAND_PRIMARY_HEX> | Hero backgrounds, header, footer |
| Secondary | <BRAND_SECONDARY_HEX> | Section backgrounds, cards |
| Accent | <BRAND_ACCENT_HEX> | CTA buttons |
| Highlight | <BRAND_HIGHLIGHT_HEX> | Links, hover states |

Typography: <BRAND_FONT> (Bold for headings, Regular for body at 16px base).

## Blog Architecture

- URL pattern: `/blog/<slug>/`
- Post template: `site/blog/post-template.html` — copy and replace `POST_*` placeholders
- Categories: [List your 3–5 blog categories here]
- Author: [Author name or team name]
- When adding a post: update blog index cards, sitemap.xml, then deploy
- Blog skill: Use `/my-blog` for end-to-end pipeline

## Content Voice

[Describe your company's tone: professional/casual, technical depth, audience.]

## Guardrail Rules (customize for your context)

- Never make claims you cannot substantiate
- Never guarantee specific performance numbers
- Never name competitors by name — use "industry alternatives" generically
```

### 4.5 Create .claude/settings.json

Create `.claude/settings.json` with the following content:

```json
{
  "permissions": {
    "allow": [
      "Bash(git:*)",
      "Bash(gh:*)",
      "Bash(python3:*)",
      "Bash(node:*)",
      "Bash(npx:*)",
      "Bash(scp:*)",
      "Bash(ssh-keyscan:*)",
      "Bash(curl:*)",
      "Bash(mkdir:*)",
      "Bash(cp:*)",
      "Bash(stat:*)",
      "Bash(grep:*)",
      "Bash(find:*)",
      "Bash(open:*)",
      "Bash(xdg-open:*)",
      "Bash(wslview:*)",
      "Bash(kill:*)",
      "Bash(pkill:*)"
    ]
  }
}
```

(`open` is macOS, `xdg-open` is Linux, `wslview` is WSL — keep all three so the same `settings.json` works across environments. `kill` is preferred over `pkill` because it works with PIDs captured at server start.)

### Verification — Phase 1

```bash
ls -la .claude/ .github/ scripts/ site/ docs/ tmp/
git status
```

Expected output: all directories present, `.gitignore` listed as untracked, no errors.

---

## 5. Phase 2: Static Site Scaffolding

### 5.1 Create site/index.html

Create `site/index.html` with the following content. Replace every `<PLACEHOLDER>` with actual values. This is a minimal but fully functional homepage — extend it after the pipeline is working.

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title><COMPANY_NAME> — <TAGLINE></title>
  <meta name="description" content="<META_DESCRIPTION>">
  <meta property="og:title" content="<COMPANY_NAME>">
  <meta property="og:description" content="<META_DESCRIPTION>">
  <meta property="og:url" content="https://<DOMAIN>/">
  <link rel="canonical" href="https://<DOMAIN>/">
  <script src="https://cdn.tailwindcss.com"></script>
  <script>
    tailwind.config = {
      theme: {
        extend: {
          colors: {
            primary: '<BRAND_PRIMARY_HEX>',
            secondary: '<BRAND_SECONDARY_HEX>',
            accent: '<BRAND_ACCENT_HEX>',
            highlight: '<BRAND_HIGHLIGHT_HEX>',
          }
        }
      }
    }
  </script>
</head>
<body class="bg-gray-50 text-gray-900 font-sans">

  <!-- Header -->
  <header class="bg-primary text-white py-4 px-6 flex items-center justify-between">
    <a href="/" class="text-xl font-bold"><COMPANY_NAME></a>
    <nav class="flex gap-6 text-sm">
      <a href="/blog/" class="hover:text-accent transition-colors">Blog</a>
      <a href="/contact/" class="hover:text-accent transition-colors">Contact</a>
    </nav>
  </header>

  <!-- Hero -->
  <section class="bg-primary text-white py-24 px-6 text-center">
    <h1 class="text-4xl font-bold mb-4"><HEADLINE></h1>
    <p class="text-xl text-gray-300 mb-8 max-w-2xl mx-auto"><SUBHEADLINE></p>
    <a href="/contact/"
       class="inline-block bg-accent text-white font-semibold px-8 py-3 rounded-lg hover:opacity-90 transition-opacity">
      <CTA_TEXT>
    </a>
  </section>

  <!-- Footer -->
  <footer class="bg-primary text-gray-400 py-8 px-6 text-center text-sm">
    <p>&copy; <span id="year"></span> <COMPANY_NAME>. All rights reserved.</p>
    <script>document.getElementById('year').textContent = new Date().getFullYear();</script>
  </footer>

</body>
</html>
```

### 5.2 Create site/.htaccess

Create `site/.htaccess` with the following content:

```apache
Options -Indexes

# Clean URLs — serve index.html for directory requests
DirectoryIndex index.html

# Security headers — guarded by mod_headers presence so the file does not
# 500 the entire site on hosts where mod_headers is disabled
<IfModule mod_headers.c>
  Header always set X-Content-Type-Options "nosniff"
  Header always set X-Frame-Options "SAMEORIGIN"
  Header always set Referrer-Policy "strict-origin-when-cross-origin"

  # Cache static assets aggressively
  <FilesMatch "\.(jpg|jpeg|png|webp|gif|ico|svg|woff2|css|js)$">
    Header set Cache-Control "public, max-age=31536000, immutable"
  </FilesMatch>
</IfModule>

# Custom 404
ErrorDocument 404 /404.html
```

### 5.3 Create site/robots.txt

Create `site/robots.txt` with the following content:

```
User-agent: *
Allow: /

Sitemap: https://<DOMAIN>/sitemap.xml
```

### 5.4 Create site/sitemap.xml

Create `site/sitemap.xml` with the following content:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <url>
    <loc>https://<DOMAIN>/</loc>
    <lastmod>YYYY-MM-DD</lastmod>
    <changefreq>weekly</changefreq>
    <priority>1.0</priority>
  </url>
  <url>
    <loc>https://<DOMAIN>/blog/</loc>
    <lastmod>YYYY-MM-DD</lastmod>
    <changefreq>weekly</changefreq>
    <priority>0.8</priority>
  </url>
</urlset>
```

Replace `YYYY-MM-DD` with today's date.

### 5.5 Create site/404.html

Create `site/404.html` with the following content:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Page Not Found — <COMPANY_NAME></title>
  <script src="https://cdn.tailwindcss.com"></script>
</head>
<body class="bg-gray-50 flex items-center justify-center min-h-screen">
  <div class="text-center">
    <h1 class="text-6xl font-bold text-gray-300 mb-4">404</h1>
    <p class="text-xl text-gray-600 mb-8">Page not found.</p>
    <a href="/" class="text-blue-600 underline">Return home</a>
  </div>
</body>
</html>
```

### 5.6 Create site/contact/index.html

The blog skill, brand guidelines, and homepage CTAs all link to `/contact/`. Create a real page at that path so the links never 404.

```bash
mkdir -p site/contact
```

Create `site/contact/index.html` with the following content:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Contact — <COMPANY_NAME></title>
  <meta name="description" content="Get in touch with <COMPANY_NAME>.">
  <link rel="canonical" href="https://<DOMAIN>/contact/">
  <script src="https://cdn.tailwindcss.com"></script>
</head>
<body class="bg-gray-50 text-gray-900 font-sans">

  <header class="py-4 px-6 flex items-center justify-between"
          style="background-color: <BRAND_PRIMARY_HEX>;">
    <a href="/" class="text-xl font-bold text-white"><COMPANY_NAME></a>
    <nav class="flex gap-6 text-sm text-white">
      <a href="/blog/" class="hover:underline">Blog</a>
      <a href="/contact/" class="hover:underline">Contact</a>
    </nav>
  </header>

  <main class="max-w-2xl mx-auto px-6 py-20">
    <h1 class="text-4xl font-bold mb-4">Get in Touch</h1>
    <p class="text-lg text-gray-600 mb-10">
      We typically respond within 1 business day.
    </p>

    <div class="bg-white border border-gray-200 rounded-2xl p-8 space-y-6">
      <div>
        <h2 class="text-sm font-semibold text-gray-500 uppercase tracking-wide mb-2">Email</h2>
        <a href="mailto:<CONTACT_EMAIL>"
           class="text-lg font-medium underline"
           style="color: <BRAND_HIGHLIGHT_HEX>;"><CONTACT_EMAIL></a>
      </div>
      <!-- [CUSTOMIZE: add a contact form, calendar link, or social handles here] -->
    </div>
  </main>

  <footer class="text-center text-sm py-8 mt-16"
          style="background-color: <BRAND_PRIMARY_HEX>; color: #9ca3af;">
    <a href="/blog/" class="underline text-gray-300">Blog</a>
    &nbsp;·&nbsp; <a href="/" class="underline text-gray-300">Home</a>
    &nbsp;·&nbsp; &copy; <COMPANY_NAME>
  </footer>

</body>
</html>
```

### 5.7 Create site/blog/index.html

Create `site/blog/index.html` as the blog listing page. This is the minimal structure the `/my-blog` skill will insert cards into.

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Blog — <COMPANY_NAME></title>
  <meta name="description" content="Insights and articles from <COMPANY_NAME>.">
  <script src="https://cdn.tailwindcss.com"></script>
  <!-- Pagefind full-text search UI styles (CI generates the index at /_pagefind/) -->
  <link rel="stylesheet" href="/_pagefind/pagefind-ui.css">
</head>
<body class="bg-gray-50 text-gray-900 font-sans">

  <header class="bg-primary text-white py-4 px-6 flex items-center justify-between"
          style="background-color: <BRAND_PRIMARY_HEX>;">
    <a href="/" class="text-xl font-bold"><COMPANY_NAME></a>
    <nav class="flex gap-6 text-sm">
      <a href="/blog/" class="hover:underline">Blog</a>
      <a href="/contact/" class="hover:underline">Contact</a>
    </nav>
  </header>

  <main class="max-w-6xl mx-auto px-6 py-16">
    <h1 class="text-4xl font-bold mb-8">Blog</h1>

    <!-- Pagefind search box -->
    <div id="search" class="mb-12"></div>

    <!-- Posts grid — /my-blog skill inserts cards immediately after the no-results line -->
    <div id="posts-grid" class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-8">
      <p id="no-results" class="col-span-3 text-gray-400 hidden">No posts found.</p>
      <!-- BLOG CARDS INSERTED HERE BY /my-blog SKILL -->
    </div>
  </main>

  <!-- Pagefind UI bootstrap. The index is built by GitHub Actions at deploy time;
       it is missing in local dev unless you run: npx pagefind --site site -->
  <script src="/_pagefind/pagefind-ui.js"></script>
  <script>
    window.addEventListener('DOMContentLoaded', () => {
      if (typeof PagefindUI !== 'undefined') {
        new PagefindUI({
          element: '#search',
          showSubResults: true,
          resetStyles: false,
          translations: { placeholder: 'Search posts…' }
        });
      }
    });
  </script>

  <footer class="text-center text-sm text-gray-400 py-8"
          style="background-color: <BRAND_PRIMARY_HEX>; color: #9ca3af;">
    &copy; <COMPANY_NAME>
  </footer>

</body>
</html>
```

### 5.8 Create site/blog/post-template.html

Create `site/blog/post-template.html`. This is the master template the skill copies and fills in for every new post. The `POST_*` tokens are all replaced automatically.

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>POST_TITLE — <COMPANY_NAME></title>
  <meta name="description" content="POST_EXCERPT">
  <meta name="robots" content="noindex"><!-- removed by skill after writing -->
  <meta property="og:title" content="POST_TITLE">
  <meta property="og:description" content="POST_EXCERPT">
  <meta property="og:image" content="https://<DOMAIN>/blog/POST_SLUG/hero.jpg">
  <meta property="og:url" content="https://<DOMAIN>/blog/POST_SLUG/">
  <meta property="og:type" content="article">
  <meta name="twitter:card" content="summary_large_image">
  <meta name="twitter:image" content="https://<DOMAIN>/blog/POST_SLUG/hero.jpg">
  <link rel="canonical" href="https://<DOMAIN>/blog/POST_SLUG/">
  <script src="https://cdn.tailwindcss.com"></script>
  <script type="application/ld+json">
  {
    "@context": "https://schema.org",
    "@type": "BlogPosting",
    "headline": "POST_TITLE",
    "description": "POST_EXCERPT",
    "image": "https://<DOMAIN>/blog/POST_SLUG/hero.jpg",
    "datePublished": "POST_DATE",
    "author": {"@type": "Organization", "name": "<COMPANY_NAME>"},
    "publisher": {"@type": "Organization", "name": "<COMPANY_NAME>"}
  }
  </script>
</head>
<body class="bg-gray-50 text-gray-900 font-sans">

  <header class="py-4 px-6 flex items-center justify-between"
          style="background-color: <BRAND_PRIMARY_HEX>;">
    <a href="/" class="text-xl font-bold text-white"><COMPANY_NAME></a>
    <nav class="flex gap-6 text-sm text-white">
      <a href="/blog/" class="hover:underline">Blog</a>
      <a href="/contact/" class="hover:underline">Contact</a>
    </nav>
  </header>

  <!-- Hero Image -->
  <div class="w-full h-64 md:h-96 overflow-hidden bg-gray-200">
    <picture>
      <source srcset="/blog/POST_SLUG/hero.webp" type="image/webp">
      <img src="/blog/POST_SLUG/hero.jpg" alt="POST_TITLE"
           class="w-full h-full object-cover" loading="eager">
    </picture>
  </div>

  <main class="max-w-4xl mx-auto px-6 py-12">
    <div class="mb-8">
      <span class="text-sm text-gray-500">POST_CATEGORY</span>
      <time datetime="POST_DATE" class="text-sm text-gray-400 ml-4">POST_DATE_DISPLAY</time>
    </div>

    <h1 class="text-4xl font-bold mb-6 leading-tight">POST_TITLE</h1>
    <p class="text-xl text-gray-600 mb-10">POST_EXCERPT</p>

    <article data-pagefind-body class="max-w-3xl prose prose-lg prose-headings:font-bold
             prose-a:text-blue-600 prose-blockquote:border-l-4 prose-blockquote:pl-4">
      <div class="prose max-w-none">
        <!-- REPLACE THIS SECTION WITH POST CONTENT -->
        <p>Post content goes here.</p>
        <!-- END POST CONTENT -->
      </div>
    </article>
  </main>

  <footer class="text-center text-sm text-gray-400 py-8 mt-16"
          style="background-color: <BRAND_PRIMARY_HEX>; color: #9ca3af;">
    <a href="/blog/" class="underline text-gray-300">Back to Blog</a>
    &nbsp;·&nbsp; &copy; <COMPANY_NAME>
  </footer>

</body>
</html>
```

### Verification — Phase 2

```bash
ls site/
ls site/blog/
python3 -c "
import re, pathlib
t = pathlib.Path('site/blog/post-template.html').read_text()
tokens = re.findall(r'POST_[A-Z_]+', t)
print('Tokens found:', set(tokens))
"
```

Expected: `site/index.html`, `site/blog/index.html`, `site/blog/post-template.html` all present. Tokens set should include `POST_TITLE`, `POST_SLUG`, `POST_EXCERPT`, `POST_DATE`, `POST_DATE_DISPLAY`, `POST_CATEGORY`.

---

## 6. Phase 3: GitHub Actions deploy.yml

Create `.github/workflows/deploy.yml` with the following content. Replace every `<PLACEHOLDER>`.

```yaml
name: Deploy to Static Host

on:
  push:
    branches: [main]
    paths: ['site/**']   # only trigger on site/ changes — docs/plans edits do not deploy
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: production-deploy
  cancel-in-progress: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683  # v4.2.2

      - uses: actions/setup-node@49933ea5288caeca8642d1e84afbd3f7d6820020  # v4.4.0
        with:
          node-version: '20'

      - name: Build Pagefind index
        run: npx pagefind@1.3 --site site --output-path site/_pagefind

      - name: Deploy via SCP
        env:
          DEPLOY_SSH_KEY_B64: ${{ secrets.DEPLOY_SSH_KEY_B64 }}
          DEPLOY_SSH_HOST: ${{ secrets.DEPLOY_SSH_HOST }}
          DEPLOY_SSH_PORT: ${{ secrets.DEPLOY_SSH_PORT }}
          DEPLOY_SSH_USER: ${{ secrets.DEPLOY_SSH_USER }}
          DEPLOY_PATH: ${{ secrets.DEPLOY_PATH }}
        run: |
          mkdir -p ~/.ssh
          ssh-keyscan -p "$DEPLOY_SSH_PORT" "$DEPLOY_SSH_HOST" >> ~/.ssh/known_hosts 2>/dev/null
          echo "$DEPLOY_SSH_KEY_B64" | base64 -d > ~/.ssh/deploy_key
          chmod 600 ~/.ssh/deploy_key
          scp -P "$DEPLOY_SSH_PORT" -i ~/.ssh/deploy_key -r site/. \
              "$DEPLOY_SSH_USER@$DEPLOY_SSH_HOST:$DEPLOY_PATH/"

      - name: Smoke check
        env:
          PUBLIC_URL: ${{ vars.PUBLIC_URL }}   # set as a repo Variable, e.g. https://example.com
        run: |
          PAGES="/ /blog/ /contact/"
          for i in 1 2 3; do
            sleep 5
            ALL_OK=true
            for page in $PAGES; do
              if ! curl -sf "${PUBLIC_URL}${page}" > /dev/null; then
                echo "  FAIL: ${page}"
                ALL_OK=false
              fi
            done
            if $ALL_OK; then
              echo "All pages responding"
              exit 0
            fi
            echo "Attempt $i failed, retrying..."
          done
          echo "Smoke check failed after 3 attempts"
          exit 1
```

> **Note on `PUBLIC_URL`**: define this as a GitHub Actions **Variable** (Settings → Secrets and variables → Actions → Variables tab), value `https://<DOMAIN>`. Variables are non-secret strings appropriate for URLs.

> **Note on action SHA pinning**: If your GitHub org enforces "Require actions to be pinned to a full-length commit SHA", the SHA values above are mandatory. Tag-based refs (`@v4`) will fail CI startup. Update the SHAs if newer versions are required by checking each action's releases page.

> **Note on SCP path for addon domains**: If `<DOMAIN>` is an addon domain under a parent hosting account (common on shared hosts), the `<DEPLOY_PATH>` typically follows the pattern `domains/<DOMAIN>/public_html` — not `public_html/`. Verify with your host's file manager before first deploy.

### Verification — Phase 3

```bash
cat .github/workflows/deploy.yml | grep -E "<[A-Z_]+>" | head -20
```

Expected: zero lines output (all placeholders replaced). Then commit and push:

```bash
git add .github/workflows/deploy.yml
git commit -m "ci: add deploy workflow"
git push -u origin main
gh run list --limit 3
```

The workflow will not trigger (no `site/**` change yet). Run `gh workflow run deploy.yml` to trigger a manual deploy and confirm the site reaches your domain.

---

## 7. Phase 4: Bootstrap Your /my-blog Skill

### 7.1 Create the Blog Skill Frontmatter and Pipeline

Create `.claude/skills/my-blog/SKILL.md` with the following content. This is the full skill definition. Customize the sections marked `[CUSTOMIZE]`.

````markdown
---
name: my-blog
description: End-to-end blog post creation — research, write, generate hero image, assemble HTML, update index, and open PR. Triggers on "/my-blog".
user_invocable: true
---

# /my-blog — Blog Post Pipeline

End-to-end blog post creation for <COMPANY_NAME>. Orchestrates research, writing,
hero image generation, HTML assembly, blog index card, sitemap update, and PR creation.

## Usage

```
/my-blog "Topic or title idea"
/my-blog "Topic" --category "Category Name"
/my-blog "Topic" --no-image --draft
/my-blog "Topic" --skip-approval
/my-blog "Topic" --format how-to
```

## Arguments

| Flag | Description | Default |
|------|-------------|---------|
| (positional) | Blog topic or title idea | Required |
| `--url` | Reference URL to fetch and cite | None |
| `--file` | Local reference file path | None |
| `--category` | One of your defined categories | Auto-detect |
| `--slug` | Override auto-generated slug | Derived from title |
| `--no-image` | Skip hero image generation | false |
| `--format` | `prose`, `how-to`, `comparison`, `listicle` | `prose` |
| `--draft` | Keep noindex, skip sitemap/card | false |
| `--skip-approval` / `-s` | Skip all user checkpoints | false |

---

## Stage 0: Preflight Validation

Run automatically, no user checkpoint.

1. Parse all arguments
2. Validate `--category` against allowed list: [CUSTOMIZE: list your categories]
3. Validate slug uniqueness: `site/blog/{slug}/` must NOT already exist
4. Validate `--format` is one of: prose, how-to, comparison, listicle
5. Check required tools:
   ```bash
   python3 -c "from PIL import Image; print('Pillow OK')"
   gh auth status
   ```
6. Check Gemini MCP available (unless `--no-image`): verify `gemini-generate-image`
   tool exists via ToolSearch
7. Check git state: if working tree has uncommitted changes, warn and ask to continue
   or abort — do NOT auto-stash

---

## Stage 1: Research and Planning

1. If `--url` provided: fetch with WebFetch and summarize
2. If `--file` provided: read and summarize
3. Propose:
   - **Title** (under 70 characters)
   - **Slug** (kebab-case, 3–5 words)
   - **Category** (one of the allowed list)
   - **Excerpt** (140–160 characters)
   - **Format** (prose / how-to / comparison / listicle)
   - **Outline** structured per format:
     - `prose`: 3–5 free-form section headings
     - `how-to`: What / Why / Prerequisites / Steps (3–7) / Pitfalls / Summary
     - `comparison`: Overview / Criteria / Option A / Option B / Comparison Table / Verdict
     - `listicle`: Introduction / Items (5–10 numbered) / Key Takeaway
4. **USER CHECKPOINT** (skip if `--skip-approval`): present proposal and wait for approval

---

## Stage 2: Content Writing

1. Write full post content as HTML inside `<div class="prose max-w-none">`
2. Follow company voice from CLAUDE.md
3. Apply answer-first structure: lead each `<h2>` with a direct 1–2 sentence answer,
   then elaborate — this improves LLM citation readiness (ChatGPT, Perplexity)
4. Target 1,200–2,000 words (8–12 minute read)
5. Apply format-specific structure (per Stage 1 outline)
6. Include:
   - `<h2>` section headings
   - At least one internal link to the site
   - At least one external `<a href="https://...">` citation
   - Closing CTA blockquote: [CUSTOMIZE: write your company's CTA blockquote here]
     ```html
     <blockquote>
       <p><strong><COMPANY_NAME></strong> is [what you do]. <a href="/contact/">Talk to us</a> to learn more.</p>
     </blockquote>
     ```
7. Apply content guardrails from CLAUDE.md
8. Run Stage 2.5 Quality Gate before presenting to user
9. **USER CHECKPOINT** (skip if `--skip-approval`): present draft and quality scorecard

---

## Stage 2.5: Content Quality Gate

Automated checks. Run before the Stage 2 checkpoint. Report as scorecard. Hard fails
stop the pipeline; soft warnings proceed.

| # | Check | Type | Rule |
|---|-------|------|------|
| 1 | Word count | Tiered | Hard fail < 1,000 or > 2,500 words. Soft warn outside 1,200–2,000 |
| 2 | Heading count | Hard | At least 3 `<h2>` headings |
| 3 | Answer-first | Soft | Each `<h2>` section's first `<p>` should be ≤ 50 words |
| 4 | Internal links | Soft | At least 1 link to `<DOMAIN>` or relative path |
| 5 | External refs | Soft | At least 1 `https://` external link |
| 6 | CTA present | Hard | Closing `<blockquote>` with company name and contact link |
| 7 | Format compliance | Hard | Per format heuristics below |
| 8 | Guardrail terms | Hard | [CUSTOMIZE: add grep patterns for banned phrases] |

**Format heuristics (Check #7):**

| Format | Hard requirement |
|--------|-----------------|
| `prose` | Pass automatically |
| `how-to` | ≥ 3 step headings matching `Step \d` in `<h3>` or ≥ 3 `<ol><li>` items |
| `comparison` | ≥ 2 `<h2>/<h3>` headings for distinct options (exclude intro/conclusion/verdict) |
| `listicle` | 5–10 numbered `<h2>` headings matching `^\d+\.` |

On hard failure: auto-revise content with specific fix instructions, re-run gate.
Maximum 2 revision cycles, then escalate to user.

---

## Stage 3: HTML Assembly

1. Read `site/blog/post-template.html`
2. Create directory: `mkdir -p site/blog/{slug}`
3. Copy template to `site/blog/{slug}/index.html`
4. Replace all `POST_*` placeholders using Edit tool with `replace_all: true`:
   - `POST_TITLE` → title text
   - `POST_SLUG` → kebab slug
   - `POST_EXCERPT` → 140–160 char excerpt
   - `POST_DATE` → `YYYY-MM-DD`
   - `POST_DATE_DISPLAY` → `Mon DD, YYYY`
   - `POST_CATEGORY` → exact category display name
5. Verify no remaining tokens:
   ```bash
   grep -c "POST_" site/blog/{slug}/index.html
   ```
   Must return 0.
6. Unless `--draft`: delete the `<meta name="robots" content="noindex">` line
7. Ensure `<article>` tag has `data-pagefind-body` attribute
8. Replace content between `<!-- REPLACE THIS SECTION WITH POST CONTENT -->` and
   `<!-- END POST CONTENT -->` with Stage 2 content

---

## Stage 4: Hero Image Generation

Three execution modes. Pick one based on the user's setup (auto-detect):

| Mode | Trigger | Behavior |
|------|---------|----------|
| `auto` | Image provider configured AND `--no-image` not set | Generate via provider |
| `manual` | `--manual-image` flag OR no provider configured | Pause and prompt user to drop a file at `tmp/hero-input.<ext>` |
| `skip` | `--no-image` flag | Bypass Stage 4 entirely; post uses no hero image |

### 4a. Auto mode (provider configured)

1. Detect which image provider is configured (see "Image Provider Setup" in the project README — supports `gemini` MCP tool, `openai` Images API, or any custom provider that the user wired up)
2. Construct image generation prompt:
   ```
   Generate a 16:9 editorial illustration for a blog post titled "{title}".
   Style: Abstract, conceptual, modern digital art.
   Color palette: <BRAND_PRIMARY_HEX>, <BRAND_SECONDARY_HEX>, <BRAND_ACCENT_HEX>.
   Rules: No text in image. No human faces. No stock photo style.
   Theme: {category-specific theme from mapping below}
   ```
   [CUSTOMIZE: add your category → theme mapping here]
3. Call the configured provider; save the raw output to `tmp/image-output/{slug}.<ext>`
4. Process the raw image (passes brand color so the script does not hardcode any color):
   ```bash
   python3 scripts/process_hero_image.py \
       tmp/image-output/{slug}.png \
       site/blog/{slug}/ \
       --brand-color <BRAND_PRIMARY_HEX>
   ```

### 4b. Manual mode (no provider, or `--manual-image`)

1. Prompt the user: "Drop a 16:9 image (>=1200x630) at `tmp/hero-input.png` (or .jpg) and press Enter"
2. Wait for the file to exist
3. Run the same processing script:
   ```bash
   python3 scripts/process_hero_image.py \
       tmp/hero-input.png \
       site/blog/{slug}/ \
       --brand-color <BRAND_PRIMARY_HEX>
   ```

### 4c. Skip mode (`--no-image`)

Skip Stage 4 entirely AND apply these changes to the post HTML in Stage 5:
- Remove the entire hero `<picture>` block from the post template
- Set `og:image` and `twitter:image` to a default site-wide fallback (e.g., `/assets/social-default.png`)
- Remove the `"image": ...` line from the JSON-LD block
- In Stage 5 blog card, omit the `<img>` tag (gradient-only card)

### Common: verify and review

5. Verify file sizes (cross-platform — uses Python instead of `stat -f%z` which is macOS-only):
   ```bash
   python3 -c "import os, sys; \
     [print(f'{f}: {os.path.getsize(f)/1024:.1f} KB') \
      for f in ['site/blog/{slug}/hero.jpg', 'site/blog/{slug}/hero-thumb.jpg']]"
   ```
   Hard limits: `hero.jpg` < 150 KB, `hero-thumb.jpg` < 50 KB.
6. If sizes exceed budget in auto mode: regenerate with simpler prompt (fewer elements, more whitespace) then re-process
7. **USER CHECKPOINT** (skip if `--skip-approval`): open image for review:
   ```bash
   # macOS: open site/blog/{slug}/hero.jpg
   # Linux: xdg-open site/blog/{slug}/hero.jpg
   # WSL:   wslview site/blog/{slug}/hero.jpg
   ```
   Allow up to 3 regeneration attempts in auto mode if rejected.

---

## Stage 5: Blog Index Card

Skip entirely if `--draft`.

1. Read `site/blog/index.html`
2. Read card template from `.claude/skills/my-blog/templates/card-template.html`
3. Replace card placeholders with actual values
4. Insert rendered card immediately after `<p id="no-results" ...>` line

---

## Stage 6: Sitemap Update

Skip entirely if `--draft`.

1. Add new `<url>` entry to `site/sitemap.xml` before closing `</urlset>`:
   ```xml
   <url>
     <loc>https://<DOMAIN>/blog/{slug}/</loc>
     <lastmod>{YYYY-MM-DD}</lastmod>
     <changefreq>monthly</changefreq>
     <priority>0.7</priority>
   </url>
   ```
2. Update the blog index entry's `<lastmod>` to today's date

---

## Stage 7: Quality Checklist and Commit

**Automated verification before commit (hard fails stop pipeline):**

- `grep -c "POST_" site/blog/{slug}/index.html` returns 0
- `grep -c "data-pagefind-body" site/blog/{slug}/index.html` returns ≥ 1
- `grep -c "noindex" site/blog/{slug}/index.html` returns 0 (unless `--draft`)
- All 4 hero image files exist (unless `--no-image`)
- OG meta `og:image` points to `https://<DOMAIN>/blog/{slug}/hero.jpg`
- Blog card inserted in `site/blog/index.html`
- Sitemap entry exists in `site/sitemap.xml`
- `git diff --cached` contains no secrets or `.env` content

**USER CHECKPOINT** (skip if `--skip-approval`): start local preview server:

```bash
cd site && python3 -m http.server 8080 &
SERVER_PID=$!

# Open in default browser (use the line for your OS):
# macOS:   open "http://localhost:8080/blog/{slug}/"
# Linux:   xdg-open "http://localhost:8080/blog/{slug}/"
# WSL:     wslview "http://localhost:8080/blog/{slug}/"
```

After approval, stop server using the PID captured above:

```bash
kill "$SERVER_PID"   # works on macOS and Linux
```

**Commit and PR:**

```bash
git checkout -b blog/{slug}
git add site/blog/{slug}/ site/blog/index.html site/sitemap.xml
git commit -m "feat(blog): add post — {Title}"
git push -u origin blog/{slug}
gh pr create --title "feat(blog): add post — {Title}" --body "..."
```

**USER CHECKPOINT** (skip if `--skip-approval`): present PR URL. User merges to trigger deploy.

---

## Stage 8: Post-Publish Verification

After PR is merged:

1. Switch to main and pull:
   ```bash
   git checkout main && git pull
   ```
2. Watch the deploy run:
   ```bash
   MERGE_SHA=$(git rev-parse main)
   gh run list --workflow=deploy.yml --branch=main --limit 5 \
       --json databaseId,headSha,status,conclusion
   gh run watch <RUN_ID>
   ```
3. Verify live URL:
   ```bash
   curl -sI https://<DOMAIN>/blog/{slug}/ | head -1
   ```
4. Report success with links to live post and blog index

---

## Error Handling

| Stage | Failure | Recovery |
|-------|---------|----------|
| Stage 0 | Slug already exists | Error and stop — user must choose a different slug |
| Stage 0 | Invalid category | Error with list of valid categories |
| Stage 0 | Dirty git state | Warn user, ask to continue or abort |
| Stage 2.5 | Hard check fails | Auto-revise (max 2 cycles), then escalate |
| Stage 3 | Template missing | Error and stop |
| Stage 4 | Gemini API error | Retry once; offer `--no-image` fallback |
| Stage 4 | Image over budget | Regenerate with simpler prompt |
| Stage 7 | Branch already exists | Append timestamp: `blog/{slug}-{YYYYMMDD}` |
````

### 7.2 Create Card Template

Create `.claude/skills/my-blog/templates/card-template.html` with the following content:

```html
<!-- Post Card: {TITLE} -->
<a href="/blog/{SLUG}/" class="blog-card bg-white rounded-2xl border border-gray-100
   overflow-hidden shadow-sm hover:shadow-md transition-shadow" data-category="{CATEGORY_SLUG}">
  <div class="h-48 overflow-hidden rounded-t-2xl relative bg-gradient-to-br {GRADIENT}">
    <div class="absolute inset-0 flex items-center justify-center">
      <div class="text-center px-6">
        <svg class="w-12 h-12 text-white/40 mx-auto mb-3" fill="none"
             stroke="currentColor" viewBox="0 0 24 24">
          <path stroke-linecap="round" stroke-linejoin="round"
                stroke-width="1.5" d="{ICON_PATH}"/>
        </svg>
        <span class="text-white/50 text-sm font-medium">{CATEGORY_DISPLAY}</span>
      </div>
    </div>
    <picture>
      <source srcset="/blog/{SLUG}/hero-thumb.webp" type="image/webp">
      <img src="/blog/{SLUG}/hero-thumb.jpg" alt=""
           class="relative w-full h-full object-cover z-10"
           loading="lazy" onerror="this.style.display='none'">
    </picture>
  </div>
  <div class="p-6">
    <div class="flex items-center gap-3 mb-3">
      <span class="text-xs font-semibold px-2.5 py-0.5 rounded-full bg-blue-100 text-blue-700">
        {CATEGORY_DISPLAY}
      </span>
      <time datetime="{ISO_DATE}" class="text-xs text-gray-400">{DATE_DISPLAY}</time>
    </div>
    <h2 class="text-lg font-bold mb-2 leading-snug">{TITLE}</h2>
    <p class="text-sm text-gray-500 leading-relaxed">{EXCERPT}</p>
  </div>
</a>
```

**Customize the category mapping** inside `SKILL.md` Stage 5. For each category, define:
- `{CATEGORY_SLUG}`: kebab-case identifier (for JS filtering)
- `{GRADIENT}`: Tailwind gradient classes e.g. `from-blue-900 to-blue-600`
- `{ICON_PATH}`: SVG path `d` attribute

### Verification — Phase 4

```bash
ls .claude/skills/my-blog/
ls .claude/skills/my-blog/templates/
head -5 .claude/skills/my-blog/SKILL.md
```

Expected: `SKILL.md` present, `templates/` directory with `post-template.html` and `card-template.html`.

---

## 8. Phase 5: Brand-Guidelines Skill

### 8.1 Create the Skill

Create `.claude/skills/my-brand-guidelines/SKILL.md` with the following content. Replace every `[CUSTOMIZE]` section with your actual brand values.

````markdown
---
name: my-brand-guidelines
description: Brand guidelines for <COMPANY_NAME> — colors, typography, voice, and content guardrails. Auto-loaded by content creation skills.
user_invocable: false
---

# <COMPANY_NAME> Brand Guidelines

This skill is auto-loaded by `/my-blog` and any other content creation skills.
It defines the non-negotiable constraints that govern all generated content.

## Color Tokens

| Token | Hex | Usage |
|-------|-----|-------|
| Primary | <BRAND_PRIMARY_HEX> | Hero backgrounds, header, footer |
| Secondary | <BRAND_SECONDARY_HEX> | Section backgrounds, cards |
| Accent | <BRAND_ACCENT_HEX> | CTA buttons, interactive elements |
| Highlight | <BRAND_HIGHLIGHT_HEX> | Links, hover states, highlights |

[CUSTOMIZE: add more color tokens as needed]

## Typography

- **Heading font**: [CUSTOMIZE: e.g., Inter, Poppins, system-ui]
- **Body font**: [CUSTOMIZE] at 16px base, 1.6 line height
- **Heading weight**: Bold (700)
- **Body weight**: Regular (400)
- **Monospace**: system monospace for code blocks

## Voice and Tone

[CUSTOMIZE: write 4–6 principles that describe how your company communicates]

Example principles:
- Lead with customer problems, not product features
- Use plain language — no jargon that insiders understand but prospects do not
- Confident but honest — distinguish what is available now from what is on the roadmap
- Avoid superlatives ("best", "most powerful") unless you can cite evidence

## Content Guardrails (Hard Rules)

The following patterns are PROHIBITED in all public content. The `/my-blog` quality
gate will reject any content that matches these patterns.

[CUSTOMIZE: replace with your actual banned patterns]

```
BANNED_PATTERNS = [
  "guaranteed",          # never guarantee performance or outcomes
  "industry-leading",    # unverifiable superlative
  "best-in-class",       # unverifiable superlative
]
```

Add patterns to the Stage 2.5 quality gate grep check in `/my-blog` SKILL.md.

## Internal Linking Strategy

Always include at least one link back to a key conversion page (contact, pricing,
or product page) within blog post content. Suggested anchor text per destination:

[CUSTOMIZE: list your key pages and preferred anchor text]

## Image Style Guidelines

For hero image generation, use the following style direction:

- Style: Abstract, conceptual, modern digital art — no stock photo aesthetic
- Prohibit: text in images, human faces, logos of other companies
- Color palette: use the brand color tokens above
- Aspect ratio: 16:9 (1200×630px for hero, 600×315px for thumbnail)

[CUSTOMIZE: add category-specific image themes if you have blog categories]

## Closing CTA Standard

Every blog post must end with a blockquote CTA:

```html
<blockquote>
  <p><strong><COMPANY_NAME></strong> is [brief company description]. 
  <a href="/contact/">Talk to us</a> to learn more.</p>
</blockquote>
```

[CUSTOMIZE: adjust the CTA text to match your company's voice]
````

### 8.2 Wire Brand Guidelines Into the Blog Skill

Add the following line to the very top of the `## Stage 2: Content Writing` section in `.claude/skills/my-blog/SKILL.md`:

```
Load and apply brand guidelines from `.claude/skills/my-brand-guidelines/SKILL.md`
before writing any content. The brand guidelines define colors, voice, tone,
and prohibited patterns that govern all output.
```

### Verification — Phase 5

```bash
ls .claude/skills/my-brand-guidelines/
grep -c "BRAND_PRIMARY_HEX" .claude/skills/my-brand-guidelines/SKILL.md
```

Expected: `SKILL.md` present, grep returns 0 (all placeholders replaced).

---

## 9. Phase 6: First Blog Post End-to-End

### 9.1 Create the Hero Image Processing Script

Create `scripts/process_hero_image.py` with the following content. This script is called by the `/my-blog` skill at Stage 4.

```python
#!/usr/bin/env python3
"""Process a raw image into blog hero and thumbnail outputs.

Usage:
    python3 scripts/process_hero_image.py <input_image> <output_dir> [options]

Options:
    --brand-color HEX    Bottom gradient color, e.g. #0B1D3A. Default: no gradient.
    --no-gradient        Explicit opt-out of gradient overlay
    -h, --help           Show this help and exit cleanly

Outputs:
    <output_dir>/hero.jpg        1200x630, JPEG 85%
    <output_dir>/hero-thumb.jpg  600x315,  JPEG 80%
    <output_dir>/hero.webp       1200x630, WebP 80%
    <output_dir>/hero-thumb.webp 600x315,  WebP 75%
"""

import argparse
import sys
from pathlib import Path

try:
    from PIL import Image, ImageOps, ImageDraw
except ImportError:
    print("Error: Pillow required. Run: pip install Pillow", file=sys.stderr)
    sys.exit(1)

HERO_W, HERO_H = 1200, 630
THUMB_W, THUMB_H = 600, 315
HERO_Q, THUMB_Q = 85, 80
WEBP_HERO_Q, WEBP_THUMB_Q = 80, 75
HERO_BUDGET = 150 * 1024   # 150 KB
THUMB_BUDGET = 50 * 1024   # 50 KB


def hex_to_rgb(hex_color: str) -> tuple:
    h = hex_color.lstrip('#')
    if len(h) != 6:
        raise ValueError(f"Invalid hex color: {hex_color!r}")
    return tuple(int(h[i:i + 2], 16) for i in (0, 2, 4))


def center_crop(img, tw, th):
    sw, sh = img.size
    if sw / sh > tw / th:
        nw = int(sh * tw / th)
        img = img.crop(((sw - nw) // 2, 0, (sw - nw) // 2 + nw, sh))
    elif sw / sh < tw / th:
        nh = int(sw * th / tw)
        img = img.crop((0, (sh - nh) // 2, sw, (sh - nh) // 2 + nh))
    return img.resize((tw, th), Image.LANCZOS)


def apply_gradient(img, brand_rgb):
    overlay = Image.new("RGBA", img.size, (0, 0, 0, 0))
    draw = ImageDraw.Draw(overlay)
    start = int(img.height * 0.75)
    for y in range(start, img.height):
        alpha = int(((y - start) / (img.height - start)) * 100)
        draw.line([(0, y), (img.width, y)], fill=(*brand_rgb, alpha))
    composited = Image.alpha_composite(img.convert("RGBA"), overlay)
    return composited.convert("RGB")


def process_image(input_path, output_dir, brand_rgb=None):
    inp = Path(input_path)
    out = Path(output_dir)
    if not inp.exists():
        print(f"Error: {inp} not found", file=sys.stderr)
        sys.exit(1)
    out.mkdir(parents=True, exist_ok=True)

    img = Image.open(inp)
    try:
        img = ImageOps.exif_transpose(img)
    except Exception:
        pass
    if img.mode == "RGBA":
        bg = Image.new("RGB", img.size, (255, 255, 255))
        bg.paste(img, mask=img.split()[3])
        img = bg
    elif img.mode != "RGB":
        img = img.convert("RGB")

    hero = center_crop(img, HERO_W, HERO_H)
    if brand_rgb is not None:
        hero = apply_gradient(hero, brand_rgb)
    thumb = hero.resize((THUMB_W, THUMB_H), Image.LANCZOS)

    hero.save(out / "hero.jpg", "JPEG", quality=HERO_Q, optimize=True)
    thumb.save(out / "hero-thumb.jpg", "JPEG", quality=THUMB_Q, optimize=True)
    hero.save(out / "hero.webp", "WEBP", quality=WEBP_HERO_Q, method=6)
    thumb.save(out / "hero-thumb.webp", "WEBP", quality=WEBP_THUMB_Q, method=6)

    for name, budget in [("hero.jpg", HERO_BUDGET), ("hero-thumb.jpg", THUMB_BUDGET)]:
        size = (out / name).stat().st_size
        flag = "WARNING: over budget" if size > budget else "OK"
        print(f"{name}: {size / 1024:.1f} KB  [{flag}]")
    print(f"\nSaved to: {out}/")


def main():
    p = argparse.ArgumentParser(description=__doc__,
                                formatter_class=argparse.RawDescriptionHelpFormatter)
    p.add_argument("input_image", help="Source image (PNG/JPG)")
    p.add_argument("output_dir", help="Destination directory for hero outputs")
    p.add_argument("--brand-color", default=None,
                   help="Hex color for bottom gradient overlay, e.g. #0B1D3A")
    p.add_argument("--no-gradient", action="store_true",
                   help="Skip gradient overlay even if --brand-color is set")
    args = p.parse_args()

    brand_rgb = None
    if args.brand_color and not args.no_gradient:
        try:
            brand_rgb = hex_to_rgb(args.brand_color)
        except ValueError as e:
            print(f"Error: {e}", file=sys.stderr)
            sys.exit(2)

    process_image(args.input_image, args.output_dir, brand_rgb=brand_rgb)


if __name__ == "__main__":
    main()
```

### 9.2 Smoke-Test the Image Script

```bash
python3 -c "from PIL import Image; print('Pillow OK')"
python3 scripts/process_hero_image.py --help   # exits 0 with usage text
```

Expected: usage block printed, exit code 0. If you see a stack trace or `error: the following arguments are required`, your `argparse` was set up incorrectly — re-check the script.

### 9.3 Run the Blog Skill

Commit and push all current files first:

```bash
git add .
git commit -m "feat: initial static site scaffold + blog skill"
git push -u origin main
```

Then invoke the skill in a Claude Code session:

```
/my-blog "Your first blog post topic here"
```

Follow the five human-in-the-loop checkpoints:

1. Approve the title, slug, category, and outline
2. Approve the written draft after reviewing the quality gate scorecard
3. Approve or regenerate the hero image
4. Approve the local browser preview of the assembled post
5. Approve and merge the PR

### Verification — Phase 6

After the PR is merged and CI completes:

```bash
curl -sI https://<DOMAIN>/blog/<your-slug>/ | head -1
curl -sI https://<DOMAIN>/blog/<your-slug>/hero.jpg | head -1
```

Expected: `HTTP/2 200` for both.

```bash
gh run list --workflow=deploy.yml --limit 3
```

Expected: most recent run shows `completed` / `success`.

---

## 10. Verification Gates Summary

| Phase | Command | Expected Output |
|-------|---------|-----------------|
| 1 | `ls .claude/ .github/ site/ scripts/` | All dirs present |
| 1 | `git status` | Clean tree or only expected untracked files |
| 2 | `python3 -c "import re, pathlib; ..."` (see Phase 2) | Correct POST_ token set found |
| 2 | `curl -sf http://localhost:8080/ >/dev/null && echo OK` | `OK` (after local server started) |
| 3 | `cat .github/workflows/deploy.yml \| grep "<[A-Z_]"` | Zero output (no unreplaced placeholders) |
| 3 | `gh run list --limit 3` | Workflow visible; manual trigger succeeds |
| 4 | `ls .claude/skills/my-blog/templates/` | Two HTML template files |
| 5 | `ls .claude/skills/my-brand-guidelines/` | SKILL.md present |
| 6 | `curl -sI https://<DOMAIN>/blog/<slug>/ \| head -1` | `HTTP/2 200` |
| 6 | `curl -sI https://<DOMAIN>/blog/<slug>/hero.jpg \| head -1` | `HTTP/2 200` |

---

## 11. Troubleshooting

### Problem 1: SSH key not accepted by host

**Symptom**: `scp` or CI deploy step fails with `Permission denied (publickey)`.

**Fix**:

```bash
# Verify the public key is in the host's authorized_keys
ssh -p <SSH_PORT> -v <SSH_USER>@<SSH_HOST> exit 2>&1 | grep "Offering"

# Re-add if missing — paste contents of your .pub file into the host control panel
# or append via SSH if you have another auth method:
ssh-copy-id -p <SSH_PORT> -i ~/.ssh/<COMPANY_NAME>_deploy_key.pub <SSH_USER>@<SSH_HOST>

# Verify the GitHub secret is correct base64 (run inside a workflow with secrets context):
echo "$DEPLOY_SSH_KEY_B64" | base64 -d | head -1
# Expected: -----BEGIN OPENSSH PRIVATE KEY-----
```

If the key re-encodes differently on different platforms, regenerate and re-add (POSIX-portable form):

```bash
ssh-keygen -t ed25519 -N "" -C "deploy-new@<DOMAIN>" -f ~/.ssh/deploy_new
base64 < ~/.ssh/deploy_new | tr -d '\n' > /tmp/key_b64.txt
# Paste /tmp/key_b64.txt contents as the GitHub secret value (DEPLOY_SSH_KEY_B64)
```

### Problem 2: CI deploys but site shows 0-byte or blank pages

**Symptom**: Deploy run succeeds, smoke check passes, but visiting `https://<DOMAIN>/` shows an empty page.

**Fix**: The SCP command `scp -r site/.` syncs all files inside `site/` to the remote path. Verify the remote path is exactly the web root, not one level above:

```bash
# SSH into host and check
ssh -p <SSH_PORT> <SSH_USER>@<SSH_HOST> "ls <DEPLOY_PATH>/ | head -20"
# You should see: index.html, blog/, assets/, etc.
# If you see a nested site/ directory, fix DEPLOY_PATH
```

For addon domains on shared hosts, the path is commonly `domains/<DOMAIN>/public_html` — not `public_html`. Check your host's file manager.

### Problem 3: Hero image exceeds 150KB size budget

**Symptom**: Stage 4 size check prints `WARNING: over budget`, skill offers to regenerate.

**Fix path 1** — Reduce image quality in `scripts/process_hero_image.py`:

```python
HERO_Q = 78      # reduce from 85
WEBP_HERO_Q = 72 # reduce from 80
```

**Fix path 2** — Regenerate with a simpler Gemini prompt. Add to the prompt: `Minimalist composition. Fewer elements. More whitespace. Abstract geometric shapes only.`

**Fix path 3** — After generation, run Pillow to aggressively re-compress:

```python
from PIL import Image
img = Image.open("site/blog/<slug>/hero.jpg")
img.save("site/blog/<slug>/hero.jpg", "JPEG", quality=72, optimize=True)
```

### Problem 4: Sitemap diff causes merge conflict

**Symptom**: Two blog posts published from separate branches each edit `site/sitemap.xml`. PR merge shows a conflict on the `</urlset>` line.

**Fix**: Before merging the second PR, rebase it on main:

```bash
git fetch origin
git checkout blog/<slug-2>
git rebase origin/main
# Manually resolve the sitemap conflict — keep both <url> entries
git add site/sitemap.xml
git rebase --continue
git push --force-with-lease
```

Prevent future conflicts by merging blog PRs sequentially, not in parallel.

### Problem 5: Blog index card inserted into wrong file or wrong location

**Symptom**: After running the skill, the homepage (`/`) shows blog content, or the blog listing shows no new card.

**Cause**: The skill inserted the card into `site/index.html` instead of `site/blog/index.html`, or SCP deployed `site/blog/index.html` to the web root (overwriting the homepage).

**Fix for wrong insertion**: The skill looks for the line `<p id="no-results"` to anchor the insertion point. Verify this line exists in `site/blog/index.html`:

```bash
grep -n "no-results" site/blog/index.html
```

If it is missing, add it back inside the posts grid container (see Phase 2 scaffolding).

**Fix for wrong SCP path**: Never run:

```bash
scp ... site/blog/index.html <SSH_USER>@<SSH_HOST>:<DEPLOY_PATH>/
```

The blog index must always deploy to `<DEPLOY_PATH>/blog/`, not `<DEPLOY_PATH>/`. CI handles this correctly via `scp -r site/.` — only manual deploys are at risk.

---

## 12. Done

The system is complete when all of the following are verifiable:

```bash
# 1. Homepage returns 200
curl -sf https://<DOMAIN>/ > /dev/null && echo "Homepage: OK"

# 2. Blog index returns 200
curl -sf https://<DOMAIN>/blog/ > /dev/null && echo "Blog index: OK"

# 3. First post returns 200
curl -sf https://<DOMAIN>/blog/<first-post-slug>/ > /dev/null && echo "Post: OK"

# 4. Hero image returns 200
curl -sf https://<DOMAIN>/blog/<first-post-slug>/hero.jpg > /dev/null && echo "Hero: OK"

# 5. Pagefind index exists (full-text search)
curl -sf https://<DOMAIN>/_pagefind/pagefind.js > /dev/null && echo "Pagefind: OK"

# 6. GitHub Actions last deploy succeeded
gh run list --workflow=deploy.yml --limit 1 --json conclusion \
  --jq '.[0].conclusion' | grep -q "success" && echo "CI: OK"

# 7. Skills are loadable by Claude Code
ls .claude/skills/my-blog/SKILL.md
ls .claude/skills/my-brand-guidelines/SKILL.md

# 8. Image processing script works
python3 scripts/process_hero_image.py 2>&1 | grep "Usage"
```

All eight checks output `OK` or expected usage text. The `/my-blog` skill is operational and can be invoked from any Claude Code session in this repository.
