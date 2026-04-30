# Contributing

Thanks for improving this tool. Read this before opening a pull request.

---

## Ground rules

**Keep it a single file.** The zero-build-step design is intentional — the tool deploys by copying one file to GitHub Pages and is auditable in its entirety. Do not introduce a bundler, framework, `package.json`, or additional files beyond docs.

**Keep it read-only.** The app must never write, modify, or delete anything in ACC. Any PR adding a write operation will not be merged.

**Keep the Client Secret out of storage.** `S.clientSecret` must only ever live in the JavaScript heap. It must not be written to `localStorage`, `sessionStorage`, the DOM, URL parameters, or any network request other than the APS token exchange.

**Test in demo mode first.** Click **Try demo data** and verify your change looks correct across all status tiers before testing against a real hub.

---

## Running locally

```bash
python3 -m http.server 8080
# then open http://localhost:8080
```

For the 3-legged sign-in flow, register `http://localhost:8080` as a Callback URL in your APS app. For 2-legged, open `index.html` directly from the filesystem.

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

### Correctness

- `didParseCell` and `didDrawCell` must use `filteredProjs[data.row.index]`, not `S.projects[data.row.index]` — these diverge when a filter is active
- C4R detection must use the **version-level** `extension.type` — item-level type is unreliable for older BIM 360 files
- `atRiskCount(p)` must remain dynamic (recomputed at render time) — never store `atRisk` on the project object
- `pStatus(p)` must remain a pure function with no side effects
- When adding a new version inference path, always set `inferredVersion=true` so the UI can show `(est.)` and users know the year is estimated

### Scan engine

- Folder skip list (`SKIP_FOLDERS`) should only contain names that are **structurally** safe to skip at any depth — not names that might contain RCW models in unusual setups
- `MAX_FOLDER_DEPTH=4` covers all documented ACC structures; increase only with documented justification
- `SKIP_THRESHOLD=1` — after just 1 confirmed C4R file, item-level RC type is trusted for skipping version calls

### PDF

- Column widths in both tables must sum to exactly **269mm** (landscape A4, 14mm margins each side)
  - Main: `100+20+18+15+14+28+38+36 = 269`
  - Detail: `88+18+24+84+30+25 = 269`
- Do not use `\u2192` (→) in PDF text — renders as "!" in jsPDF Helvetica. Use `\u2265`, `\u2264`, `\u2014`, `\u00b7` or plain ASCII

### Security

- `S.clientSecret` must never be assigned to any variable that gets serialised, logged, or persisted
- Do not add code that reads `S.clientSecret` for purposes other than the token exchange and token refresh

---

## Testing checklist

- [ ] Demo mode: all 7 status tiers display correctly (Critical, Outdated, Current, Mixed, No RCW, No Access, No Version)
- [ ] Files with `inferredVersion=true` show `(est.)` label in expanded row
- [ ] Amber banner appears in expanded row when any file has `inferredVersion=true`
- [ ] PDF export with active filter: correct project count, correct row colours
- [ ] PDF export per group: only that group's projects appear
- [ ] CSV export with active filter: matches table
- [ ] Threshold slider change: status chips, filter labels, metric cards, and PDF all update instantly
- [ ] Sort by all sortable columns (including MIXED and null versions)
- [ ] Sign-out clears all state including `refreshToken` and `tokenIssuedAt`
- [ ] No Version projects appear in filter dropdown and can be exported

---

## Code style

- Vanilla JS — no TypeScript, no JSX
- All state in `S` — no module-level mutable variables
- Call `render()` after every state mutation
- Comments explain *why*, not *what*

---

## Reporting issues

Include:
- Browser and version
- Auth mode (3-legged or 2-legged)
- Whether it reproduces in demo mode
- For API errors: status code, endpoint path, error message from expanded row
- For scan inaccuracy: project name, expected files vs found files, whether files show `(est.)` version
