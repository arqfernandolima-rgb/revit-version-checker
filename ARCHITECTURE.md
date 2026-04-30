# Architecture

Technical internals for contributors and anyone extending or adapting the code.

---

## Overview

A single HTML file (~2,200 lines). HTML, CSS, and JavaScript in one file — no build step, no framework, no `node_modules`. Renders by writing strings into `innerHTML` via a single `render()` function. Two CDN-loaded jsPDF libraries are the only runtime dependencies.

```
index.html
├── <style>        CSS custom properties + all component styles
├── <body>         Static shell — topbar, sidebar, main content area
└── <script>
    ├── Version constants     VERSION, RELEASE_DATE, CHANGELOG_URL
    ├── PKCE OAuth            3-legged token acquisition + refresh
    ├── 2-legged auth         Client Secret token acquisition + refresh
    ├── refreshSessionIfNeeded()
    ├── RateLimit             Token bucket — proactive throttling
    ├── api()                 Fetch wrapper: throttle, timeout, retry, 401/429/5xx
    ├── apiAll()              Paginated fetch (follows links.next)
    ├── Scan constants        SKIP_FOLDERS, MAX_FOLDER_DEPTH, FOLDER_PRIORITY
    ├── CONCURRENCY           VERSION_FETCHES, FOLDER_FETCHES, YIELD_EVERY
    ├── GROUP_SIZE            Projects per sequential scan group
    ├── buildGroups()         Split projects into GROUP_SIZE chunks
    ├── scanProject()         BFS folder walk + version classification
    ├── pool()                Concurrency primitive
    ├── scanHub()             Top-level orchestrator
    ├── deriveProject()       Aggregate per-project stats
    ├── State (S)             Single source of truth
    ├── pStatus() / vClass()  Pure status/tier functions
    ├── Renderers             rSetup / rHubs / rDash / rGroup / rRow / rAbout
    ├── Exports               exportCsv / exportPdf / PDF_C / pdfBadge
    └── Boot                  OAuth callback / auto-login
```

---

## Key constants

```js
const GROUP_SIZE = 100;    // projects per sequential group

const CONCURRENCY = {
  VERSION_FETCHES: 8,      // parallel version API calls per project
  FOLDER_FETCHES:  3,      // parallel folder content fetches per project
  YIELD_EVERY:     5,      // yield to browser every N folder batches
};

const MAX_FOLDER_DEPTH = 4; // BFS stops at topFolder/discipline/phase/subphase

// 35+ folder names skipped at any BFS depth — no API calls made for these
const SKIP_FOLDERS = new Set([
  'plans','photos','submittals','rfis','issues','closeout','reports',
  'transmittals','markups','specifications','correspondence','meetings','contracts',
  'archive','archives','old','superseded','backup','backups','deprecated',
  'previous','history','historical','legacy','obsolete',
  'admin','administration','templates',
  'images','pictures','photos & videos','video','videos',
  'documents','doc','docs','sheets','drawings published',
  'incoming','outgoing','for review','for approval','for information',
  'issued','superseded drawings','record drawings',
]);

// Scan order within each BFS level — discipline folders before archive/admin
const FOLDER_PRIORITY = (name) => {
  // 0 = Project Files, 1 = Shared, 2 = discipline names, 3 = everything else
};
```

---

## State

All application state lives in `S`. Nothing is mutated outside clearly labelled functions. `render()` is always called after any state mutation.

```js
const S = {
  // Auth
  phase: 'setup',          // 'setup'|'hubs'|'dash'|'about'
  token: '',               // APS access token
  authMode: '3-legged',    // '3-legged'|'2-legged'
  clientId: '',
  clientSecret: '',        // 2-legged only — JS heap, never stored
  refreshToken: null,      // 3-legged only — from offline_access scope
  tokenIssuedAt: null,     // Date.now() when token was issued
  tokenExpired: false,     // set true on 401; cleared on new sign-in
  uname: '', userId: '',

  // Hub + projects
  hubs: [], hub: null,
  projects: [],            // flat array — all groups reference slices
  groups: [],              // group metadata (see below)
  currentGroup: null,      // group currently scanning

  // Scan progress
  scanning: false, aborted: false,
  pct: 0, pdone: 0, ptotal: 0,
  msg: '', scanStart: null, eta: null,
  activeNames: new Set(),  // unused directly — group.activeNames is used

  // UI
  expanded: new Set(),
  filterOpen: false, search: '',
  statusFilters: new Set(),
  sortCol: 'risk', sortDir: -1,
  threshold: 2021,
  error: null,
};
```

### Group object

