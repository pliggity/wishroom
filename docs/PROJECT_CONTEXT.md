# Vignette — Project Context & Handoff

_A living reference for anyone (human or AI) picking up this project. Captures the
**why** behind the architecture, the non-obvious gotchas, and the current state —
the things you can't infer by reading the code alone._

Last updated: 2026-07-12

> **Naming note:** the app was originally **Wishroom** and was rebranded to
> **Vignette** on 2026-07-12 (editorial redesign from a Claude Design prototype +
> full repo/URL rename). Some *internal, invisible* identifiers deliberately still
> say `wishroom` — see "The rename" below. Don't "fix" them blindly.

---

## What Vignette is

A personal shopping moodboard web app. You save products (by link or manually),
organize them into **vignettes** (freeform, color-palette-driven collages — the
concept formerly called "boards"), group them by user-editable **scenes** (the
concept formerly called "folders"), tag them, and track prices. Single user: the
owner is GitHub user **`pliggity`**.

- **Live app:** https://pliggity.github.io/vignette/
- **App repo:** `pliggity/vignette` (public; renamed from `pliggity/wishroom`)
- **OAuth/proxy worker:** `wishroom-oauth.pliggity.workers.dev` (Cloudflare; name kept as-is — it's an invisible endpoint. Source in the separate `wishroom-oauth` project, not committed to the app repo)
- **Local clone:** still `~/wishroom` on the owner's machine (just a folder name)

It started as a Claude Design prototype and was turned into a real, persistent,
auth-gated, mobile-installable app; then a second Claude Design pass produced the
Vignette editorial redesign that was merged back over the live backend.

## Design language (Vignette)

Editorial / fashion-magazine: **Fraunces** (serif display) + **Inter** (sans),
a cool-gray palette (`#F7F7F5` bg, `#232326` ink, `#9B9793`/`#726F6C` muted,
`#9A6B72` dusty-rose for danger). Sharp 2px corners, uppercase letter-spaced
labels, outline buttons that fill and swap to **Fraunces italic lowercase** on
hover, and a literal **vignette shadow framing the viewport** (the brand). Custom
Fraunces "V" wordmark + matching app icon.

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
  ├──────────────► GitHub Contents API  (repo: pliggity/vignette, file: data.json)
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
| **Auth** | GitHub OAuth web flow. Redirect → `?code=` → Worker exchanges for token → app verifies `login === 'pliggity'` → token stored in `localStorage['wishroom-pat']` (key deliberately kept through the rename — see below). The OAuth App's **Authorization callback URL** must equal the live app URL (`https://pliggity.github.io/vignette/`). |
| **Data** | Entire state (items, boards, **scenes**, tags, archived, nextId) is one JSON blob committed to `data.json` via the Contents API. Debounced 1.5s; SHA-tracked with 409-retry. `_saveData()` is the single source of the payload shape. |
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

- **PWAs cannot be iOS share targets.** That's why the "share to Vignette" feature
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

- **Commits to `data.json` are auto-generated** ("update vignette") by the app every
  time state changes. Expect lots of them in the history; that's normal.

---

## The rename (Wishroom → Vignette), and what deliberately stayed

The repo/URL was renamed `pliggity/wishroom` → `pliggity/vignette` on 2026-07-12.
Renaming a GitHub repo carries `data.json` + Releases (images) along automatically,
and git remotes auto-redirect. What that touched — and what was intentionally left:

- **Changed:** the 6 hardcoded `api.github.com/repos/pliggity/wishroom/...` paths in
  `index.html`, the manifest `scope`/`start_url` (`/wishroom/`→`/vignette/`), and the
  SW cache name (`wishroom-v1`→`vignette-v1`, to force a clean takeover).
- **⚠️ Manual step, no API:** the **OAuth App's Authorization callback URL** must be
  updated in GitHub Developer Settings to `https://pliggity.github.io/vignette/`.
  The authorize link in code sends **no `redirect_uri`**, so GitHub redirects to
  whatever callback is registered — a stale one 404s after sign-in. (Classic OAuth
  Apps have no management REST API, so this can't be scripted.)
- **Kept `wishroom` on purpose (do NOT rename):**
  - `localStorage['wishroom-pat']` (+ `wishroom-oauth-state`, `wishroom-pending-add`)
    — localStorage is per-*origin*, not per-path, so keeping the key means the
    existing login survived the URL change. Renaming it would force a re-auth.
  - The Cloudflare Worker `wishroom-oauth.pliggity.workers.dev` — invisible endpoint;
    renaming it means redeploying + updating the app URL for zero user-visible gain.
  - The local clone folder `~/wishroom` — just a directory name.
- **After a rename, the user must also:** re-add the home-screen PWA at the new URL
  (old `/wishroom/` now 404s) and update the iOS Shortcut's `?add=` URL.

## Data model: scenes & vignettes

The Vignette redesign renamed concepts *and* changed the data shape:
- items' `folder` → **`scene`**; there's a top-level user-editable **`scenes`** list;
  boards (now "vignettes") carry an optional `scene`.
- **Migration on load** (`loadFromGitHub`): `item.scene = item.scene || item.folder ||
  'Home'`, and `scenes` is seeded from any saved list or derived from item scenes.
  Legacy `data.json` (folder-only) loads unchanged. The save payload now includes
  `scenes` (see `_saveData()` and the `componentDidUpdate` change-detection list).

---

## File map

**`pliggity/vignette` (app):**
- `index.html` — the whole app: Claude Design template + inline `DCLogic` React class. This is where ~all logic lives.
- `support.js` — the Claude Design runtime (React loader, template compiler, component wrapper). Third-party; generally don't edit, but read it to understand runtime behavior.
- `data.json` — the entire dataset (items/boards/tags/…). Written by the app.
- `sw.js` — service worker (network-first, same-origin cache; never intercepts GitHub API / images / worker).
- `manifest.webmanifest`, `icon-*.png`, `apple-touch-icon.png`, `icon.svg` — PWA install assets (Fraunces "V" wordmark icon).
- `.nojekyll` — disables Jekyll on Pages.
- `docs/PROJECT_CONTEXT.md` — this file.

**`wishroom-oauth` (Cloudflare Worker, separate project):**
- `src/index.js` — OAuth token exchange (`POST /`) + `POST /fetch-meta` (auth-gated scraper that returns title + re-hostable image bytes).
- `wrangler.toml` — Worker config. Secrets `GITHUB_CLIENT_ID` / `GITHUB_CLIENT_SECRET` are set in Cloudflare, not in the repo.
- Deploy with `npx wrangler deploy`.

---

## Current state (2026-07-12)

Working and deployed at **https://pliggity.github.io/vignette/**:
- Vignette editorial redesign live (merged from a Claude Design prototype over the
  existing backend). Scenes + vignettes data model; permanent-delete in archive.
- OAuth login, GitHub-backed persistence with cross-device sync, 409-retry.
- Add by link (with URL-based name/size/color/qty guessing when a site blocks
  scraping) and manual custom tiles — both with tags, vignettes(boards), notes, price.
- Image upload → GitHub Releases; scraped images re-hosted.
- Mobile mode (responsive CSS), PWA install, touch drag/resize, board zoom with
  bounded spread, palette reorder on touch.
- Security hardening pass complete (SSRF gate, OAuth CSRF `state`, safe links,
  CSS-injection sanitize, save-race serialization).
- Full repo/URL rename complete; OAuth callback URL updated to `/vignette/`.

Known open items / TODO:
- **iOS Shortcut for share-sheet** — app side is done and live (`?add=` handler is
  dormant until used). The owner still needs to build the Shortcut on their phone
  (recipe: Receive URLs from Share Sheet → URL Encode → Text
  `https://pliggity.github.io/vignette/?add=[Encoded]` → Open URLs). Caveat: opens
  in Safari, so must be signed in there too.
- **`data.json` size** — image migration to Releases runs on load (`migrateImages`)
  and shrinks it over time; check it's fully migrated (few KB, image URLs not base64).
- **"Real" native app (Capacitor)** — deferred option if a true share-sheet entry /
  app-store presence is wanted later. ~1–3 days + $99/yr Apple Developer.
- **Cosmetic leftovers** — the Worker name (`wishroom-oauth`), the `wishroom-*`
  localStorage keys, and the local `~/wishroom` folder still say wishroom on purpose
  (see "The rename"). Only rename them if you accept the tradeoffs noted there.

---

## Conventions

- Commits: work is committed and pushed to `pliggity/vignette` `main`. The GitHub
  token for pushing is fetched at runtime from 1Password (`op read`), never stored
  in plaintext or pasted into chat. The OAuth `client_secret` lives only in the
  Cloudflare Worker. (The git remote was also de-tokenized during the rename — push
  with an explicit `https://pliggity:$TOKEN@…` URL, not a baked-in credential.)
- After changing `index.html`, GitHub Pages takes ~30s to rebuild; hard-refresh
  (Cmd/Ctrl+Shift+R) to bypass the service worker cache.
