# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository layout

This working directory **is itself the git repo root** (remote: `github.com/supasiao7896TH/Regular-Auto-Update`, branch `main`). Run all git commands directly here — there is no nested subfolder to `cd` into.

- `Regular_auto_update.html` — original/base app, "CTA & PTA No.1 Plant"
- `Regular_auto_update_PTA_No2.html` — fork for "GCM PTA No.2 Plant"
- `Regular_auto_update_PTA_No3.html` — fork for "PTA No.3 Plant"
- `index.html` — small landing page linking to the 3 apps (the Worker's root URL)
- `wrangler.jsonc`, `.assetsignore`, `.github/workflows/deploy.yml` — Cloudflare Workers auto-deploy (see **Deploy**)
- `README.md` — user-facing usage doc (Thai); partly stale (still mentions SheetJS/Excel export, ~74 tasks and reference `.csv`/`.xls` files that are not in the repo) — trust this file and the code over it

Note for anyone working from this user's other machine: some past notes describe this repo living inside a `Regular-Auto-Update/` subfolder with a manually-synced loose copy of the No.2 file one level up. That layout is not what's on disk here — check `git rev-parse --show-toplevel` if a setup on another machine looks different, rather than assuming a mirrored-copy workflow is still needed.

## What this is

Three independent, self-contained, single-file HTML web apps (no build step, no backend, no package.json) that generate a monthly regular-maintenance/inspection schedule for GC-M PTA plant departments — a table of tasks vs. days-of-month with recurring "done" marks, exportable to Excel/print/PDF/image.

**The 3 files are forks of the same codebase and share byte-identical module structure and function names.** They differ only in: branding strings (plant name, title, PDF/backup file names such as `GCM_PTA2_…`), `APP_CONFIG.DB_NAME`, the `DEFAULT_TASKS` data array (task list), `SECTION_REMARKS` (No.2 only), and the forks have no header "signature" block. When fixing a bug or adding a feature, the same edit almost always applies verbatim to all 3 files — check this before assuming per-file logic needs to be re-derived. Line numbers differ slightly between files because `DEFAULT_TASKS` has a different length in each.

**Keep the 3 `DB_NAME`s distinct** (`gcm_sched_v13`, `gcm_sched_pta2_v1`, `gcm_sched_pta3_v1`): once deployed, all three apps share one origin, so identical names would make them overwrite each other's IndexedDB data.

**Porting main → forks after the forks fall behind** (they did once: the Studio design system and several UI fixes only landed in the main file first). Don't hand-copy hunks; do a 3-way merge with `git merge-file <fork> <base> <main>` where `<base>` is `git show <last-synced-commit>:Regular_auto_update.html` (last sync: main's HTML at `9cc192a`, 2026-09-20). First cut the pre-rendered table snapshot (see below) out of all three inputs — otherwise every one of its ~60 rows conflicts — and put each fork's own copy back afterwards. Expect exactly these conflicts and resolve them in favour of the fork: header plant-name line, the header signature block (forks have none), the PDF filename. Then verify: no plant string of another file leaked in (`grep`), the script block passes `node --check`, and `buildPrintDoc`/`printSchedule`/`calcScheduledDays` are byte-identical to before.

**This is not the same 9-module architecture as this user's general Vibe Coding standard** (no Firebase, no `CLOUD_SYNC_MANAGER`, no `AUTH_PROVIDER`, no `GEMINI_AI_BRIDGE`). Persistence is 100% local via IndexedDB. Don't assume those modules exist here.

## Commands

No build/lint/test tooling exists. Everything is manual:

- **Syntax-check after editing**: extract the inline `<script>...</script>` block and run it through Node — this catches JS syntax errors without a browser:
  ```
  node -e "const fs=require('fs'); const html=fs.readFileSync('Regular_auto_update_PTA_No2.html','utf8'); fs.writeFileSync('_check.js', html.match(/<script>([\s\S]*)<\/script>/)[1]);"
  node --check _check.js
  ```
- **Run/test in a browser**: these apps must be served over `http://` (a local static server), not opened via `file://` — several features (chiefly the IndexedDB persistence path) behave differently or are unavailable under `file://`. A one-off Node static server is enough:
  ```
  node -e "require('http').createServer((req,res)=>{const fs=require('fs');fs.readFile('.'+decodeURIComponent(req.url==='/'?'/Regular_auto_update_PTA_No2.html':req.url),(e,d)=>{if(e){res.writeHead(404);res.end();return;}res.writeHead(200,{'Content-Type':'text/html'});res.end(d);});}).listen(PORT)"
  ```
  Note: `file://` mode is still a real, supported usage mode for end users (double-click to open) — the app detects it (`window.location.protocol === 'file:'`) and falls back to an in-memory store instead of IndexedDB (see `STORAGE_ENGINE`). Just don't rely on that mode for testing browser-API-dependent features during development.
