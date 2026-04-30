# Architecture

Technical internals for contributors and anyone extending or adapting the code.

---

## Overview

A single HTML file (~1,900 lines). HTML, CSS, and JavaScript in one file — no build step, no framework, no `node_modules`. Renders by writing strings into `innerHTML` via a single `render()` function. Two CDN-loaded jsPDF libraries are the only runtime dependencies.

```
index.html
├── <style>        CSS custom properties + all component styles
├── <body>         Static shell — topbar, sidebar, main content area
└── <script>
    ├── Version constants
    ├── PKCE OAuth        3-legged token acquisition
    ├── 2-legged auth     Client Secret token acquisition
    ├── Token refresh     refreshSessionIfNeeded()
    ├── Rate limiter      RateLimit token bucket
    ├── api() / apiAll()  Fetch wrappers with retry, timeout, pagination
    ├── Scan engine       buildGroups / scanProject / BFS / pool
    ├── State (S)         Single source of truth
    ├── Renderers         rSetup / rHubs / rDash / rGroup / rRow / rAbout
    ├── Exports           exportCsv / exportPdf
    └── Boot              OAuth callback / auto-login
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
  clientId: '',            // APS Client ID (localStorage)
  clientSecret: '',        // 2-legged only — JS heap, never stored
  refreshToken: null,      // 3-legged only — from offline_access scope
  tokenIssuedAt: null,     // Date.now() when token was issued
  tokenExpired: false,     // set true on 401; cleared on new sign-in
  uname: '',
  userId: '',

  // Hub + projects
  hubs: [],
  hub: null,
  projects: [],            // flat array, all groups reference into this
  groups: [],              // group metadata objects (see below)
  currentGroup: null,      // group currently being scanned

  // Scan progress
  scanning: false,
  aborted: false,
  pct: 0,                  // overall 0–100
  pdone: 0,                // total projects completed
  ptotal: 0,               // total projects
  msg: '',                 // current activity string
  activeNames: new Set(),  // not used directly — group.activeNames is used
  scanStart: null,
  eta: null,

  // UI
  expanded: new Set(),     // project IDs with expanded file detail
  filterOpen: false,
  search: '',
  statusFilters: new Set(),
  sortCol: 'risk',
  sortDir: -1,
  threshold: 2021,
  error: null,
};
```

### Group object

```js
{
  id: 'g0',
  label: 'Group 1 · Projects 1–200',
  start: 0,                // index into S.projects (inclusive)
  end: 200,                // exclusive
  status: 'pending',       // 'pending'|'scanning'|'done'
  expanded: true,
  pdone: 0,
  ptotal: 200,
  scanStart: null,
  eta: null,
  activeNames: new Set(),  // names of projects currently scanning in this group
}
```

### Project object

```js
{
  id: 'b.<uuid>',          // DM API format — b. prefix required
  adminId: '<uuid>',       // bare UUID for Admin API
  name: 'Project Name',
  status: 'active',
  accUrl: 'https://acc.autodesk.com/docs/files/projects/<uuid>',
  pending: true,           // queued but not yet picked up by a worker
  scanning: false,         // actively being scanned right now
  noAccess: false,         // 403 on topFolders — pre-marked or detected at scan
  scanError: null,         // string if project-level error occurred
  c4rFiles: [],            // Cloud Workshared .rvt files
  cloudFiles: [],          // Non-workshared .rvt files
  failedFiles: [],         // .rvt files where version call failed
  c4rVersion: null,        // dominant version: string | 'MIXED' | null
  c4rVersions: [],         // all distinct versions
  c4rCount: 0,
  cloudCount: 0,
  skippedFolders: 0,       // 403'd subfolders — contents unknown
  lastMod: null,
  // NOTE: atRisk is NOT stored. Always computed via atRiskCount(p).
}
```

---

## Authentication

### 3-legged PKCE flow

```
Browser                              Autodesk
  │                                     │
  │  1. Generate code_verifier (64 random bytes, base64url)
  │  2. code_challenge = base64url(SHA-256(verifier))
  │  3. Store verifier + clientId in sessionStorage
  │                                     │
  │── GET /authorize?...&code_challenge=...&scope=data:read account:read offline_access ──▶│
  │◀── redirect to callback URL with ?code= ────────────────────────────────────────────│
  │                                     │
  │  4. Exchange code + verifier for token
  │── POST /token (grant_type=authorization_code) ──────────────────────────────────────▶│
  │◀── { access_token, refresh_token, expires_in: 3600 } ───────────────────────────────│
  │                                     │
  │  5. Store access_token in localStorage
  │  6. Store refresh_token in S.refreshToken (memory)
  │  7. Record S.tokenIssuedAt = Date.now()
```

The `offline_access` scope causes APS to include a `refresh_token` in the response. This is used by `refreshSessionIfNeeded()` between scan groups.

### 2-legged client_credentials flow

