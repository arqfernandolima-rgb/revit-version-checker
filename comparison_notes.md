# Comparison: `tsb2127/revit-version-checker` vs `arqfernandolima-rgb/revit-version-checker`

Date: 2026-04-30

## Scope and confidence
This comparison is based on:
- Local source files in this repo (`index.html`, `README.md`).
- Public metadata and README content visible at `https://github.com/arqfernandolima-rgb/revit-version-checker`.

Because direct git clone is blocked in this execution environment, this is a **high-confidence feature comparison from docs/repo structure**, not a full line-by-line code diff.

## Improvements found in your colleague's repo

### 1) Better project structure and maintainability docs
Your repo is currently very lightweight (single-page app + README). Your colleague added:
- `ARCHITECTURE.md`
- `CHANGELOG.md`
- `CONTRIBUTING.md`

Why this matters:
- Easier onboarding for collaborators
- Clearer change tracking and release discipline
- Lower bus-factor for future maintenance

### 2) Clearer positioning around ACC Admin API coverage
Your colleague explicitly documents that scan coverage uses ACC Admin API for all active projects and calls out the Account Admin requirement.

Why this matters:
- Sets correct access expectations up front
- Explains why scan results can differ for admin vs non-admin users

### 3) Documented parallelism / performance model
Your colleague documents explicit concurrency controls:
- Parallel projects
- Parallel folder traversal
- Parallel version lookups

And warns about API rate-limit trade-offs (429 risk).

Why this matters:
- Better scalability on large hubs
- Easier future tuning without regressions

### 4) Stronger status model and filtering taxonomy
Compared to your current Critical/Outdated/Current-centric UX, your colleague documents expanded categories such as:
- Mixed
- No RCW
- Multi-status filtering combinations

Why this matters:
- Better operational triage
- More realistic representation of mixed-version projects

### 5) PDF export capability (in addition to CSV)
Your repo already includes CSV export. Your colleague additionally highlights:
- Filter-aware PDF output
- Project summary + file-level detail sections for critical projects

Why this matters:
- Executive-ready reporting format
- Easier sharing with non-technical stakeholders

### 6) More explicit RCW detection details
Your colleague documents exact RCW signature patterns (`C4RModel` variants) and keeps RC exclusion logic explicit.

Why this matters:
- Better trust and auditability of model classification
- Easier debugging when ACC metadata variants appear

### 7) Improved developer-facing documentation quality
Your colleague's README includes:
- More explicit setup details (required API products)
- More detailed “How scan works” flow
- Tuning guidance for concurrency

Why this matters:
- Fewer setup failures
- Less support burden

## What I recommend merging back first (priority order)

### Quick wins (low effort, high value)
1. Add `CHANGELOG.md` and start structured release notes.
2. Add `CONTRIBUTING.md` with local run/test rules.
3. Expand README with explicit role/access caveats and API product checklist.
4. Add status definitions for Mixed/No RCW in UI legend and export columns.

### Medium effort
5. Add multi-status filter behavior (if not already fully implemented).
6. Add PDF export matching current table filter/sort state.
7. Add documented concurrency constants and expose safe tuning points.

### Higher effort / architectural hardening
8. Introduce architecture doc + scan pipeline diagram.
9. Add minimal regression checks (even simple fixture-driven checks for status calculation).

## If you want a strict code-level diff
Provide either:
- a ZIP of your colleague repo in this environment, or
- a local checkout path to that repo.

Then I can generate an exact file-by-file diff and a patch plan (what to cherry-pick vs re-implement cleanly).
