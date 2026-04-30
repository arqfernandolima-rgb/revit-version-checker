# Merge-Back Implementation Plan

Date: 2026-04-30
Source comparison target: `arqfernandolima-rgb/revit-version-checker`

## Goal
Adopt the highest-value improvements from the colleague repo into this repo with minimal regression risk.

## Work packages

### WP1 — Documentation foundation (Quick win)
**Tasks**
1. Add `CHANGELOG.md` with Keep-a-Changelog style sections.
2. Add `CONTRIBUTING.md` with:
   - local run instructions
   - coding conventions for `index.html`
   - PR checklist
3. Add `ARCHITECTURE.md` with:
   - scan flow (hub → projects → folders → items → versions)
   - RCW/RC classification logic
   - status computation logic

**Effort**: 2–4 hours
**Risk**: Low
**Dependencies**: None

---

### WP2 — Status model expansion (Quick/Medium)
**Tasks**
1. Define status enums clearly:
   - `Current`
   - `Outdated`
   - `Critical`
   - `Mixed`
   - `No RCW`
2. Update legend + badges to include new states.
3. Add multi-select/combined status filters.
4. Ensure CSV export includes status values exactly as shown in UI.

**Acceptance criteria**
- A project with mixed file versions is labeled `Mixed`.
- A project with only RC files is labeled `No RCW`.
- Filters can isolate each status and combinations.

**Effort**: 4–8 hours
**Risk**: Medium
**Dependencies**: WP1 docs update for terminology

---

### WP3 — PDF export (Medium)
**Tasks**
1. Add PDF export button near CSV export.
2. Export reflects current filters/sort/search state.
3. PDF sections:
   - Summary metrics
   - Project table snapshot
   - Critical-project file details
4. Include generation timestamp + threshold setting in footer/header.

**Acceptance criteria**
- Same filtered dataset in UI and PDF.
- PDF remains readable for large hubs (pagination + wrapping).

**Effort**: 6–12 hours
**Risk**: Medium
**Dependencies**: WP2 status consistency

---

### WP4 — Performance tuning controls (Medium)
**Tasks**
1. Introduce explicit constants for concurrency:
   - `PROJECT_CONCURRENCY`
   - `FOLDER_CONCURRENCY`
   - `VERSION_CONCURRENCY`
2. Add adaptive backoff for HTTP 429.
3. Add scan telemetry counters:
   - total calls
   - retries
   - throttle events
4. Document safe tuning ranges in README.

**Acceptance criteria**
- Reduced scan stalls under rate limits.
- Stable completion on large hubs with no unhandled failures.

**Effort**: 8–16 hours
**Risk**: Medium/High (rate-limit behavior)
**Dependencies**: None (can run in parallel with WP3)

---

### WP5 — Validation harness (Higher effort)
**Tasks**
1. Extract status computation into pure functions.
2. Add fixture-based tests for:
   - RCW/RC detection
   - status labeling
   - threshold edge cases
3. Add simple regression script for export shape consistency.

**Acceptance criteria**
- Deterministic pass/fail for status logic before release.

**Effort**: 8–14 hours
**Risk**: Medium
**Dependencies**: WP2

## Suggested execution order
1. WP1
2. WP2
3. WP3 + WP4 (parallel)
4. WP5

## Release strategy
- **Release A**: WP1 + WP2 (docs + status UX improvements)
- **Release B**: WP3 (PDF export)
- **Release C**: WP4 + WP5 (scalability + reliability hardening)

## Definition of done (overall)
- README and supporting docs reflect new behavior.
- Status taxonomy is consistent across UI, CSV, and PDF.
- Large-hub scan behavior is resilient under throttling.
- Regression checks exist for core classification and status logic.
