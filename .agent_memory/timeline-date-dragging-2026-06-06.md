<!--
SPDX-FileCopyrightText: 2026 The PMForge Contributors
SPDX-License-Identifier: CC0-1.0
-->

# Timeline Date Dragging - 2026-06-06

- The broad release hardening and validation checkpoint was committed as `4c7f7d9`.
- The Timeline date-dragging slice was committed after the release gate passed.
- The root `.gitignore` build-output rule was narrowed from `pmforge` to `/pmforge` so it no longer hides the `cmd/pmforge` source tree. The generated embed copy under `cmd/pmforge/frontend/dist/` remains ignored.
- `make license-check` now removes generated `cmd/pmforge/frontend/dist/` output and `.DS_Store` files before running REUSE, because REUSE scans ignored generated files when they exist.
- `scripts/check-release.sh` now removes generated embed output before direct REUSE linting, builds the frontend, copies `frontend/dist` into `cmd/pmforge/frontend/dist`, then runs release-scope, memory-safety, race, and build checks against an available Wails embed tree.
- `cmd/pmforge/main.go` now exposes `MoveTimelineEntry(kind, sourceID, dateISO)` for editable timeline boundaries.
- Supported moves are date-only updates for `project_start`, `project_end`, `sprint_start`, and `sprint_end`. Deployments remain read-only timeline events because they are DORA history.
- `internal/timeline.Entry` now exposes `editable` and `edit_field` metadata so the UI does not infer write permissions from presentation-only labels.
- `TimelineView.svelte` supports pointer dragging, date input editing, and left/right keyboard nudges for editable project and sprint date events.
- Regression coverage: `cmd/pmforge/timeline_move_test.go` verifies project and sprint date moves, read-only deployment rejection, invalid date ordering, source mismatch handling, and editable metadata in the returned timeline.

Verification run:

- `go test -count=1 ./cmd/pmforge -run 'TestMoveTimelineEntry'`
- `go test -count=1 ./cmd/pmforge ./internal/timeline`
- `make frontend-stability`
- `go test -count=1 ./cmd/... ./internal/...`
- `make license-check`
- `make check-release`
