# Gym Tracker — Specification v1

A personal, single-user workout logger built as a Progressive Web App (PWA).
It replaces a Google Sheets log. The user opens it on his Android phone in the
gym (installed to the home screen) and occasionally on his laptop.

## 1. Background: how the user trains

- **Full-body training**, about 2 sessions per week, 60–70 minutes, 5–6 exercises
  with ~3 sets each.
- **About one exercise per muscle group per session.** The user picks the
  exercises freshly each time; there is no fixed plan.
- Two criteria when picking: (a) exercises he likes, (b) exercises he has not
  done for a while.
- Some exercises must not be skipped for more than 2–3 sessions; others may
  rest for 4–5 sessions. This **skip limit is counted in sessions, not days**,
  because training frequency is irregular.
- Exercises can belong to several muscle groups (e.g. Face Pulls = Shoulders +
  Back).

The app's core job is to make the picking step fast: on arrival at the gym,
show every muscle group with its exercises and how many sessions ago each was
last done, flagging the ones at or near their limit.

## 2. Platform and tech constraints

- Static website, hosted on **GitHub Pages**. No backend, no server code.
- **Plain HTML/CSS/JavaScript. No framework, no build step, no npm
  dependencies.** The app should still run unchanged in 10 years.
- Main app in a single `index.html` (inline CSS + JS). Extra files only where
  the platform requires them: `manifest.webmanifest`, `sw.js`, icons.
- Installable PWA (manifest + service worker). Must work well on a phone screen
  (Android Chrome) and acceptably on a laptop.
- **UI language: English.**
- Mobile ergonomics matter: large tap targets, numeric keyboards for numbers
  (`inputmode="decimal"` / `"numeric"`), accept a decimal comma ("12,5"), and no
  layout that needs horizontal scrolling.
- Light and dark theme following the system setting.

## 3. Data

All data lives in **one JSON file**, `gym.json`, stored in the user's Google
Drive (see §6). The app keeps a local copy in `localStorage` and syncs it.

The file must stay readable and hand-editable: pretty-printed with 2-space
indentation, stable key order, ISO dates. The user will fix past mistakes by
editing this file directly, so the app must tolerate hand edits. For example,
it should ignore unknown fields and survive missing optional fields.

### 3.1 Schema

```json
{
  "format": "gym-tracker",
  "version": 1,
  "updatedAt": "2026-11-09T17:40:00.000Z",
  "groups": [
    { "id": "shoulders", "name": "Shoulders", "exercises": ["lateral-raise-machine", "face-pulls"] },
    { "id": "chest",     "name": "Chest",     "exercises": ["chest-press-b"] },
    { "id": "back",      "name": "Back",      "exercises": ["face-pulls", "lat-pulldown"] },
    { "id": "legs",      "name": "Legs",      "exercises": [] },
    { "id": "biceps",    "name": "Biceps",    "exercises": [] },
    { "id": "triceps",   "name": "Triceps",   "exercises": [] },
    { "id": "abs",       "name": "Abs",       "exercises": [] }
  ],
  "exercises": [
    {
      "id": "face-pulls",
      "name": "Face Pulls",
      "setting": "Top; double rope",
      "maxSkip": 3,
      "archived": false,
      "photo": "face-pulls-m3k9x2a1.jpg"
    }
  ],
  "sessions": [
    {
      "id": "2026-11-09-a1b2",
      "date": "2026-11-09",
      "finished": true,
      "items": [
        {
          "exercise": "face-pulls",
          "note": "left shoulder slightly sore",
          "sets": [
            { "weight": 17.5, "reps": 15, "plus": false },
            { "weight": 17.5, "reps": 15, "plus": true },
            { "weight": null, "reps": 12, "plus": false }
          ]
        }
      ]
    }
  ]
}
```

Field rules:

- **groups**: fixed list of 7, in this order: Shoulders, Chest, Back, Legs,
  Biceps, Triceps, Abs. Groups cannot be added or removed in the UI.
  `groups[].exercises` is the **ordered** list of exercise ids in that group.
  It is the single source of truth for both group membership and display order.
  An exercise in several groups appears in several lists, and can have a
  different position in each.
- **exercises[].id**: a slug made from the name at creation time (made unique
  with a suffix if needed). It never changes, even if the exercise is renamed.
- **exercises[].setting**: the machine or seat setting, for example
  "Seat 5; top 1". It belongs to the exercise, not to a session, because it is
  "my setting on this machine". Free text.
- **exercises[].maxSkip**: an integer or `null`. It is the maximum number of
  consecutive sessions the exercise may be skipped. `null` means no limit.
