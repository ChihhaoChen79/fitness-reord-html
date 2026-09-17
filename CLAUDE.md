# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A zh-TW fitness workout tracker delivered as a single self-contained `index.html` (no backend, no build step, no npm). All data lives in the browser's IndexedDB/localStorage — there is no sync, so exported/imported JSON backups are the only durable copy. See `README.md` for the full user-facing feature list and deployment instructions (GitHub Pages + "Add to Home Screen" on iOS Safari is the primary target).

## Commands

There is no package.json, build step, linter, or test suite — this is a static site. To develop:

```bash
python3 -m http.server 8934   # serve the repo root
# open http://localhost:8934/index.html
```

There is no automated test suite. Verify changes manually in a browser (or drive it with Playwright against the local server above) — click through: add a custom exercise, build a routine, run a live session, run a backfill session, check stats charts render, export/import a backup. Watch the browser console for errors; there should be none.

## Architecture

Everything lives in `index.html` inside one IIFE. Two files ship in total:
- `index.html` — markup, CSS, and all JS
- `vendor/chart.umd.js` — Chart.js, vendored locally on purpose (no CDN scripts). Tailwind was likewise replaced by a small hand-written utility-class layer in the `<style>` block. **Do not reintroduce CDN `<script>`/`<link>` dependencies** — the app must keep working fully offline once loaded, since it's meant to be added to the iOS home screen and used at the gym.

### Data layer
- IndexedDB (`fitnessTrackerDB`) has five object stores, all keyed by `id`: `exercises`, `templates`, `records`, `bodyMetrics`, `tags`. Thin promise wrappers (`dbGet`, `dbGetAll`, `dbPut`, `dbBulkPut`, `dbDelete`, `dbClear`) wrap the raw IDB calls — always go through these rather than opening transactions directly.
- `localStorage` holds only two things: app settings (`ft_settings`) and the in-progress workout draft (`ft_draft_session`, autosaved via the debounced `persistDraft()` so a session survives a browser reload).
- On first load, `seedIfEmpty()` inserts the built-in exercise library (`buildSeedExercises()`, ~67 exercises) and default tags if the stores are empty.
- Records/templates snapshot exercise names at save time (`exerciseNameSnapshot`, `templateNameSnapshot`) so renaming/deleting an exercise or template later never changes historical records.

### App state and rendering
- `State` (module-level, **not** exposed on `window`) is the single source of truth: cached DB reads (`State.exercises`, `State.templates`, `State.records`, `State.bodyMetrics`, `State.tags`), per-tab UI state (`State.exercisesView`, `State.templatesView`, `State.historyView`, `State.statsView`), and `State.session` for the active/in-progress workout.
- `App` (exposed as `window.App` so inline `onclick="App.foo(...)"` handlers work) holds every action/handler. There is no framework — `App.render()` picks a `render<Tab>Tab()` function by `State.tab` and replaces `#app.innerHTML` wholesale; `State.session` truthy overrides all tabs and renders the full-screen workout session view instead.
- Reload the DB cache after any write with `await App.reloadAll()` before re-rendering, so `State.*` stays consistent with IndexedDB.

### Important gotcha: full re-render vs. text inputs
Because `App.render()` replaces the whole tab's `innerHTML`, any text/number input whose value is only read from the DOM at submit time will be **silently wiped** if something else (e.g. opening the exercise picker modal) triggers a full re-render while the user is mid-typing. This bit us once with the template-name field. The fix, and the pattern to follow for any new input: give it an `oninput`/`onchange` handler that writes straight into `State` on every change (e.g. `App.tplUpdateName`, `App.sessUpdateSet`, `App.sessUpdateOverallNotes`), and render the input's `value` from that same state. Never rely on reading `document.getElementById(...).value` only at save time.

### Units
Internally everything is stored metric (kg / km). `State.settings.unit` (`metric`/`imperial`) only affects display: `toDisplayWeight`/`toDisplayDistance` convert for rendering, `toStoreWeight`/`toStoreDistance` convert user input back to metric before it's saved. Never store imperial values.

### Workout sessions (live vs. backfill)
`State.session` represents both a live, "started now" workout and a backfilled past workout — they share one render/save path (`renderSession()`, `App.finishSession()`), branching on `session.isBackfill`:
- Live sessions get a running `#sessionTimer` (via `App.startSessionTimer()` / `attachSessionTimerDisplay()`), and `finishSession()` uses `nowISO()` as `endTime`.
- Backfill sessions (started from the History tab's "+ 補記錄" button, via `App.startBackfillFromTemplate`/`App.startBackfillBlank`) instead show editable date/time/duration fields (`App.sessUpdateBackfillDate/Time/Duration`), and `finishSession()` derives `endTime` from `startTime + backfillDurationMinutes`.

When finishing a session started from a template, `templateDiffersFromSession()` decides whether to prompt "update the routine?"; when finishing a blank session, the user is prompted to optionally save it as a new template. Both prompts reuse `App.showConfirm`/`App.showModalHtml`.

### Security
User-supplied strings (exercise names, notes, tags, custom exercise descriptions, etc.) are rendered into `innerHTML` template strings — always pass them through `escapeHtml()` before interpolating. IDs (UUIDs from `uid()`) are safe to interpolate into inline `onclick="App.foo('${id}')"` strings unescaped since they never contain user input.

## Git workflow

This repo has a single branch, `main` — commit and push directly to it (no PR-based workflow is in use here).
