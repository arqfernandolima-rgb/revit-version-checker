# Revit Version Checker

[![Version](https://img.shields.io/badge/version-1.0.0-blue)](https://github.com/tsb2127/revit-version-checker/releases/tag/v1.0.0)
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
| RCW vs RC | Separates Cloud Workshared (version-locked, at risk) from plain cloud uploads (not affected) |
| Project groups | Hub split into groups of 200; groups scan sequentially; results appear progressively |
| Parallel scanning | 3–4 projects in parallel within each group; 5 version fetches per project |
| Proactive rate limiting | Token-bucket tracks calls/min; inserts precise waits before hitting APS limits |
| Token auto-refresh | Silently refreshes between groups — scans survive past the 1-hour token expiry |
| Per-request timeout | 30s AbortController on every fetch; hung connections abort and retry rather than stalling |
| Threshold picker | Choose 2021 or 2022 as the critical cutoff; all tiers adjust instantly without re-scanning |
| Multi-status filter | Combine any statuses (e.g. Critical + Outdated) for targeted exports |
| Accuracy statement | PDF includes file-level accuracy % and caveats for inaccessible folders/projects |
| PDF report | Landscape A4, colour-coded rows and status badges, clickable ACC links, per-group or hub-wide |
| CSV export | Project summary + per-file RCW detail, filter-aware, per-group or hub-wide |
| Demo mode | Nine sample projects covering every status tier — no login required |

---

## Deprecation tiers

With the default threshold set to **2021**:

| Status | Revit version | Meaning |
|--------|--------------|---------|
| 🟢 **Current** | ≥ 2024 (threshold + 3) | No action needed |
| 🟡 **Outdated** | 2022–2023 (threshold + 1 or + 2) | Plan an upgrade cycle |
| 🔴 **Critical** | ≤ 2021 (at or below threshold) | Upgrade before FY27 Q3/Q4 |
| 🟣 **Mixed** | Multiple C4R versions in same project | Expand row to inspect files individually |
| ⚫ **No RCW** | No Cloud Workshared files found | Not affected |
| 🔘 **No Access** | Project visible but files inaccessible | See [Troubleshooting](#troubleshooting) |

Set threshold to **2022** to shift all bands: Critical ≤ 2022, Outdated 2023–2024, Current ≥ 2025.

Only **RCW (Revit Cloud Workshared)** models count toward risk. Plain cloud uploads (RC) are shown separately and excluded from all calculations.

---

## Authentication — choosing the right mode

This is the most important decision before using the tool. The two modes have meaningfully different access levels and security profiles.

### Mode 1 — Sign in with Autodesk (3-legged PKCE)

Standard OAuth 2.0 Authorization Code flow with PKCE. You are redirected to Autodesk's login page, authenticate there, and the app receives a time-limited access token. No password or secret ever touches this app.

**APS app type:** Desktop, Mobile, Single-Page App

**Access:**
- All projects listed by the Admin API (requires Account Admin role)
- File contents **only for projects you are a member of** — the Admin API lists all projects, but the Data Management API enforces project-level membership for folder and file access

**Token lifetime:** 1 hour. Auto-refreshed between groups using a refresh token (granted via `offline_access` scope since v1.0.0).

**Security profile:** Low risk. Token is short-lived. No secret stored anywhere. Autodesk's own login page handles credentials. Suitable for shared or team environments.

**Best for:** Hubs where you are a member of most projects, or where complete hub coverage is less important than a standard sign-in experience.

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
- The session token is stored in `sessionStorage` (tab-scoped) and clears on tab close
- While the session is active, the secret is readable in browser memory by anyone with DevTools access on that machine
- **Never use this mode on a shared, public, or screen-shared computer**
- **Never use this mode if your APS app has write scopes** — this tool only requests `data:read account:read`, but verify your app's scope configuration before connecting
- If your Client Secret is compromised, rotate it immediately in the APS developer portal

**Best for:** Account Admins scanning large hubs with many older BIM 360 projects where 3-legged mode leaves significant numbers of projects as "No Access". The productivity gain is real, but the security tradeoff must be understood.

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
1. Paste Client ID into the Client ID field
2. Paste Client Secret into the password field
3. Click **Connect with Client Secret** → immediate, no login page
4. Select hub → scanning begins

---

## How the scan works

Projects are split into **groups of 200** and scanned sequentially. Within each group, 3–4 projects run in parallel. Results appear group by group, so you can export Group 1 while Group 2 is scanning.

```
scanHub()
 ├─ getAllAdminProjects()        ACC Admin API — all active projects, paginated
 ├─ getMemberProjectIds()        DM API — pre-marks non-member projects (3-legged only)
 └─ For each group (sequential)
     ├─ refreshSessionIfNeeded() Proactive token refresh if >50 min old
     └─ pool(projects, 3–4)      Parallel within group
         └─ scanProject()
             ├─ Pass 1: BFS folder walk, zero version calls
             │    topFolders → queue → apiAll() per folder (handles pagination)
             └─ Pass 2: version fetches, pool(items, 5)
                  /items/{id}/versions → extension.type → RCW or RC
                                       → revitProjectVersion → year
```

### RCW classification

Classification uses the **version-level** `extension.type` — never the item-level type. Older BIM 360 files often carry `items:autodesk.bim360:File` at item level even when Cloud Workshared, making item-level classification unreliable.

```
versions:autodesk.bim360:C4RModel   BIM 360 era — most common for affected files
versions:autodesk.a360:C4RModel     Early A360 era
versions:autodesk.core:C4RModel     ACC native
```

### Scan accuracy

```
accuracy % = classified / (classified + failed)
```

Failed files (version call failed after retries) appear in each project's expanded row and in the PDF accuracy box. Inaccessible folders (403) and no-access projects are flagged separately — their contents are genuinely unknown and excluded from both numerator and denominator.

---

## Exports

### PDF report

- Client-side generation via jsPDF + jsPDF-AutoTable — no server
- Two-tone navy/blue header, colour-coded row backgrounds, status badge pills
- Scan accuracy statement with file counts and caveats
- Clickable "Open ACC" buttons link to each project in ACC
- Per-group export (group header button) or whole-hub export (toolbar)
- File-level detail pages for critical projects

### CSV export

| Row type | Columns |
|---|---|
| Project summary | Project, RCW Version, RCW Files, At Risk Files, RC Files, Status, Last Modified |
| Per-file detail | Project, Type, File, Version, Risk, Path, Last Modified, Modified By |

Both exports respect the active search, status filter, and sort order.

---

## Troubleshooting

**Fewer projects than expected / many "No Access"**
Use 2-legged Client Secret mode for complete hub coverage. In 3-legged mode, file access is limited to projects you are personally a member of.

**Sign-in callback error**
The Callback URL in your APS app does not exactly match the page URL. Check trailing slashes, `http` vs `https`. The expected URL is printed on the setup screen.

**Groups 3+ all show "No RCW"**
Token expired mid-scan. Upgrade to v1.0.0 — the auto-refresh mechanism silently renews the token between groups.

**2-legged: secret not accepted / 401 error**
Check that your APS app type is **Traditional Web App** or **Service App (M2M)** — Single-Page App type does not generate a Client Secret. Also verify the app is registered as a Custom Integration in ACC.

**Many "timed out after 5 minutes"**
Projects with very deep folder structures or slow API responses. The 30s per-request timeout prevents any single hung call from consuming the full project budget. Check the expanded row for `skippedFolders` and `failedFiles` counts.

**PDF link buttons don't open**
Use Adobe Acrobat or macOS Preview. Some browser-based PDF viewers strip link annotations.

**Client Secret clears on page refresh**
By design — the secret lives in browser memory only. Re-enter credentials after each browser session. The session token (without the secret) survives in `sessionStorage` for up to 1 hour, so you can refresh the page and continue viewing results without re-scanning.

---

## Project structure

```
/
├── index.html       Single-file app — all HTML, CSS, and JS
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

Community enhancements: ACC Admin API, 2-legged auth, project groups, rate limiting, token auto-refresh, parallel scanning, PDF redesign, accuracy tracking, resilience improvements.

---

## License

MIT — see [LICENSE](LICENSE) for details.
