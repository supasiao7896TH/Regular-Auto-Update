# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository layout

This working directory is **not itself a git repository**. The actual git repo (remote: `github.com/supasiao7896TH/Regular-Auto-Update`) lives in the `Regular-Auto-Update/` subfolder. All git commands (status, add, commit, push) must be run from inside that subfolder.

- `Regular-Auto-Update/` — the git repo; this is "the codebase"
  - `Regular_auto_update.html` — original/base app, "CTA & PTA No.1 Plant"
  - `Regular_auto_update_PTA_No2.html` — fork for "GCM PTA No.2 Plant"
  - `Regular_auto_update_PTA_No3.html` — fork for "PTA No.3 Plant"
  - `README.md` — user-facing usage doc (Thai)
- `Regular_auto_update_PTA_No2.html` (loose copy at this root level) — a working-copy mirror of the repo's file of the same name, kept in sync manually. When editing this app, edit here first, verify, then copy over `Regular-Auto-Update/Regular_auto_update_PTA_No2.html` before committing. Don't let the two drift — always diff/sync before pushing.
- `Regular Work  Monthly CTA2  PTA2_-2.xls` — source data reference for the No.2 plant's task list; not needed to run any app.

## What this is

Three independent, self-contained, single-file HTML web apps (no build step, no backend, no package.json) that generate a monthly regular-maintenance/inspection schedule for GC-M PTA plant departments — a table of tasks vs. days-of-month with recurring "done" marks, exportable to Excel/print/PDF/image.

**The 3 files are forks of the same codebase and share byte-identical module structure and function names.** They differ only in: branding strings (plant name, title, CC line), `APP_CONFIG.DB_NAME`, and the `DEFAULT_TASKS` data array (task list). When fixing a bug or adding a feature, the same edit almost always applies verbatim to all 3 files — check this before assuming per-file logic needs to be re-derived. Line numbers differ slightly between files because `DEFAULT_TASKS` has a different length in each.

**This is not the same 9-module architecture as this user's general Vibe Coding standard** (no Firebase, no `CLOUD_SYNC_MANAGER`, no `AUTH_PROVIDER`, no `GEMINI_AI_BRIDGE`). Persistence is 100% local via IndexedDB. Don't assume those modules exist here.

## Commands

No build/lint/test tooling exists. Everything is manual:

- **Syntax-check after editing**: extract the inline `<script>...</script>` block and run it through Node — this catches JS syntax errors without a browser:
  ```
  node -e "const fs=require('fs'); const html=fs.readFileSync('Regular_auto_update_PTA_No2.html','utf8'); fs.writeFileSync('_check.js', html.match(/<script>([\s\S]*)<\/script>/)[1]);"
  node --check _check.js
  ```
- **Run/test in a browser**: these apps must be served over `http://` (a local static server), not opened via `file://` — several features (`navigator.clipboard.write`, and generally the IndexedDB persistence path) behave differently or are unavailable under `file://`. A one-off Node static server is enough:
  ```
  node -e "require('http').createServer((req,res)=>{const fs=require('fs');fs.readFile('.'+decodeURIComponent(req.url==='/'?'/Regular_auto_update_PTA_No2.html':req.url),(e,d)=>{if(e){res.writeHead(404);res.end();return;}res.writeHead(200,{'Content-Type':'text/html'});res.end(d);});}).listen(PORT)"
  ```
  Note: `file://` mode is still a real, supported usage mode for end users (double-click to open) — the app detects it (`window.location.protocol === 'file:'`) and falls back to an in-memory store instead of IndexedDB (see `STORAGE_ENGINE`). Just don't rely on that mode for testing browser-API-dependent features during development.
