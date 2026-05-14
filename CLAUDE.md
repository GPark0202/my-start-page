# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Personal browser start page (new tab replacement). UI is Korean (`lang="ko"`).

## Architecture

**Single-file app.** The entire project is one file: `index.html` (~26KB). HTML, CSS, and JavaScript are all inline. There is no build step, bundler, package manager, test suite, or framework.

**To run / preview:** open `index.html` directly in a browser, or serve the directory with any static server (e.g. `python -m http.server`). No install step.

**To deploy a change:** edit `index.html`, commit, push. There is no CI.

## State model

A single object — `{shortcuts, quests, todos, lastdate}` — is the source of truth. It is written to **both** `localStorage['hp_data']` **and** Firebase Realtime Database at `users/<uid>` on every mutation via `saveData()`.

- Logged out: localStorage only (`loadLocal()` on init).
- Logged in: `dbRef.on('value', loadSnap)` subscribes to remote changes, which overwrite local state and trigger `renderAll()`. Remote is authoritative when signed in.

Every mutation function (`addQuest`, `completeQuest`, `addTodo`, `toggleTodo`, `deleteTodo`, `saveShortcutEdit`) must call `saveData()` then the relevant `render*()`. Don't mutate state without `saveData()` — the change will be lost on next snapshot.

Firebase config in the source is the standard web-side public config (apiKey, projectId, etc.) — it's not a secret and is safe to commit. Access control is enforced by Firebase Database Rules, not by hiding this config.

## Conventions worth knowing

- **KST timezone is hardcoded.** `getKSTDateString()` adds 9h offset and slices to `YYYY-MM-DD`. Any date logic (D-Day, daily todo reset) must go through this — don't use raw `new Date()` for date comparisons.
- **Daily todo reset:** `checkReset(lastdate)` clears `done` flags when the KST date changes. A 60-second `setInterval` re-checks this so the page doesn't need a reload at midnight.
- **Shortcut grid is 10 fixed slots**, not a dynamic list. `DEFAULT_SHORTCUTS` is the seed. Right-click on a shortcut opens the edit modal (`contextmenu` handler); there is no add/delete.
- **`__CAL__` sentinel:** if a shortcut's `icon` field equals the string `'__CAL__'`, `calIcon()` generates a live canvas favicon showing today's date number. Preserve this when touching `renderShortcuts()`.
- **Favicons** come from `https://www.google.com/s2/favicons?domain=<host>&sz=64`. New shortcuts auto-derive this from the URL hostname in `saveShortcutEdit()`.
- **XSS:** any user-entered string rendered inside `innerHTML` (quest text, todo text) must go through `escapeHtml()`. Static template strings don't need it.
- **Quest sort order:** priority ascending (P1 first), then due date ascending, nulls last. Applied in `renderQuests()` — do not pre-sort the stored array.
