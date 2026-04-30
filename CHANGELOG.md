# Changelog

All notable changes are documented here.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).
Versioning follows [Semantic Versioning](https://semver.org/).

---

## [1.2.7] — 2026-04-30

### Fixed
- **`latest` block-scope bug** — `const latest` was declared inside the `try{}` block in Pass 2 and referenced outside it on the `modelGuid` line. This caused a `ReferenceError` for every successfully-fetched file, which the `pool()` worker silently caught (`catch(e){results[i]=undefined}`), dropping every file before it could be pushed to `c4rFiles` or `cloudFiles`. All projects appeared as No RCW. Fix: moved `latest` to the outer `let` declaration and changed the inner assignment from `const` to a plain assignment.
- **Stale comment block removed** — the "Conservative item-level RC skip / isDefinitelyRC" comment block introduced in v1.2.4 described an optimization that was already removed. Replaced with a single accurate note.

---

## [1.2.3] — 2026-05-01

### Changed
- **Setup page redesigned** — two fully self-contained method cards (Option A: Sign in with Autodesk; Option B: Use Client Secret), each with their own step-by-step instructions, their own Client ID field, and their own connect button. Eliminates the previous ambiguity where a shared Client ID field sat between two different flows separated by an "or" divider. Each card explains the correct APS app type for that flow, the Callback URL requirement, and the Hub Admin registration step. Client ID fields sync between cards so users don't need to retype if they switch methods.

---

## [1.2.2] — 2026-05-01

### Added
- **`hidden: true` item filter** — APS DM API folder contents include system/derivative items with `attributes.hidden = true` that are invisible in the ACC web UI. These are now filtered in `isRvt()` before filename or type checks. Zero false-negative risk: ACC itself marks them as non-user-facing.
- **GUID/system filename detection (`isSystemName`)** — detects four patterns: full UUID (8-4-4-4-12), 32-char hex, `vf.`-prefixed URN fragments, and 20+ char hex strings. Covers Design Automation output files, Desktop Connector conflict backups, and internal storage objects. These files are not silently skipped — they go to `proj.systemFiles[]` and appear in the expanded row as: *"N auto-named files found — not counted toward risk."*
- **`modelGuid` deduplication in `deriveProject()`** — Design Collaboration creates copies of the same model in multiple folders (working copy, Shared folder, consumed package). All copies share the same `modelGuid` in `extension.data`. `deriveProject()` now builds a Map keyed by `modelGuid`, keeping the copy with the lowest (worst) version for risk assessment. `c4rCount` and `atRiskCount` operate on `uniqueC4RFiles` (deduplicated), not the raw `c4rFiles` array.
- **Copy annotation** — files sharing a `modelGuid` with another file in the same project are marked `isCopy: true` and show a grey **copy** badge in the expanded file table.
- **`c4rCopyCount` field** — stored on the project; shown in an informational purple banner in the expanded row: *"3 unique RCW models (5 additional copies — risk metrics use unique model count)."*
- **PDF accuracy caveats** — copy count and system file count added as bullet lines in the accuracy statement.
- **Hub Admin terminology** — all occurrences of "Account Admin" updated to "Hub Admin" throughout the app and docs to match the current Autodesk platform terminology.
- **Live page link in README** — `https://arqfernandolima-rgb.github.io/revit-version-checker/` added prominently below the badge row.

### Changed
- **`proj.systemFiles[]`** — new bucket on the project object; initialised alongside `c4rFiles`, `cloudFiles`, `failedFiles`
- **`proj.uniqueC4RFiles[]`** — deduplicated C4R file list; used by `atRiskCount` and `c4rCount`
- **`proj.c4rCopyCount`** — count of Design Collaboration copies excluded from unique count

---

## [1.2.1] — 2026-05-01

### Changed
- **Version inference extended to full tier range** — previously only files with modification date ≤ threshold were inferred as Critical. Now also infers Outdated for files modified at threshold+1 or threshold+2 (e.g. 2022/2023 with threshold=2021). Files modified before 2020 are always Critical. Files modified after threshold+2 remain as No Version (cannot safely assume Current without API data).

---

## [1.2.0] — 2026-05-01

### Added
- **Parallel folder fetches (`CONCURRENCY.FOLDER_FETCHES=3`)** — BFS processes 3 folders simultaneously per project using `Promise.all` batching. Each iteration pulls up to 3 items from the priority queue, fetches contents in parallel, collects subfolder discoveries, re-sorts by priority, and prepends to the queue. Safe: JS single-threaded model means post-`await` synchronous loops execute atomically, so `visitedFolders` and `batchSubFolders` mutations are race-free. Reduces folder-walk time 2–3×.
- **`?include=version` inline classification (`apiAllWithInclude`)** — folder contents calls append `?include=version`, embedding each item's latest version data in the same response. Items whose inline version data contains `extension.type` and `revitProjectVersion` are classified immediately — no separate version API call needed. Pass 2 runs only for items without inline data. A project with 5 folders and 15 RVT files could reduce from 20 API calls to 5.
- **No Version status** — new `no-version` project status (amber dashed chip) for projects with confirmed RCW files but no version year available from the API. Includes metric tile, filter option, PDF badge, CSV label, About page entry, and expanded row explanation with schema version context.
- **Version inference from modification date** — when `revitProjectVersion` is absent (pre-2023 BIM 360 schema v1.1.x/v1.2.x), the file's `lastModifiedTime` year is used as a conservative estimate. Applied in both the inline classification path and Pass 2. Files with inferred versions show an `(est.)` label. An amber banner explains the inference in the expanded row.

### Fixed
- **Project showed "No RCW" with confirmed RCW files** — `pStatus` fell through to `no-c4r` when `c4rVersion` was null even if `c4rFiles.length > 0`. New `no-version` check fires first.
- **Files showed "Outdated" with `—` version** — `parseInt(null||0) = 0 ≤ threshold+2` was always true. File-level chip now checks `!f.version` and shows "No Version" chip for null versions.

---

## [1.1.0] — 2026-05-01

### Added
- **`SKIP_FOLDERS` at all depths** — skip list extended from top-level only to every BFS depth. 35+ folder names added covering archive/obsolete (Old, Superseded, Backup, Deprecated, Previous, History, Legacy), admin (Admin, Administration, Templates), media (Images, Pictures, Video, Documents, Sheets), and workflow (Incoming, Outgoing, For Review, For Approval, Issued, Record Drawings) — eliminating API calls for any folder that cannot contain active RCW models.
- **`MAX_FOLDER_DEPTH=4`** — BFS capped at 4 levels from topFolders, matching the deepest documented real-world ACC structure. Eliminates the dominant cause of 5-minute timeouts on deep/demo account projects.
- **`FOLDER_PRIORITY` — priority-first BFS** — folders scanned in priority order: `Project Files` (0) → `Shared` (1) → discipline names (2) → other (3). Sibling folders at each BFS level sorted before being prepended to the queue. Results appear quickly and timeouts leave the project with the most important files intact.
- **`CONCURRENCY.FOLDER_FETCHES` constant** — documents the parallel folder fetch count.
- **401 refresh-and-retry** — `api()` 401 response triggers `refreshSessionIfNeeded(force=true)` and retries the failed request once before aborting. Previously a mid-group token expiry immediately halted the entire scan.
- **`force` parameter on `refreshSessionIfNeeded()`** — bypasses the 50-minute age check for on-demand recovery.

### Changed
- **`GROUP_SIZE` 200 → 100** — groups complete in 5–15 minutes, well within the 60-minute token window. More frequent progressive results on large hubs.
- **3-legged rate limit 50 → 100 req/min** — previous limit caused continuous throttle waits and made scans significantly slower. 100/min is still conservative (actual limit ~150/min).
- **`VERSION_FETCHES` 5 → 8** — safe with rate limiter at 100/min (2 projects × 8 = 16 peak).
- **`SKIP_THRESHOLD` 3 → 1** — after 1 confirmed C4R file, trust item-level RC type for classification.

---

## [1.0.0] — 2026-05-01

First stable release.

### Added
- **Versioning system** — `VERSION`, `RELEASE_DATE`, `CHANGELOG_URL` constants; topbar version badge linking to GitHub releases; version card in About page
- **Project groups** — hub split into groups of 200 (later 100); sequential group scanning; collapsible group sections; per-group CSV/PDF export
- **Per-request 30s timeout** — `AbortController` on every `fetch()`; hung requests abort and retry
- **Proactive rate limiting** — `RateLimit` token-bucket; auth-mode-aware limits; API load % in progress bar
- **Token expiry detection** — 401 mid-scan stops cleanly and prompts re-auth
- **Token auto-refresh** — `refreshSessionIfNeeded()` called before each group; 2-legged silent re-auth; 3-legged `refresh_token` grant; `offline_access` scope
- **Scan accuracy statement in PDF** — file-level accuracy %, failed-file count, skipped-folder count, no-access caveat
- **Conservative item-level RC skip** — after confirmed C4R files, known-RC item types skip version call
- **Current metric tile** — green "Current ≥ threshold+3" card on dashboard and PDF
- **Queued / Scanning / Done project states** — "Queued" chip before worker pickup, "Scanning" spinner while active
- **Auth-mode-aware scan concurrency** — 4 parallel projects for 2-legged, 3 for 3-legged
- **PDF visual redesign** — two-tone header; colour-coded row backgrounds; status badge pills; version text coloured by tier; red left accent on critical rows; "Open ACC" button; footer with version number

### Fixed
- **Critical metric showed green when zero** — now shows grey/hint when 0, red when > 0
- **PDF "View !" broken link** — `\u2192` not in Helvetica WinAnsi; replaced with drawn "Open ACC" button
- **Version badge rendered as literal text** — `${VERSION}` in static HTML cannot be interpolated; fixed by setting from JS in `render()`
- **`visitedFolders` guard** — prevents duplicate BFS traversal of shared/linked folders
- **`documents` item type** — `.rvt` files surfacing as `type: "documents"` were silently skipped
- **`apiAll` URL parsing** — `new URL(href, APS)` replaces brittle string replace
- **Silent file drop on version failure** — failed files now appear in `failedFiles` list
- **Pool worker fairness** — `setTimeout(0)` yield prevents workers racing through no-access tasks

---

## [0.9.0] — 2026-04-30

### Added
- 2-legged auth — Client Secret, memory-only storage, session token in `sessionStorage`
- Admin API integration — enumerates all active projects regardless of membership
- Member project pre-filter — pre-marks non-member projects as No Access (3-legged)
- Multi-status filter, threshold picker, PDF export, CSV export
- Parallel BFS scan, group scanning, failedFiles, skippedFolders
- Demo mode, No Access status, pending/scanning/done states

### Fixed
- `atRisk` stale on threshold change — dynamic `atRiskCount(p)` replaces stored field
- MIXED project status — explicit status; `parseInt('MIXED')` = NaN no longer silently fails
- BFS recursion → iterative BFS queue
- Folder pagination — full `apiAll()` pagination

---

## [0.1.0] — Original release (Tanmay Bhalerao)

Initial version: Data Management API, member projects only, basic version display and deprecation status indicators.
