# Revit Version Checker

A lightweight, zero-backend web app for scanning Revit file versions across Autodesk Forma / ACC (Autodesk Construction Cloud) projects.

![Screenshot](screenshot.png)

## Features

- **Recursive folder scanning** — crawls all subfolders automatically from any top-level folder
- **Version badges** — colour-coded by recency (latest → one behind → two behind → outdated)
- **Workshared detection** — flags cloud workshared (C4R) models
- **Full path column** — shows exactly where in the folder tree each file lives
- **Sortable table** — click any column header to sort
- **Search + filter** — filter by version year or search by filename, path, or user
- **CSV export** — one-click export of all results with date-stamped filename
- **Saved token** — stores your token in `localStorage` so you don't re-enter every session
- **Demo mode** — try the full UI without an Autodesk account

---

## How it works

Uses the [APS Data Management API](https://aps.autodesk.com/developer/overview/data-management-api) to walk the folder tree of an ACC project. For each `.rvt` file found, it calls `GET /data/v1/projects/{projectId}/items/{itemId}/versions` and reads:

```
attributes.extension.data.revitProjectVersion  →  e.g. 2025
```

This field is the authoritative Revit year for cloud workshared models. For non-cloud models the field may be absent (shown as `—`).

---

## Getting started

### 1. Prerequisites

- An [Autodesk Platform Services (APS) app](https://aps.autodesk.com) with `data:read` scope
- Access to at least one ACC / Forma project
- A 3-legged OAuth Bearer token for the user whose projects you want to browse

### 2. Get a token

Follow the [APS 3-legged token tutorial](https://aps.autodesk.com/en/docs/oauth/v2/tutorials/get-3-legged-token) or generate one via [APS Explorer](https://aps-explorer.autodesk.io/).

Scopes required: `data:read`

### 3. Run locally

No build step needed — it's a single HTML file.

```bash
# Clone the repo
git clone https://github.com/YOUR_USERNAME/revit-version-checker.git
cd revit-version-checker

# Open directly in your browser
open index.html

# Or serve with any static server
npx serve .
python3 -m http.server 8080
```

---

## Deploying to GitHub Pages

1. Push this repo to GitHub
2. Go to **Settings → Pages**
3. Set **Source** to `Deploy from a branch` → `main` → `/ (root)`
4. Your app will be live at `https://YOUR_USERNAME.github.io/revit-version-checker`

No CI, no build pipeline needed.

---

## Usage

1. Open the app and paste your APS Bearer token
2. Select your hub (auto-skipped if you only have one)
3. Select a project
4. Click any top-level folder to start a **recursive scan**
5. All `.rvt` files in all subfolders will be enumerated and their Revit version displayed
6. Use the filter dropdown, search box, or sort columns to explore results
7. Click **Export CSV** to download the full results

---

## Version colour key

| Badge | Meaning |
|-------|---------|
| Green | Latest version found in the project |
| Blue  | One version behind latest |
| Amber | Two versions behind latest |
| Red   | Three or more versions behind |
| Gray  | Version could not be read |

---

## API reference

| Endpoint | Purpose |
|----------|---------|
| `GET /project/v1/hubs` | List ACC hubs |
| `GET /project/v1/hubs/{hubId}/projects` | List projects in hub |
| `GET /project/v1/hubs/{hubId}/projects/{projectId}/topFolders` | Get root folders |
| `GET /data/v1/projects/{projectId}/folders/{folderId}/contents` | List folder contents (recursive) |
| `GET /data/v1/projects/{projectId}/items/{itemId}/versions` | Get file version metadata |

All calls use a 3-legged token. No server-side component is needed.

---

## Limitations

- `revitProjectVersion` is only populated for **Revit Cloud Models** (workshared files published to ACC). Local `.rvt` files uploaded as-is will show `—`.
- Token must be obtained externally — this app does not implement an OAuth login flow itself. See [roadmap](#roadmap) below.
- Large projects with hundreds of files will make many API calls; rate limits may apply.

---

## Roadmap

- [ ] Built-in OAuth login flow (PKCE)
- [ ] Required version enforcement with pass/fail column
- [ ] Scan progress persistence (resume after reload)
- [ ] Version drift chart over time
- [ ] Slack / email digest webhook

Contributions welcome — open an issue or PR.

---

## License

MIT