```
Browser                              Autodesk
  │                                     │
  │  1. User types clientId + clientSecret into form
  │  2. Secret is read from DOM, stored in S.clientSecret (heap only)
  │  3. DOM input is immediately cleared
  │                                     │
  │── POST /token (grant_type=client_credentials) ─────────────────────────────────────▶│
  │◀── { access_token, expires_in: 3600 } ──────────────────────────────────────────────│
  │                                     │
  │  4. Store access_token in sessionStorage (tab-scoped)
  │  5. Record S.tokenIssuedAt = Date.now()
  │  6. S.clientSecret remains in memory for auto-refresh
```

**Security model:** The Client Secret never leaves the browser except in the POST body to `developer.api.autodesk.com` over HTTPS. It is never written to `localStorage`, `sessionStorage`, URL parameters, or the DOM after initial use. It persists in JavaScript heap memory (`S.clientSecret`) until the tab closes. This is equivalent in risk to a password manager auto-filling a form — acceptable on a personal machine, unacceptable on a shared machine.

### Token refresh

`refreshSessionIfNeeded()` is called before each scan group (except the first). It checks `S.tokenIssuedAt`; if the token is older than 50 minutes it refreshes:

- **2-legged:** calls `client_credentials` again with `S.clientId` and `S.clientSecret`. Silent, ~200ms.
- **3-legged with refresh token:** calls `refresh_token` grant. Silent, ~300ms. APS may return a new refresh token (rolling); if so, `S.refreshToken` is updated.
- **3-legged without refresh token:** logs a warning and continues. The 401 handler in `api()` is the backstop — it sets `S.aborted=true` and renders an error if the token does expire.

---

## API layer

### `api(path, retries=3)`

Every API call goes through `api()`, which provides:

1. **Proactive rate throttling** — `RateLimit.throttle()` checks the 60s call window before each request
2. **30-second AbortController timeout** — `fetch()` has no built-in timeout; hung connections abort after 30s and retry
3. **401 detection** — immediately sets `S.tokenExpired=true`, `S.aborted=true`, clears the scan timer, renders error
4. **429 backoff** — waits `1500 × 2^attempt` ms before retrying (1.5s, 3s, 6s)
5. **5xx retry** — waits `1000 × (attempt+1)` ms before retrying (1s, 2s, 3s)
6. **403/404** — throws immediately with tagged `.status` for callers to distinguish no-access from transient errors

### `apiAll(path)`

Follows `links.next` pagination for DM API folder contents. Uses `new URL(href, APS)` for robust URL normalisation — handles relative URLs, different host prefixes, and query parameters correctly. On transient error mid-pagination, returns items collected so far rather than discarding everything.

### Rate limiter

```js
const RateLimit = {
  calls: [],           // timestamps of calls in the last 60s
  limit() { return S.authMode === '2-legged' ? 180 : 50; },
  async throttle() {
    // prune old calls
    // if calls.length >= limit: wait until oldest call ages out
    // push Date.now() to calls
  }
};
```

Conservative limits (90% of actual APS limits) prevent cascade stalls. The 2-legged limit is higher because app tokens have separate rate limit quotas from user tokens.

---

## Scan engine

### Groups

`buildGroups(projects)` splits `S.projects` into chunks of `GROUP_SIZE=200`. Groups scan **sequentially** in a `for...of` loop. Within each group, `pool(tasks, projConcurrency)` runs 3 projects (3-legged) or 4 (2-legged) in parallel.

Sequential groups provide:
- Progressive results — Group 1 is exportable while Group 2 scans
- Natural token refresh points — refresh happens between groups when the hub is idle
- Resilience — a crash or abort in Group 3 preserves Groups 1–2 results
- Browser responsiveness — only `projConcurrency` projects ever compete for resources simultaneously

### `pool(tasks, limit)`

```js
async function pool(tasks, limit) {
  const results = new Array(tasks.length);
  let next = 0;
  async function worker() {
    while (next < tasks.length) {
      const i = next++;
      try { results[i] = await tasks[i](); } catch(e) { results[i] = undefined; }
      await new Promise(res => setTimeout(res, 0)); // yield between tasks
    }
  }
  await Promise.all(Array.from({ length: Math.min(limit, tasks.length) }, worker));
  return results;
}
```

The `setTimeout(0)` yield between tasks ensures workers alternate fairly. Without it, a worker processing synchronous (no-access) tasks can consume the entire queue before parallel workers start real scan work.

### `scanProject(proj, hub)`

Two-pass design:

**Pass 1 — BFS folder walk, zero version calls**

```
topFolders
  └─ BFS queue [{id, path}, ...]
       └─ For each folder (yield every 5 via setTimeout(0)):
            apiAll() → folder contents (paginated, handles >200 items)
              ├─ subfolders → push to queue (visitedFolders Set prevents loops)
              └─ .rvt items → push to allRvtItems
```

