# Changelog

All notable changes are documented here.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.0.0/).
Versioning follows [Semantic Versioning](https://semver.org/).

Maintained by **Fernando Lima** (Sr. Technical Specialist, Autodesk) — built on original work by **Tanmay Bhalerao**.
This is a personal project and is not official Autodesk software. No warranty or liability from Autodesk.

---

## [1.6.2] — 2026-05-01

### Added — New scan / reset flow + filter clear button

Completes the filter-then-scan UX:

- **× clear button** appears inside the filter input whenever text is present — one click resets the filter without clearing other state
- **"New scan" button** appears in the threshold bar after a scan completes — resets all projects back to Queued instantly using cached admin project list and member data (no API re-fetch)
- **"Stop & new scan" button** appears alongside "Stop scan" during scanning — aborts the current scan and resets to Queued in a single click
- `resetScan()` uses `S.allAdminProjects` + `S._memberIds` / `S._hasMemberList` stored at hub load time — reset is immediate regardless of hub size
- `S.search` is cleared on reset so the filter input starts fresh

---

## [1.6.1] — 2026-05-01

### Fixed — Group header removed for small scans; scan button left; filter typing

- **No group accordion for ≤ 100 projects:** when `S.groups.length === 1`, `rGroup()` skips the group header and renders a flat project table directly — cleaner for targeted scans
- **Threshold bar reordered:** `[Scan N projects]` → `[🔍 filter input]` → `[Critical threshold]`
- **Filter input typing fixed:** `id="sinput"` added; `bind()` restores focus and cursor position after every re-render triggered by `oninput`, so typing is uninterrupted

---

## [1.6.0] — 2026-05-01

### Added — Load project names first, user-initiated scan

Hub selection now loads all project names immediately without starting file scanning:

- `loadHubProjects(hub)` fetches the admin project list and member IDs, then populates `S.projects` with all active projects in **Queued** state — displayed immediately in the table
- `startScan()` applies the name filter from `S.search` before scanning — non-matching projects are removed from `S.projects`; only matched projects are scanned
- **"Scan N projects"** button in the threshold bar replaces the automatic scan trigger — count reflects the active filter live
- `buildProjectList()` extracted as a shared helper used by both `loadHubProjects` and `resetScan`

---

## [1.5.9] — 2026-05-01

### Changed — Project name filter moved into threshold bar

Replaced the separate pre-scan filter screen (v1.5.8) with an inline filter input directly in the threshold bar. Flow is unchanged — hub click scans everything immediately. The name filter is now a post-scan display filter:

- **Left:** "Filter by project name" search input (filters the project list in the results)
- **Right:** Critical threshold version picker (≤ 2021 / ≤ 2022)
- Duplicate search input removed from table header — one search field, one place

---

## [1.5.8] — 2026-05-01

### Added — Project filter before scan (superseded by v1.5.9)

---

## [1.5.7] — 2026-05-01

### Fixed — Design Collaboration consumed files hidden from scan

**Root cause (ACC-TS-Campbell-Demo "No RCW" with all dashes):**

The `isRvt` filter previously rejected any item with `attributes.hidden=true`. Design Collaboration (DC) "Consumed" and managed model files carry `hidden=true` in the APS Data Management API response even though they are real, user-facing Revit files. This silently removed all DC-managed `.rvt` items before they reached Pass 2, leaving `allRvtItems=[]` and the project showing "No RCW" with all columns at `—`.

**Fixes:**

- Removed `if(i.attributes?.hidden) return false` from `isRvt`. The `isSystemName` filter already handles UUID-named system artifacts; the hidden flag is not a reliable indicator of a non-user file.
- `hasFiles` now includes `failedFiles` and `systemFiles` in the expand-button condition — so if all version calls fail or only system files are found, the expand row is still shown for debugging.
- Added console logging after BFS (`allRvtItems` count) and after Pass 2 (RCW / cloud / failed / system counts) for future diagnosis.

---

## [1.5.6] — 2026-05-01

### Fixed — Large project scan timeout race condition

**Root cause (ACC-TS-Campbell-Demo and similar large projects):**

The previous timeout used `Promise.race([scanProject(), timeout])`. When the timeout won, the outer thread moved on to `deriveProject(proj)` — but `scanProject()` was still running in the background, continuing to push files into `proj.c4rFiles`. The `deriveProject()` call therefore ran on an empty or partial `c4rFiles`, producing `uniqueC4RFiles=[]` and `c4rCount=0`. Shortly after, the background scan deposited C4R files into `c4rFiles`, making `pStatus` read `'no-version'` (c4rFiles non-empty, c4rVersion null) while the expand button and RCW file count both showed nothing (c4rCount=0).

**Fix: soft-stop timeout replacing `Promise.race`:**

- Timeout now sets a per-project `proj._timeout=true` flag instead of resolving a race
- The BFS `while` loop condition includes `&&!proj._timeout` — the moment the flag is set, no new folders are added to the queue; the current batch completes normally
- Pass 2 version fetches check `proj._timeout` and skip remaining calls, preserving all data already collected
- `scanProject()` then returns naturally — `deriveProject()` always runs on the full, consistent `c4rFiles` array

**Timeout extended from 5 min to 10 min** — gives large hubs with deeply nested folder structures more runway before the soft stop kicks in.

**Result:** Projects like ACC-TS-Campbell-Demo now correctly show their RCW file count and the expand row with all files found before the cutoff. The scan error message is preserved in the expanded row so the user knows the scan stopped early.

---

## [1.5.5] — 2026-05-01

### Fixed — Counting precision + no-version visibility

**Deduplication fix (root cause of multiple counting errors):** the `deriveProject()` deduplication previously treated a null version as `0`, so a null-version copy of a model would displace a versioned copy (e.g. 2022) as the representative in `uniqueC4RFiles`. This caused:
- Projects with DC copies to be classified as "No Version" instead of their true version tier
- Group header Outdated/Current badges to undercount
- No-version counts to be inflated by files whose version IS known via another copy

**Fix:** null version is now treated as `Infinity`. A versioned copy always beats an unresolved copy of the same model. Among versioned copies, the lowest version (worst case risk) is kept.

**Group header counts reverted to `uniqueC4RFiles`:** safe now that the deduplication treats null correctly. Design Collaboration copies of the same model are no longer double-counted in the No Version badge.

**Summary metric boxes now show file counts (not project counts):**
- Outdated and Current boxes now show RCW file counts with a project count subtitle, consistent with the Critical box. All three boxes use `uniqueC4RFiles` (deduplicated unique models).

**Hub-wide no-version file list:** a collapsible alert appears below the metric boxes whenever there are unresolved files. Lists every unresolved RCW model across the hub with project name, file name, and path — without requiring the user to expand each project row individually. Only visible after scanning completes.

**Per-project amber box updated:** uses `uniqueC4RFiles` instead of `c4rFiles` — lists genuinely unresolved models, not DC copies whose version is known via another copy.

---

## [1.5.4] — 2026-05-01

### Fixed — Group header outdated count was zero for mixed-version projects

- **Root cause:** group header file counts used `uniqueC4RFiles` (deduplicated by model GUID). The deduplication logic picks the copy with the lowest version, treating `null` as `0`. When a model had one copy with `version=null` and another with `version=2022`, the null-version copy replaced the versioned one in `uniqueC4RFiles`. The group header's filters (`f.version && ...`) then saw only the null copy and counted it as "No Version" instead of "Outdated" — causing the outdated badge to show 0 even with visible outdated files in the expanded row.
- **Fix:** group header now aggregates `c4rFiles` (all copies, including Design Collaboration duplicates) rather than `uniqueC4RFiles`. This matches what the expanded project row shows and ensures every versioned copy contributes to the correct tier bucket.

---

## [1.5.3] — 2026-05-01

### Fixed — Group header counts now reflect file totals, not project totals

- **Root cause:** the Critical / Outdated / Current / No Version badges in the group header were counting *projects* by status. Mixed-version projects (status `mixed`) contributed zero to every badge and were invisible in the totals.
- **Fix:** badges now aggregate `uniqueC4RFiles` across every project in the group and count files by version tier. A mixed project's 2021 files increment Critical; its 2024 files increment Current. Design Collaboration copies are excluded (unique model count, same as the risk table).
- Total file counts now correctly reflect every RCW model in the group regardless of whether the project is single-version, mixed, or fully unresolved.

---

## [1.5.2] — 2026-05-01

### Changed — Extended binary scan to 20 MB

- Binary header parse now tries two additional chunk sizes: **10 MB** and **20 MB** (progression: 64 KB → 256 KB → 1 MB → 5 MB → 10 MB → 20 MB). Each pass is only attempted if the previous one found nothing. Targets the small percentage of files where OLE2 sector allocation placed `BasicFileInfo` deeper than 5 MB — typically very old BIM 360 files upgraded through multiple Revit versions.
- Files that still return no version after 20 MB are marked **No Version** and listed by name + path in the expanded project row for manual follow-up in ACC.

---

## [1.5.1] — 2026-05-01

### Changed — PDF metric renamed and corrected

- **RCW version ID rate** replaces "Scan accuracy" in the PDF footer. The old metric measured version API call success (all classified `.rvt` files / total `.rvt` files) and always showed 100% when no API calls failed — even when many RCW files had unknown versions. The new metric is: `resolved C4R versions / total C4R files × 100`, clearly showing how many workshared files got a version identified across all three passes.
- PDF footer line now reads: `RCW version ID rate: N%  |  N of N workshared files identified  |  N unresolved`
- **No Version badge** added to group header summary alongside Critical / Outdated / Current, so unresolved projects are visible at a glance without expanding the group.
- **No-version file list** in expanded project row: replaced the vague "Revit year could not be determined" banner with a named file + path table for all unresolved RCW files, making manual ACC follow-up straightforward.

---

## [1.5.0] — 2026-05-01

### Changed — Fully automatic three-pass detection, no inference

- **Binary file header parse (Pass 3) is now fully automatic** — runs after the Model Derivative manifest check for every project. No button, no manual action. Scans all RCW files that still have no version, concurrently (3 files at a time).
- **Date inference removed entirely** — the fallback that estimated Revit year from `lastModifiedTime` has been dropped. Active projects on old Revit versions (e.g. Revit 2011, synced recently) were silently mis-classified as current. Every version shown is now authoritative: from the DM API, the MD manifest, or the binary file header.
- **5 MB as final chunk size** — chunk progression is now 64 KB → 256 KB → 1 MB → 5 MB (was 1 MB max). Resolves files where the OLE2 `BasicFileInfo` stream sits deep in the compound document. Covers >98% of real-world `.rvt` files.
- **Per-chunk resolution logging** — console logs a summary of how many files were resolved at each chunk size (e.g. `(64KB:81, 256KB:8, 1MB:3, 5MB:1)`) for diagnostics.
- **Version badges cleaned up** — `(file)`, `(est.)`, and `(MD)` annotations removed from the file table version column. Informational banners in the expanded project row still indicate which method resolved a file.
- **Deep Scan UI removed** — the manual Deep Scan button and group-level progress bar have been removed. All three passes are automatic and integrated into the normal scan flow.
- **No Version chip tooltip updated** — reflects that all three passes run automatically; no manual action is required.
- **About page and README updated** — three passes documented as fully automatic; inference and manual Deep Scan removed.
- **CSV Version Source updated** — `Estimated (date)` removed; values are now `API`, `Model Derivative`, or `File Header (OLE2)`.

---

## [1.4.1] — 2026-05-01

### Added — Deep Scan (OLE2 binary file header parse)
- **Deep Scan button** appears in the group header after scanning completes when RCW files remain unresolved. Reads the Revit version directly from the `.rvt` binary via OSS signed URL + HTTP Range requests — no server, no library.
- **Three-pass version detection now complete:** DM API → Model Derivative manifest → binary file header (OLE2 `BasicFileInfo` stream).
- Searches for UTF-16LE pattern `Format: YYYY` (Revit 2019+) and `Autodesk Revit YYYY` (older build strings). Progressive chunk sizes: 64 KB → 256 KB → 1 MB.
- Runs in the background (non-blocking) — other groups can continue scanning in parallel.
- Files resolved via binary header marked `(file)` in green in the table and `File Header (OLE2)` in CSV exports.
- Per-project banners: green strip for binary-resolved files; amber strip listing files that failed all three passes with a prompt to verify manually in ACC.
- `storageUrn` now captured in Pass 2 from `latest.relationships.storage.data.id`.

---

## [1.4.0] — 2026-05-01

### Changed — Manifest check scope expanded
- **Manifest check now covers estimated files** — previously only ran on C4R files with no version at all (`version===null`). Now also runs on files where version was inferred from `lastModifiedTime` (`inferredVersion===true`). This catches the high-risk case: active projects still running Revit 2011–2022 that have a recent modification date, causing the date-inference to return the wrong year. If the manifest resolves a real version, it replaces the estimate and is marked `(MD)` in the table.

### Fixed
- **Stale timeout corrupting completed project results** — the 5-minute scan timeout `setTimeout` was never cancelled when a project finished early. It would fire minutes later and write `scanError` to an already-completed project, showing a false timeout message alongside a valid scan time (e.g. "78.2s" + "Timed out after 5 minutes"). Fixed with `clearTimeout` after `Promise.race` resolves.
- **Duplicate scan error banner** — error was shown twice in the expanded project detail (once inline in the scan stats line, once as a large red banner below). The redundant bottom banner was removed.

---

## [1.3.2] — 2026-05-01

### Added — Pass 3 manifest check (automatic)
- **Automatic Model Derivative manifest scan** runs after Pass 2 for every project. Files where `revitProjectVersion` was absent from the DM API (pre-2023 BIM 360 schema) are now checked against the APS Model Derivative manifest. If the file has been translated and the manifest contains `revitProductVersion` under `Document Information`, the Revit year is extracted and applied. Marked `(MD)` in the file table and `Model Derivative` in CSV export.
- **Version Source column in CSV** replaces the `Estimated` column — now shows `API`, `Model Derivative`, or `Estimated (date)` so the data lineage is explicit in exports.
- **Deep Scan button** (stub, per group) — appears after a group finishes when unresolved files remain. Will trigger OLE2 binary file header parsing in the next release.
- **Manifest-resolved banner** — project detail shows a blue info strip when files were resolved via Model Derivative, separate from the amber `(est.)` warning for date-inferred files.
- **Updated `(est.)` warning text** — now explicitly notes that Model Derivative was checked first and suggests Deep Scan for remaining unknowns.

---

## [1.3.1] — 2026-04-30

### Fixed / Changed — CSV export
- **UTF-8 BOM added** — Excel was rendering `—` and accented characters as `â€"` / garbage because it was reading the file as Windows-1252. A BOM (`﻿`) at the start of the file signals UTF-8 to Excel explicitly.
- **Switched from `data:` URL to Blob URL** — `encodeURIComponent` on a large CSV can exceed browser URL length limits on big hubs. `URL.createObjectURL` is the correct approach; object URL is revoked after 1s.
- **Column mismatch fixed** — project summary rows (7 cols) and file detail rows (8 cols) shared the same header, so nothing aligned in the spreadsheet. The CSV now has two clearly labelled sections (`PROJECT SUMMARY` / `RCW FILE DETAIL`), each with its own header row and consistent column count.
- **Em-dash replaced with empty cell** — `—` used as a null placeholder was both causing encoding issues and making numeric columns non-sortable in Excel. All null/missing values now produce empty cells.
- **ACC Link column added** — project summary rows include the direct ACC URL for each project.
- **Estimated and Is Copy columns added** — file detail rows now include `Estimated` (Yes/No — version inferred from mod date) and `Is Copy` (Yes/No — Design Collaboration copy vs source model).
- **RCW Version(s) shows all versions** — mixed-version projects now list all distinct versions (e.g. `2021 / 2023`) instead of `MIXED`.
- **Risk column uses full tier labels** — file rows now show `Critical / Outdated / Current / No Version` instead of `Critical / OK`.

---

## [1.3.0] — 2026-04-30

### Changed
- **Rate limiter replaced with token bucket** — the previous sliding-window limiter allowed bursting to the full capacity (100 or 180 calls) then stalled for up to 60s waiting for old calls to age out. Under sustained load with 3–4 concurrent projects, a small project's calls could each wait 10–15s in that stall window, making a 3-second project take 57s.

  The token bucket refills at a steady rate (capacity ÷ 60s = ~1.67 tokens/s for 3-legged, ~3/s for 2-legged). Each call waits only until the next token accumulates — never longer. The same total throughput is maintained (same capacity ceiling, same conservative limits) but waits are spread evenly rather than concentrated in post-burst stalls. A small project's 5 calls each wait ~600ms maximum instead of 10–15s each.

  The `RateLimit.calls` sliding window is replaced by `tokens` + `lastRefill`. Three reset sites updated from `RateLimit.calls=[]` to `RateLimit.reset()`. The API load indicator in the progress bar now reads from `RateLimit.load()` (token depletion %) rather than call count.

  The APS rate limit ceiling is unchanged — this change cannot cause more 429s than before.

---

## [1.2.9] — 2026-04-30

### Changed
- **Removed verbose skipped-folders banner** — the "N top-level folders skipped (Plans, Photos, etc.)" info block has been removed from the expanded row. The compact skip count next to the scan timer (⏱ Scanned in Xs · N folders skipped) is sufficient.

---

## [1.2.8] — 2026-04-30

### Fixed
- **Per-project scan timer not recording** — `proj.scanTime` was displayed in the expanded row but never assigned. `proj.scanStart` was also never set, so the timer always showed nothing. Added `proj.scanStart = Date.now()` before `Promise.race` and `proj.scanTime = elapsed` immediately after. The expanded row now correctly shows e.g. *⏱ Scanned in 4.2s*.

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
