# Astro Starlight → GitHub Pages: How It Works

## Layer 0: What Astro Starlight actually produces

Astro is a **static site generator**. It reads your source files (Markdown, `.astro` components) and outputs a folder of plain HTML, CSS, and JS. No database. No server required to serve it. Just files.

```
Source files (.md, .astro, .ts)
        ↓  [Astro processes these]
dist/ folder (HTML + CSS + JS + assets)
        ↓  [any web server serves this]
Browser sees your site
```

Starlight is just an Astro theme/integration — it contributes sidebar layouts, search, etc. The output is still static HTML.

---

## Layer 1: The two Astro pipelines

### `npm run dev` — the dev server

```
Your source files
      ↓
Astro dev server (Node.js, port 4321)
      ↓  renders pages on-the-fly as you request them
Your browser at localhost:4321
```

- Watches for file changes and hot-reloads instantly
- Never writes a `dist/` folder
- Only lives on your machine
- Not for publishing — it's a live preview tool

### `npm run build` — the production build

```
Your source files
      ↓
Astro processes everything once
      ↓
dist/  ← a folder of static files (HTML, CSS, JS)
      ↓
Done. Astro exits.
```

- Runs once and exits — no server, no watching
- `dist/` is what you actually publish
- `npm run preview` after this starts a *temporary* local server that serves `dist/` so you can verify what the real site will look like

**The key distinction:** `dev` is a server you talk to. `build` is a one-shot process that writes files.

---

## Layer 2: `site` and `base` — the URL math problem

When Astro writes HTML, it has to write links to assets:

```html
<link rel="stylesheet" href="/styles/main.css">
<script src="/scripts/app.js"></script>
<a href="/guides/example/">Read guide</a>
```

These links are **relative to a root**. The question is: what IS the root?

### Case A: your site IS the root of the domain

Site lives at `https://example.com/`

```
https://example.com/styles/main.css  ✓  (/ maps to domain root)
https://example.com/guides/example/  ✓
```

No problem. The `/` prefix works perfectly.

### Case B: your site lives at a subpath