```js
{
  id: 'g0',
  label: 'Group 1 · Projects 1–100',
  start: 0, end: 100,       // slice indices into S.projects
  status: 'pending',        // 'pending'|'scanning'|'done'
  expanded: true,
  pdone: 0, ptotal: 100,
  scanStart: null, eta: null,
  activeNames: new Set(),
}
```

### Project object

```js
{
  id: 'b.<uuid>',           // DM API format — b. prefix required
  adminId: '<uuid>',        // bare UUID for Admin API
  name: 'Project Name',
  accUrl: 'https://acc.autodesk.com/docs/files/projects/<uuid>',
  pending: true,            // queued, not yet picked up by a worker
  scanning: false,          // actively being scanned
  noAccess: false,          // 403 on topFolders
  scanError: null,
  c4rFiles: [],             // Cloud Workshared .rvt files (all copies)
  cloudFiles: [],           // Non-workshared .rvt files
  failedFiles: [],          // .rvt files where version call failed
  systemFiles: [],          // GUID-named files (DA outputs, conflict backups) — not counted toward risk
  uniqueC4RFiles: [],       // deduplicated c4rFiles by modelGuid — populated by deriveProject()
  c4rVersion: null,         // dominant version: string | 'MIXED' | null
  c4rVersions: [],
  c4rCount: 0,              // unique RCW model count (not file copy count)
  c4rCopyCount: 0,          // number of Design Collaboration copies excluded from c4rCount
  cloudCount: 0,
  skippedFolders: 0,        // 403'd subfolders
  skippedTopFolders: 0,     // structurally skipped (SKIP_FOLDERS match)
  lastMod: null,
  scanTime: null,           // seconds from pick-up to deriveProject() completion
  // NOTE: atRisk is NEVER stored — always computed via atRiskCount(p)
}
```

### File object (c4rFiles / cloudFiles entries)

```js
{
  id: 'urn:...',
  name: 'Architecture.rvt',
  path: 'Project Files / Architecture',
  version: '2021',          // String year, or null if unknown
  isC4R: true,
  modTime: '2021-03-11T...',
  modBy: 'John Janzen',
  inferredVersion: false,   // true when year came from modTime, not revitProjectVersion
  modelGuid: 'e5a59497-...', // from extension.data.modelGuid — shared across Design Collaboration copies
  isCopy: false,            // true if another file in the same project shares this modelGuid
}
```

---

## Authentication

### 3-legged PKCE flow

```
Browser                              Autodesk
  │  1. Generate code_verifier + code_challenge
  │  2. Store verifier + clientId in sessionStorage
  │── GET /authorize?scope=data:read account:read offline_access ──▶│
  │◀── redirect with ?code= ──────────────────────────────────────────│
  │── POST /token (grant_type=authorization_code) ──────────────────▶│
  │◀── { access_token, refresh_token, expires_in: 3600 } ────────────│
  │  3. Store access_token in localStorage
  │  4. Store refresh_token in S.refreshToken (memory only)
  │  5. S.tokenIssuedAt = Date.now()
```

### 2-legged client_credentials flow

```
Browser                              Autodesk
  │  1. User enters clientId + clientSecret
  │  2. Secret stored in S.clientSecret (heap only), DOM input cleared
  │── POST /token (grant_type=client_credentials) ────────────────▶│
  │◀── { access_token, expires_in: 3600 } ──────────────────────────│
  │  3. Store access_token in sessionStorage (tab-scoped)
  │  4. S.tokenIssuedAt = Date.now()
  │  5. S.clientSecret retained in memory for auto-refresh
```

### Token refresh

`refreshSessionIfNeeded(force=false)` is called before each group (except the first). Guards: if token age < 50 minutes AND `force=false`, returns immediately.

- **2-legged:** calls `client_credentials` with stored `clientId`/`clientSecret`. ~200ms, silent.
- **3-legged with refresh token:** calls `refresh_token` grant. ~300ms, silent. Updates `S.refreshToken` if APS returns a new one (rolling refresh).
- **3-legged without refresh token:** continues; `api()` 401 handler is the backstop.
- **Force mode:** called by the `api()` 401 handler to attempt recovery mid-scan — bypasses the age check.

---

## API layer

### `api(path, retries=3)`

Every API call flows through `api()`:

1. `RateLimit.throttle()` — waits if the 60s call window is at capacity
2. `AbortController` + `setTimeout(30s)` — cancels hung connections after 30 seconds
3. **401** → `refreshSessionIfNeeded(force=true)` + retry once; if fails → abort scan
4. **429** → exponential backoff: 1.5s, 3s, 6s
5. **5xx** → linear retry: 1s, 2s, 3s
6. **403/404** → throw immediately (permanent, caller handles)

### `apiAll(path)`

Follows `links.next` pagination. Uses `new URL(href, APS)` for robust URL normalisation. Returns partial results on transient error rather than discarding everything.

