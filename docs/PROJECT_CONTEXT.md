# Wishroom — Project Context & Handoff

_A living reference for anyone (human or AI) picking up this project. Captures the
**why** behind the architecture, the non-obvious gotchas, and the current state —
the things you can't infer by reading the code alone._

Last updated: 2026-07-11

---

## What Wishroom is

A personal shopping wishlist / moodboard web app — "a consumerism mind map." You
save products (by link or manually), organize them into **boards** (freeform,
color-palette-driven collages), tag them, and track prices. Single user: the
owner is GitHub user **`pliggity`**.

- **Live app:** https://pliggity.github.io/wishroom/
- **App repo:** `pliggity/wishroom` (public)
- **OAuth/proxy worker:** `wishroom-oauth.pliggity.workers.dev` (Cloudflare; source in the separate `wishroom-oauth` project, not committed to the app repo)

It started as a Claude Design prototype and was turned into a real, persistent,
auth-gated, mobile-installable app.

---

## Architecture at a glance

```
Browser (GitHub Pages static site)
  │
  │  OAuth login ("Sign in with GitHub")
  ├──────────────► wishroom-oauth Worker ──► GitHub OAuth (holds client_secret)
  │                        │
  │                        └─ /fetch-meta: scrape og:title/og:image for link-adds
  │
  │  read/write data.json (the entire dataset)
  ├──────────────► GitHub Contents API  (repo: pliggity/wishroom, file: data.json)
  │
  │  image storage (uploaded, not base64)
  └──────────────► GitHub Releases (tag "images") as release assets
```

**There is no backend server and no database.** GitHub *is* the storage layer.
The Cloudflare Worker exists only to (a) hold the OAuth `client_secret` off the
client and (b) act as an authenticated scraping proxy for link metadata.

### Key pieces

| Concern | How it works |
|---|---|
| **Hosting** | GitHub Pages, static. `.nojekyll` present so Jekyll doesn't choke on `{{ }}` template syntax. |
| **Runtime** | Claude Design runtime (`support.js`) — loads React 18 + Babel from unpkg (with SRI hashes + `crossOrigin`), then mounts a `DCLogic`/`StreamableLogic` React class component defined inline in `index.html`. |
| **Auth** | GitHub OAuth web flow. Redirect → `?code=` → Worker exchanges for token → app verifies `login === 'pliggity'` → token stored in `localStorage['wishroom-pat']`. |
| **Data** | Entire state (items, boards, tags, archived, nextId) is one JSON blob committed to `data.json` via the Contents API. Debounced 1.5s; SHA-tracked with 409-retry. |
| **Images** | Resized to ≤640px JPEG client-side, uploaded to the repo's "images" Release, stored as URLs (not base64) to keep `data.json` small. |
| **Install** | PWA (manifest + service worker + apple-touch-icon). Installable to iOS home screen **via Safari only**. |

---

## Decisions & the *why* behind them

These are the choices that would look arbitrary without context:

1. **Public-read, private-write is intentional.** The repo is public, so `data.json`
   is world-readable (anyone with the URL can GET it). The owner explicitly does
   **not** care about read privacy — only about preventing edits. Write protection
   is real and enforced by **GitHub**, not the UI: only `pliggity` has push access,
   and OAuth hands each visitor a token for *their own* account, which GitHub
   rejects for writes to this repo. The `login === 'pliggity'` check in the UI is
   cosmetic; the real boundary is GitHub's permission model + the owner's 2FA.

2. **OAuth, not a PAT.** Originally gated with a Personal Access Token typed into
   the app. Replaced with proper OAuth so there's a real "Sign in with GitHub"
   button and no token typing. The `client_secret` lives only in the Worker's
   encrypted env vars — never in the repo or client.

3. **Link-add opens the modal; it does NOT silently save.** When you share/paste a
   link, it opens the add modal pre-filled rather than saving in the background.
   Reason: silent-save would require the GitHub token to live in the iOS Shortcut
   or a new write-capable Worker endpoint — i.e. **new token-exposure surface**,
   which the owner has repeatedly and explicitly ruled out. Opening the app reuses
   the token already safely in the browser.

4. **Images are re-hosted, never hot-linked.** Scraped `og:image` bytes are fetched
   server-side by the Worker and uploaded to GitHub Releases like a manual upload.
   Avoids leaking the owner's IP to source sites on every view, and avoids images
   rotting when the source removes them.

5. **The Worker `/fetch-meta` is auth-gated + SSRF-hardened.** It was briefly an
   open proxy (CORS does NOT protect a server-side fetch). Now it requires the
   caller's GitHub token, verifies `login === 'pliggity'`, and blocks
   non-http(s) schemes and private/loopback/link-local IPs before fetching.

---

## Gotchas (the expensive-to-rediscover stuff)

