# Gym Tracker

A personal workout logger: an installable PWA (plain HTML/CSS/JS, no build step)
that stores its data in `gym.json` in Google Drive, with a local copy in the
browser. See [SPEC.md](SPEC.md) for the full spec and [SETUP.md](SETUP.md) for
the one-time GitHub Pages + Google OAuth setup.

## Run locally

```
python3 -m http.server 8000
```

Open http://localhost:8000. The app works fully without Google Drive. For Drive
sync, set `GOOGLE_CLIENT_ID` at the top of the script in `index.html` and add
`http://localhost:8000` as an authorized JavaScript origin of the OAuth client.

## Deploy

Push to `main`; GitHub Pages serves the repo root. To push a change to
installed apps immediately, bump `CACHE` in `sw.js`.
