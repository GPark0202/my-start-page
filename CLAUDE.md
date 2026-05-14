# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Personal browser start page (new tab replacement). UI is Korean (`lang="ko"`). Desktop-only — no mobile breakpoints, drag-and-drop assumes mouse.

## Architecture

**Single-file app + 2 static JSON.** Entirety is `index.html` (~40KB inline HTML/CSS/JS) plus `data/shortcuts.json` and `data/bookmarks.json` fetched at load. No build, no bundler, no framework.

**Deploy target: GitHub Pages.** Page must be served over HTTP/HTTPS for `fetch()` to work — `file://` double-click does NOT work (CORS blocks fetching the JSON files; the page silently renders as empty).

**To run locally:** `python -m http.server 8000` from the repo root, then `http://localhost:8000`. The OAuth Client's authorized origins must include the local origin (currently `http://localhost:8000`) and `https://lyb2106.github.io`.

## Two storage layers

| Layer | Storage | Lifetime | Edited by |
|---|---|---|---|
| Static | `data/*.json` in git | Permanent in repo | Manual edit + commit |
| Dynamic | Google Drive `appDataFolder` (per-user, hidden) + localStorage cache | Per-user | Page UI |

**Static**: bookmark pills, shortcut grid. To change, edit the JSON, commit, push. No UI for editing — the right-click edit modal was removed.

**Dynamic**: Daily Job, Brain Dump, Time Plan. Read/written via Drive REST API after Google login.

Drive files (all in `appDataFolder` — invisible in user's Drive UI, app-private):
- `dailyJob.json` — `{items:[{id,text,done}], lastResetDate}`. Resets at 06:00 KST (full clear, not just unchecking).
- `brainDump.json` — `[{id, category, name, createdAt}]`. `createdAt` is immutable after creation.
- `timeplan-YYYY-MM-DD.json` — `[{id, start, end, text, category, sourceType}]`. One file per day. `sourceType ∈ {'brainDump','dailyJob','manual'}` indicates origin of the block.

## State → persistence → render flow

Every mutation calls its save function (`saveDailyJob`, `saveBrainDump`, `saveTimeplan(date)`) then re-renders. Save = synchronous localStorage write (`cacheSet`) + 300ms-debounced Drive PUT (`scheduleSave` → `driveSave`). When logged out, Drive PUT is skipped; localStorage still updates.

Load on init:
1. `cacheGet` for `dailyJob`/`brainDump`/`tp:<currentDate>` → paint immediately (cache-first).
2. If session token valid, `loadAllFromDrive()` fetches latest from Drive → may overwrite cache → re-render.

When date changes via nav (prev/next/today/picker), `onDateChange()` paints from cache for that date, then `loadTimeplan(date)` syncs from Drive.

## Auth

Google Identity Services token client with scope `openid email profile https://www.googleapis.com/auth/drive.appdata`. Access token kept in `localStorage` (`drive_token`, `drive_token_expiry`, `drive_user`) so a single login persists across tabs and browser restarts. On 401 from Drive, `refreshToken()` does one silent retry via `requestAccessToken({prompt:''})`. On page load, if a stored `drive_user` exists but the token is missing/expired, `onGsiLoad` runs the same silent refresh before showing the login button — so a returning user auto-reconnects without UI.

`CLIENT_ID` constant is the OAuth Web Client public ID — safe to commit because origin restriction (`https://lyb2106.github.io` + `http://localhost:8000`) enforces security. To rotate, regenerate in GCP project `my-start-page-a48c0` and update the constant.

Logged-out state: page works read-only against localStorage cache. Edits still mutate cache but do not sync. On next login, Drive content **overwrites** local — local-only edits are lost.

## Conventions worth knowing

- **"Today" rolls over at 06:00 KST**, not midnight. `getPlannerDate()` returns the active planner day; Daily Job reset and `tpToday()` both use this. Don't use raw `new Date()` or `getKSTDateString` (removed) for planner-day comparisons.
- **Time grid:** 06:00 – 24:00, 30-min slots → 36 rows × 30px. Block layout: `top = (startMin - 360) / 30 × 30px`, `height = (endMin - startMin) / 30 × 30px`. Constants in code: `SLOT_MIN=30`, `SLOT_PX=30`, `HOUR_START=6`, `HOUR_END=24`, `SLOT_COUNT=36`.
- **`__CAL__` sentinel:** if a shortcut's `icon` field equals the string `'__CAL__'`, `calIcon()` generates a live canvas favicon showing today's calendar date number. Preserve this when touching `renderShortcuts()`.
- **Favicons** in `shortcuts.json` use `https://www.google.com/s2/favicons?domain=<host>&sz=64` — set manually when adding a shortcut.
- **XSS:** any user-entered string rendered via `innerHTML` (Daily Job text, Brain Dump name, block text) must go through `escapeHtml()`. Static template strings and OAuth-provided fields (user name, picture URL) are trusted.
- **Drag-and-drop moves the source.** Dragging Brain Dump or Daily Job item into Time Plan creates a new block AND deletes the source from its panel in a single action (`saveBrainDump`/`saveDailyJob` fire alongside `saveTimeplan`). `sourceType` field records origin but is informational only — the source no longer exists in the panel. No reverse drag (Time Plan → panel).
- **Block resize and drop snap to 30-min grid.** `Math.round(deltaY / SLOT_PX)`; never sub-slot precision.
- **Brain Dump's `createdAt` is immutable.** `bdSave` only writes `name` and `category` on edit — never overwrites `createdAt`.
- **Block overlap is allowed** but renders stacked (later block on top by z-index, hover raises). No side-by-side cluster layout.
- **`id` generation:** `uid()` = `Date.now().toString(36) + 6 random chars`. Avoids the collision risk of plain `Date.now()` in fast successive adds.
- **Shortcuts/bookmarks length is not enforced.** `shortcuts.json` may contain any number; UI just renders what's there. The original "10 fixed slots" rule was dropped along with the edit modal.