- **⚠️ The Claude Design runtime calls `componentDidUpdate(prevProps)` with
  `prevProps` ONLY — there is no `prevState` argument** (see `support.js` ~line
  912). Code that reads `prevState[k]` throws, and **the runtime swallows the error
  in a try/catch**, so it fails silently. This caused GitHub saves to never fire
  (no "saving…", nothing persisted) and cost hours to find. Detect state changes by
  snapshotting onto an instance var (see `_prevData` / `_snapshotData`), never via a
  `prevState` param.

- **PWA install is Safari-only on iOS.** Chrome, Arc, DuckDuckGo etc. on iOS cannot
  create a true standalone PWA (Apple restriction). "Add to Home Screen" must be
  done from Safari. Android/Chrome installs fine.

- **PWAs cannot be iOS share targets.** That's why the "share to Wishroom" feature
  is done via an iOS **Shortcut** that opens `?add=<url>`, not a native share
  extension. A real share-sheet entry would require wrapping the app in Capacitor
  (native shell) — deferred.

- **`keepalive` fetch has a 64KB body cap.** The `beforeunload` flush uses it as a
  best-effort last save; it can fail if `data.json` is large. The real fix was
  moving images out of `data.json` (Releases) so the payload stays small.

- **Template attribute casing:** the runtime maps lowercased HTML attrs back to
  React camelCase via an `EVENT_MAP` + a generic `on*` fallback (`support.js` ~line
  407). `onPointerDown` etc. work, but if you add an exotic handler check it maps.

- **Touch drag needs pointer capture + no re-render mid-gesture.** Board tile
  move/resize and palette swatch reorder use Pointer Events. On touch, calling
  `setState` mid-drag re-renders and breaks the implicit pointer capture, freezing
  the drag. Solution: `setPointerCapture`, mutate the DOM directly during the drag,
  and `setState` once on release. (Mouse doesn't have this problem, so bugs here
  look "desktop works, mobile doesn't.")

- **Commits to `data.json` are auto-generated** ("update wishroom") by the app every
  time state changes. Expect lots of them in the history; that's normal.

---

## File map

**`pliggity/wishroom` (app):**
- `index.html` — the whole app: Claude Design template + inline `DCLogic` React class. This is where ~all logic lives.
- `support.js` — the Claude Design runtime (React loader, template compiler, component wrapper). Third-party; generally don't edit, but read it to understand runtime behavior.
- `data.json` — the entire dataset (items/boards/tags/…). Written by the app.
- `sw.js` — service worker (network-first, same-origin cache; never intercepts GitHub API / images / worker).
- `manifest.webmanifest`, `icon-*.png`, `apple-touch-icon.png`, `icon.svg` — PWA install assets (flower icon).
- `.nojekyll` — disables Jekyll on Pages.
- `docs/PROJECT_CONTEXT.md` — this file.

**`wishroom-oauth` (Cloudflare Worker, separate project):**
- `src/index.js` — OAuth token exchange (`POST /`) + `POST /fetch-meta` (auth-gated scraper that returns title + re-hostable image bytes).
- `wrangler.toml` — Worker config. Secrets `GITHUB_CLIENT_ID` / `GITHUB_CLIENT_SECRET` are set in Cloudflare, not in the repo.
- Deploy with `npx wrangler deploy`.

---

## Current state (2026-07-11)

Working and deployed:
- OAuth login, GitHub-backed persistence with cross-device sync, 409-retry.
- Add by link (with URL-based name/size/color/qty guessing when a site blocks
  scraping) and manual custom tiles — both with tags, boards, notes, price.
- Image upload → GitHub Releases; scraped images re-hosted.
- Mobile mode (responsive CSS), PWA install, touch drag/resize, board zoom with
  bounded spread, palette reorder on touch.
- Security hardening pass complete (SSRF gate, OAuth CSRF `state`, safe links,
  CSS-injection sanitize, save-race serialization).

Known open items / TODO:
- **iOS Shortcut for share-sheet** — app side is done and live (`?add=` handler is
  dormant until used). The owner still needs to build the Shortcut on their phone
  (recipe: Receive URLs from Share Sheet → URL Encode → Text
  `https://pliggity.github.io/wishroom/?add=[Encoded]` → Open URLs). Caveat: opens
  in Safari, so must be signed in there too.
- **`data.json` still ~458KB** — image migration to Releases may not have fully run
  for all legacy base64 images. Loading the app while signed in runs `migrateImages`
  which should shrink it over time.
- **"Real" native app (Capacitor)** — deferred option if a true share-sheet entry /
  app-store presence is wanted later. ~1–3 days + $99/yr Apple Developer.
- **Homebrew vs npm version lag** — unrelated to the app; just the owner's Claude
  Code install channel.

---

## Conventions

- Commits: work is committed and pushed to `pliggity/wishroom` `main`. The GitHub
  token for pushing is fetched at runtime from 1Password (`op read`), never stored
  in plaintext or pasted into chat. The OAuth `client_secret` lives only in the
  Cloudflare Worker.
- After changing `index.html`, GitHub Pages takes ~30s to rebuild; hard-refresh
  (Cmd/Ctrl+Shift+R) to bypass the service worker cache.
