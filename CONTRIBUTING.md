# Contributing

Thanks for improving this tool. Read this before opening a pull request.

---

## Ground rules

**Keep it a single file.** The zero-build-step design is intentional — the tool deploys by copying one file to GitHub Pages and is auditable in its entirety. Do not introduce a bundler, framework, `package.json`, or additional files beyond docs.

**Keep it read-only.** The app must never write, modify, or delete anything in ACC. Any PR adding a write operation will not be merged.

**Keep the Client Secret out of storage.** `S.clientSecret` must only ever live in the JavaScript heap. It must not be written to `localStorage`, `sessionStorage`, the DOM, URL parameters, or any network request other than the APS token exchange.

**Test in demo mode first.** Click **Try demo data** and verify your change looks correct across all status tiers (Critical, Outdated, Current, Mixed, No RCW, No Access, Queued, Scanning) before testing against a real hub.

---

## Running locally

```bash
# Simple static server — Python 3
python3 -m http.server 8080
# then open http://localhost:8080
```

For the 3-legged sign-in flow to work locally, register `http://localhost:8080` as a Callback URL in your APS app. For 2-legged you can open `index.html` directly from the filesystem.

---

## What is in scope

- Bug fixes in scanning, classification, or API handling
- Resilience and rate-limit improvements
- UI and accessibility improvements
- PDF and CSV export improvements
- Performance improvements to the scan engine
- Better error messages and edge-case handling
- Documentation improvements

## What is out of scope

- Write operations against ACC or APS
- Storing the Client Secret in any persistent medium
- Adding a build system, bundler, or non-CDN dependencies
- Changing the rendering model (innerHTML + `render()`)
- Server-side components of any kind

---

## Before opening a PR

**Correctness:**
- `didParseCell` must use `filteredProjs[data.row.index]`, not `S.projects[data.row.index]` — these diverge when a filter is active
- C4R detection must use the **version-level** `extension.type` — item-level type is unreliable for older BIM 360 files
- `atRiskCount(p)` must remain dynamic (computed at render time from `p.c4rFiles`) — never store `atRisk` on the project object
- `pStatus(p)` must remain a pure function with no side effects — it is called repeatedly in render loops

**PDF:**
- Column widths in both tables must sum to exactly **269mm** (landscape A4, 14mm margins each side)
- Main: `100+20+18+15+14+28+38+36 = 269`
- Detail: `88+18+24+84+30+25 = 269`
- Do not use `\u2192` (→) in PDF text — it renders as "!" in jsPDF Helvetica. Use `\u2265`, `\u2264`, `\u2014`, `\u00b7` or plain ASCII

**Security:**
- `S.clientSecret` must never be assigned to any variable that gets serialised, logged, or persisted
- Do not add any code that reads `S.clientSecret` for purposes other than the token exchange

**Testing checklist:**
- [ ] Demo mode works and all 7 status tiers display correctly
- [ ] PDF export with active filter: correct project count, correct row colours
- [ ] PDF export per group: only that group's projects appear
- [ ] CSV export with active filter: matches table
- [ ] Threshold slider change updates status chips, filter labels, metric cards, and PDF instantly
- [ ] Sort by all sortable columns works correctly (including MIXED and null versions)
- [ ] Sign-out clears all state including `refreshToken` and `tokenIssuedAt`

---

## Code style

- Vanilla JS — no TypeScript, no JSX
- All state in `S`. No module-level variables acting as state
- Call `render()` after every state mutation
- Descriptive names for module-scope functions; short names are fine inside tight loops
- Comments explain *why*, not *what*

---

## Reporting issues

Include:
- Browser and version
- Auth mode (3-legged or 2-legged)
- Whether it reproduces in demo mode
- For API errors: status code, endpoint path, error message from the UI or expanded row
- For scan inaccuracy: project name, expected files vs found files, whether the project shows skipped folders
