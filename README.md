# Revit Version Checker

[![Version](https://img.shields.io/badge/version-1.2.1-blue)](https://github.com/tsb2127/revit-version-checker/releases/tag/v1.2.1)
[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)

**Scan every project in an Autodesk Forma / ACC hub for Revit cloud worksharing deprecation risk — no server required.**

A single-file HTML app that runs entirely in the browser. Authenticate with Autodesk Platform Services (APS), walk every folder in every project, classify Revit files by worksharing type and version, and export colour-coded PDF and CSV reports.

> ⚠ **Not an official Autodesk product.** Read-only access only. Use at your own discretion.

---

## Why this exists

Autodesk is deprecating cloud worksharing access for Revit versions older than **current minus five** starting **FY27 Q3/Q4**. Projects still on Revit 2021 or earlier (Cloud Workshared models only) will lose the ability to sync after that deadline.

Finding affected projects manually — hub by hub, project by project — is impractical at scale. This tool uses the **ACC Admin API** to enumerate every active project regardless of your personal project membership, then recursively scans folder trees to find `.rvt` files and classify them by worksharing type and version.

**Official Autodesk announcement:** [forums.autodesk.com](https://forums.autodesk.com/t5/revit-cloud-worksharing-forum/important-update-deprecation-of-revit-cloud-models-access-for/td-p/14093972)

---

## Features

| Feature | Detail |
|---|---|
| Full hub coverage | ACC Admin API lists all active projects regardless of personal membership |
| Two auth modes | 3-legged PKCE OAuth (standard) or 2-legged Client Secret (full hub access, session only) |
| RCW vs RC | Correctly separates Cloud Workshared (version-locked, at risk) from plain cloud uploads (not affected) |
| Smart folder skipping | 35+ known non-RVT folder names skipped at any depth (Plans, Photos, Archive, Specs, etc.) — no API calls wasted |
| Depth limit | BFS capped at depth 4 — matches deepest real-world ACC layout; eliminates timeout on deep/demo projects |
| Priority-first scan | Project Files and discipline folders scan before archive/admin folders — results appear fast |
| Parallel folder fetches | 3 folder content calls per project in parallel — 2–3× faster folder walk |
| Inline version data | `?include=version` on folder calls returns version data in the same response — eliminates separate version calls for many files |
| Project groups | Hub split into groups of 100; groups scan sequentially; results appear progressively; exportable per group |
| Parallel scanning | 3–4 projects in parallel within each group; 8 version fetches per project |
| Proactive rate limiting | Token-bucket tracks calls/min; inserts precise waits before hitting APS limits |
| Token auto-refresh | Silently refreshes between groups — scans survive past the 1-hour token expiry |
| Per-request timeout | 30s AbortController on every fetch; hung connections abort and retry rather than stalling |
| Version inference | When `revitProjectVersion` is absent (pre-2023 schema), year is inferred from file date and flagged as estimated |
| Threshold picker | Choose 2021 or 2022 as the critical cutoff; all tiers adjust instantly without re-scanning |
| Multi-status filter | Combine any statuses (e.g. Critical + Outdated) for targeted exports |
| Accuracy statement | PDF includes file-level accuracy % and caveats for inaccessible folders/projects |
| PDF report | Landscape A4, colour-coded rows and status badges, clickable ACC links, per-group or hub-wide |
| CSV export | Project summary + per-file RCW detail, filter-aware, per-group or hub-wide |
| Demo mode | Sample projects covering every status tier — no login required |

---

## Deprecation tiers

With the default threshold set to **2021**:

| Status | Revit version | Meaning |
|--------|--------------|---------|
| 🟢 **Current** | ≥ 2024 (threshold + 3) | No action needed |
| 🟡 **Outdated** | 2022–2023 (threshold + 1 or + 2) | Plan an upgrade cycle |
| 🔴 **Critical** | ≤ 2021 (at or below threshold) | Upgrade before FY27 Q3/Q4 |
| 🟣 **Mixed** | Multiple C4R versions in same project | Expand row to inspect files individually |
| ⚫ **No RCW** | No Cloud Workshared files found | Not affected by the deprecation |
| 🟠 **No Version** | RCW confirmed, year not in API | Pre-2023 schema — inferred from file date if possible, otherwise verify manually |
| 🔘 **No Access** | Project visible but files inaccessible | See [Troubleshooting](#troubleshooting) |

Set threshold to **2022** to shift all bands: Critical ≤ 2022, Outdated 2023–2024, Current ≥ 2025.

Only **RCW (Revit Cloud Workshared)** models count toward risk. Plain cloud uploads (RC) are displayed separately and excluded from all risk calculations.

### Version inference for pre-2023 files

The `revitProjectVersion` field was added to the APS API response in the C4R extension schema v1.3.x (circa 2023). Files on schema v1.1.x or v1.2.x — which includes most BIM 360 files created before 2023 — do not include this field.

When the field is absent, the tool infers the version from the file's last-modified date:

| Modified date | Inferred classification |
|---|---|
| ≤ threshold year (e.g. ≤ 2021) | **Critical** — version set to modification year |
| threshold+1 or threshold+2 (e.g. 2022–2023) | **Outdated** — version set to modification year |
| > threshold+2 (e.g. ≥ 2024) | **No Version** — cannot safely assume Current without proof |
| Before 2020 (any threshold) | **Critical** — pre-modern BIM 360 era |

Inferred versions are marked **(est.)** in the expanded file row and an amber banner explains the methodology. Files that cannot be inferred remain as **No Version** for manual review in ACC.

---

## Authentication — choosing the right mode

This is the most important decision before using the tool. The two modes have meaningfully different access levels and security profiles.

### Mode 1 — Sign in with Autodesk (3-legged PKCE)

Standard OAuth 2.0 Authorization Code flow with PKCE. You are redirected to Autodesk's login page, authenticate there, and the app receives a time-limited access token. No password or secret ever touches this app.

**APS app type:** Desktop, Mobile, Single-Page App

**Access:**
- All projects listed by the Admin API (requires Account Admin role)
- File contents **only for projects you are a member of** — the Admin API lists all projects, but the Data Management API enforces project-level membership for folder and file access

**Token lifetime:** 1 hour. Auto-refreshed between groups using a refresh token (granted via `offline_access` scope).

**Security profile:** Low risk. Token is short-lived. No secret stored anywhere. Autodesk's own login page handles credentials. Suitable for shared or team environments.

**Best for:** Hubs where you are a member of most projects, or where a standard sign-in experience is preferred.

---

### Mode 2 — Client Secret (2-legged)

Machine-to-machine `client_credentials` grant. The app exchanges your Client ID + Client Secret directly for an application token. No login page. The token represents the APS application, not a user — it bypasses per-project membership restrictions entirely.

**APS app type:** Traditional Web App or Service App (M2M)

**Access:**
- All projects in the hub regardless of personal membership
- File contents in all projects — no membership check

**Token lifetime:** 1 hour. Auto-refreshed between groups by calling `client_credentials` again with the stored credentials — fully silent, no user interaction required.

**Security profile:**

> ⚠ **The Client Secret grants application-level read access to your entire ACC hub. Treat it like a password.**

- The secret is held **in browser memory only** (the `S.clientSecret` JavaScript variable) and is never written to `localStorage`, `sessionStorage`, the DOM, server logs, or the network after the initial token exchange
- The secret is **cleared when the tab closes** and does not survive a page refresh
- While the session is active, the secret is readable in browser memory by anyone with DevTools access on that machine
- **Never use this mode on a shared, public, or screen-shared computer**
- **Never use this mode if your APS app has write scopes** — this tool only requests `data:read account:read`
- If your Client Secret is compromised, rotate it immediately in the APS developer portal

**Best for:** Account Admins scanning large hubs with many older BIM 360 projects where 3-legged mode leaves significant numbers of projects as "No Access".

---

## Setup

### Step 1 — Create an APS application

Go to [aps.autodesk.com](https://aps.autodesk.com) → **My Apps** → **Create App**. Choose the type based on your auth mode:

**3-legged (Sign in with Autodesk):**
- App type: **Desktop, Mobile, Single-Page App**
- No Client Secret is generated — that is expected and correct
- Set **Callback URL** to your exact deployment URL (no trailing slash):
  ```
  https://yourusername.github.io/revit-version-checker
  ```

**2-legged (Client Secret):**
- App type: **Traditional Web App** or **Service App (M2M)**
- Copy your Client Secret — it is only shown once
- Callback URL is not required for 2-legged

Enable API products for both: **Data Management**, **ACC Account Administrator**

### Step 2 — Register as a Custom Integration in ACC

1. ACC → **Account Admin → Custom Integrations → Add custom integration**
2. Paste your Client ID and complete the flow
3. The user performing this step must have **Account Admin** role

### Step 3 — Deploy to GitHub Pages

1. Fork or clone this repo
2. File must be `index.html` at the repo root
3. **Settings → Pages** → source: `main` branch, root folder
4. App live at `https://<username>.github.io/<repo-name>/`
5. GitHub Pages URL must exactly match the Callback URL (3-legged only)

### Step 4 — Connect and scan

**3-legged:**
1. Paste Client ID → **Sign in with Autodesk**
2. Autodesk login page → authenticate with an Account Admin account
3. Select hub → scanning begins

**2-legged:**
1. Paste Client ID and Client Secret into the setup fields
2. Click **Connect with Client Secret** → immediate, no login page
3. Select hub → scanning begins

---

## How the scan works

Projects are split into **groups of 100** and scanned sequentially. Within each group, 3–4 projects run in parallel. Results appear group by group so you can export Group 1 while Group 2 is scanning.

```
scanHub()
 ├─ getAllAdminProjects()        ACC Admin API — all active projects, paginated
 ├─ getMemberProjectIds()        DM API — pre-marks non-member projects (3-legged only)
 └─ For each group of 100 (sequential)
     ├─ refreshSessionIfNeeded() Proactive token refresh if token >50 min old
     └─ pool(projects, 3–4)      3 parallel (3-legged) or 4 (2-legged)
         └─ scanProject()
             ├─ topFolders call  — skip known non-RVT folders immediately
             ├─ Pass 1: Parallel BFS folder walk (3 folders at once)
             │    Queue ordered by FOLDER_PRIORITY (Project Files → Shared → disciplines → other)
             │    SKIP_FOLDERS applied at every depth (Archive, Old, Specs, Admin, etc.)
             │    MAX_FOLDER_DEPTH=4 — never descends beyond discipline/phase/subphase
             │    apiAllWithInclude() — fetches contents + inline version data (?include=version)
             │    Items with inline version data → classified immediately, no Pass 2 call
             │    Items without inline version data → allRvtItems (needs Pass 2)
             └─ Pass 2: Version fetches — pool(allRvtItems, 8) — only for items not yet classified
                  /items/{id}/versions → extension.type → RCW or RC
                                       → revitProjectVersion → year
                                       → if absent: infer from lastModifiedTime
```

### Folder walk optimisations

Three layers prevent wasted API calls on folders that cannot contain RCW models:

**Skip list (`SKIP_FOLDERS`)** — 35+ folder names skipped at any depth in the BFS:
- ACC system folders: Plans, Photos, Submittals, RFIs, Issues, Closeout, Reports, Transmittals, Markups, Specifications, Correspondence, Meetings, Contracts
- Archive/obsolete: Archive, Archives, Old, Superseded, Backup, Backups, Deprecated, Previous, History, Legacy, Obsolete
- Admin: Admin, Administration, Templates
- Media: Images, Pictures, Video, Videos, Documents, Sheets
- Workflow: Incoming, Outgoing, For Review, For Approval, Issued, Record Drawings

**Depth limit (4)** — BFS never descends past `topFolder / discipline / phase / subphase`. Covers every documented real-world ACC structure; eliminates timeouts on deep demo/test accounts.

**Priority-first BFS** — Folders scan in priority order within each BFS level:
- Priority 0: `Project Files` (working models)
- Priority 1: `Shared` (Design Collaboration consumed models)
- Priority 2: Known discipline names (Architecture, Structural, MEP, Civil, etc.)
- Priority 3: Everything else

### RCW classification

Classification always uses the **version-level** `extension.type` — never item-level. Older BIM 360 files commonly carry `items:autodesk.bim360:File` at the item level even when Cloud Workshared, making item-level classification unreliable.

```
versions:autodesk.bim360:C4RModel   BIM 360 era — most common for affected files
versions:autodesk.a360:C4RModel     Early A360 era
versions:autodesk.core:C4RModel     ACC native
```

### Scan accuracy

```
accuracy % = classified / (classified + failed)
```

Failed files (version call failed after all retries) appear in each project's expanded row and in the PDF accuracy statement. Inaccessible folders (403) and no-access projects are flagged separately — their contents are unknown and excluded from accuracy calculations.

---

## Exports

### PDF report

- Client-side generation via jsPDF + jsPDF-AutoTable — no server involved
- Two-tone navy/blue header, colour-coded row backgrounds (red/amber/green/purple by status)
- Filled status badge pills, version text coloured by tier, 2.5mm red accent bar on critical rows
- Scan accuracy statement: file-level accuracy %, failed file count, skipped folder caveat
- Clickable "Open ACC" buttons link directly to each project in ACC
- Per-group export (buttons appear in each group header when that group finishes) or whole-hub

### CSV export

| Row type | Columns |
|---|---|
| Project summary | Project, RCW Version, RCW Files, At Risk Files, RC Files, Status, Last Modified |
| Per-file detail | Project, Type, File, Version, Risk, Path, Last Modified, Modified By |

Both exports respect the active search, status filter, and sort order.

---

## Troubleshooting

**Many "No Access" projects**
Use 2-legged Client Secret mode. In 3-legged mode, file access is limited to projects you are personally a member of. This is common on hubs with older BIM 360 projects.

**Sign-in callback error**
The Callback URL in your APS app does not exactly match the page URL. Check trailing slashes, `http` vs `https`. The expected URL is shown on the setup screen.

**Token expired mid-scan**
The tool auto-refreshes between groups (every ~100 projects). If a single project takes longer than 60 minutes (very unusual), the 401 handler retries once with a fresh token. If you still see this error, sign out and back in.

**2-legged: "401 Unauthorized" on connect**
Verify your APS app type is **Traditional Web App** or **Service App (M2M)** — Single-Page App does not generate a Client Secret. Also verify the app is registered as a Custom Integration in ACC Account Admin.

**Projects show "No Version" despite being RCW**
These files use the pre-2023 BIM 360 schema where `revitProjectVersion` is absent. The tool infers the version from the file's last-modified date where possible (≤ threshold+2 years). Files modified after that remain as No Version for manual review in ACC.

**Files show "(est.)" next to their version**
The version year was inferred from the file's modification date rather than read directly from the API. This occurs on pre-2023 schema files. The inference is conservative — treat these files as requiring verification before assuming they are safe.

**Many "timed out after 5 minutes"**
Usually indicates a project with an unusually deep or large folder structure. The depth limit (4 levels) and skip list should prevent most of these. Check the expanded row for `skippedFolders` count — if high, the project may have restricted subfolders.

**Scan feels slow on small projects**
The irreducible cost is one API call per folder (~500ms each). For a typical project with 4 discipline folders, that's ~2 seconds of folder walk plus version fetches. The parallel folder fetches (3 at once) and inline version data (`?include=version`) minimise this as much as the API allows.

**PDF link buttons don't open**
Use Adobe Acrobat or macOS Preview. Some browser-based PDF viewers strip link annotations.

**Client Secret clears on page refresh**
By design — the secret lives in browser memory only. Re-enter credentials after each browser session. The session token survives in `sessionStorage` for up to 1 hour.

---

## Project structure

```
/
├── index.html       Single-file app — all HTML, CSS, and JavaScript
├── README.md
├── ARCHITECTURE.md  Technical internals for contributors
├── CONTRIBUTING.md
├── CHANGELOG.md
└── LICENSE
```

No build step, no `node_modules`, no bundler. Open `index.html` directly for local testing — register `http://localhost` as a Callback URL in your APS app.

---

## Credits

Original tool by **Tanmay Bhalerao** — Senior Account Technical Lead, Autodesk.

Community enhancements: ACC Admin API, 2-legged auth, project groups, smart folder scanning, parallel BFS, inline version classification, rate limiting, token auto-refresh, version inference, PDF redesign, accuracy tracking, resilience improvements.

---

## License

MIT — see [LICENSE](LICENSE) for details.
