# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Static HTML dashboards/forms for Total Productive Maintenance (TPM) tracking at PT AT Indonesia's Casting 2 plant (Indonesian-language UI). There is no build system, package manager, or test suite — every page is a single self-contained `.html` file with inline `<style>` and `<script>`, deployed as-is via GitHub Pages from `main` (repo: `andonoimagine-dev/self-maintenance`). "Development" means editing an HTML file directly and opening it in a browser (or pushing to `main` to see it live on Pages).

There is no lint/build/test command to run. When verifying a change, open the file directly in a browser (or use the `run`/`verify` skill) and click through the affected tab(s).

## Files

- `hsb-management-system.html` — TPM HSB system dashboard (impeller ampere monitoring, sparepart, limbah/waste, steelshot, trolley, preventive maintenance). Password-gated (`hsb2026`).
- `cnd-management-system.html` — TPM CND system dashboard, structurally parallel to the HSB one but a separate app/spreadsheet/password (`cnd2026`).
- `hsb-test-koneksi.html` — standalone diagnostic tool: runs a 5-step check (connectivity → auth → sheet init → read → write) against a given Apps Script URL/password. Use this when debugging why an HSB/CND-style backend isn't syncing.
- `1.html` — "Self MTC Dashboard" (Chart.js-based overview dashboard).
- `2.html` — "Form Temuan Self-Maintenance" (abnormality/finding report form). Shares the same Apps Script deployment/spreadsheet as `1.html`.
- `index.html`, `dashboard_selfmtc.html` — meta-refresh redirect stubs pointing at an external portal (`portal.at-indonesia.co.id`); not active app pages, kept for old links/QR codes.

## Architecture (per management-system app)

Each of `hsb-management-system.html` / `cnd-management-system.html` (and to a lesser extent `1.html`/`2.html`) follows the same pattern:

- **Single-page app, one file.** Tabs/pages are `<div class="pg">` elements toggled by adding/removing an `on` class; navigation buttons (`.tb`) call a `goTab(name)` function that shows `#p-<name>` and invokes that tab's refresh function (e.g. `rDash`, `rLb`, `rSp` in `hsb-management-system.html`; `rDash`, `rPrev`, `rSp`, ... in `cnd-management-system.html`). When adding a new tab, wire it into both the nav buttons and the `goTab`/dispatch table.
- **Theming** via CSS custom properties on `[data-theme="light"]` / `[data-theme="dark"]`, toggled by a button and persisted to `localStorage`.
- **Backend = Google Apps Script Web App over a Google Sheet.** Each app hardcodes its own Apps Script deployment URL as `GAS_URL` (or `APPS_URL`/`API_URL` in `1.html`/`2.html`) plus a plaintext app password checked both client-side and by the Apps Script. There is no real auth — the password is a shared factory-floor PIN, not a security boundary.
- **API calls go over GET, not POST**, with `action`, `pwd`, and any extra fields JSON-stringified into query params — this is deliberate, to dodge CORS preflight against Apps Script (see comments in `hsb-management-system.html` around the `api()` function). `cnd-management-system.html` deviates for a few detail-update actions, which POST a JSON body instead.
- **Offline-first.** App state is cached in `localStorage` under an app-specific prefix (e.g. `hsb9_*`, `cnd9_*`), and failed writes are pushed onto a retry queue (`hsb9_queue` / equivalent) that gets flushed on a timer (`flushQueue`, sync interval configurable via `localStorage` under `hsb9_sync_interval`, default 300s). The server (Google Sheet, via `getAll`/`syncFromSheets`) is treated as the source of truth on sync — local arrays are overwritten wholesale, including with empty results, so deletes on the sheet propagate.
- Domain vocabulary you'll see throughout: `limbah` (waste), `sparepart`, `ampere` (motor current draw monitoring), `steelshot`, `trolley`, `preventive` (scheduled maintenance), `hanger`, `impeller` — these are casting/foundry machine components and consumables, not generic naming.

## Working in this repo

- Files are large (the management-system HTML files run 1,900–5,400 lines) with everything — markup, CSS, JS — in one file. Use `Grep`/offsets to jump to the relevant section (e.g. search for the tab id, or for `GAS_URL`/`api(`) rather than reading top-to-bottom.
- Each management-system file is independent: don't assume a fix in `hsb-management-system.html` needs to be mirrored in `cnd-management-system.html` (or vice versa) unless the user asks — they have separate GAS backends/spreadsheets and have diverged over time, even though the tab/theming scaffolding is structurally similar.
- Since there's no bundler, don't introduce `import`/module syntax, npm dependencies, or a build step — keep additions as inline vanilla JS/CSS consistent with the surrounding file. External libraries already in use are loaded via CDN `<script>`/`<link>` tags (e.g. Chart.js, Google Fonts) in `cnd-management-system.html`.