All `.rvt` items are collected first. The BFS is uninterrupted by version API calls.

**Pass 2 — version fetches, `pool(items, 5)`**

```
For each .rvt item:
  Conservative RC skip check (if ≥3 C4R confirmed AND item type is known-RC type → skip)
  api(/items/{id}/versions)
    └─ versions[0].attributes.extension.type → .includes('c4r') → isC4R
    └─ versions[0].attributes.extension.data.revitProjectVersion → year
  → push to proj.c4rFiles or proj.cloudFiles or proj.failedFiles
```

**Why version-level C4R detection is critical:** Older BIM 360 files (2019–2022) frequently have `items:autodesk.bim360:File` as their item-level `extension.type` even when they are Cloud Workshared. Using item-level type as an exclusion filter causes false negatives — missed RCW files — for exactly the files this tool exists to find. The version response is always authoritative.

**Conservative RC skip:** After ≥3 confirmed C4R files have been version-checked in a project (proving item-type taxonomy is reliable for this hub), items with known definitively-non-C4R item types (`bim360:file`, `bim360:document`, `bim360:transmittal`) skip the version call and are added to `cloudFiles` directly from item attributes. This saves calls on RC-heavy projects without accuracy risk.

---

## Rendering

`render()` rewrites `document.getElementById('mc').innerHTML` on every state change. `bind()` re-attaches event handlers after each render. `renderProgress()` is a fast path that only writes to 5 stable DOM elements — called by the 600ms `setInterval` during scanning to update progress without a full page rebuild.

There is no virtual DOM, no diffing. For the scale of data this tool handles (hundreds of projects), full re-render is fast enough and the simplicity is worth it.

**Version badge** is a special case: it is in the static HTML `<body>` (not in a JS template literal), so `${VERSION}` cannot be interpolated directly. `render()` sets the badge `href` and `textContent` from the JS constants on first call, guarded by a `_set` flag so it only runs once.

---

## PDF export

### Color palette

All colors are defined in `PDF_C` and mirror the screen CSS custom properties:

```js
const PDF_C = {
  blue: [20,115,230], blueHdr: [8,60,155],
  red: [220,38,38], redBg: [255,247,247],
  amber: [217,119,6], amberBg: [255,252,242],
  green: [13,125,95], greenBg: [244,253,249],
  purple: [91,79,207], purpleBg: [248,246,255],
  // ...
};
```

### Row color coding

`didParseCell` sets `fillColor` on every cell of a row based on `pStatus(p)`. The status cell gets a distinct solid fill (the badge color) with white text. `didDrawCell` draws the rounded-rect status badge and the "Open ACC" link button.

**Critical:** `didParseCell` and `didDrawCell` must index into `filteredProjs[data.row.index]`, not `S.projects[data.row.index]`. When a filter is active, row 0 in the PDF is not necessarily project 0 in `S.projects`.

### Column width budget

Landscape A4 = 297mm. With 14mm margins, usable width = **269mm**.

Main table: `100 + 20 + 18 + 15 + 14 + 28 + 38 + 36 = 269`

Detail table: `88 + 18 + 24 + 84 + 30 + 25 = 269`

### Unicode in Helvetica

jsPDF uses WinAnsi encoding. Safe characters: basic Latin, `\u2014` (—), `\u2264` (≤), `\u2265` (≥), `\u00b7` (·). **Not safe:** `\u2192` (→) — renders as "!" in most PDF viewers. Use plain ASCII or drawn shapes instead.

---

## Key invariants

These must be preserved across all changes:

- `atRisk` is **never stored** — always computed via `atRiskCount(p)` so the threshold slider updates instantly
- `pStatus(p)` is **always recomputed** — never cached — so threshold changes instantly reclassify everything
- `didParseCell` must use `filteredProjs[row.index]`, never `S.projects[row.index]`
- C4R detection must use **version-level** `extension.type`, not item-level
- The Client Secret (`S.clientSecret`) must never be written to any persistent storage
- Column widths in both PDF tables must sum to exactly **269mm**

---

## Adding a new table column

1. Add `th()` call to `rGroup()` thead — if sortable, add a case to the sort comparator
2. Add the value to `rRow()` td cells
3. In `exportPdf`: add the column to `head:` and `body:`, add a `columnStyles` entry, verify widths still sum to 269
4. In `exportCsv`: add the column to the CSV row

---

## Deployment checklist

- [ ] APS app type matches auth mode (SPA for 3-legged, Traditional/Service for 2-legged)
- [ ] Callback URL exactly matches GitHub Pages URL (no trailing slash) — 3-legged only
- [ ] App registered as Custom Integration in ACC Account Admin
- [ ] `offline_access` scope granted (automatic since v1.0.0 for 3-legged)
- [ ] `index.html` at repo root
- [ ] GitHub Pages source: `main` branch, root folder
