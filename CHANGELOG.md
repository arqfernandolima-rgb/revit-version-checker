# Changelog

All notable changes to this project are documented here.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).
Versioning follows [Semantic Versioning](https://semver.org/).

---

## [1.1.0] — 2026-05-01

### Added
- **Folder skip list (`SKIP_TOP_FOLDERS`)** — 22 ACC system and administrative top-level folder names (Plans, Photos, Submittals, RFIs, Issues, Closeout, Reports, Transmittals, Markups, Specifications, Correspondence, Meetings, Contracts, Admin, Archives, Templates, and variants) are skipped at the `topFolders` stage with zero API calls. These are system-created or convention-only folders where Revit models cannot exist. Counted separately as `skippedTopFolders` and shown as an informational note in the expanded row.
- **Depth limit of 4 (`MAX_FOLDER_DEPTH`)** — BFS traversal capped at depth 4 from the top-level folder, matching the deepest real-world ACC structure documented by Autodesk (`Project Files / Discipline / Phase / Subphase / model.rvt`). Eliminates the dominant cause of 5-minute project timeouts on deep/demo account projects.
- **Priority-first BFS (`FOLDER_PRIORITY`)** — Folders are scanned in priority order: `Project Files` (depth 0 → priority 0), `Shared` (Design Collaboration consumed copies, priority 1), known discipline names — Architecture, Structural, MEP, Mechanical, Electrical, Plumbing, Civil, Interior, Landscape, Facade, Fire, IT, Technology (priority 2), everything else (priority 3). Sibling folders at each BFS level are sorted by priority before being queued. Ensures discipline folders with working models scan before archive/admin folders, so results appear quickly and timeouts still leave the project with useful data.
- **401 refresh-and-retry** — When `api()` receives a 401 mid-scan, it calls `refreshSessionIfNeeded(force=true)` and retries the failed request once before aborting. Previously a mid-group token expiry immediately stopped the entire scan.
- **`force` parameter on `refreshSessionIfNeeded()`** — bypasses the 50-minute age check for on-demand refresh triggered by a 401 response.

### Changed
- **GROUP_SIZE 50 → 100** — Larger groups reduce the number of between-group refreshes on small hubs while still completing well within the 60-minute token window.
- **3-legged rate limit 50 → 100 req/min** — Previous limit was overly conservative and caused the rate throttle to insert waits continuously, making scans significantly slower. 100/min is still well below the actual APS limit (~150/min for user tokens) while eliminating unnecessary drag.

### Fixed
- Groups 3+ showing "No RCW" even after v1.0.0 auto-refresh — caused by token expiry within a single group (>60 min on large or deep projects); resolved by depth limit + skip list reducing per-project scan time, and by the 401 refresh-and-retry as a backstop.

---

## [1.0.0] — 2026-05-01

First stable release. Scanning accuracy confirmed on production hubs. Token auto-refresh resolves multi-group expiry. All known resilience issues addressed.

### Added

**Authentication**
- `offline_access` added to PKCE scope so APS returns a refresh token alongside the access token
- `refreshSessionIfNeeded()` — called before each scan group; refreshes the token silently if >50 minutes old
  - 2-legged: re-authenticates using stored `S.clientId` + `S.clientSecret`, no user interaction
  - 3-legged: uses `refresh_token` grant; supports rolling refresh tokens
  - 3-legged without refresh token: degrades gracefully — scan continues, 401 handler is backstop
- `S.tokenIssuedAt` and `S.refreshToken` added to state
- 401 response in `api()` now immediately stops the scan and renders a clear re-auth prompt

**Scan engine**
- `RateLimit` token-bucket — proactive rate limiting before each request; auth-mode-aware limits (50/min for 3-legged, 180/min for 2-legged); eliminates 429 cascade stalls
- 30-second `AbortController` timeout on every `fetch()` — hung connections abort and retry rather than blocking a scan worker indefinitely
- `API load X%` indicator in scan progress stats when approaching rate limit
- `RateLimit.calls` reset at the start of each new scan

**PDF report**
- Full visual redesign matching the screen UI:
  - Two-tone navy/blue header (title + subtitle strip)
  - Colour-coded row backgrounds per status (light red/amber/green/purple tints)
  - Drawn status badge pills (filled rounded rect, white text) in the status column
  - Version column text coloured to match status
  - 2.5mm red left accent bar on critical rows
  - "Open ACC" blue button replaces the broken "View !" link text
  - Metric tile separators in the metrics strip
  - Footer with version number and horizontal rule
- Scan accuracy statement above the projects table: file-level accuracy %, failed file count, skipped folder count, no-access project count, "not an official audit" disclaimer

**UI**
- Version badge (`v1.0.0`) in topbar, links to GitHub releases — set correctly from JS constants (not broken `${VERSION}` HTML literal)
- Version card in About page showing version number, release date, and changelog link
- Current metric tile (green) added to dashboard and PDF metrics strip
- Critical metric tile now shows red text when > 0, grey/hint when 0 (was incorrectly green when 0)
- Project state model: Queued → Scanning → result (pending/scanning/done flags)
- Pool worker yield between tasks prevents a worker from racing through no-access projects synchronously
- Scan progress shows "X done · Y scanning · Z queued" breakdown

### Fixed
- **Groups 3+ showing "No RCW"** — token expired mid-scan; resolved by auto-refresh between groups
- **Critical metric showing green** — was `atRiskTotal > 0 ? red : green`; now `atRiskTotal > 0 ? red : hint`
- **"View !" in PDF** — `\u2192` (→) not in Helvetica WinAnsi charset; replaced with drawn "Open ACC" button
- **Version badge showing `v${VERSION}` literally** — template literal in static HTML body cannot be interpolated; fixed by setting from JS in `render()`

---

## [0.9.0] — 2026-04-30

### Added

**Authentication**
- 2-legged auth — Client Secret entry on setup screen; token held in JS heap only, never stored persistently; session token in `sessionStorage` only
- Auth-mode-aware instructions throughout the setup screen, dashboard, and About page
- `connectTwoLegged()` function

**Scan engine**
- Project groups — `buildGroups()` splits hub into groups of 200; groups scan sequentially; within-group parallelism via `pool()`
- Auth-mode-aware concurrency — 4 parallel projects for 2-legged, 3 for 3-legged
- `refreshSessionIfNeeded()` skeleton (completed in v1.0.0)
- Per-group progress tracking — `grp.pdone`, `grp.eta`, `grp.activeNames`
- `renderProgress()` fast path — 600ms interval updates 5 DOM elements without full `innerHTML` rebuild
- Two-pass scan — Pass 1: BFS folder walk (zero version calls); Pass 2: version fetches batched across whole project
- Conservative item-level RC skip — after ≥3 confirmed C4R files, known-RC item types skip version call
- `visitedFolders` Set — prevents duplicate BFS traversal of shared/linked folders
- `apiAll()` with `new URL()` pagination — follows `links.next`, handles all URL forms, returns partial data on transient error
- `failedFiles` tracking — `.rvt` files whose version call failed shown in expanded row
- `skippedFolders` counter — 403'd subfolders counted and shown
- Both `items` and `documents` item types accepted for `.rvt` filtering
- 429 backoff and 5xx retry in `api()`

**UI**
- Collapsible group sections with group header: status chip, summary badges, mini progress bar, per-group CSV/PDF buttons
- No Access status chip and expanded row explanation (auth-mode-aware message)
- Queued / Scanning / Done project states
- Multi-status filter — filter dropdown is a multi-select Set; filter labels update with active threshold
- Per-group and whole-hub CSV/PDF export buttons
- Sort buttons in global controls bar

**PDF**
- Per-group or whole-hub export (optional `groupIdx` parameter)
- Filter-aware — applies active search + status filter + sort
- `didParseCell` uses `filteredProjs[row.index]` — correct when filter is active
- Auto-height risk banner and project header banners via `splitTextToSize`
- Scan accuracy statement (completed in v1.0.0)

### Fixed
- `atRisk` stale on threshold change — removed stored field; always computed dynamically
- MIXED project status — `parseInt('MIXED')` = NaN caused silent fallthrough; explicit status added
- BFS recursion stack overflow — replaced recursive `scanFolder` with iterative BFS queue
- Folder pagination — `api()` previously only returned first page of folder contents
- Silent file drop on version call failure — files now go to `failedFiles` bucket

---

## [0.8.x] — Pre-release

### Added
- ACC Admin API integration — `GET /construction/admin/v1/accounts/{accountId}/projects`
- Member project pre-filter (3-legged) — `getMemberProjectIds()` pre-marks non-member projects
- Admin notice on setup screen and dashboard (auth-mode-aware)
- Threshold picker — 2021 or 2022 critical cutoff
- Status tiers — threshold-relative Critical/Outdated/Current classification
- Multi-status filter dropdown (multi-select Set)
- PDF export with jsPDF + jsPDF-AutoTable — landscape A4, filter-aware
- CSV export — project summary + per-file RCW detail
- Demo mode — 9 sample projects covering all status tiers

---

## [0.1.0] — Original release (Tanmay Bhalerao)

Initial version: single-page app, Data Management API only (member projects), basic `.rvt` version display, deprecation status indicators.