- **exercises[].archived**: an archived exercise is hidden from planning and
  from the exercise list (it appears in an "Archived" section there). It is
  never deleted, so history stays consistent.
- **exercises[].photo**: file name of the exercise's photo, or `null`. One photo
  per exercise, to recognise the machine later. The image itself is not in
  `gym.json` (see §6, Photos). Each new or replaced photo gets a new name
  (`<exercise-id>-<timestamp>.jpg`), so other devices notice the change.
- **sessions**: kept sorted by `date` ascending. `finished: false` marks the
  one in-progress workout. There is at most one, and it is stored in the file
  so it syncs between devices.
- **sets[].weight**: a number in kg, or `null` for bodyweight exercises.
  **sets[].reps**: an integer or `null`. **sets[].plus**: `true` means "more
  reps would have been possible".
- **items[].note**: free text per exercise per session. Optional; empty string
  allowed.

### 3.2 Derived values (computed, never stored)

- **Finished sessions**: sessions with `finished: true`, sorted by `date`, then
  by array order.
- **Sessions since** for an exercise: the number of finished sessions after
  the most recent finished session that contains the exercise. If it was in the
  last session, the value is 0. If it has never been done, the value is
  "never". **Every session counts as one session**, however many exercises it
  had. An exercise counts as done in a session if it appears in `items`.
- **Status** for an exercise with a `maxSkip`:
  - `due` if sessions since ≥ maxSkip. It must be done today, or the limit is
    exceeded.
  - `soon` if sessions since = maxSkip − 1.
  - Otherwise `ok`.
  - Exercises without `maxSkip` are always `ok`; `never` is its own state.
- **Last performance** of an exercise: the item from the most recent finished
  session that contains it. Used to prefill sets.

## 4. Screens

Navigation: a bottom tab bar with **Plan / Workout**, **Exercises** and
**Settings**. Plan and Workout share one tab: it shows Workout while a session
is in progress, otherwise Plan. A small sync-status indicator sits in the
header and is also a tap target (§6).

### 4.1 Plan (no workout in progress)

- One section per muscle group, in the fixed order. Each section lists its
  non-archived exercises in the user's order.
- Each exercise row shows:
  - the name,
  - "last session", "N sessions ago" or "never",
  - the limit if set ("max 3"),
  - a colour state: **due** = red, **soon** = amber, ok = neutral, never =
    muted.
  - If the exercise belongs to other groups too, also show them small (e.g.
    "also Back").
  - If it has a photo, a thumbnail next to the row. Tapping the thumbnail
    shows the photo; it does not toggle the selection.
- Tapping a row toggles its selection. An exercise in two groups is selected in
  both places at once.
- A per-group counter shows how many exercises are selected.
- A sticky button "Start workout (N)" creates the in-progress session with the
  selected exercises, in selection order.
- Header line: date of the last session and how many days ago it was.
- Empty catalog: show a hint pointing to the Exercises tab, plus a
  "Load starter exercises" button (§7).

### 4.2 Workout (session in progress)