### Rate limiter

```js
const RateLimit = {
  calls: [],
  limit() { return S.authMode === '2-legged' ? 180 : 100; }, // conservative limits
  async throttle() {
    // Prune calls older than 60s
    // If at limit: wait until oldest call ages out (exact, not approximate)
    // Record call timestamp
  }
};
```

- **3-legged:** 100/min (actual APS limit ~150/min)
- **2-legged:** 180/min (actual APS limit ~250/min)
- `RateLimit.calls` reset at start of each scan and after each token refresh

---

## Scan engine

### Groups

`buildGroups(projects)` slices `S.projects` into `GROUP_SIZE=100` chunks. Groups scan **sequentially**; within each group `pool(tasks, projConcurrency)` runs concurrently:
- 3-legged: `projConcurrency = 3`
- 2-legged: `projConcurrency = 4`

Sequential groups give:
- Progressive results — Group 1 is exportable while Group 2 scans
- Resilience — abort in Group 3 preserves Groups 1–2
- Natural token refresh points — refresh fires between groups
- Bounded API load — at most `projConcurrency` projects compete simultaneously

### `pool(tasks, limit)`

```js
async function pool(tasks, limit) {
  let next = 0;
  async function worker() {
    while (next < tasks.length) {
      const i = next++;
      await tasks[i]();
      await new Promise(res => setTimeout(res, 0)); // yield — prevents no-access races
    }
  }
  await Promise.all(Array.from({ length: Math.min(limit, tasks.length) }, worker));
}
```

The `setTimeout(0)` yield ensures all workers get scheduled before any worker races through synchronous (no-access) tasks.

### `scanProject(proj, hub)`

#### Folder skip and priority setup

```
topFolders API call
  └─ Filter: SKIP_FOLDERS removes known non-RVT containers
  └─ Sort: FOLDER_PRIORITY(name) → 0 (Project Files) … 3 (other)
  └─ Build initial queue with depth=1
```

#### Pass 1 — Parallel BFS

```
while queue not empty:
  batch = queue.splice(0, FOLDER_FETCHES)   // take up to 3
  await Promise.all(batch.map(async entry =>
    apiAll(contents URL)                    // plain folder contents fetch
    ├─ Subfolders:
    │    Skip if in visitedFolders (duplicate guard)
    │    Skip if name in SKIP_FOLDERS
    │    Skip if depth >= MAX_FOLDER_DEPTH
    │    Add to batchSubFolders with priority
    └─ .rvt items:
         Skip if attributes.hidden === true  (ACC system/derivative artifacts)
         If isSystemName(name) → push to proj.systemFiles[] (not counted toward risk)
         Otherwise → push to allRvtItems (Pass 2)
  ))
  batchSubFolders.sort by priority → queue.unshift (depth-first, priority-ordered)
```

JavaScript is single-threaded: post-`await` synchronous code within each batch item executes atomically, so `visitedFolders` and `batchSubFolders` mutations are race-free.

> **Why not `?include=version`?** Appending `?include=version` forces APS to resolve version metadata server-side for every item in the folder — including PDFs, DWGs, images. On a folder with 200 non-Revit files this pushes response time well past the 30s per-request timeout. The plain `apiAll` call is reliable; version data is fetched selectively in Pass 2 only for `.rvt` files.

#### Version inference (both inline and Pass 2 paths)

When `revitProjectVersion` is absent from the API response (pre-2023 BIM 360 schema):

```
modYear = new Date(lastModifiedTime).getFullYear()
if modYear ≤ threshold+2:
  version = String(modYear)
  inferredVersion = true
else:
  version = null  // cannot safely assume Current — leave as No Version
```

This covers:
- Files modified ≤ threshold → Critical (e.g. 2021 → Critical with threshold=2021)
- Files modified threshold+1 or +2 → Outdated (e.g. 2022/2023 with threshold=2021)
- Files modified before 2020 → Critical (always ≤ any threshold we use)
- Files modified after threshold+2 → left as No Version (manual review required)

#### Pass 2 — Version fetches

```
pool(allRvtItems, VERSION_FETCHES=8):
  api(/items/{id}/versions)
    └─ versions[0].extension.type → .includes('c4r') → isC4R
    └─ versions[0].extension.data.revitProjectVersion → year
    └─ if absent: apply date inference from lastModifiedTime
    └─ versions[0].extension.data.modelGuid → stored for deduplication
  Push to c4rFiles, cloudFiles, or failedFiles
```

**Why every `.rvt` item gets a version call:** Older BIM 360 files (2019–2022) commonly carry `items:autodesk.bim360:File` at the item level even when Cloud Workshared. Item-level type is not reliable. The version-level `extension.type` is authoritative — no item-level shortcuts are taken.

