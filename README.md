# Revit Version Checker

A free, zero-backend hub-level dashboard for scanning Revit model versions across all projects in an Autodesk Forma / ACC hub at once.

![Screenshot](screenshot.png)

---

## Why this exists

Version visibility has been one of the most common requests from ACC customers for years — and the product team still hasn't shipped it natively. With Autodesk announcing deprecation of cloud worksharing access for older Revit versions ([FY27 Q3/Q4 announcement](https://forums.autodesk.com/t5/revit-cloud-worksharing-forum/important-update-deprecation-of-revit-cloud-models-access-for/td-p/14093972)), teams need to know exactly where they stand across all projects at once — not one folder at a time.

---

## RCW vs RC — the key distinction

Not all `.rvt` files in ACC are the same. The app separates two types:

| Type | Label | What it means | Deprecation risk |
|------|-------|---------------|-----------------|
| Revit Cloud Workshared | `RCW` | Cloud Worksharing is enabled. The entire team is locked to one Revit version — nobody can open the file in a different version. | **Yes** — affected by Autodesk's deprecation deadline |
| Revit Cloud Model | `RC` | A Revit file uploaded to ACC without worksharing. No version lock — anyone can open it in any version. | **No** — excluded from risk calculations entirely |

Only `RCW` models are counted toward deprecation risk. `RC` files appear in the expanded project view as informational only.

Detection is done via `attributes.extension.type` — if it contains `C4RModel`, the file is Revit Cloud Workshared.

---

## Features

- **Hub-level scan** — scans every project in your hub in one go, no project-by-project navigation
- **RCW / RC separation** — correctly distinguishes workshared models from plain cloud uploads; only RCW files count toward risk
- **Deprecation banner** — automatically flags projects with files on Revit 2021 or older (configurable threshold)
- **Configurable threshold** — set your own cutoff year (2019–2022) to get ahead of the deadline
- **Per-project expand** — click any project row to see every RCW and RC file with version, path, and last modified
- **Version colour badges** — green (latest in hub) → blue (1 behind) → amber (2 behind) → red (3+ behind)
- **Sortable table** — sort by version, files, risk count, or last modified
- **Search + status filter** — find projects by name or filter to Critical / Outdated / Current
- **Direct ACC links** — each project row links straight to that project in ACC
- **CSV export** — exports project summary and full file detail, with at-risk flag per file
- **Saved token** — stores your token in `localStorage` so you don't re-enter every session
- **Demo mode** — try the full UI without an Autodesk account

---

## How it works

Connects to your ACC hub and scans every project simultaneously. For each `.rvt` file found it calls:

```
GET /data/v1/projects/{projectId}/items/{itemId}/versions
```

And checks two things on the latest version:

1. `attributes.extension.type` — determines whether the file is RCW (contains `C4RModel`) or RC (anything else)
2. `attributes.extension.data.revitProjectVersion` — the Revit year, e.g. `2025`

The version is only shown and counted for RCW files. RC files are surfaced separately with no version lock implied.

### API endpoints used

| Endpoint | Purpose |
|----------|---------|
| `GET /project/v1/hubs` | List ACC hubs |
| `GET /project/v1/hubs/{hubId}/projects` | List all projects in hub |
| `GET /project/v1/hubs/{hubId}/projects/{projectId}/topFolders` | Get root folders |
| `GET /data/v1/projects/{projectId}/folders/{folderId}/contents` | Recurse folder tree |
| `GET /data/v1/projects/{projectId}/items/{itemId}/versions` | Read version + type metadata |

All calls use a 3-legged token. No server-side component is needed — every API call is made directly from the browser.

---

## Getting started

### Prerequisites

- An [APS app](https://aps.autodesk.com) with `data:read` scope
- A 3-legged OAuth Bearer token for your ACC user
- Access to at least one ACC / Forma hub

### Get a token

Follow the [APS 3-legged token tutorial](https://aps.autodesk.com/en/docs/oauth/v2/tutorials/get-3-legged-token) or use Postman with the `/authentication/v2/token` endpoint. Scope required: `data:read`.

Tokens expire after 1 hour. If the app stops loading data, generate a fresh one.

### Use the hosted version

[**tsb2127.github.io/revit-version-checker**](https://tsb2127.github.io/revit-version-checker)

### Run locally

```bash
git clone https://github.com/tsb2127/revit-version-checker.git
cd revit-version-checker
open index.html        # Mac
start index.html       # Windows
```

No build step, no dependencies, no Node required.

---

## Deploying your own copy to GitHub Pages

1. Fork or push to a GitHub repo
2. Go to **Settings → Pages**
3. Set source to `Deploy from a branch` → `main` → `/ (root)`
4. Live at `https://YOUR_USERNAME.github.io/revit-version-checker`

---

## Usage

1. Paste your APS Bearer token and click **Connect & scan hub**
2. If you have multiple hubs, select one — single-hub accounts skip this step automatically
3. The app scans all projects recursively. Watch the progress bar as each project is processed
4. The dashboard shows every project with its RCW version, file counts, at-risk count, and status
5. Click any project row to expand and see individual RCW and RC files
6. Use the threshold bar to change the deprecation cutoff year (default: 2021)
7. Filter to **Critical** to see only at-risk projects, then click the ACC link to go upgrade them
8. Click **Export CSV** to download a full report with project summary and per-file detail

---

## Version colour key

| Badge colour | Meaning |
|---|---|
| Green | Latest RCW version found across the hub |
| Blue | One version behind the hub latest |
| Amber | Two versions behind |
| Red | Three or more versions behind |
| Red (outlined) | At or below deprecation threshold — action required |

---

## Limitations

- `revitProjectVersion` is only populated for **Revit Cloud Workshared** models. RC (non-workshared) uploads do not have a version lock and are not counted toward risk.
- Token must be obtained externally — no built-in OAuth login flow yet (see roadmap).
- Large hubs with hundreds of projects and thousands of files will make many API calls and may take several minutes to scan. The APS Data Management API is free with no per-call cost.

---

## Roadmap

- [ ] Built-in OAuth login (PKCE) — one-click "Sign in with Autodesk", no manual token copy
- [ ] Upgrade deadline countdown — days remaining per project based on Autodesk's FY27 timeline
- [ ] Required version enforcement — set a hub standard and surface any project that deviates
- [ ] Scan history — persist results so you can compare hub health week over week

Contributions welcome — open an issue or PR.

---

## Built by

Tanmay Bhalerao — Senior Account Technical Lead at Autodesk, working with AEC teams across the US and India. Built this because customers kept asking for it and it didn't exist. Not a software engineer by background — proof that the APS APIs are approachable for anyone willing to experiment.

---

## License

MIT
