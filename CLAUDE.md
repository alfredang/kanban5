# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file IT PMO Kanban board (`index.html`) for a **fictitious** bank's internal demo/training use. All markup, the `<style>` block and the `<script>` block live in that one file.

## Hard constraints (from the original brief — keep them)

- Vanilla HTML/CSS/JS only: no frameworks, libraries, build step, bundler or npm.
- Must run by double-clicking the file (`file://`); no server.
- No external resources: no CDNs, web fonts or image files. Use the system font stack and inline SVG/Unicode icons.
- **No persistence.** Do not use localStorage, sessionStorage, IndexedDB or cookies. A refresh resets to the seed data by design, and the header note says so.
- The only network call is FormSubmit's AJAX endpoint (`FORMSUBMIT_ENDPOINT`, the first constant in the script). It stays a placeholder (`YOUR_EMAIL@example.com`) unless the user supplies an address.
- **Neutral branding:** no real bank names, logos or trademarks. Task IDs use the `ITPM-####` format and email subjects start with `[IT PMO]`.
- No `alert()`/`confirm()` dialogs. Deletion is confirmed inline in the card, and form errors appear inline under each field.
- No `!important`. Colours and spacing come from CSS custom properties on `:root`.

## Running / checking

There is no build, lint or test tooling.
- Open the app: `start index.html` (Git Bash) or `Start-Process index.html` (PowerShell).
- Syntax-check the script block:
  `node -e "const h=require('fs').readFileSync('index.html','utf8');new Function(h.match(/<script>([\s\S]*?)<\/script>/)[1]);console.log('ok')"`
- Constraint scan (should match only the FormSubmit URL):
  `grep -nE 'localStorage|sessionStorage|indexedDB|document\.cookie|alert\(|confirm\(|!important|UOB|https?://' index.html`
- Headless screenshot:
  `"/c/Program Files (x86)/Microsoft/Edge/Application/msedge.exe" --headless=new --window-size=1440,1100 --screenshot=out.png "file:///<abs path>/index.html"`
  Headless Edge clamps very narrow windows, so check the <768px stacked layout at around 700px wide.

## Architecture (script block)

- **One-way data flow.** `state = { tasks, filters }` is the single source of truth. A small `ui` object holds view-only state: `openMoveId`, `pendingDeleteId` and `focusAfterRender`.
- **Actions** (`addTask`, `moveTask`, `deleteTask`) mutate `state`, then call `renderBoard()`. Nothing else should touch card DOM.
- **Rendering.**
  - `renderBoard()` runs `applyFilters()`, rebuilds each column's `innerHTML` from `renderCard()` strings, updates the column counts and calls `renderSummary()`.
  - Column counts reflect the filtered tasks; the summary strip counts all tasks.
  - It then restores keyboard focus via `ui.focusAfterRender`, a CSS selector string.
- **Escaping.** Every task field interpolated into `renderCard()` must go through `escapeHtml()`. Toasts are built with `textContent`.
- **Events.** Event delegation on `#board` handles:
  - HTML5 drag-and-drop: card `data-id` goes in `dataTransfer`, and the column's `data-status` is the drop target.
  - Clicks on `button[data-action]`: `move-toggle`, `move-to`, `delete-ask`, `delete-yes`, `delete-no`.
- **Statuses.** Status keys (`backlog`, `inprogress`, `blocked`, `done`) are tied to element IDs: `list-<key>`, `count-<key>`, `sum-<key>` and `col-<key>-title`. Changing `STATUSES` means updating the markup too.
- **Dates.** Dates are local `YYYY-MM-DD` strings compared lexically (`todayISO()`, `isOverdue()`). Seed due dates are relative to today so the overdue examples always show.
- **Add-task flow.**
  1. Validate.
  2. Add the task optimistically and reset the form.
  3. Disable submit with "Sending…" while `notifyNewTask()` runs.
  4. On failure, show a warning toast and keep the card.
  5. Close the modal.

  `notifyNewTask()` also treats a 200 response with `success: "false"` (for example, an unactivated FormSubmit address) as a failure.
