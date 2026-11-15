# Gym Tracker

A personal workout logger: a PWA on GitHub Pages, with data in a single
`gym.json` in Google Drive. The full spec is in **SPEC.md**. Read it before
changing anything.

## Ground rules

- No framework, no build step, no npm dependencies. Plain HTML/CSS/JS. The app
  is one `index.html` with inline CSS and JS, plus `manifest.webmanifest`,
  `sw.js` and icons.
- UI text in English.
- Local-first: the app must be fully usable without Google Drive. Drive sync
  lives in one clearly separated section of the script.
- `gym.json` is also edited by hand. Write it pretty-printed (2 spaces) with
  stable key order, and read it tolerantly (unknown or missing optional fields
  must not break anything).
- Mobile first: test at a 390×844 viewport. There must be no horizontal scroll
  at 360 px.
- Keep it small and readable. This is a personal tool, not a product.

## Run locally

```
python3 -m http.server 8000
```

Open http://localhost:8000. `http://localhost:8000` must be registered as an
authorized JavaScript origin of the Google OAuth client for Drive to work
locally (see SETUP.md).

## Testing

Use Playwright (Chromium) against the local server with a phone viewport, and
go through the acceptance checks in SPEC.md §9. Drive sync is tested manually
by the user.

## Deploy

Push to `main`. GitHub Pages serves the repo root. When a change has to reach
installed apps immediately, bump the cache name in `sw.js`.
