# Vignette

Personal shopping moodboard web app (single user: GitHub `pliggity`). Static site
on GitHub Pages, GitHub-as-database, small Cloudflare Worker for OAuth + link-scrape.
Was formerly "Wishroom" — fully rebranded 2026-07-12.

**→ Read `docs/PROJECT_CONTEXT.md` first.** It has the architecture, the *why*
behind decisions, the runtime gotchas, the file map, and an operational quickstart
(deploy commands, verify steps). Don't start editing without it — several things
look arbitrary or wrong until you know the context (e.g. the Claude Design runtime
calls `componentDidUpdate` with **no `prevState`**; failures there are swallowed
silently).

## Fast facts
- **Live:** https://pliggity.github.io/vignette/ · **Repo:** `pliggity/vignette`
- **Worker:** `~/vignette-oauth` → `vignette-oauth.pliggity.workers.dev`
- **The app is one file:** `index.html` (Claude Design template + inline React class).
- **Deploy:** commit + `git push` with a token from `op read` (see quickstart in the doc).
  Pages rebuilds ~30s. The app auto-commits to `data.json`, so `git pull --rebase`
  before pushing is usually needed.
- **Don't rename** the `vignette-*` localStorage keys or worker without reading
  "The rename" section — there's a login-migration shim and origin/callback wiring
  that will break silently.