- The session date is shown and editable (defaults to today).
- One card per selected exercise, containing:
  - The name, and a remove button (✕) that takes the exercise out of this
    session.
  - ▲/▼ buttons that move the card up or down in the workout. Logged sets
    and the note move with it.
  - The **setting** field, editable inline. Edits write to the exercise, not
    the session.
  - Next to it, the photo thumbnail (tap to view full size, with "Replace
    photo"), or a 📷 button to take one right at the machine.
  - The line "Last time (18.10.): 70×15 · 70×15 · 70×15+".
  - **Set rows**: `[weight] × [reps] [+] [✓]`.
    - Inputs are empty, with the last performance as **placeholders**: set i
      uses last time's set i, or last time's final set if there were fewer.
    - Tapping ✓ copies the placeholders into any empty field and marks the row
      as done. Tapping ✓ on a done row un-marks it and keeps the values. The
      common case "same as last time" is then a single tap per set.
    - [+] toggles the `plus` flag.
    - Typing a value directly also works; the row counts as done once it has a
      value.
    - The number of rows defaults to last time's set count, or 3 for a new
      exercise. "+ Set" and "− Set" buttons adjust it.
  - A **note** field (single line, expands if needed).
- An "+ Add exercise" button opens the Plan view in "add" mode. Exercises
  already in the workout are marked and cannot be selected again. Confirming
  appends the new selection.
- "Finish workout" button:
  - Drops set rows that are neither done nor have any value, then drops items
    left with no sets.
  - Sets `finished: true`, triggers a sync, and returns to Plan.
- "Discard workout" (needs a confirmation tap) deletes the in-progress session.
- **Everything autosaves** on every change. Closing the app mid-workout loses
  nothing.

### 4.3 Exercises

- The exercises are grouped by muscle group, in the user's order. Each row has
  ▲/▼ arrows that move it within that group's list.
- "+ New exercise" opens a form with:
  - Name (required),
  - Muscle groups (multi-select chips, at least one),
  - Max skip (number, optional),
  - Setting (text),
  - Photo (add / replace / remove; camera or gallery).
  New exercises are appended to the end of each selected group.
- Tapping an exercise opens the same form for editing. Changing its groups adds
  it to or removes it from those group lists. "Archive" / "Unarchive" are in
  the form as well.
- An "Archived (N)" section at the bottom is collapsed by default.

### 4.4 Settings

- Google Drive:
  - The connection status, a "Connect" / "Sync now" / "Disconnect" button, the
    time of the last sync, and the last error, if any.
- Data:
  - **Export JSON** downloads `gym-YYYY-MM-DD.json`.
  - **Import JSON** replaces local data after a confirmation tap, then syncs.
  - "Load starter exercises" (§7). It only adds exercises whose name does not
    exist yet.
- App version string.

## 5. Out of scope for v1

- Charts and progress graphs.
- Editing or deleting past sessions in the app. The user edits `gym.json`
  directly.
- History browsing beyond "last time" on the workout cards.
- Migrating the old spreadsheet data.
- Multiple users, sharing, accounts.
- Drop sets, supersets, rest timers, per-side values.

## 6. Storage and sync: Google Drive

- **Google Identity Services, token model** (`google.accounts.oauth2.initTokenClient`).
  There is no backend and no refresh tokens.
- Scope: **`https://www.googleapis.com/auth/drive.file`** only. The app can see
  only files it created itself.
- `GOOGLE_CLIENT_ID` is a constant at the top of the script. It is not a
  secret.
- Drive layout: a folder **"Gym Tracker"** in My Drive containing `gym.json`.
  The ids of the folder and the file are cached in `localStorage`. If the cached
  ids are gone (404), look up the folder and file again by name, and recreate
  them if missing.
- REST calls via `fetch` against `https://www.googleapis.com/drive/v3` and
  `https://www.googleapis.com/upload/drive/v3`:
  - download with `alt=media`,
  - update with `PATCH ...?uploadType=media`,
  - create with `uploadType=multipart`.
  Use `cache: "no-store"`.

**Local-first behaviour**

1. Every change is written to `localStorage` immediately and marks the data as
   "dirty".
2. If a valid token exists, the app pushes after a 3-second debounce. It also
   pushes on "Finish workout".
3. It pulls on app start and when the app becomes visible again
   (`visibilitychange`), if a valid token exists.
4. **Conflict detection**: remember the Drive file's `modifiedTime` from the
   last pull or push. Before pushing, fetch the metadata; if `modifiedTime`
   differs, the file changed elsewhere (e.g. a hand edit on the laptop).
   - Local data not dirty: take the remote copy silently.
   - Local data dirty: show a choice, "Keep this device's version" or "Use Drive
     version". There is a single user, so this is rare and should stay simple.
     No field-level merging.
5. **Daily backup**: before the first overwrite of `gym.json` on a given day,
   copy the current remote file to `gym-backup-YYYY-MM-DD.json` in the same
   folder (`files.copy`). Backups are never deleted automatically.

**Photos**

- A picked photo is scaled down to max 1024 px and stored as JPEG (quality
  0.8, about 100–150 KB).
- Locally, photos live in IndexedDB, keyed by file name. They work without
  Drive, but then only on that device.
- In Drive, each photo is a separate file in **"Gym Tracker/photos"**.
  New photos are uploaded during a push, before `gym.json` is written, so the
  file never names a photo that is not there yet. Replaced or removed photos
  are moved to the Drive trash after the write. Pending uploads and removals
  are queued in `localStorage` until the next push.
- A device that shows a photo it does not have downloads it from Drive and
  caches it in IndexedDB.
- Photos are not part of Export / Import JSON, and not covered by the daily
  backup.

**Tokens and login**

- Store the access token and its expiry in `localStorage`, so a reload within
  the hour does not ask again.
- Tokens last about 1 hour. When the token has expired, sync stops quietly and
  the indicator shows "tap to reconnect". **The interactive token request must
  happen inside a user gesture**, so preload the GIS script on startup; awaiting
  a script load inside the click handler gets the popup blocked.
  - Use `prompt: "consent"` only on the first connect, and `prompt: ""` after
    that.
- The app is fully usable without Drive, local-only. Drive is an add-on and the
  app must never block on it.

**Sync indicator states**

| State | Meaning |
|---|---|
| local only | Drive not configured or not connected |
| synced ✓ | No local changes since the last sync |
| unsynced ● | Local changes not yet pushed |
| syncing ⟳ | Sync in progress |
| reconnect ⚠ | Token expired, or the last sync failed |

Tapping the indicator runs an interactive sync.

## 7. Starter exercises (optional seed)

The user starts with fresh data, but his existing exercise list is a useful
starting point. "Load starter exercises" adds these. maxSkip is `null`, and the
setting is taken from his old sheet where one existed.

| Group(s) | Name | Setting |
|---|---|---|
| Shoulders | Lateral Raise Machine | Bottom 4, top 4 |
| Shoulders | DB Lateral Raise | |
| Shoulders | Shoulder Press | Top 5 |
| Shoulders, Back | Face Pulls | Very top; D = double, H = hammer |
| Shoulders, Back | Reverse Butterfly | Bottom 5 |
| Shoulders | Cable Crossover (rear) | Top 8 |
| Chest | Chest Press (B) | Bottom 5 |
| Chest | Cable Chest (high) | Top |
| Chest | Cable Chest (low) | From below |
| Chest | Butterfly | Top 1, bottom 8 |
| Chest | Incline Bench Press (G) | Bottom 8 |
| Chest | Decline Chest Machine | Middle, narrow grip |
| Chest | Chest Fixator (B) | ca. 5 |
| Chest | DB Incline Press | |
| Chest | Bench Press BB | Wide grip |
| Chest | Push Ups | |
| Chest, Triceps | Seated Dip | Bottom 8 |
| Back, Legs | Deadlift | Rack 12 |
| Back | Lat Pulldown | Exactly middle |
| Back | Low Row | |
| Back | Row Machine | Lower grip |
| Back | Gravitron | |
| Back | Lat Pulley (B) | Between 3 and 4, back upright |
| Back | Ground Base Combo Decline (H) | |
| Back | One-Arm Row | Exactly middle |
| Back | Back Extension | No hyper, very bottom |
| Back | Hex Shrugs | |
| Legs | Squat | Top 5, bottom 14 |
| Legs | Leg Press | |
| Legs | Lunges BB | |
| Legs | Hamstring Curl Seated | Back 1, front 4 |
| Legs | Glute Machine | Top 5 |
| Legs | Abduction (B) | |
| Legs | Adduction | |
| Legs | Leg Extension | |
| Legs | Calves | 6 |
| Legs | Hip Thrust Machine | |
| Biceps | Biceps Curl (DB/SZ) | |
| Biceps | Biceps Machine | Bottom 3 / G5 |
| Triceps | Triceps Cable / SZ | |
| Triceps | Triceps Rope (low) | |
| Abs | Abs | |
| Abs | Woodchoppers | Bottom 12 |

## 8. Files

```
index.html             the whole app (HTML + inline CSS + inline JS)
manifest.webmanifest   name "Gym Tracker", short_name "Gym", display standalone,
                       start_url "./", scope "./", theme/background colors, icons
sw.js                  caches the app shell; network-first for index.html so
                       deploys show up immediately; never caches googleapis.com
icon-192.png           app icon (also maskable)
icon-512.png
README.md              short: what it is, how to run locally, how to deploy
```

## 9. Acceptance checks

These are the things to verify before calling v1 done. Use Playwright with a
phone viewport (e.g. 390×844) against `python3 -m http.server 8000`, in
local-only mode.

1. Load the starter exercises, and the Plan shows 7 groups with all
   exercises marked "never".
2. Select 5 exercises across groups, including Face Pulls, which is selected
   in both Shoulders and Back. "Start workout (5)" creates the workout with 5
   cards.
3. Fill sets, using ✓ on an empty prefilled row, typing values, "12,5" with a
   comma, + flag, "+ Set" / "− Set", a note and a setting change. Reload the
   page and everything is still there.
4. Add an exercise mid-workout and remove one. Finish. Empty rows and items
   are dropped.
5. Start a second workout. Placeholders show last time's values, and ✓ fills
   them in. After finishing, Plan shows "last session" and "1 session ago".
6. With maxSkip = 1 on an exercise not done last session, it shows red "due".
   With maxSkip = 2, it shows amber "soon".
7. Reorder with ▲/▼, archive and unarchive. The order persists, and archived
   exercises disappear from Plan.
8. Export JSON, hand-edit it (e.g. change a weight), and import it again. The
   change shows up in "Last time".
9. No horizontal scrolling at 360 px width. Light and dark mode are both
   readable.

Drive sync cannot be tested automatically, because it needs the user's Google
account. Keep the Drive code isolated (one module/section with a small
interface: `connect()`, `pull()`, `push()`, `status`) so it can be reviewed
separately. The user tests it manually after deploying.
