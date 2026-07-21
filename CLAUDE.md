# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Painel de Gestão da DSV — a management dashboard for Brazil's *Diretoria de Segurança Viária (DSV)* at Detran-SP. It tracks projects, contracts, strategic/management indicators, dispatch meeting agendas (pautas), a Gantt schedule, a deadline calendar, and the org chart from Portaria Detran-SP nº 44/2025. All content and UI copy is in Brazilian Portuguese.

## Repository structure

This repo has no build system, no package manager, and no test suite. It is exactly two files:

- `index.html` — the entire application (~8,800 lines): markup, Tailwind utility classes, custom CSS, and all JavaScript in one inline `<script type="module">` block starting around line 4600.
- `README.md` — feature overview in Portuguese.

There is no `package.json`, no bundler, no transpiler, no linter config, and no test runner. **Do not introduce one unless asked** — the project is intentionally a single deployable static file.

## Development workflow

- Edit `index.html` directly with a text editor; there is no compile step.
- To preview, open `index.html` in a browser or serve the directory with any static file server (e.g. `python3 -m http.server`). A real HTTP origin is required for the Firebase SDK and ES module imports to work correctly (not `file://`).
- There is no automated test suite. Verify changes manually in the browser: log in, exercise the affected tab/view, and check the browser console for errors.
- Commits historically are single-file changes with generic messages ("Update index.html") — there's no enforced commit convention.

## Architecture

### Single-file, no-framework app

There is no component framework (no React/Vue/etc). The app is plain DOM manipulation:

- Global mutable state lives in top-level `let`/`const` bindings inside the `<script type="module">` block (e.g. `dataStore`, `activeTab`, `currentFilter`, `selectedMeetingId`, `projectContractViewMode`).
- `dataStore` is the in-memory cache of all Firestore data: `{ projects, contracts, operational, calendarEvents, indicators, dispatchMeetings, dispatchCustomItems }`.
- Render functions (`window.renderData`, `window.renderIndicators`, `window.renderDespachos`, `window.renderCalendar`, `window.renderGantt`, `window.renderPortaria`, etc.) read from `dataStore` and rebuild `innerHTML` for their section.
- Nearly every handler is attached to `window.*` (e.g. `window.saveProject`, `window.toggleModal`, `window.switchTab`) because generated HTML strings wire up behavior via inline `onclick="..."` attributes rather than `addEventListener`. When adding a new interactive element, follow this same pattern — expose the handler on `window` and reference it inline.

### Views / tabs

`window.switchTab(tab)` (around line 6542) is the router. It sets `activeTab`, toggles the `.hidden` class on top-level view containers, and restyles the nav. The view containers are plain `<div>`s, not routes:

- `#standardView` — projects/contracts/operational (cards, table, or Gantt-adjacent list views), driven by `currentProjectCollection`
- `#despachosView` — dispatch meetings / pautas
- `#indicatorsView` — strategic & management indicators
- `#portariaView` — org chart / Portaria nº 44/2025 viewer
- `#calendarView` — deadlines/events calendar
- `#ganttView` — Gantt schedule

### Firebase (Auth + Firestore)

Firebase is loaded via ES module imports directly from `gstatic.com` CDN URLs (no npm), version pinned (`firebasejs/11.6.1`). Config and `initializeApp` happen inline at the top of the script block (~line 4600).

- **Auth**: email/password only (`signInWithEmailAndPassword`). `#loginGate` blocks the UI until `onAuthStateChanged` reports a signed-in user. New users must be provisioned directly in the Firebase console — there is no self-registration flow.
- **Data model**: every collection lives under a shared namespace path `artifacts/{appId}/public/data/{collectionName}` where `appId = 'painel-gestao-main'`. This is a multi-tenant-style Firestore layout (likely inherited from a template) even though there is only one tenant.
- **Real-time sync**: `initListeners()` (~line 5263) attaches one `onSnapshot` listener per collection (`projects`, `contracts`, `operational`, `calendar_events`, `indicators`, `dispatch_meetings`, `dispatch_custom_items`), maps snapshot docs into `dataStore`, and re-renders only the currently active view. When adding a new persisted collection, register it in this same loop rather than fetching ad hoc.
- Writes use `setDoc`/`deleteDoc`/`writeBatch` directly against Firestore from the client — there is no backend API layer. Errors are surfaced with a raw `alert('Erro ao ... no Firebase. Veja o console.')`; follow this existing (unpolished but consistent) error-handling style rather than inventing a new UX for it.
- `buildAuditFields()` generates created/updated-by metadata (using `currentUser`) that gets merged into documents on save — reuse it for any new record type rather than hand-rolling audit fields.

### Status/normalization conventions

Project, contract, and operational-item status values are unified across types (`UNIFIED_STATUS` = Backlog, Em Execução, Em Execução (com risco), Cancelado, Concluído). `normalizeStatus()` and `normalizeStoredStatusOnItem()` coerce legacy/inconsistent stored values into this set on read — new status-related code should go through these rather than comparing raw strings.

### Notable third-party CDN dependencies (loaded via `<script>`/`<link>` tags, no local copies)

- Tailwind CSS (JIT via CDN script, not a build step)
- Font Awesome 6 (icons)
- `html2pdf.js` — used by `window.exportSelectedMeetingPdf` to export dispatch meeting agendas as PDF
- Google Fonts (Open Sans)
