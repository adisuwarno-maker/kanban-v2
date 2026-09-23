# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file IT project Kanban board (`index.html`) for an internal "UOB IT PMO" demo/training tool. All markup, CSS (`<style>`) and JS (`<script>`) live in that one file. There is no build, package manager, linter or test suite.

## Running

Open `index.html` directly in a browser (double-click / `file://`). No server is required. To verify changes, open it in the browser pane and exercise the UI or use `javascript_tool` to drive it (e.g. `taskForm.requestSubmit()`, dispatching `DragEvent`s on `.column` elements).

## Hard constraints (from the original spec — do not violate)

- Vanilla HTML/CSS/JS only: no frameworks, libraries, CDNs, web fonts, image files, bundlers or npm. Icons are Unicode glyphs or inline SVG; fonts use the system stack.
- Stays a single file.
- **No persistence**: no `localStorage`, `sessionStorage`, IndexedDB or cookies. A refresh resetting the board to the seed data is intended (the header shows a note saying so).
- No `alert()` / `confirm()`: validation errors are inline, and delete uses an inline "Delete? Yes / No" toggle rendered on the card.
- No `!important`; colours and spacing come from CSS custom properties in `:root`.
- Branding: neutral "UOB IT PMO" text wordmark and blue palette only. Never add real UOB logos or trademarks, and never imitate an official system.
- The only network call is FormSubmit, made via AJAX `fetch`, so the page never navigates away.

## Architecture (inside `<script>`)

- **Config at the top**: `FORMSUBMIT_ENDPOINT` is the single place the notification email address lives. `STATUSES`, `PRIORITIES`, `PROJECTS` and `CATEGORIES` drive the selects, validation and move menus. Column markup in the HTML is static and keyed by `data-status`, which must match `STATUSES`.
- **Single source of truth**: `state = { tasks, filters, ui }`. `ui.openMoveId` and `ui.pendingDeleteId` hold the transient per-card toggles (Move menu, delete confirm) so that they survive re-renders.
- **Render from state**: mutations (`addTask`, `moveTask`, `deleteTask`, filter changes, UI toggles) update `state` and then call `renderBoard()`. That function rebuilds every column's card list from `renderCard()` template strings and updates the counts and summary. Do not mutate card DOM directly. Because cards are re-created, handlers restore focus afterwards via `focusEl(cardSelector(id) + ...)`.
- **Escaping**: every user-supplied value inserted into HTML must go through `escapeHtml()`.
- **Events are delegated**: one click handler on `#board` dispatches on `data-action`, and drag-and-drop uses native HTML5 DnD with the task ID in `dataTransfer`. Escape and outside-click close the open toggles.
- **Dates** are local `YYYY-MM-DD` strings compared lexically (`todayISO()`, `toISODate()`). Avoid `toISOString()` for dates because of UTC off-by-one errors. Seed due dates are relative to today so the overdue badges stay live.
- **IDs**: `generateId()` produces `UOB-ITPM-####` from an incrementing counter. The 8 seed tasks use 0001–0008.
- **Add-task flow** (`handleTaskSubmit`): validate (`validateForm` → inline errors), then add the card optimistically, then call `notifyNewTask()` in `try/catch` while the submit button shows "Sending…". A failed call only shows a warning toast; the card stays on the board.
- **FormSubmit**: `notifyNewTask()` deliberately throws when the endpoint still contains the `YOUR_EMAIL@example.com` placeholder, so nothing is sent until it is configured. The first real submission triggers FormSubmit's one-time activation email, and nothing is delivered until that link is clicked.

## Layout breakpoints

- ≥1280px: form in a sticky left sidebar.
- <1280px: form stacks above the board.
- <768px: the four board columns stack vertically.