---

## `pStatus(p)` — status taxonomy

```js
function pStatus(p) {
  if (p.pending)                                      return 'pending';
  if (p.scanning)                                     return 'scanning';
  if (p.noAccess)                                     return 'no-access';
  if (p.c4rFiles.length > 0 && !p.c4rVersion)        return 'no-version';
  if (!p.c4rVersion)                                  return 'no-c4r';
  if (p.c4rVersion === 'MIXED')                       return 'mixed';
  const v = parseInt(p.c4rVersion);
  if (v <= S.threshold)                               return 'critical';
  if (v <= S.threshold + 2)                           return 'warn';
  return 'ok';
}
```

The `no-version` status fires when a project has confirmed C4R files but no version year — either none were inferable from modification date, or all inferences produced years > threshold+2.

---

## Rendering

`render()` rewrites `document.getElementById('mc').innerHTML` on every state change. `bind()` re-attaches event handlers. `renderProgress()` is a fast path updating 5 stable DOM IDs — called by the 600ms scan interval without a full page rebuild.

**Version badge:** lives in static HTML (outside JS template literals), so `${VERSION}` cannot be interpolated by the browser. `render()` sets `href` and `textContent` from JS constants on first call, guarded by a `_set` flag.

---

## PDF export

### Colour palette

`PDF_C` mirrors the screen CSS variables:

```js
const PDF_C = {
  blue:[20,115,230], blueHdr:[8,60,155],
  red:[220,38,38],   redBg:[255,247,247],
  amber:[217,119,6], amberBg:[255,252,242],
  green:[13,125,95], greenBg:[244,253,249],
  purple:[91,79,207],purpleBg:[248,246,255],
};
```

### Row rendering

`didParseCell` sets `fillColor` on every cell based on `pStatus(p)`. `didDrawCell` draws:
- Rounded-rect status badge pill (status column)
- 2.5mm red left accent bar (critical rows, project name column)
- Blue "Open ACC" button with `doc.link()` annotation (link column)

**Critical invariant:** both callbacks must index `filteredProjs[data.row.index]`, not `S.projects[data.row.index]`. These diverge whenever a filter is active.

### Column widths

Landscape A4 = 297mm, 14mm margins each side → **269mm** usable.

Main table: `100 + 20 + 18 + 15 + 14 + 28 + 38 + 36 = 269` ✓

Detail table: `88 + 18 + 24 + 84 + 30 + 25 = 269` ✓

Column widths must sum to exactly 269 after any change.

### Unicode safety

jsPDF Helvetica WinAnsi: safe characters include `\u2014` (—), `\u2264` (≤), `\u2265` (≥), `\u00b7` (·). `\u2192` (→) renders as "!" — never use in PDF text. Draw shapes instead.

---

## Key invariants

These must be preserved across all changes:

| Invariant | Why |
|---|---|
| `atRisk` never stored on project | `atRiskCount(p)` recomputes dynamically so threshold slider is instant |
| `pStatus(p)` is always pure | No side effects — called in render loops, filter, sort, export |
| `didParseCell`/`didDrawCell` use `filteredProjs[row.index]` | Active filter makes `S.projects` indices wrong |
| C4R detection uses **version-level** `extension.type` | Item-level type is unreliable for pre-2023 BIM 360 files |
| No item-level type shortcuts in Pass 2 | `isDefinitelyRC` caused false negatives for older BIM 360 C4R files |
| `latest` declared outside the `try{}` block | Keeps `modelGuid` in scope after the try-catch; pool silently swallows ReferenceErrors |
| `S.clientSecret` never written to any storage | Security invariant — heap memory only |
| PDF column widths sum to 269mm | Layout breaks if mismatched |
| `inferredVersion=true` whenever version came from modTime | UI must show `(est.)` label to be transparent |
| `c4rCount` uses `uniqueC4RFiles`, not `c4rFiles` | Design Collaboration copies inflate raw count; risk assessment uses unique models |

---

## Adding a new table column

1. Add `th()` call in `rGroup()` thead — add sort case if sortable
2. Add value to `rRow()` td cells
3. In `exportPdf`: add to `head:`, `body:`, and `columnStyles`; verify both width sums are still 269
4. In `exportCsv`: add to the CSV row string

---

## Deployment checklist

- [ ] APS app type matches auth mode (SPA for 3-legged, Traditional/Service for 2-legged)
- [ ] Callback URL exactly matches GitHub Pages URL (no trailing slash) — 3-legged only
- [ ] App registered as Custom Integration in ACC Hub Admin
- [ ] `offline_access` scope granted automatically for 3-legged
- [ ] `index.html` at repo root
- [ ] GitHub Pages: `main` branch, root folder
