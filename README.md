# Revit Version Checker

A free, zero-backend hub-level dashboard for scanning Revit model versions across all projects in an Autodesk Forma / ACC hub at once.

![Screenshot](screenshot.png)

---

## Why this exists

Version visibility has been one of the most common requests from ACC customers for years — and the product team hasn't shipped it natively. With Autodesk announcing deprecation of cloud worksharing access for older Revit versions ([FY27 Q3/Q4 announcement](https://forums.autodesk.com/t5/revit-cloud-worksharing-forum/important-update-deprecation-of-revit-cloud-models-access-for/td-p/14093972)), teams need to know exactly where they stand across all projects at once — not one folder at a time.

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
- **One-click sign-in** — PKCE OAuth via Autodesk's official login page, no tokens to copy
- **Saved Client ID** — stored in your browser so you only enter it once
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

Each user connects using their own APS app credentials. This means the app works for any ACC hub without a central whitelist — you control your own access.

### What you need

1. **An APS app** — create one at [aps.autodesk.com](https://aps.autodesk.com) (free)
2. **Data Management API + Authentication API** enabled on that app
3. **Your GitHub Pages URL set as the Callback URL** in the app settings
4. **Your app added as a Custom Integration** in ACC Account Admin

Full step-by-step instructions are built into the app's connect screen.

### Use the hosted version

[**tsb2127.github.io/revit-version-checker**](https://tsb2127.github.io/revit-version-checker)

1. Follow the 3-step setup guide on the connect screen
2. Paste your APS Client ID (saved in your browser after first use)
3. Click **Sign in with Autodesk** — logs in via Autodesk's official login page
4. Hub scan starts automatically

### Run locally

```bash
git clone https://github.com/tsb2127/revit-version-checker.git
cd revit-version-checker
open index.html        # Mac
start index.html       # Windows
```

The callback URL shown in the setup guide will update automatically to match wherever you're running the app.

---

## How sign-in works (PKCE OAuth)

The app uses the **PKCE (Proof Key for Code Exchange)** OAuth 2.0 flow — the industry standard for browser-based apps with no backend server. There is no hardcoded client ID in the code. Each user brings their own APS app, which means:

- No central whitelist required — your APS app only needs to be added to your own hub
- You control exactly which hubs can be accessed by your app
- If you stop using the app, simply remove the Custom Integration from ACC

**The flow step by step:**

1. You enter your Client ID and click Sign in
2. The app generates a random one-time `code_verifier` stored only in your browser's `sessionStorage`
3. Your browser redirects to Autodesk's official login page
4. You log in and approve `data:read` access
5. Autodesk redirects back with a short-lived `code` in the URL
6. The app exchanges that code + `code_verifier` for an access token — no client secret needed
7. Token is stored in `localStorage`, expires after 1 hour

Your Client ID is saved in your browser so you only enter it once.

---

## ⚠️ Use at your own risk

**This is not an official Autodesk product.** It is an open-source tool built by an individual Autodesk employee in a personal capacity. Please read this before using it or sharing it with customers:

- This app has **not** undergone a security audit, penetration test, or Autodesk's internal application review process
- The token scope is `data:read` only — the app **cannot** write, modify, delete, or change anything in your ACC hub
- No data is sent to any third-party server — all API calls go directly from your browser to Autodesk's servers
- Your token is stored in your browser's `localStorage` and never transmitted anywhere except Autodesk's own API endpoints
- The full source code is publicly available at this repository — your IT or security team can review exactly what the app does before use
- Tokens expire after 1 hour and must be refreshed by signing in again
- By using this app you accept that it is provided as-is with no warranty, support, or liability implied by Autodesk

If your organisation has strict policies about third-party OAuth applications accessing ACC data, check with your IT or security team before use.

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
- Large hubs with hundreds of projects and thousands of files will make many API calls and may take several minutes to scan. The APS Data Management API is free with no per-call cost.
- Each user needs their own APS app (free to create) registered as a **Desktop, Mobile, Single-Page App** type, with the app added as a Custom Integration in their ACC hub. The connect screen walks through this in 3 steps.
- Tokens expire after 1 hour — sign in again when prompted.
- The app requires the APS app to be of type **Single-Page App** (not Traditional Web App) to support PKCE. Traditional Web App types will fail authentication.

---

## Roadmap

- [x] ~~Built-in OAuth login (PKCE)~~ — shipped
- [ ] Upgrade deadline countdown — days remaining per project based on Autodesk's FY27 timeline
- [ ] Required version enforcement — set a hub standard and surface any project that deviates
- [ ] Scan history — persist results so you can compare hub health week over week

Contributions welcome — open an issue or PR.

---

## Built by

Tanmay Bhalerao — Senior Account Technical Lead at Autodesk, working with AEC customers across the US. Built this because customers kept asking for it and it didn't exist. Not a software engineer by background — proof that the APS APIs are approachable for anyone willing to experiment.

---

## License

MIT