- **No package manager** — all third-party code loads from CDN in `<head>`. `html2canvas` (1.4.1) and `jsPDF` (4.2.1) are pinned with verified `integrity` (SRI) hashes; Tailwind CSS loads via the unversioned play-CDN script (`cdn.tailwindcss.com`), which doesn't support pinning/SRI. Do not add a *new* dependency without a verified `integrity` hash where the CDN supports one — fetch the file and compute/cross-check it (e.g. against cdnjs's published SRI API) rather than guessing one.

## Architecture (per file, ~5870-5900 lines)

Static HTML/CSS in the first ~3300 lines (header, toolbar, modals, print CSS), followed by a single `<script>` block containing 9 IIFE modules in this order, each exposed as a `const MODULE_NAME = (() => { ...; return {...}; })();`.

**Do not edit the pre-rendered table snapshot** (`<thead id="tableHead">` … `</tbody>`, ~3,000 lines starting around line 268 in every file). It is a stale copy of the *main* app's table (the forks carry main's rows, not their own) and is fully overwritten by `APP_CORE.generate()` → `renderHeader`/`renderBody` on load; `#scheduleTable` is `display:none` until `generate()` shows it, so the snapshot is never seen. Change table markup in `UI_RENDERER`, never in the snapshot. Modules, in order:

1. **`APP_CONFIG`** — `DB_NAME`, `DB_VERSION` (2), month/day label arrays, `DEFAULT_TASKS` (the task list — array of `{id, no, section, name, shift, freqType, freqValue, freqLabel, lastDate, scheduleAnchor, scheduleNote}` plus optional `scheduleCustomDays`/`scheduleAnchor2`/`scheduleAnchorNth`/`scheduleMode`/`scheduleBiannualMonths`), and (No.2 only) `SECTION_REMARKS` — free-text footnotes keyed by section name, rendered right after that section's rows in Excel/print/PDF output.
2. **`STATE_STORE`** — plain pub/sub state container.
3. **`STORAGE_ENGINE`** — IndexedDB CRUD across 3 object stores (`tasks`, `settings`, `scheduleOverrides`, all keyed by `id`/`key`). Falls back to an in-memory object when `window.location.protocol === 'file:'` (IndexedDB is unreliable/restricted from `file://`).
4. **`FREQ_CALC`** (comment label "FREQUENCY_CALCULATOR") — `calcScheduledDays(task, year, month)` computes which days of a given month get auto-marked, purely from `task.freqType`/`freqValue`/`scheduleAnchor`(s)/`scheduleCustomDays` — never from hardcoded dates, so it works for any month/year. `freqType` vocabulary: `daily`/`shift` (every day), `days` (every N days), `weekly` (one weekday), `biweekly` (two weekdays), `monthly` (fixed date or nth-weekday), `bimonthly` (two dates, `anchor`/`anchor+15`, or nth-weekday), `customDays` (explicit day list — the fallback for patterns that don't fit any formula), `dcs`/`batch`/`special` (no auto-mark; shows `task.scheduleNote` text instead, for DCS-timer-driven or non-calendar tasks), `quarterly`/`biannual`.
5. **`UTILS`** — small helpers (`escapeHtml`, etc.); aliased to bare top-level names (e.g. `const escapeHtml = UTILS.escapeHtml;`) right after the module, so most call sites use the bare name, not `UTILS.foo`.
6. **`UI_RENDERER`** — largest module: main table render, all modals (task manager, add/edit task, image-preview, in-app manual/help — `showManual()`), toasts (`showToast`), and the shared helpers `ICONS`/`svgIcon(name, size)` (use these instead of pasting `<svg>` markup), `showConfirmModal(message, onConfirm, {confirmLabel, danger, onCancel})` (in-app confirm dialog with `role="dialog"`, Esc and a Tab trap — never use native `confirm()`; pass `onCancel` when the dialog is opened from inside another modal, otherwise cancelling closes that modal too) and `syncFreezeOffsets()` (see **Frozen columns**). Sections in the on-screen table (and in Excel/print/PDF) are derived dynamically from the data — `[...new Set(tasks.map(t => t.section))]` — never hardcoded, so the section list adapts automatically to whatever `section` strings appear in `DEFAULT_TASKS`.
7. **`DEBUG_MODULE`** — `log(level, msg, data)` console wrapper, `[GCM]`-prefixed.
8. **`EXPORT_ENGINE`** — `exportJSON`/`importJSON` (full data backup/restore) and `saveStandaloneHTML` (re-serializes the current app + data into a new downloadable `.html` file, embedding data via `window.__GCM_EMBEDDED__` for the `file://` fallback path to pick up on first load). Excel export (`exportExcel`) existed here originally but was removed from all 3 files as unused — don't re-add it without being asked.
9. **`APP_CORE`** — month navigation, `generate()` (computes the displayed schedule = `FREQ_CALC` base schedule with any per-day manual overrides from `scheduleOverrides` applied on top), `buildPrintDoc()` (builds the full print/PDF HTML document as a string — shared by `printSchedule()` and `exportPDF()`, so both stay visually identical to the on-screen data), `exportPDF()` (renders `buildPrintDoc()`'s HTML into a hidden iframe, rasterizes it with `html2canvas` and embeds the image in an A3-portrait PDF via `jsPDF`, then `pdf.save(...)`; this replaced the earlier clipboard-image `exportImage()`, which no longer exists), `init()`.

## Conventions specific to this codebase

- **Inline styles are the deliberate convention**, not an oversight — everything is one HTML file with no external CSS, so `style="..."` attributes are used throughout intentionally. Don't "clean up" by extracting to a stylesheet; that breaks the single-file distribution model these apps are built around (double-click to open, or drop on any static host).
- **Line endings can differ between files and between machines** — as of the last check here all 3 files are CRLF, but don't assume that's fixed: verify with `file *.html` (or check for `\r` before newlines) before scripting a cross-file text edit via Node, and normalize to `\n` for matching, restoring the original ending per-file, or a naive string-replace will silently fail on whichever file doesn't match your assumed line ending.
- **`buildPrintDoc()` is the single source of truth for anything print/PDF/image-shaped.** If a print/PDF/exported-image bug is reported, the fix almost always belongs inside this one function (and its embedded `<style>` string), not in the on-screen `UI_RENDERER` table renderer — the two are visually similar but are two entirely separate render paths.
- **A3 portrait page-fit is computed, not fixed**: the print template calculates row height (`rowH`) as `usableBodyH / rowCount`, where `rowCount` must equal the exact number of `<tr>` rows the template emits (section headers + task rows + any `SECTION_REMARKS` rows, including their own header row). If you add a new kind of row to the print table, you must also add it to this count, or the table silently overflows onto a second page.
- **Frozen columns (No./Work/Time/Frequency) are measured, not hardcoded.** The table is auto-layout, so those columns grow past their `min-width` with long task names or wide screens. `UI_RENDERER.syncFreezeOffsets()` reads the header cells' real widths into the CSS variables `--fz2/--fz3/--fz4` that the sticky `left:` values use. It must run *after* the table is visible (`generate()` calls it right after un-hiding `#scheduleTable`; a hidden table measures 0 and the function then keeps the last good values), and again on `resize` and `document.fonts.ready`. Body cells `td.freeze-*` and the sticky "จัดการ" cell need their own `z-index:1`, otherwise the `position:relative` O-badges in later cells paint over them while scrolling. Below 768px only No.+Work stay frozen (`@media` block in `<style>`).
- **Colors come from the Studio tokens in `:root`** (`--navy`, `--surface`, `--text2`, …); don't hardcode hex values in new UI. Every icon-only button needs an `aria-label` (these apps follow a measured ≥4.5:1 contrast / 44px tap-target / `aria-label` standard).
- **Editing a CRLF file with the Edit tool:** multi-line `old_string`s don't match CRLF. Convert the file to LF with a small Node script first, edit, then convert back (`\n` → `\r\n`) and confirm `git diff --stat` shows only the real changes (a whole-file diff means endings were left mixed).
- **Testing responsive CSS:** in the browser-automation setup used here `resize_window` did not shrink the viewport (`innerWidth` stayed huge), so `@media` rules never applied. Test the ≤768px layout by loading the app in a ~400px-wide `<iframe>` on the same origin and inspecting `iframe.contentWindow`.

## Deploy

Pushing to `main` **is** a production deploy: `.github/workflows/deploy.yml` runs `cloudflare/wrangler-action` and publishes the repo root as an assets-only Cloudflare Worker `regular-auto-update` (config in `wrangler.jsonc`, ~20 s). Live at `https://regular-auto-update.supasiao.workers.dev` (landing page) with the apps at `/Regular_auto_update`, `/Regular_auto_update_PTA_No2`, `/Regular_auto_update_PTA_No3` (a request for `….html` 307-redirects to the extensionless URL). It needs the repository secret `CLOUDFLARE_API_TOKEN` (set by the user, never by Claude).

- **`.assetsignore` decides what becomes public.** Any new root-level file that is not part of the site must be added there (it already excludes `.git`, `.github`, `*.md` docs, `wrangler.jsonc`, and `package.json`/`package-lock.json`, which the deploy step creates in the repo root and which were briefly served before being ignored). After changing deploy files, probe the live URL with `curl` for files that should 404.
- **Validate config without deploying:** `npx --yes wrangler@latest deploy --dry-run --outdir <tmp dir>` (no login needed). Its log lists every file as `Ignoring asset:` or keeps it; delete the `.wrangler/` folder it leaves in the repo root afterwards.
- GitHub Pages is also enabled on this repo (serves `main` from `/`), so the same files exist at `https://supasiao7896th.github.io/Regular-Auto-Update/` — and there `CLAUDE.md`, `README.md` and `wrangler.jsonc` are served too (checked with `curl`: 200), because `.assetsignore` only applies to the Worker. Don't put anything non-public in this repo.
- Each app keeps its data in the browser's IndexedDB on that device; nothing syncs across devices or between the two URLs. The apps also load Tailwind/jsPDF/html2canvas from external CDNs, so they need those hosts reachable.
