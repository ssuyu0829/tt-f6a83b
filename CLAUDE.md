# CLAUDE.md

Guidance for AI assistants working in this repository.

## What this is

A single-page **study-time dashboard** — a zero-dependency, installable PWA that reads a
CSV of time-tracking events and renders weekly study statistics. The UI is entirely in
Traditional Chinese (`zh-Hant`) and is designed for a phone screen (`max-width: 560px`),
styled to look like grid paper marked up with blue pen, red pen, and a highlighter.

**There is no build system, no package manager, no test suite, and no dependencies.**
Everything is hand-written HTML/CSS/JS in one file. Do not introduce npm, a bundler, or a
framework unless explicitly asked — the whole point of this repo is that the deploy artifact
*is* the source.

## Files

| File | Role |
| --- | --- |
| `index.html` | The entire application — markup, CSS, and JS in one file (~350 lines). |
| `data.csv` | **Generated data.** Append-only event log written by an external sync script. |
| `sw.js` | Service worker: cache-first for the app shell, network-first for `data.csv`. |
| `manifest.webmanifest` | PWA manifest (standalone display, paper-white theme `#FAFAF6`). |
| `icon-192.png`, `icon-512.png` | App icons referenced by the manifest and `apple-touch-icon`. |
| `.gitignore` | Ignores `sync/`, `README.md`, `*.zip`, `*.tmp`. |

Note that `README.md` and `sync/` are **deliberately gitignored** — do not add a README or
commit anything under `sync/` without being asked.

Every path in the app is relative (`./index.html`, `data.csv`, `sw.js`), so the repo root is
served as-is by any static host (GitHub Pages style). Adding a subdirectory or absolute path
will break both the SW scope and offline mode.

## Data pipeline

`data.csv` is produced **outside this repository** by a sync script running on the owner's
Mac (the `sync/` directory is gitignored). That script appends rows and commits with the
message `sync YYYY-MM-DD HH:MM`. 49 of the 50 commits in history are exactly that; only the
initial commit touched application code.

**Never hand-edit, reorder, or reformat `data.csv`.** Treat it as an external input. If you
need test data, work on a scratch copy outside the repo.

### CSV schema

```
timestamp,action,source,category,detail
2026-08-20T17:30:00,start,telegram,生活,
```

- `timestamp` — naive local ISO 8601, **no timezone suffix**. Parsed with `new Date(...)`,
  so it is interpreted in the viewer's local timezone. All week/day math is local-time.
- `action` — `start` or `end`.
- `source` — where the event came from: `ipad_notability`, `mac_tracker`, `manual_add`,
  `telegram`, `gcal`, `iphone_shortcut`, `claude_backfill`. Informational only; the app
  never reads this field.
- `category` — Chinese activity label. Observed values: `讀書`, `實驗室`, `通勤`, `家教`,
  `睡眠`, `運動`, `生活`, `女友`, `家事`, `專案`, `用餐`, `休息`, `休閒`, `娛樂`.
  A legacy lowercase `study` value exists and is aliased to `讀書` at parse time.
- `detail` — free text, usually empty. Some rows contain only a space.

Rows are **not** guaranteed to be paired or chronologically sorted; the app sorts and
reconciles them at load time.

## How the app works

`load()` → `parseCSV()` → `buildSessions()` → `render()` → `drawWeekly()`

1. **`parseCSV(text)`** — splits on newlines and naive `,`. Skips the header, blank lines,
   rows with fewer than 4 fields, and unparseable timestamps. Normalizes `study` → `讀書`,
   defaults an empty category to `其他`. Sorts ascending by time.
2. **`buildSessions(rows)`** — pairs `start`/`end` per category using an `open{}` map:
   - A second `start` for a category that is already open **replaces** the earlier one
     (later start wins).
   - An `end` with no matching open `start` is silently dropped.
   - Sessions of `<= 0` minutes or `> MAX_SESSION_MIN` (8 hours) are discarded as bad data.
   - Categories are tracked independently, so overlapping activities are fine.
   - Currently this drops ~2 unmatched `end` rows out of ~868; that is expected, not a bug.
3. **`render(sessions)`** draws four regions, all derived from the same session list:
   - **Hero number** — hours of `讀書` this week, plus the delta vs. last week and a
     `正`-character tally (one stroke per study session, via `tally()`).
   - **`drawWeekly()`** — hand-built SVG bar chart of the last 8 weeks of `讀書` hours,
     current week in red. No charting library; the SVG string is assembled inline.
   - **Category bars** — *all* categories for the current week, sorted descending, widths
     relative to the largest. `讀書` is highlighted in red.
   - **Recent list** — last 12 sessions, newest first.