GitHub Pages project sites (any repo that isn't `<username>.github.io`) live at:
```
https://tongli4.github.io/temp-pub-pages-test/
```

Now those same links break:
```
https://tongli4.github.io/styles/main.css   ✗  (404 — nothing there)
https://tongli4.github.io/guides/example/   ✗  (404)
```

They should be:
```
https://tongli4.github.io/temp-pub-pages-test/styles/main.css   ✓
https://tongli4.github.io/temp-pub-pages-test/guides/example/   ✓
```

This is what `base` fixes. You tell Astro: "prepend `/temp-pub-pages-test` to every internal link you generate."

```js
// astro.config.mjs
base: '/temp-pub-pages-test'
```

And `site` is a separate concern — the full public URL, used for:
- Canonical `<link rel="canonical">` tags (SEO)
- `sitemap.xml` generation
- Open Graph `og:url` meta tags
- RSS feed absolute URLs

```js
site: 'https://tongli4.github.io/temp-pub-pages-test'
```

**Summary:**

| Config | What it controls | Example |
|--------|-----------------|---------|
| `site` | The full public URL of your site | `https://tongli4.github.io/temp-pub-pages-test` |
| `base` | The path prefix for internal links | `/temp-pub-pages-test` |

If you ever add a custom domain (`docs.example.com`), both become irrelevant for subpathing — your site IS the root, `base` goes away, and `site` becomes `https://docs.example.com`.

---

## Layer 3: What GitHub Pages IS

GitHub Pages is just a **static file host**. You give it a folder of HTML/CSS/JS, it serves them at a URL. That's it.

Two flavors:
- **User site**: repo named `tongli4.github.io` → served at `https://tongli4.github.io/` (root, no subpath)
- **Project site**: any other repo → served at `https://tongli4.github.io/<reponame>/` (always a subpath!)

Your repo `tongli4/temp-pub-pages-test` is a project site → subpath → `base` is required.

### Three historical ways to get files into Pages

```
Approach 1 (old): gh-pages branch
  You push built HTML to a special branch
  GitHub reads from that branch
  Problem: built output in git — large, messy, confusing history

Approach 2 (also old): /docs folder on main
  Commit built output into /docs on main branch
  Same problem — built output committed to git

Approach 3 (modern): GitHub Actions → Pages deployment API
  Workflow builds dist/ → uploads as an ephemeral artifact
  GitHub's deployment system publishes the artifact
  Built output NEVER goes into git
  ← This is what we'll use
```

---

## Layer 4: How GitHub Actions + Pages works

When you push to `main`, GitHub starts a workflow. Here's what actually happens:

```
You: git push origin main
         ↓
GitHub detects push to main
         ↓
Starts GitHub Actions workflow
         ↓
┌──────────────────────────────────────┐
│  Job 1: build  (runs on ubuntu VM)  │
│                                      │
│  1. checkout — copies your source    │
│     files onto the VM                │
│                                      │
│  2. configure-pages — queries GitHub │
│     API: "where will Pages serve     │
│     this repo?" Gets back:           │
│       origin = https://tongli4.      │
│               github.io              │
│       base_path = /temp-pub-pages-   │
│               test                   │
│                                      │
│  3. npm ci — installs node_modules   │
│                                      │
│  4. astro build                      │
│     --site "https://tongli4.         │
│             github.io"               │
│     --base "/temp-pub-pages-test"    │
│     → writes dist/ folder            │
│                                      │
│  5. upload-pages-artifact            │
│     packages dist/ as a GitHub       │
│     artifact (stored internally)     │
└──────────────────────────────────────┘
         ↓
┌──────────────────────────────────────┐
│  Job 2: deploy  (special perms)     │
│                                      │
│  deploy-pages — GitHub's internal   │
│  deployment system picks up the     │
│  artifact and publishes it to the   │
│  Pages CDN                          │
└──────────────────────────────────────┘
         ↓
https://tongli4.github.io/temp-pub-pages-test/ is live
```

### Why two separate jobs?

This is GitHub's security model, not optional:

- The `deploy` job runs in a special `github-pages` **Environment** with elevated permissions to publish
- Separating build from deploy means the build step (which runs arbitrary code — your package.json scripts, etc.) runs with minimal permissions
- The deployment uses **OIDC** — GitHub issues a short-lived cryptographic token proving "this is workflow X running on repo Y right now." No stored secrets, no PATs. Token expires when the workflow finishes.

### Why `configure-pages` instead of hardcoding?

You *could* hardcode in `astro.config.mjs`:
```js
site: 'https://tongli4.github.io/temp-pub-pages-test',
base: '/temp-pub-pages-test',
```

But `configure-pages` queries GitHub dynamically, so:
- Rename the repo → base path updates automatically
- Add a custom domain → base path becomes `/` automatically
- Fork the repo to another account → site URL adjusts automatically

It's the difference between a magic string and a live query.

---

## The complete picture side by side

```
LOCAL DEV                    PRODUCTION (GitHub Actions)
───────────────────────────────────────────────────────

npm run dev                  git push origin main
    ↓                              ↓
Astro dev server           GitHub Actions workflow starts
(Node.js, port 4321)             ↓
    ↓                       Job 1: build
Your browser at                 ↓ checkout source
localhost:4321                  ↓ configure-pages (gets site/base)
                                ↓ npm ci
Files never leave               ↓ astro build → dist/
your machine                    ↓ upload artifact
                                ↓
                           Job 2: deploy
                                ↓ deploy-pages (OIDC auth)
                                ↓
                           GitHub Pages CDN
                                ↓
                           https://tongli4.github.io/
                           temp-pub-pages-test/

No dist/ in git            No dist/ in git
No server needed later     No server needed later
                           (Pages serves static files)
```

---

## Summary of all the "builds" / pipelines

| Command / Step | What runs | Output | Purpose |
|----------------|-----------|--------|---------|
| `npm run dev` | Node.js dev server | nothing on disk | Local editing |
| `npm run build` | Astro one-shot | `dist/` folder | Produces publishable files |
| `npm run preview` | Local file server | nothing new | Test built output locally |
| Actions `build` job | `astro build` on GitHub's VM | artifact uploaded to GitHub | CI build |
| Actions `deploy` job | GitHub's Pages API | site goes live on CDN | Publishing |

There's only ever **one build**: `astro build`. Everything else is either a dev tool or a way to move/serve the output of that build.

> **Note:** Astro 6 requires Node.js ≥ 22.12.0. GitHub Actions `ubuntu-latest` defaults to Node 20 — always pin `node-version: 22` in the `setup-node` step or the build will fail immediately.
