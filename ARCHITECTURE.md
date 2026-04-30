# Architecture

## Overview
This is a static single-page app (`index.html`) that performs Autodesk APS/ACC API calls directly from the browser.

## Data flow
1. Authenticate with APS using PKCE OAuth.
2. List hubs and projects.
3. Traverse top folders recursively.
4. Identify `.rvt` items.
5. For each item, fetch versions and inspect latest metadata.

## Model classification
- **RCW**: `attributes.extension.type` contains `C4RModel` signature.
- **RC**: non-workshared Revit cloud uploads.

Only RCW files participate in version-risk logic and threshold alerts.

## Status computation
Project status is derived from RCW findings:
- `critical`: has RCW files at or below selected threshold.
- `warn`: RCW version is behind hub latest.
- `ok`: RCW version is latest.
- `no-c4r`: no RCW files detected.

## Exports
- CSV export serializes project-level data rows.
- PDF export generates a print-friendly HTML report honoring current filters.