Only `讀書` feeds the hero number and the weekly chart. Every other category shows up only
in the current-week allocation bars.

### Week boundaries

`weekStart(d)` snaps to **Monday 00:00 local time** (`(getDay()+6)%7`). Keep this convention
for any new time bucketing.

### Client state

The only persisted state is `localStorage['examDate']` (a `YYYY-MM-DD` string), set through a
`prompt()` on the countdown bar and used to render a days-remaining number. All
`localStorage` access is wrapped in `try/catch` because private-mode Safari throws — keep
that guard on any new storage access.

## Service worker — the main deployment gotcha

`sw.js` opens a cache named by the `VERSION` constant (currently `'tt-v1'`) and precaches the
shell: `./`, `index.html`, `manifest.webmanifest`, and both icons. Activation deletes every
cache whose key is not `VERSION`.

**If you change `index.html`, `manifest.webmanifest`, `sw.js`, or the icons, bump `VERSION`
in `sw.js` in the same commit** (`tt-v1` → `tt-v2` → …). The shell is served cache-first, so
without a bump, installed clients keep the old HTML indefinitely.

`data.csv` is the exception: it is network-first with a cache fallback, so fresh data lands
without a version bump. `index.html` also requests it with `?t=<now>` and `cache: 'no-store'`.
The SW matches on `pathname.endsWith('data.csv')`, so the cache-buster does not bypass it.

Google Fonts are loaded from the CDN and are **not** in the precache list, so a first-ever
offline load falls back to system fonts. That is intentional; the `--serif`/`--sans`/`--mono`
stacks all name a fallback.

## Conventions

- **Language.** UI strings *and* source comments are Traditional Chinese. Match that when
  editing; do not translate existing strings to English.
- **Styling.** Colors go through the CSS custom properties on `:root` (`--paper`, `--ink`,
  `--red`, `--hilite`, `--line`, …). The three-font system is deliberate: serif
  (Noto Serif TC) for headings and big numbers, sans (Noto Sans TC) for body, mono
  (IBM Plex Mono) for timestamps and numeric values. Do not hardcode hex values.
- **DOM access** goes through the `$ = id => document.getElementById(id)` helper.
- **Rendering** is full-innerHTML replacement per region, recomputed from `sessions`. There
  is no incremental update path and no state object — keep it that way.
- **Accessibility.** The countdown is a keyboard-operable `role="button"`, the chart carries
  `role="img"` with an `aria-label`, and the highlighter swipe is disabled under
  `prefers-reduced-motion`. Preserve these when touching those elements.
- **Mobile-first.** Layout is capped at 560px and uses `env(safe-area-inset-*)` padding for
  iOS standalone mode.

## Known sharp edges

- **CSV parsing is naive.** `line.split(',')` means a comma inside `detail` shifts fields.
  If richer detail text becomes a requirement, the parser needs quote handling.
- **No HTML escaping.** Category and `detail` values are interpolated straight into
  `innerHTML`. The data is self-authored so the risk is theoretical today, but escape any
  newly rendered field rather than widening the surface.
- **Silent data loss.** Rows dropped by `buildSessions()` never surface in the UI; the only
  hint is the `共 N 筆有效紀錄` footer count. When debugging "missing hours", compare that
  count against `wc -l data.csv`.
- **Local-time timestamps.** Viewing the dashboard in a different timezone shifts every
  session and can move it across a week boundary.

## Working locally

The app needs a real HTTP origin — service workers and `fetch('data.csv')` do not work from
`file://`:

```sh
python3 -m http.server 8000    # then open http://localhost:8000
```

When iterating on the UI, unregister the service worker (or hard-reload with "Update on
reload" checked in DevTools → Application) so you are not testing a cached shell. Test at a
narrow viewport — the desktop layout is not a design target.

There are no linters, formatters, or tests to run. Verification means loading the page and
checking that the hero number, chart, category bars, and recent list all render with no
message in `#err`.

## Git

- Work on the branch you were assigned; never push to `main` directly.
- `sync ...` commits belong to the automated data sync. Code changes get their own commits
  with real descriptive messages — do not mimic the `sync` format.
- Do not amend, rebase, or reorder existing `sync` commits.
- If a code change and a `data.csv` update land together, still keep the code change in its
  own commit.