- **No package manager** — all third-party code loads from CDN with pinned versions + Subresource Integrity hashes in `<head>` (Tailwind CSS, html2canvas 1.4.1). Do not add a dependency without a verified `integrity` hash — fetch the file and compute/cross-check it (e.g. against cdnjs's published SRI API) rather than guessing one.

## Architecture (per file, ~5900-6000 lines)

Static HTML/CSS in the first ~3300 lines (header, toolbar, modals, print CSS), followed by a single `<script>` block containing 9 IIFE modules in this order, each exposed as a `const MODULE_NAME = (() => { ...; return {...}; })();`:

1. **`APP_CONFIG`** — `DB_NAME`, `DB_VERSION` (2), month/day label arrays, `DEFAULT_TASKS` (the task list — array of `{id, no, section, name, shift, freqType, freqValue, freqLabel, lastDate, scheduleAnchor, scheduleNote}` plus optional `scheduleCustomDays`/`scheduleAnchor2`/`scheduleAnchorNth`/`scheduleMode`/`scheduleBiannualMonths`), and (No.2 only) `SECTION_REMARKS` — free-text footnotes keyed by section name, rendered right after that section's rows in Excel/print/PDF output.
2. **`STATE_STORE`** — plain pub/sub state container.
3. **`STORAGE_ENGINE`** — IndexedDB CRUD across 3 object stores (`tasks`, `settings`, `scheduleOverrides`, all keyed by `id`/`key`). Falls back to an in-memory object when `window.location.protocol === 'file:'` (IndexedDB is unreliable/restricted from `file://`).
4. **`FREQ_CALC`** (comment label "FREQUENCY_CALCULATOR") — `calcScheduledDays(task, year, month)` computes which days of a given month get auto-marked, purely from `task.freqType`/`freqValue`/`scheduleAnchor`(s)/`scheduleCustomDays` — never from hardcoded dates, so it works for any month/year. `freqType` vocabulary: `daily`/`shift` (every day), `days` (every N days), `weekly` (one weekday), `biweekly` (two weekdays), `monthly` (fixed date or nth-weekday), `bimonthly` (two dates, `anchor`/`anchor+15`, or nth-weekday), `customDays` (explicit day list — the fallback for patterns that don't fit any formula), `dcs`/`batch`/`special` (no auto-mark; shows `task.scheduleNote` text instead, for DCS-timer-driven or non-calendar tasks), `quarterly`/`biannual`.
5. **`UTILS`** — small helpers (`escapeHtml`, etc.); aliased to bare top-level names (e.g. `const escapeHtml = UTILS.escapeHtml;`) right after the module, so most call sites use the bare name, not `UTILS.foo`.
6. **`UI_RENDERER`** — largest module: main table render, all modals (task manager, add/edit task, image-preview, in-app manual/help — `showManual()`), toasts (`showToast`). Sections in the on-screen table (and in Excel/print/PDF) are derived dynamically from the data — `[...new Set(tasks.map(t => t.section))]` — never hardcoded, so the section list adapts automatically to whatever `section` strings appear in `DEFAULT_TASKS`.
7. **`DEBUG_MODULE`** — `log(level, msg, data)` console wrapper, `[GCM]`-prefixed.
8. **`EXPORT_ENGINE`** — `exportJSON`/`importJSON` (full data backup/restore) and `saveStandaloneHTML` (re-serializes the current app + data into a new downloadable `.html` file, embedding data via `window.__GCM_EMBEDDED__` for the `file://` fallback path to pick up on first load). Excel export (`exportExcel`) existed here originally but was removed from all 3 files as unused — don't re-add it without being asked.
9. **`APP_CORE`** — month navigation, `generate()` (computes the displayed schedule = `FREQ_CALC` base schedule with any per-day manual overrides from `scheduleOverrides` applied on top), `buildPrintDoc()` (builds the full print/PDF HTML document as a string — shared by `printSchedule()` and `exportImage()`, so both stay visually identical to the on-screen/Excel data), `exportImage()` (renders `buildPrintDoc()`'s HTML into a hidden iframe, rasterizes it with `html2canvas`, and writes the PNG straight to the clipboard via `navigator.clipboard.write()` so the user can just Ctrl+V into an email — falls back to a preview modal with manual right-click-to-copy if the Clipboard API is unavailable or rejected), `init()`.

## Conventions specific to this codebase

- **Inline styles are the deliberate convention**, not an oversight — everything is one HTML file with no external CSS, so `style="..."` attributes are used throughout intentionally. Don't "clean up" by extracting to a stylesheet; that breaks the single-file distribution model these apps are built around (double-click to open, or drop on any static host).
- **Line endings are inconsistent between files** (`Regular_auto_update.html` is CRLF; `Regular_auto_update_PTA_No2.html` and `_No3.html` are LF) — a leftover from how each was originally created/edited. When scripting a text edit to apply across multiple files (e.g. via Node), normalize to `\n` for matching and only restore `\r\n` for files that originally had it, or a naive string-replace will silently fail on whichever files don't match your assumed line ending.
- **`buildPrintDoc()` is the single source of truth for anything print/PDF/image-shaped.** If a print/PDF/exported-image bug is reported, the fix almost always belongs inside this one function (and its embedded `<style>` string), not in the on-screen `UI_RENDERER` table renderer — the two are visually similar but are two entirely separate render paths.
- **A3 portrait page-fit is computed, not fixed**: the print template calculates row height (`rowH`) as `usableBodyH / rowCount`, where `rowCount` must equal the exact number of `<tr>` rows the template emits (section headers + task rows + any `SECTION_REMARKS` rows, including their own header row). If you add a new kind of row to the print table, you must also add it to this count, or the table silently overflows onto a second page.
