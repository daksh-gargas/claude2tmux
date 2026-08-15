# claude2tmux — site

Single-page static site for claude2tmux. Plain HTML/CSS, no build step, no
analytics, no third-party scripts, no network fonts. Deploys to GitHub Pages
as-is.

## Layout

```
site/
├── index.html   ← landing page
└── style.css    ← single stylesheet (light + dark via prefers-color-scheme)
```

## Design

The hero is a **terminal window** showing real `claude2tmux --list` output,
because the disposition column (MOVE / STAGE / SKIP) *is* the product — it's
what makes the tool safe to run. Showing it beats describing it.

- **Terminal surfaces stay dark in both light and dark mode.** They're product
  shots of a terminal; a light-mode terminal would misrepresent the tool.
- **One accent colour**, a terminal green, reserved for the MOVE badge, the
  primary CTA, and the eyebrow. STAGE gets the amber warn colour. Nothing else
  is coloured.
- **Font** is the macOS system stack with `SF Mono` for terminal output.
- **Light/dark** via `@media (prefers-color-scheme)` — no JS toggle.
- No emoji, no illustrations, no marketing language.

The page leads with the constraint (a live PTY can't be reassigned on macOS, so
this resumes rather than moves) rather than burying it. Anyone evaluating the
tool needs to know what `--go` costs before they run it.

## Local preview

```sh
python3 -m http.server -d site 8000
```

then open <http://localhost:8000>. Toggle **System Settings → Appearance** with
the page open to check both themes.

## Deploy (GitHub Pages)

**Settings → Pages →** Source: *Deploy from a branch*, Branch: `main`,
Folder: `/site`. Pushing to `main` is enough; no GitHub Action needed.

The repo must be **public** for Pages to serve on a free plan — and the install
commands on the page (raw.githubusercontent, the release tarball, Homebrew)
all 404 while the repo is private.
