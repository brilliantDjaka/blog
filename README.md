# Brilliant's Blog

Personal blog built with [Hugo](https://gohugo.io/) and the [DoIt theme](https://github.com/HEIGE-PCloud/DoIt), hosted on GitHub Pages at **https://brilliantdjaka.github.io/blog/**.

## Requirements

- [Devbox](https://www.jetify.com/devbox) (provides Hugo extended + Node.js, see `devbox.json`)

## Development

```bash
devbox shell                 # enter the devbox environment
hugo server -D               # live dev server (drafts included) -> http://localhost:1313/blog/
hugo --minify                # production build to public/
```

Notes:

- Site is served from the `/blog/` base path (see `baseURL` in `hugo.toml`), so the dev server URL includes `/blog/`.
- The theme is a git submodule in `themes/DoIt`. Update it with:
  ```bash
  git submodule update --remote --merge themes/DoIt
  ```
- DoIt requires **Hugo extended ≥ 0.146** and Dart Sass (the devbox `hugo` package bundles both; the CI workflow installs them explicitly).
- Custom styles go in `assets/css/_custom.scss` (new styles) and `assets/css/_override.scss` (variable overrides).

## Content

- Posts: `content/posts/<slug>.md` — create one with `hugo new posts/<slug>.md` (archetype in `archetypes/default.md`).
- Static pages: `content/about.md`.
- Permalink pattern for posts: `/<slug>/` (see `[permalinks]` in `hugo.toml`).

## Deployment (GitHub Pages)

The repo is named `blog`, so GitHub Pages serves it at `https://brilliantdjaka.github.io/blog/`. The workflow `.github/workflows/hugo.yaml` builds the site and deploys it to Pages on every push to `main`.

### 1. Create the repo and push

From this directory (remote `origin` = `git@github.com:brilliantDjaka/blog.git` is already configured):

```bash
# create the empty repo (public) on GitHub
gh repo create brilliantDjaka/blog --public

# push (submodule pointer is part of the commit; the theme repo is public, so Pages can clone it)
git push -u origin main
```

### 2. Enable GitHub Pages (Actions source)

Via the web UI:

1. Open the repo on GitHub → **Settings** → **Pages**.
2. Under **Build and deployment** → **Source**, select **GitHub Actions**.
3. Done. The existing `Build and deploy` workflow takes over deployment.

Or via the CLI (equivalent):

```bash
gh api repos/brilliantDjaka/blog/pages -X POST -f build_type=workflow
```

### 3. Deploy

Push to `main` and the workflow runs:

```bash
git add -A && git commit -m "post: my new post" && git push
```

- Watch the run under **Actions** → "Build and deploy".
- First deployment takes ~1–2 minutes; subsequent ones ~30–60s.
- Site goes live at https://brilliantdjaka.github.io/blog/ (allow a few minutes for CDN propagation).

### Workflow details

`.github/workflows/hugo.yaml`:

- Checks out with `submodules: recursive` (fetches `themes/DoIt`) and full history (`fetch-depth: 0`, needed for `enableGitInfo`).
- Installs Dart Sass, Hugo extended, Node.js, and npm dependencies (if a lockfile is present).
- Builds with `--gc --minify` and `--baseURL` taken from the Pages config (resolves to `https://brilliantdjaka.github.io/blog`).
- Uploads `public/` as the Pages artifact and deploys it.

To change the Hugo/Sass/Node versions used in CI, edit the `env:` block at the top of the workflow.

## Project layout

```
├── .github/workflows/hugo.yaml   # Pages build & deploy workflow
├── archetypes/                   # post template (hugo new)
├── assets/css/                   # _custom.scss / _override.scss (site overrides)
├── content/                      # posts + static pages
├── hugo.toml                     # site config (baseURL, theme, menus, markup)
├── static/                       # static files (favicon) copied as-is
├── themes/DoIt                   # git submodule -> HEIGE-PCloud/DoIt
├── devbox.json / devbox.lock     # dev environment (hugo + node)
└── package.json                  # placeholder (no runtime npm deps needed)
```
