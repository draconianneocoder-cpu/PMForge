<!--
SPDX-FileCopyrightText: 2026 James L. Burns and The PMForge Contributors
SPDX-License-Identifier: GFDL-1.3-or-later
-->

# Session Notes - 2026-06-05

## Active Checkout

- Work continued in `/Users/jamesburns/Documents/GitLab/PMForge - Go + Typescript`.
- The older Claude-project checkout path should not be assumed valid for release verification.
- The worktree is intentionally dirty with a large staged bundle plus additional unstaged edits. Avoid whole-file staging in files that already have unrelated unstaged changes.

## Key Decisions

- Keep the PAdES hardening path as the highest-value bounded work before moving to larger V3 items.
- Treat local CMS and ByteRange verification as required local invariants, while keeping Acrobat/DSS/veraPDF interoperability as a separate external-validation milestone.
- Keep PAdES signing as the final PDF mutation. PDF/A/XMP/OutputIntent work must happen before `pdfmeta.InjectPAdESSignature`.
- Use project-local validation gates where possible. External tools can be soft or manual, but local deterministic gates should fail release checks when their invariants break.
- Preserve REUSE compliance for every newly tracked file. Session notes in the repo root follow the documentation license pattern, `GFDL-1.3-or-later`.

## Completed This Session

- Installed and verified `reuse` 6.2.0 via `pipx`.
- Added/organized tracked license and asset metadata for REUSE compliance.
- Replaced stale ICC download handling with a tracked compact CC0 sRGB ICC profile and updated `scripts/fetch-icc.sh`.
- Hardened PAdES AcroForm integration so the invisible signature widget is reachable through existing direct and indirect AcroForm trees.
- Added CMS signer regressions in `internal/crypto/pdf_sign_test.go`.
- Added embedded CMS/ByteRange verification coverage in `internal/pdfmeta/pdfmeta_test.go`.
- Added `scripts/validate-pades.sh`, which generates `.tmp/pmforge-pades-test/signed-sample.pdf` and locally verifies the embedded CMS signature against the declared `/ByteRange`.
- Added `make check-pades` and wired the PAdES local gate into `scripts/check-release.sh`.
- Updated README, AGENT, and project memory notes for the PAdES local validation gate.

## Verification Evidence

- `make check-pades`
- `bash -n scripts/validate-pades.sh scripts/check-release.sh`
- `go test -count=1 ./internal/crypto`
- `go test -count=1 ./internal/pdfmeta`
- `go test -count=1 ./internal/crypto ./internal/pdfmeta ./internal/export`
- `make license-check`
- `git diff --check && git diff --cached --check`
- `make check-release`

The latest `make check-release` passed and included:

- Release gate scope verification.
- REUSE licensing check.
- Memory-safety gate.
- Frontend stability gate.
- Frontend build budget.
- Race detector.
- Production build.
- PDF/A-3 soft validation gate.
- PAdES local validation gate.

## Staging Notes

- Staged: `.agent_memory/reuse-tracking-2026-06-05.md`, `scripts/validate-pades.sh`, and the existing staged release bundle from earlier work.
- Unstaged by design: `Makefile`, `scripts/check-release.sh`, `README.md`, `AGENT.md`, and `internal/pdfmeta/pdfmeta_test.go` because those files already had pre-existing unstaged changes. Stage their relevant hunks deliberately before committing.
- Generated validation artifacts are under `.tmp/pmforge-pades-test/` and remain ignored.

## Next Steps

1. Normalize the staging boundary before any commit. In particular, decide whether the PAdES changes in `internal/pdfmeta/pdfmeta.go`, `internal/pdfmeta/pdfmeta_test.go`, `Makefile`, `scripts/check-release.sh`, `README.md`, and `AGENT.md` should be staged with the current release bundle.
2. Run external PAdES validators against `.tmp/pmforge-pades-test/signed-sample.pdf` or a signed export sample. Record results for Acrobat, DSS, and veraPDF when available.
3. Continue PDF/A-3 strict-conformance work by making representative generated PDFs pass the veraPDF gate reliably enough to promote `make check-pdfa` from soft to hard.
4. Decide the per-user encryption-at-rest path: SQLCipher with native build complexity, or a documented OS-level encryption stopgap for V2.
5. Defer PDM date-dragging on the Timeline until the current signing/PDF validation surface is staged and stable.
6. Before commit or handoff, rerun `make check-pades`, `make license-check`, `git diff --check && git diff --cached --check`, and `make check-release`.

## Follow-up - 2026-06-07 DSS Baseline-B Validation

- Installed DSS 6.4 separately as `dss-validation-tool` and wired `scripts/validate-pades-external.sh` to invoke it when available.
- Replaced the generic `pkcs7.AddSigner` CMS path for PDF signing with a narrow detached CMS encoder that omits CMS `signing-time` for PAdES baseline-B while retaining `contentType`, `messageDigest`, `signingCertificateV2`, and embedded certificates.
- Added a signed PDF signature dictionary `/M (D:YYYYMMDDHHmmSSZ)` timestamp so DSS no longer reports the PAdES baseline-B `/M` cardinality warning.
- Tightened the local and external PAdES gates: the local gate now requires `/M`, and the DSS branch fails on PAdES baseline requirements warnings or a non-`PAdES-BASELINE-B` `signature.format`.
- Current DSS result for `.tmp/pmforge-pades-test/signed-sample.pdf`: one signature, `signature.format=PAdES-BASELINE-B`, expected `NO_CERTIFICATE_CHAIN_FOUND` because the gate sample is self-signed and no trust source is configured.
- Follow-up docs cleanup: README and AGENT now describe DSS as executed coverage rather than a remaining TODO, with the remaining gap narrowed to Acrobat plus trusted-chain validation using a real trusted signing source.
- `scripts/release-gate-scope-check.sh` now requires README/AGENT to document the DSS `PAdES-BASELINE-B` result and rejects stale wording that treats DSS as unrun.

Verification evidence:

- `go test -count=1 ./internal/crypto ./internal/pdfmeta ./internal/export`
- `bash scripts/validate-pades-external_test.sh`
- `bash scripts/validate-pades-parallel_test.sh`
- `bash -n scripts/validate-pades.sh scripts/validate-pades-external.sh scripts/validate-pades-external_test.sh scripts/validate-pades-parallel_test.sh`
- `make check-pades`
- `make check-pades-external`
- `make license-check`
- `git diff --check`
- `git diff --cached --check`
- `make check-release`
- `make release-scope`
- `bash -n scripts/release-gate-scope-check.sh`

## Follow-up - 2026-06-08 PDF/A-3 gate: strict tooling-presence

- Closed the remaining soft hole in the PDF/A-3 gate: `validate-pdfa.sh` previously `exit 0`d when veraPDF could not be obtained, the ICC profile was missing, or no samples were generated, so the "hard" wrapper in `check-release.sh` passed vacuously in any environment without Docker/veraPDF.
- Added an explicit strictness switch `PMFORGE_PDFA_STRICT` (default `1`). Unmet preconditions now route through `pdfa_precondition_unmet`: strict -> `FAIL`/exit 1; non-strict -> `SKIP`/exit 0. A genuinely non-compliant sample still fails in either mode.
- `check-release.sh` invokes the gate with `PMFORGE_PDFA_STRICT=1` explicitly. `Makefile` help text changed from "(soft gate)" to "(hard gate; PMFORGE_PDFA_STRICT=0 to skip locally)".
- Made `ICC_PROFILE` overridable via `PMFORGE_ICC_PROFILE` for hermetic testing of the precondition branches.
- Renderers needed no changes: all three representative samples already pass veraPDF 1.30.2 PDF/A-3b (`isCompliant="true"`, 146 passed / 0 failed rules).
- Files touched: `scripts/validate-pdfa.sh`, `scripts/check-release.sh`, `Makefile`, `README.md`, `AGENT.md`, `session-notes.md`. No new tracked files (REUSE unaffected).

Verification evidence (run against a `/tmp` tmpfs copy because the working mount blocks `rm`; Go 1.26.4 + veraPDF 1.30.2 fetched into the sandbox):

- `bash -n scripts/validate-pdfa.sh scripts/check-release.sh`
- Full `validate-pdfa.sh` happy path with real veraPDF, strict default -> all three samples OK, gate PASSED (exit 0).
- `bash scripts/validate-pdfa-lib_test.sh` -> passed.
- ICC-missing: strict -> exit 1 (FAIL); non-strict -> exit 0 (SKIP).
- veraPDF-unavailable (empty PATH, no Docker): strict -> exit 1 (FAIL); non-strict -> exit 0 (SKIP).

Next handoff: on a machine with Docker or a preinstalled `verapdf` (and Go), rerun `make check-pdfa` and `make check-release` to confirm the strict gate end-to-end, then `make license-check` and `git diff --check` before commit. (The veraPDF GitHub-releases auto-download URL is dead/404 — provide Docker or a `verapdf` CLI on PATH in CI.)

## Follow-up - 2026-06-08 Documents package unit tests

- Confirmed handoff checks from the PDF/A-3 strict gate: `make check-pdfa` (strict default, veraPDF 1.28.1 on PATH), `make license-check` (274/274 files compliant), `make check-release` (all 9 gates pass). Working tree was clean.
- Added `internal/documents/documents_test.go` with 33 tests covering the document registry (`All`, `Get`, `ByPhase`), `DefaultContent` round-trip for all 25 kinds (including the two Word/Excel alias pairs), and `TestRender_AllKindsProduceValidPDF` which smoke-tests all 25 dispatcher branches. All 33 pass race-clean.
- Closed stale AGENT.md TODO #9 (bespoke renderers pending) — all 23 bespoke renderers + 2 aliases are confirmed wired into `renderRaw`.

Verification evidence:

- `go test -v -count=1 ./internal/documents/` -> 33 PASS
- `go test -count=1 -race ./internal/documents/` -> ok
- `make check-pdfa` -> PASSED (all three samples, strict mode)
- `make license-check` -> compliant
- `git diff --check && git diff --cached --check` -> clean
- `make check-release` -> PMForge is ready for release

## Follow-up - 2026-06-09 stats package remaining engine tests

- Identified `charts/stats` at 42% coverage (only Pareto and Control were tested from the 2026-06-04 session). The six remaining engines (Line, Bar, Pie, BurnUp, BurnDown, CumulativeFlow) had 0% coverage despite being pure parse+layout math.
- Added `internal/charts/stats/stats_remaining_test.go` with 40 tests covering `ParseXxx` (empty, `{}`, invalid JSON, valid doc for Line), `LayoutXxx` for all six engines, and `computeIdealBurnDown` (n=0, empty remaining, n=1, n=5 trajectory).
- Package now at 95.3%, race-clean. Remaining 4.7% is `ParseXxx` valid-doc success paths (implicitly exercised by layout tests) and the unreachable `out[i] < 0` clamp in `computeIdealBurnDown`.

Verification evidence:

- `go test -count=1 ./internal/charts/stats/` -> ok (coverage 95.3%)
- `go test -count=1 -race ./internal/charts/stats/` -> ok
- `go test ./internal/... ./cmd/...` -> ALL PASS (29 packages)

## Follow-up - 2026-06-08 Matrix engine layout tests

- Surveyed coverage across all packages: test *presence* is saturated (only `sigma/domain`, pure type defs with zero functions, has no tests — intentional). Used coverage % to find untested *depth*.
- Found `charts/matrix` at 29.5% vs sibling engines at 83-95%. Cause: only `raci.go` tested; `swot.go`, `stakeholder.go`, `generic.go` Parse/Layout were 0% and are pure parse+layout logic (not gofpdf glue).
- Added `swot_test.go`, `stakeholder_test.go`, `generic_test.go`. Matrix package now 95.8%, race-clean. Remaining uncovered lines are unreachable defensive guards.
- Rejected a tempting-but-wrong `cli` refactor (ParseFlags at 5%): those lines are `flag` registration boilerplate, uncoverable by nature; refactoring the launch entry point to test stdlib behaviour is risk without reward.

Verification evidence:

- `go test -count=1 ./internal/charts/matrix/` -> ok (coverage 95.8%)
- `go test -count=1 -race ./internal/charts/matrix/` -> ok
- `go test ./internal/... ./cmd/...` -> ALL PASS

## Follow-up - 2026-06-09 charts dispatcher and pdfmeta trivial tests

- Surveyed coverage across all packages after the update+auth session. Identified `internal/charts` at 77.0% (engines.go dispatcher at 74.5%) and two zero-coverage helpers in `internal/pdfmeta` (`icc.go`, `xmlEscape`) as the next pure-logic targets.
- Added 7 tests to `internal/charts/charts_test.go` (file now 25 functions): `TestLayout_AllKinds_RejectsBadJSON` (table test, all 20 kinds with invalid JSON), `TestLayout_Network_CycleError`, `TestLayout_PERT_CycleError`, `TestLayout_CPM_CycleError`, `TestLayout_CauseAndEffect_NilRootError` (empty doc → ErrNoRoot), `TestLayout_Workflow_CycleError`, `TestLayout_Activity_CycleError`. All parse-error and layout-error arms in `engines.go:Layout()` are now covered.
- Added 7 tests to `internal/pdfmeta/pdfmeta_test.go`: `TestXmlEscape_Empty`, `TestXmlEscape_AllSpecialChars`, `TestXmlEscape_NoSpecialChars`, `TestXmlEscape_Mixed`, `TestDefaultICCProfile_NonNil`, `TestDefaultICCProfile_ReturnsCopy`, `TestHasDefaultICC_ReturnsTrue`.

Verification evidence:

- `go test -count=1 ./internal/charts/ ./internal/pdfmeta/` -> ok (both)
- `go test -count=1 -race ./internal/charts/ ./internal/pdfmeta/` -> ok (both)
- `go test ./internal/...` -> ALL PASS (28 packages, 1 no-test-files)
- `make license-check` -> 279/279 compliant
- `git diff --check && git diff --cached --check` -> clean

## Follow-up - 2026-06-09 agile/dora and calendar coverage

- Surveyed coverage across all packages. Two pure-logic targets identified: `agile/dora.go` (`formatHours` at 41.7%, `ComputeDORA` zero-now branch at 97.1%) and `calendar/calendar.go` (`For` at 54.5%: 5 untested country switch cases; `WorkdaysFrom` at 80%: negative-days path uncovered).
- Added 2 tests to `internal/agile/dora_test.go` (file now 14 functions): `TestFormatHours` (table test, 6 cases covering `≤0`, `<1h→min`, `<48h→h`, `<30d→d`, `≥30d→wk`) and `TestComputeDORAZeroNowFallsBack` (covers the `if now.IsZero()` guard). All `dora.go` functions now 100%.
- Added 2 tests to `internal/calendar/calendar_test.go` (file now 7 functions): `TestFor_AllSupportedCountries` (table test over GB, UK, CA, DE, FR, AU — each verified non-nil, correct CountryCode, Christmas 2026-12-25 is a holiday) and `TestWorkdaysFrom_BackwardWalk` (negative days: one workday before Monday 2026-01-05 = Friday 2026-01-02). Calendar coverage: 78.1% → 100%.

Verification evidence:

- `go test -count=1 ./internal/agile/ ./internal/calendar/` -> ok (both)
- `go test -count=1 -race ./internal/agile/ ./internal/calendar/` -> ok (both)
- `go test ./internal/...` -> ALL PASS (28 packages, 1 no-test-files)
- `make license-check` -> 279/279 compliant
- `git diff --check && git diff --cached --check` -> clean

## Follow-up - 2026-06-09 update + auth package tests

- Identified `internal/update` (isNewer/splitVer/atoi at 0%; VerifyManifest at 0%) and `internal/auth` (NeedsRehash at 0%; VerifyPassword missing several error branches; HashPassword missing empty-password path) as the next pure-logic coverage targets using the glue-vs-logic discriminator.
- Added 23 new tests to `internal/update/check_test.go` (file now 25 total): `VerifyManifest` 7 tests (happy path, wrong key, bad public key length, invalid manifest JSON, bad payload base64, bad signature base64, invalid payload JSON after successful signature verification); `isNewer` 7 tests; `splitVer` 4 tests; `atoi` 5 tests. `VerifyManifest` now 100%, `isNewer`/`splitVer`/`atoi` all 100%.
- Added 18 new tests to `internal/auth/password_test.go` (file now 21 total): `HashPasswordRejectsEmptyPassword`, 8 `VerifyPassword` error-branch tests (wrong part count, wrong algorithm, bad version scan, wrong version, bad param scan, bad salt base64, zero memory, zero time), 8 `NeedsRehash` tests (malformed, wrong algorithm, bad param format, weaker memory/time/threads, current params, stronger params), `TestHashVerifyPassword_RoundTrip` (single argon2 call covering HashPassword + VerifyPassword correct + ErrMismatch). `HashPassword` 100%, `NeedsRehash` 100%, `VerifyPassword` 96.4%.

Verification evidence:

- `go test -count=1 ./internal/update/ ./internal/auth/` -> ok (both)
- `go test -count=1 -race ./internal/update/ ./internal/auth/` -> ok (both)
- `go test ./internal/...` -> all internal packages pass race-clean; `cmd/pmforge` not tested (requires built `frontend/dist`)
- `make license-check` -> 279/279 compliant
- `git diff --check && git diff --cached --check` -> clean

## Follow-up - 2026-06-09 sigma/stats capability bands and timeline parseDate

- Surveyed function-level coverage. Two pure-logic gaps: `sigma/stats/basic.go` `CalculateCapability` at 76.9% (only the top DPMO band was tested) and `timeline/timeline.go` `parseDate` at 66.7% plus the `Build` failed-deployment branch.
- Rewrote `TestCalculateCapability_DPMOBands` into a 6-row table covering every DPMO band. Drives sigma level deterministically via the dataset `{-1, 1}` (sample StdDev exactly √2) with centered spec `USL=√2*k, LSL=-√2*k`, giving `sigmaLevel = k + 1.5`. `CalculateCapability` now 100%.
- Added `TestBuildFailedDeploymentTitle` (covers the `!d.Successful` -> "(failed)" branch in `Build`) and `TestParseDate` (direct table: empty, ISO date, RFC3339, RFC3339Nano, garbage). `Build` now 100%; `parseDate` 88.9% (the RFC3339 fallback at 137-139 is unreachable: RFC3339Nano is a superset, left as defensive code).
- Coverage: sigma/stats 86.0% -> 100%; timeline 86.7% -> 96.7%.

Verification evidence:

- `go test -count=1 ./internal/sigma/stats/ ./internal/timeline/` -> ok (both)
- `go test -count=1 -race ./internal/sigma/stats/ ./internal/timeline/` -> ok (both)
- `go test ./internal/...` -> ALL PASS (no failures)
- `make license-check` -> 279/279 compliant
- `git diff --check && git diff --cached --check` -> clean

## Follow-up - 2026-06-09 charts/dag encoders and layout wrappers

- `charts/dag` was the laggard pure-logic engine at 83.7% (siblings flow/stats/matrix at 94-96%). Function-level survey found the gaps: four `Encode*` functions at 0%, the `Layout{CPM,Network,PERT}` wrappers at 0% in-package (they are exercised only via the `charts` dispatcher, and per-package coverage ignores cross-package callers), `NewLayeredNode` at 0%, and the `walk` nil guard.
- Added 12 tests to `internal/charts/dag/dag_test.go`: four Encode round-trips (`Parse(Encode(doc))`, which also close the matching Parse success paths), `NewLayeredNode`, `LayoutNetwork` (chain + cycle), `LayoutPERT` (fills Expected/Duration, asserts on the in-place-mutated node slice), `LayoutCPM` (linear chain marks every node critical + cycle), and `walk(nil)`.
- Coverage: charts/dag 83.7% -> 94.5%. Remaining gaps are the `json.Marshal` error guards in `Encode*` (unreachable for these plain structs, capped at 75%) and pre-existing layout-branch partials outside this task's 0%-function scope.

Verification evidence:

- `go test -count=1 ./internal/charts/dag/` -> ok (coverage 94.5%)
- `go test -count=1 -race ./internal/charts/dag/` -> ok
- `go test ./internal/...` -> ALL PASS (no failures)
- `make license-check` -> 279/279 compliant
- `git diff --check` -> clean

## Follow-up - 2026-06-09 documents pure helpers

- `internal/documents` is ~95% gofpdf rendering glue, but the renderers are fed by a seam of pure transforms that were untested. Identified nine: `normaliseExecutionTasks`, `sumExecutionCost`, `computeProjectWindow`, `parseDate` (execution_plan.go), `partitionIssues`/`isIssueResolved`/`issueSeverityOrder` (issue_log.go), `procurementTotal`, `budgetSubtotal`.
- Added `internal/documents/helpers_test.go` (new file, SPDX header) with focused tests: parseDate formats, computeProjectWindow (empty/inclusive-Days/multi-task/start-only-extends-window), the three cost aggregations, isIssueResolved (trim+case-fold), issueSeverityOrder (each + default), partitionIssues (split + severity sort order), and one representative accessor default-branch test (`TestNormaliseExecutionTasks_DefaultsOnBadInput`) standing in for the ~20 near-identical normalise/getStringX/getFloatX copies.
- All nine targeted helpers now 100%. Package coverage 39.3% -> 40.5% (small delta expected: gofpdf glue dominates the statement count and is intentionally untested).
- Note: this empties the pure-logic well. Remaining low-coverage packages (cli, export, charts/pdfrender, sigma/service, db) are predominantly glue already rejected by the discriminator; a future survey may legitimately find no target.

Verification evidence:

- `go test -count=1 ./internal/documents/` -> ok (coverage 40.5%)
- `go test -count=1 -race ./internal/documents/` -> ok
- `go test ./internal/...` -> ALL PASS (no failures)
- `make license-check` -> 279/279 compliant
- `git diff --check` -> clean

## Follow-up - 2026-06-09 stale-doc TODO cleanup

- Pure-logic coverage well is dry (per prior follow-up). Re-read the task as "complete the TODO list": grepped TODO/FIXME/"this v1"/"follow-up"/"not yet" across internal+cmd and cross-checked against README's "Real TODOs in the V2 scaffold" list. README's open items are all non-code (PDF/A-3 soak, PAdES Acrobat validation, SQLCipher V3-deferred). The actionable items were two stale comments contradicting shipped code:
  - `internal/documents/report.go`: comment claimed charts were "referenced only by ID in this V1" with raster embedding "a follow-up". The code already embeds each chart_ref as a vector visualisation via `pdfrender.RenderChartToPDF` (matches README TODO #12 Done). Rewrote the comment.
  - `internal/charts/engines.go`: comment claimed "Stats / Matrix / Flow families return ErrEngineNotImplemented" and "DAG fully implemented in V2.1". All 20 kinds have switch arms. Rewrote to list all four families implemented; clarified that `ErrEngineNotImplemented` is the defensive default for an unregistered-renderer kind (still live: returned at the switch default and handled non-fatally in main.go), not dead code.
- Comment-only changes, no behavior change. README needs no edit: its TODO list already marks #9 and #12 done.

Verification evidence:

- `go build ./internal/...` -> ok
- `go vet ./internal/charts/ ./internal/documents/` -> clean
- `go test ./internal/...` -> ALL PASS (no failures)
- `git diff --check` -> clean

## Follow-up - 2026-06-09 pdfrender error-sentinel robustness

- Broad TODO/FIXME scan across all tracked files (frontend, scripts, Go) confirmed no actionable feature TODO remains; README's open items stay non-code. The scan did surface a real latent bug: `internal/charts/pdfrender/dispatcher.go` `isEngineNotImpl` matched the charts not-implemented error by string literal (`err.Error() == "charts: engine renderer not yet implemented"`).
- Replaced the string compare with `errors.Is(err, charts.ErrEngineNotImplemented)` (pdfrender already imports charts). The old form silently breaks if the message text drifts and does not unwrap, so a wrapped sentinel would be treated as a hard render failure instead of a skip.
- Added `TestIsEngineNotImpl` to `pdfrender_test.go`: nil/sentinel/wrapped-sentinel/unrelated cases. The wrapped-sentinel row is the behavior the fix buys and would fail against the old string compare.

Verification evidence:

- `go vet ./internal/charts/pdfrender/` -> clean
- `go test -count=1 ./internal/charts/pdfrender/` -> ok
- `go test -count=1 -race ./internal/charts/pdfrender/` -> ok
- `go test ./internal/...` -> ALL PASS (no failures)
- `make license-check` -> 279/279 compliant
- `git diff --check` -> clean

## Follow-up - 2026-06-09 frontend UI/UX: critical mount fix + global polish

- Task: ensure frontend UI/UX operability, elegance, polish. Established baseline (svelte-check 0 errors/203 files, vite build clean), then actually LAUNCHED the app via the preview tool. Discovered the app did not mount at all (`#app` empty, `childCount: 0`).
- ROOT CAUSE (critical): `src/lib/toast.ts` used the `$state` rune in a plain `.ts` file. Svelte 5 only compiles runes in `.svelte`/`.svelte.js`/`.svelte.ts`; in a plain `.ts`, `$state` throws `rune_outside_svelte` at module load. App -> ToastContainer -> toast import crashed the whole mount. svelte-check passed (ambient rune types) and vite build passed (esbuild bundles the call) - the break is runtime-only, invisible to the release gates which never launch the UI.
- FIX: renamed `toast.ts` -> `toast.svelte.ts` (git mv) and updated all 12 importers across 11 files to `'../toast.svelte'` / `'../../toast.svelte'` (matching the `session.svelte.ts` -> `'session.svelte'` convention). After the fix the app mounts: `#app` children = [login form, ToastContainer]; verified live.
- POLISH (all global, in app.css / index.html, so every screen benefits without per-component edits):
  - Keyboard focus ring: 40 files used `outline-none` (transparent outline) and 0 used focus-visible; added an unlayered `:focus-visible` rule (outranks the layered `.outline-none` utility) with a 2px accent ring scoped to interactive elements.
  - `prefers-reduced-motion` media block neutralises animation/transition durations app-wide.
  - `color-scheme: dark` + `accent-color` on `:root` (native scrollbars/checkboxes/date pickers render dark + on-brand).
  - `-webkit-font-smoothing: antialiased` for crisper text.
  - index.html: inline `background-color:#020617` on `<html>` + `<meta name=color-scheme content=dark>` to kill the flash-of-white on cold start.
  - App.svelte route loader: replaced bare "Loading view..." with a spinner + retained "Loading" text label (so reduced-motion users keep the signal).
  - Login.svelte: autofocus the username field on mount (verified activeElement = username input).
- Committed `.claude/launch.json` (frontend dev server config for the preview tool) with a REUSE.toml annotation (JSON cannot carry an inline SPDX header).

Verification evidence:

- `npm run check` (svelte-check) -> 203 files, 0 errors, 0 warnings
- `npm run build` (vite) -> clean
- Live preview (npm run dev + browser): `#app` mounts [login + ToastContainer]; focus-visible rule loaded with `rgb(0,212,255) solid 2px`; `color-scheme: dark`; html bg `rgb(2,6,23)`; reduced-motion media rule present; username autofocused
- `make license-check` -> compliant (launch.json annotated in REUSE.toml)
- Note: `go build ./cmd/...` reports the pre-existing `all:frontend/dist` embed-path condition (present since session start; the Wails/make build handles dist placement), unrelated to these changes.

## Follow-up - 2026-06-09 frontend runtime smoke gate

- Implemented the runtime smoke-check recommended after the toast-rune mount bug (which svelte-check + vite build both passed). Goal: fail the release if the app would not mount.
- `frontend/scripts/smoke-mount.mjs`: starts a Vite dev server (middleware mode), `ssrLoadModule('/src/App.svelte')` to execute the whole synchronous module graph through the real Svelte compiler, then SSR-renders App via `svelte/server`. Any load-time or synchronous render throw fails. Zero new dependencies (reuses Vite); no jsdom/Playwright/vitest, matching the project's minimal-dep ethos.
- SSR is the right mode: onMount (window.go) and $effect (dynamic route imports) are skipped, so the foundation loads in Node without the Wails backend. Targets App.svelte, not main.ts (main.ts calls mount() against document at top level -> would false-positive).
- Wiring: `scripts/frontend-smoke-check.sh` (bash wrapper), `make frontend-smoke`, and `check-release.sh` step 4b after the stability gate. Both new files carry SPDX headers.
- Proved it catches the bug end-to-end: injected a plain `.ts` using `$state` into App's graph -> `make frontend-smoke` exits 1 ("the app failed to load or render ... #app would not mount"); restored via git checkout -> exits 0. The throw message differs by context (`rune_outside_svelte` in browser vs `$state is not defined` under Node SSR); the gate keys off any throw.

Verification evidence:

- `make frontend-smoke` -> "App loaded and rendered (482 bytes of HTML)", exit 0
- broken-graph probe -> gate exit 1 (then restored, exit 0)
- `bash -n scripts/frontend-smoke-check.sh scripts/check-release.sh` -> syntax ok
- `make license-check` -> compliant

## 2026-06-10 — Scheduling core roadmap + calendar-anchored CPM (roadmap step 1)

- Added the **Scheduling core roadmap (V3)** to README "Real TODOs" (items 14–20) and AGENT.md §8, in dependency order: date anchoring, dependency types + lag, constraints, progress/baselines, EVM, resource layer, MSPDI import + first-class Gantt. PMForge stays local-first; no cloud features on the roadmap.
- **Doc-accuracy fix:** README intro and the kernel package comment claimed EVM as implemented scheduling math; no PV/EV/AC/SPI/CPI computation exists. Both now state EVM is roadmap item 18. Rule recorded in AGENT.md: docs must not claim EVM until item 18 ships.
- **Implemented roadmap item 14, step 1 — `kernel.AnchorSchedule`** (`internal/kernel/anchor.go`): maps CPM day-offsets onto real calendar dates from the project start date, skipping non-working days via an injected `WorkdayFunc` (kernel stays pure; `internal/calendar.Calendar.IsWorkday` is the production predicate). Offset 0 = first workday on/after project start; finish = last occupied day; milestones start/finish same day; defensive 366-day cap prevents pathological calendars from hanging. New `Task.StartDate`/`Task.FinishDate` (YYYY-MM-DD, omitempty) + mirrored optional fields on `KernelTask` in `wails-window.d.ts`.
- **MSPDI export upgraded** (`internal/export/mspdi.go`): emits anchored dates (T08:00:00 start / T17:00:00 finish) when present, falling back to the legacy offset-from-today epoch for un-anchored maps; tasks now emitted in deterministic (ES, ID) order for reproducible archives.
- **Wired in `cmd/pmforge/main.go`:** `exportScheduleReportAs` calls new `anchorScheduleToProject` (project start date + country calendar) after `CalculateCPM`; `parseProjectDate` accepts YYYY-MM-DD and RFC3339. Remaining for item 14: thread anchored dates into the CPM chart editor and Timeline/Gantt views.

Verification evidence:

- `go vet ./internal/kernel ./internal/export` -> clean
- `go test ./internal/kernel ./internal/export` and same with `-race` -> ok (new: 6 AnchorSchedule tests incl. weekend skip, weekend-start roll-forward, milestone, nil calendar, pathological-calendar termination; 3 MSPDI tests incl. anchored dates and determinism)
- `go build ./cmd/...` -> compiles (dist copied per Makefile convention, then cleaned up)

## Follow-up - 2026-06-10 roadmap item 14 completed (anchored dates in the CPM editor)

- **`charts.LayoutWithSchedule(kind, raw, projectStart, isWorkday)`** (engines.go): CPM-only anchored layout entry point. Non-CPM kinds and a zero projectStart delegate to plain `Layout`, so existing callers and behaviour are untouched. New `dag.AnchorCPMDates(&doc, start, isWorkday)` rebuilds the kernel task map from the annotated doc and copies StartDate/FinishDate back to nodes; `dag.LayeredNode` gains `start_date`/`finish_date` (omitempty).
- **`App.LayoutChart`** now resolves the open project's start date + country calendar and calls `LayoutWithSchedule`; projects without a start date keep the plain day-offset layout.
- **`CPMEditor.svelte`**: canvas node's cyan ES/EF row shows `start → finish` real dates when anchored (offsets remain in the detail panel); detail panel gains Start date / Finish date rows; helper text documents that dates appear once Project Settings has a start date. `LayeredNode` TS interface in the shell + `KernelTask` d.ts extended with the optional fields.
- README roadmap item 14 marked done; the date-axis Gantt strip is deliberately deferred to item 20 (first-class Gantt chart kind). AGENT.md tracker updated to match.
- Style note: gofmt 1.26 wants comment-block/struct-alignment reflows in several pre-existing files (dag_test.go, fishbone.go, registry.go, odt.go, ...). Left untouched to avoid diff churn; only new/modified hunks are formatted.

Verification evidence:

- `go vet ./internal/charts/... ./internal/kernel` -> clean
- `go test -race ./internal/charts/... ./internal/kernel ./internal/export` -> ok (new: 2 AnchorCPMDates tests, 3 LayoutWithSchedule tests incl. non-CPM byte-identical delegation)
- `go build ./cmd/...` -> compiles (dist staged then cleaned; root binary not kept)
- `npm run check` -> 0 errors, 0 warnings; `make frontend-smoke` -> App loaded and rendered, exit 0

## Follow-up - 2026-06-10 roadmap item 15 (dependency types FS/SS/FF/SF + lag)

- **`kernel.Link` / `kernel.LinkType`** (scheduler.go): typed PDM links with lag (negative = lead). `Task.Links` added alongside legacy `Precedents` (FS+0); a typed Link for the same predecessor wins (`effectiveLinks`). Unknown types normalise to FS.
- **PDM passes**: forward candidates FS: pEF+lag, SS: pES+lag, FF: pEF+lag-dur, SF: pES+lag-dur, ES clamped >= 0 (leads cannot schedule before project start). Backward: every task's LF starts at projectEF and successor candidates only tighten it -- required because SS successors do not constrain a predecessor's finish and FS leads can produce candidates beyond project finish (caught by TestCPM_StartToStart during development; the old terminal-only projectEF default was insufficient). topoSort builds edges from effectiveLinks.
- **`dag.ParseLinkLabel`** (cpm.go): "FS", "SS+2", "ff - 1.5", "+3" -> (type, lag); free text / malformed labels fail soft to FS+0 so annotation labels never break scheduling. `LayoutCPM` now feeds typed links to the kernel; `cpmChartDataToKernelTasks` in main.go carries edge labels so schedule-report and MSPDI exports honour them (new import: pmforge/internal/charts/dag).
- **Layered shell UI**: "Incoming links" section in the detail panel -- every edge into the selected node gets a label input (placeholder FS); CPM editors additionally show the link-grammar hint. Generic across Network/PERT/CPM since labels already exist in the data model.
- README roadmap item 15 + AGENT.md tracker marked done; CPMEditor helper text documents the grammar.

Verification evidence:

- `go test -race ./internal/kernel ./internal/charts/... ./internal/export` -> ok (new: 10 kernel PDM tests incl. all four link types, lead clamping, typed-link-wins, link cycles, mixed-link backward pass; 2 dag tests for label parsing + LayoutCPM honouring SS+1)
- `go vet` clean; `go build ./cmd/...` -> compiles (dist staged then cleaned)
- `npm run check` -> 0 errors, 0 warnings; `make frontend-smoke` -> App loaded and rendered, exit 0

## Follow-up - 2026-06-10 roadmap item 16 (task constraints with violation surfacing)

- **`kernel.ConstraintType`** (scheduler.go): ASAP (default/empty), ALAP, SNET, FNLT, MFO. Task gains Constraint, ConstraintDate, ConstraintDay/ConstraintArmed (set by arming), ConstraintViolated (computed).
- **`kernel.ApplyConstraintDates`** + **`kernel.DayOffset`** (anchor.go): DayOffset is the inverse of AnchorSchedule's mapping (date -> working-day index; pre-start clamps to 0; non-workdays map to the next workday; 100k-day walk cap). Finish constraints store day+1 because EF is exclusive (finishOffset = ceil(EF)-1). Date constraints require a project start date; un-anchored schedules leave them dormant. ALAP needs no date.
- **Pass semantics** (links always win): SNET lifts ES in the forward pass; MFO pulls ES to date-dur or flags violation when links force past the pin; backward pass pins LF to MFO date (flagging if successors need it earlier) and caps LF at FNLT (flagging when EF overruns). FNLT/MFO squeezes produce negative float; **IsCritical changed from |Float|<eps to Float < eps** so super-critical tasks read as critical. ALAP post-pass moves a task to its late dates (ES=LS, EF=LF, Float=0) after both passes so no other task shifts.
- **dag** (cpm.go): LayoutCPM refactored into cpmTasksFromDoc + copyCPMResults; new **`LayoutCPMScheduled(doc, start, isWorkday)`** = arm constraints -> CPM -> AnchorSchedule -> copy-back (incl. ConstraintViolated). LayeredNode gains constraint/constraint_date (input) + constraint_violated (computed). `charts.LayoutWithSchedule` now calls LayoutCPMScheduled (AnchorCPMDates kept as public API).
- **main.go**: anchorScheduleToProject replaced by **`scheduleProjectTasks`** (arm -> CPM -> anchor; plain CPM when no start date); cpmChartDataToKernelTasks parses node constraint fields.
- **CPMEditor.svelte**: constraint dropdown (ASAP/ALAP/SNET/FNLT/MFO), date picker for the date-bearing kinds, amber violation explainer panel, amber dashed outline + "!" canvas marker; helper text documents the rules. Shell LayeredNode TS interface extended.
- README roadmap item 16 + AGENT.md tracker marked done.

Verification evidence:

- `go test -race ./internal/kernel ./internal/charts/... ./internal/export` -> ok (new: 9 kernel constraint tests incl. DayOffset table, SNET delay + anchored date, MFO pull/violation, FNLT quiet/violation + negative float criticality, ALAP float consumption, unarmed/bad-date no-ops; 2 dag tests for LayoutCPMScheduled honouring case-insensitive constraints and plain-path ignoring them)
- One test expectation corrected during development (single-task FNLT: float is bounded by projectEF, not the constraint cap - rewrote with a parallel driver task)
- `go build ./cmd/...` -> compiles; `npm run check` -> 0 errors; `make frontend-smoke` -> renders, exit 0
- gofmt: only pre-existing deviations remain (engines.go comment block etc.), untouched per style-preservation note above

## Follow-up - 2026-06-10 roadmap item 17 (progress, milestones, baseline snapshots)

- **Kernel** (scheduler.go, baseline.go): Task gains PercentComplete (clamped 0-100 inside CalculateCPM; reporting-only, never moves dates), Milestone flag, ActualStart/ActualFinish (DateLayout; consumed by EVM in item 18 - no entry UI yet). New `kernel.CompareSchedules(current, baseline)` -> map[taskID]ScheduleVariance with start/finish variance in working days (positive = slip) and the baseline's anchored dates; tasks present in only one map are skipped.
- **DB** (sqlite.go, baselines.go): new `baselines` table (FK to project + charts with CASCADE; indexed by chart). Rows are immutable snapshots: SaveBaseline (insert-only), GetBaseline, ListBaselines (newest first), DeleteBaseline. Additive CREATE TABLE IF NOT EXISTS keeps the V1->V2 migration story intact.
- **Wails surface** (main.go §Schedule baselines): SetScheduleBaseline snapshots the chart's FULLY scheduled task map (constraints armed -> CPM -> anchored) as JSON; ListScheduleBaselines; DeleteScheduleBaseline; CompareScheduleBaseline (latest when baselineID empty, returns {} when none) re-schedules current chart data and diffs via kernel.CompareSchedules.
- **dag**: LayeredNode gains percent_complete/milestone (inputs persisted in chart JSON); cpmTasksFromDoc passes them to the kernel so exports see progress.
- **Shell**: new optional `toolbarExtra` snippet prop rendered before + Node (generic; CPM uses it for Set baseline).
- **CPMEditor**: Set baseline / Re-baseline (n) toolbar button with transient status; % Complete input + Milestone checkbox; canvas progress strip (cyan, bottom edge) + cyan diamond milestone marker; baseline variance block in detail panel (baseline dates, start/finish vs baseline, red late / green early / slate on-plan). Baseline fetches fail soft - the editor never blocks on them.
- d.ts: BaselineRecord, ScheduleVariance, 4 App methods, KernelTask progress/constraint fields.

Verification evidence:

- `go test -race ./internal/kernel ./internal/db ./internal/charts/... ./internal/export` -> ok (new: CompareSchedules slip/new-task/baseline-dates tests, progress clamping test, baselines CRUD test with FK fixture - first attempt failed FOREIGN KEY until the fixture created real project/chart rows - and newest-first ordering + empty-chart cases)
- `go build ./cmd/...` compiles; `npm run check` -> 0 errors; `make frontend-smoke` -> renders
- gofmt clean on all touched files (settings_test.go flag is pre-existing)

## Follow-up - 2026-06-10 roadmap item 18 (Earned Value Management)

- **`kernel.ComputeEVM`** (evm.go): per-task PV = BudgetedCost x planned fraction at the status day (linear across ES..EF; zero-duration milestones earn fully at ES), EV = BudgetedCost x PercentComplete/100, AC = ActualCost. Totals + SV/CV, SPI/CPI (0 = "n/a" when denominator 0), EAC = BAC/CPI (BAC fallback), ETC, VAC. Per-task breakdown sorted by ID for determinism. Task gains BudgetedCost/ActualCost (scheduler.go).
- **Status-date mapping**: `App.ComputeScheduleEVM(chartID, asOfDate)` ("" = today) re-schedules the chart's tasks, maps the date through `kernel.DayOffset` with the project country calendar, and **errors without a project start date** rather than emit offset-less numbers.
- **Threading**: dag LayeredNode + cpmTasksFromDoc and main.go's cpmChartDataToKernelTasks now carry percent/milestone/actuals/costs, so the export path sees them too (engine.go ReportPayload comment updated to point at ComputeEVM for future report sections).
- **UI**: CPMEditor detail panel gains Budgeted/Actual cost and Actual start/finish inputs (the actual-date UI deferred from item 17). New `asideExtra` shell snippet hosts the chart-level "Earned value" card: status-date picker + Compute button, BAC/PV/EV/AC grid, SV/CV/SPI/CPI with red/green semantics, EAC/ETC/VAC, and a plain-language footnote. d.ts: EVMetrics/TaskEV + method + node cost fields.
- **Doc-accuracy claims closed**: kernel package comment now lists EVM as implemented (the rule "docs must not claim EVM until item 18" is retired); README intro re-expanded to include EVM/baselines/constraints; README item 18 + AGENT.md tracker marked done. Optional follow-up noted: EVM sections in the Status Report renderer / combined report builder.

Verification evidence:

- `go test -race ./internal/kernel ./internal/charts/... ./internal/export ./internal/db` -> ok (new: 5 EVM tests - textbook totals incl. SPI 0.75 / CPI 0.6 / EAC = BAC/CPI, mid-task linear PV, zero-denominator conventions, milestone PV step, deterministic per-task order)
- `go build ./cmd/...` compiles; `npm run check` -> 0 errors (after adding asideExtra to the shell's props destructure - svelte-check caught the omission); `make frontend-smoke` -> renders

## Follow-up - 2026-06-10 roadmap item 19 slice 1 (kernel resource core)

- **`internal/kernel/resources.go`**: Assignment {resource, units} on Task (units <= 0 normalise to 1.0); Task.Overallocated computed flag.
- **`ResourceUsage(tasks)`**: per-resource per-day demand profiles (shared horizon = last occupied day + 1; integer-day spans via the AnchorSchedule convention: round(ES) .. ceil(EF)-1; zero-duration tasks occupy nothing).
- **`DetectOverallocations(tasks, capacities)`**: capacity map (missing = 1.0), breaches sorted by (resource, day) with offender task IDs sorted; clears then sets Task.Overallocated so repeated runs are idempotent.
- **`LevelResources(tasks, capacities)`**: serial method - internal CalculateCPM (cycle -> false), ready queue picks least (LS, ID), precedence-earliest start recomputed against LEVELLED predecessors with the full FS/SS/FF/SF + lag candidate formulas, never earlier than the constrained ES, then pushed day-by-day (10k-day horizon) until every assignment fits booked capacity. Impossible demand (units > capacity) stays at its earliest start and remains visible to DetectOverallocations rather than being shoved to the horizon. Documented simplifications: integer-day booking; post-leveling LS/LF/Float still describe the precedence-only schedule.
- README item 19 marked "kernel core landed, UI remaining"; AGENT.md tracker matches. Remaining slices: assignment UI wired to stakeholders, resource histogram chart kind, Level-resources action, per-resource calendars.

Verification evidence:

- `go test -race ./internal/kernel ./internal/charts/... ./internal/export ./internal/db` -> ok (new: 10 resource tests - usage profile, overallocation detection + capacity + flag idempotence, leveling serialisation, least-float priority, links+lag after leveling, fractional-unit sharing, impossible demand, cycle, unassigned tasks)
- `go vet` clean; `go build ./cmd/...` compiles; gofmt clean on new files

## Follow-up - 2026-06-10 roadmap item 19 slice 2 (assignment UI + overallocation surfacing)

- **dag**: LayeredNode gains `assignments` ([]kernel.Assignment, input) + `overallocated` (computed). Both CPM layout paths (plain + scheduled) run `kernel.DetectOverallocations(tasks, nil)` after CalculateCPM - overallocation needs only offsets, so it works un-anchored. copyCPMResults copies the flag back.
- **main.go**: cpmChartDataToKernelTasks parses node assignments, so schedule-report/MSPDI/EVM/baseline paths all see resource demand.
- **CPMEditor**: "Assignments" section in the detail panel - per-row resource input with a stakeholder `<datalist>` (loaded via App.ListStakeholders('') fail-soft; free text always works), units input (1 = full-time), remove button, "+ Assign resource". Overallocated nodes get an orange left-edge strip on the canvas and an explainer panel; helper text documents the capacity-1.0 default.
- d.ts/shell types: ResourceAssignment, assignments/overallocated on KernelTask + LayeredNode.
- Remaining for item 19 (slice 3): resource usage/histogram chart kind, Level-resources action (design: persist levelled starts as SNET constraints), per-resource capacities/calendars UI.

Verification evidence:

- `go test -race ./internal/kernel ./internal/charts/... ./internal/export ./internal/db` -> ok (new dag test: LayoutCPM flags parallel same-resource tasks and leaves unassigned nodes clean)
- `go build ./cmd/...` compiles; `npm run check` -> 0 errors; `make frontend-smoke` -> renders

## Follow-up - 2026-06-10 roadmap item 19 slice 3 (Level action + resource histogram)

- **`App.LevelChartResources(chartID)`** (main.go): baseline precedence-only CPM pass vs a LevelResources pass on a fresh task map; every task levelling delayed gets `constraint=SNET` + `constraint_date=<levelled start>` written into the chart doc and saved. User-set non-SNET constraints are never overridden; stale SNET pins from earlier levelling runs are cleared when no longer needed. Requires a project start date (offsets -> dates). Returns pinned count.
- **`App.GenerateResourceHistogram(chartID)`**: kernel.ResourceUsage -> Bar chart (categories = real dates when anchored via a synthetic 1-day-task AnchorSchedule trick, else "Day n"; one series per resource, sorted). Snapshot semantics: the bar chart's config carries `{"source_chart_id":...}` so regeneration updates the same chart instead of accumulating copies; being a normal Bar chart it inherits the editor, pdfrender, and combined-report embedding for free (no 21st chart kind needed - decision: reuse beats new render surface).
- **Shell**: extracted `loadChart()` and exported **`reloadFromDB()`**; CPMEditor binds the shell instance and reloads after levelling so the in-memory doc can't clobber backend-written SNET pins on the next Ctrl+S (hazard caught during design review).
- **CPMEditor toolbar**: Level + Histogram buttons with transient status messages; d.ts declarations added.
- README item 19 + AGENT.md tracker updated: remaining = per-resource capacities (stakeholder record) and per-resource calendars.

Verification evidence:

- `go test -race ./cmd/... ./internal/kernel ./internal/charts/dag` -> ok (new cmd tests: levelling pins exactly the delayed task at the right date - B SNET 2026-06-03 behind A from a Monday start; no-start-date error; histogram series/dates/idempotent-regeneration; no-assignments error)
- `go build ./cmd/...` compiles; `npm run check` -> 0 errors; `make frontend-smoke` -> renders

## Follow-up - 2026-06-10 roadmap item 20 slice 1 (MSPDI import + round-trip export)

- **`export.FromMSPDI`** (mspdi_import.go): parses MSPDI XML into ImportedProject/ImportedTask. Conversions: PT<h>H<m>M<s>S durations -> working days at 8h/day; PredecessorLink Type 0=FF/1=FS(default)/2=SF/3=SS; LinkLag tenths-of-a-minute -> days (4800 = 1 day); Summary=1 and IsNull=1 rows skipped with dangling links to them dropped; assignments flattened to resource NAMES with Units passing through; StartDate reduced to YYYY-MM-DD. Errors on zero importable tasks and malformed XML.
- **`ToMSPDI` enriched for round-trip**: emits PredecessorLink (via new exported `kernel.EffectiveLinks` merge), Milestone (explicit flag OR zero duration), PercentComplete, and Resources/Assignments (stable name->UID table). Verified by TestMSPDIRoundTrip: export -> import preserves durations, SS+1 lag, FS from legacy Precedents, milestone flag, and 0.5-unit assignment.
- **`dag.FormatLinkLabel`**: ParseLinkLabel's inverse ("" for plain FS, "SS+1", "FF-1.5") used when materialising imported links as edge labels.
- **`App.ImportMSPDIChart`** (file dialog) + testable `importMSPDIFromBytes`: builds a CPM chart doc from the import, adopts the file's start date ONLY when the project has none, saves as kind=cpm. Dashboard "New chart" header gains an "Import schedule (MSPDI)" button that routes straight into the CPM editor (cancel is silent; errors show inline).
- cmd tests prove the full path end-to-end: imported chart -> loader -> scheduleProjectTasks gives Move ES=1 (SS+1) anchored at 2026-07-07 from the adopted Monday 2026-07-06 start; existing project start dates are preserved.
- README item 20 marked "interchange half done"; .mpp binary import documented as out of scope (MSPDI XML is the interchange format). Remaining: first-class Gantt chart kind.

Verification evidence:

- `go test -race ./cmd/... ./internal/export ./internal/kernel ./internal/charts/dag` -> ok (new: 3 export tests incl. round-trip, 2 cmd tests; one spurious FAIL was the embed-dist staging order in my own test command, not a code defect - re-ran with dist staged, green)
- `go build ./cmd/...` compiles; `npm run check` -> 0 errors; `make frontend-smoke` -> renders

## Follow-up - 2026-06-10 roadmap item 20 slice 2 (first-class Gantt chart kind) - ROADMAP FEATURES COMPLETE

- **`gantt` is the 21st chart kind**, sharing the layered/CPM data model so every scheduling feature (typed links + lag, constraints, progress, assignments, overallocation, baselines, levelling, histogram, MSPDI import) works on Gantt charts with zero extra plumbing.
- **dag/gantt.go**: GanttRow/GanttDep/GanttLayout; LayoutGantt (full CPM + DetectOverallocations, rows sorted (ES, ID), horizon = max EF) and LayoutGanttScheduled (+constraints armed, anchored dates, Anchored flag). engines.go: KindGantt arm + LayoutWithSchedule generalised to CPM|Gantt.
- **pdfrender/gantt.go**: bespoke renderer (label column, day grid via pickGridStep, critical-red bars, progress strip, milestone diamonds, anchored date captions, row cap to frame height) dispatched alongside fishbone's bespoke path; embeds in combined reports.
- **GanttEditor.svelte**: editable task grid (label/duration/%/milestone, delete), link list + add (from/to selects + FS/SS/FF/SF±lag label), zoomable (8-80 px/day) SVG canvas with day grid, dependency elbow paths, critical colouring, progress overlay, baseline ghost bars (via CompareScheduleBaseline variance back-computation), overallocation outlines, constraint "!" markers, anchored date captions, Set-baseline button, Ctrl+S.
- Wiring: session view union, App.svelte route, Dashboard card/starter/route. Registry count tests updated 20 -> 21 (All/ByEngine DAG 6 -> 7); README/AGENT.md "20 chart kinds" claims swept to 21 (historical lessons-learned entry left as history; TODO item 9 reworded to note the 21st landed via roadmap item 20).
- **Scheduling core roadmap items 14-20 are now all functionally complete.** Remaining polish (item 19): per-resource capacities (stakeholder record) + per-resource calendars; optional: EVM sections in Status Report / combined report renderers; V3 hardening per AGENT.md section 8.

Verification evidence:

- `go test -race ./internal/charts/... ./internal/kernel ./internal/export ./cmd/...` -> ok (new: 4 dag gantt tests incl. scheduled dates + cycle + overallocation rows; 3 pdfrender tests incl. empty-chart placeholder and grid-step picker; registry counts)
- `go build ./cmd/...` compiles; `npm run check` -> 0 errors; `npm run build` -> clean; `make frontend-smoke` -> renders

## Follow-up - 2026-06-10 roadmap item 19 polish (stakeholder availability as resource capacity)

- **db**: stakeholders gain `availability REAL NOT NULL DEFAULT 1` (units; 1 = full-time, 0.5 = half-time, 2 = two-person pool). Additive migration via the existing columnSet/ALTER TABLE probe pattern; SaveStakeholder defaults <= 0 to 1; all SELECT/INSERT/scan sites updated.
- **Threading**: `stakeholderCapacities(d, projectID)` in main.go (name -> availability; fail-soft nil). Consumed by: `App.LayoutChart` -> `charts.LayoutWithSchedule` -> `dag.LayoutCPMScheduled`/`LayoutGanttScheduled` (new `capacities` parameter; plain un-anchored Layout paths keep the 1.0 default since they lack project context) and `App.LevelChartResources` -> `kernel.LevelResources`. Non-stakeholder resource names keep the kernel's 1.0 default.
- **UI**: Stakeholder manager gains an Availability field with plain-language hint; d.ts Stakeholder interface updated.
- README item 19 marked complete (per-resource calendars explicitly deferred to V3 with the design reason: resource-specific non-working days interact with the anchoring layer); AGENT.md tracker matches. **All roadmap items 14-20 are now complete.**

Verification evidence:

- `go test -race ./cmd/... ./internal/db ./internal/charts/...` -> ok, plus -count=1 fresh runs of the new tests (capacity-2 stakeholder absorbs the contention fixture -> 0 pins; availability round-trip incl. default-to-1)
- `go build ./cmd/...` compiles; `npm run check` -> 0 errors; `make frontend-smoke` -> renders

## Follow-up - 2026-06-10 EVM sections in schedule-report exports + orphan cleanup

- **`ReportPayload.EVM *kernel.EVMetrics`** (engine.go) + shared **`evmSummaryLines`** helper (11 label:value lines; nil when metrics are nil OR BAC is 0, so cost-less schedules keep byte-identical reports). PDF (renderPDF), DOCX (renderDocumentDOCX), and ODT (renderODTReportBody) schedule reports append an "Earned Value (status date: today)" section from the same lines.
- **main.go exportScheduleReportAs**: computes EVM at today's working-day offset (kernel.DayOffset against project start + country calendar) only when the project is anchored. CSV/XLSX/MSPDI formats deliberately unchanged (row-schema stability; MSPDI has no EVM elements in our subset).
- **Orphan retired**: legacy `frontend/src/lib/components/GanttChart.svelte` (V1 read-only bar component, referenced nowhere since the first-class gantt kind shipped) deleted; svelte-check/build confirm nothing depended on it.
- README item 18 follow-up note updated: schedule-report EVM landed; Status Report document renderer / combined report EVM remains a possible later enhancement (needs chart_ref resolution design since document renderers see content JSON, not schedule payloads).

Verification evidence:

- `go test -count=1 -race ./internal/export` -> ok (new: evmSummaryLines values incl. SPI 0.75/CPI 0.60, suppression on nil + zero BAC, ODT body contains/omits the section, PDF+DOCX render smoke with EVM)
- `go test -count=1 ./cmd/...` -> ok; `go build ./cmd/...` compiles
- `npm run check` -> 0 errors; `npm run build` -> clean; `make frontend-smoke` -> renders (confirms orphan deletion safe)

## 2026-06-11 - Full release gate GREEN on macOS (capstone over the scheduling-core expansion)

James ran `make check-release` on the Mac mini against everything shipped 2026-06-10 (roadmap items 14-20 + EVM report sections + orphan cleanup). All gates passed: version match, REUSE licensing, frontend build budget, release-gate scope, frontend stability, runtime smoke, memory-safety scan, race detector, build, PDF/A-3 validation, PAdES local validation. "PMForge is ready for release."

Observation for the next release decision: the version string is still `1.1.0-V1-Expansion` (wails.json `productVersion` + internal/cli/parser.go `Version`). Given the scheduling-core expansion (typed dependencies, constraints, baselines, EVM, resource layer, MSPDI interchange, Gantt - 7 roadmap items), a bump (e.g. 1.2.0 with a V2-Scheduling tag, or whatever naming James prefers) would better describe the binary. Both sites must change together - check-release's version gate compares them. Left untouched: version bumps are a release decision, not a code fix.

## 2026-06-11 - ADR-001: database encryption at rest (V3 design)

- New `docs/design/ADR-001-database-encryption-at-rest.md` (GFDL header; first file in docs/design/). Proposed decision: SQLCipher page-level encryption for per-user .pmforge databases; system.db deliberately stays plaintext (only Argon2id hashes + wrapped keys; avoids the login bootstrapping problem).
- Key design point discovered while grounding the doc in code: `ResetWithRecoveryCode` changes the password WITHOUT the old one, so a password-derived key alone would orphan all data on recovery. Hierarchy: random 32-byte DEK -> wrapped by KEK(password) AND by a KEK per active recovery code (reusing the crypto package's Argon2id + AES-256-GCM patterns); password change = re-wrap, recovery reset = unwrap-via-code + re-wrap; all-codes-spent + forgotten password = unrecoverable by design (UI must say so).
- Options evaluated: A SQLCipher binding (chosen; binding/license/perf explicitly deferred to a Phase 0 spike - no dependency lands before evidence), B whole-file envelope (rejected: plaintext WAL window = the documented crash hazard), C field-level encryption (rejected: plaintext metadata + smeared complexity), D status quo OS FDE (remains the documented baseline + defence in depth).
- Migration mirrors the proven SwapInSnapshot atomic pattern via sqlcipher_export with integrity_check before and after, .bak retention, no downgrade path (matches V1->V2 stance).
- README TODO #8 and AGENT.md "Still deferred to V3" now point at the ADR. `release-gate-scope-check.sh` re-run -> still green (the README encryption guidance it guards is intact).

Verification evidence: scope gate green; doc-only change, no code touched.

## 2026-06-12 - ADR-001 Phase 0 spike executed (linux/arm64)

- Spiked `mutecomm/go-sqlcipher/v4 v4.4.2` in an ISOLATED module (sandbox scratch; repo go.mod/go.sum untouched - verified zero diff). Sources + reproduction README stored as `.go.txt` under `docs/design/spike-sqlcipher/` so they never enter the build or module graph; results recorded as Appendix A in ADR-001.
- **All functional checks PASS** against PMForge's usage profile: encrypted create via raw-keyspec DSN, WAL + foreign keys active, integrity_check ok + cipher_integrity_check 0 failures, wrong-key/keyless opens rejected, file header randomised (IsEncrypted true), plaintext->encrypted migration via ATTACH + sqlcipher_export with matching row counts. Clean 15 s build, no system deps (libtomcrypt bundled, no OpenSSL). Confirmed: the binding registers driver name "sqlite3" -> REPLACES mattn, cannot coexist in one binary (perf baseline needed a second module).
- **Performance**: insert-5000-rows tx ~6.0-6.1 ms plaintext vs ~15.6-22.6 ms encrypted (~2.6-3.7x write overhead); full scan ~330-343 us vs ~380-410 us (~15-20%); binaries comparable (6.85 vs 6.68 MB). Negligible in absolute terms for PMForge's single-user KB-scale workload.
- **Principal finding (against adopting v4.4.2 as-is): staleness.** MAINTENANCE pins mattn v1.14.5 / SQLCipher 4.4.2 / libtomcrypt 2020-08-29; bundled engine reports sqlite_version() 3.33.0 (2020) vs 3.45.1 in our current mattn v1.14.22. PMForge's SQL needs nothing newer than 3.33, but an encryption feature on a 2020-frozen crypto stack misses 5+ years of upstream fixes. ADR Appendix A orders next evaluation: (A1) maintained fork tracking current SQLCipher, (A2) mattn -tags libsqlite3 against vendored current SQLCipher, (A3) accept v4.4.2 with documented risk. Key hierarchy/migration design unaffected.
- Remaining Phase 0: James reproduces on macOS arm64 via docs/design/spike-sqlcipher/README.md (also Windows when CI exists).
- Sandbox lesson: background processes do not survive between tool calls here; the amalgamation compiles in ~15 s anyway, so foreground builds suffice.

Verification evidence: spike `SPIKE PASS` (3 runs) + baseline `BASELINE PASS` (3 runs); repo go.mod/go.sum diff = 0 lines; release-gate scope check green after docs additions.

## 2026-06-12 - Phase 0 macOS results + fork survey (Phase 0 build evidence COMPLETE for dev platforms)

- James reproduced the spike on the Mac mini (arm64): **SPIKE PASS x3 and BASELINE PASS x3**, functional results identical to linux. Build 9.5 s wall. Perf: insert5000 14.5-20.6 ms encrypted vs 7.7-13.8 ms plaintext (~1.5-1.9x writes); scans within noise (encrypted 349-673 us vs plaintext 439-792 us); binaries 6.70 vs 6.84 MB. Appendix A updated with the macOS table. Windows remains for when a Windows target exists.
- README paste bug fixed along the way: the spike README used a `<repo>` placeholder that zsh parsed as a redirect; replaced with a literal $REPO variable block (lesson: reproduction docs must be paste-safe, no angle-bracket placeholders).
- **A1 fork survey**: no fork demonstrably tracks current SQLCipher; grassto/go-sqlcipher only renames the driver to avoid the mattn conflict. Realistic decision is now A2 (mattn -tags libsqlite3 + vendored current SQLCipher; fresh engine, owns packaging) vs A3 (adopt v4.4.2 as-is; proven on both dev platforms, 2020-frozen engine documented as risk, sqlcipher_export as future escape hatch). Recorded in Appendix A; decision is James's.

## 2026-06-12 - ADR-001 ACCEPTED (A3) + bbolt assessment + key hierarchy implemented (step 3)

- **A3 accepted by James**: adopt mutecomm/go-sqlcipher v4.4.2 as-is; 2020-frozen engine documented as known risk, sqlcipher_export as escape hatch. ADR Status -> Accepted with acceptance note. NOTE: the dependency itself still does NOT land until step 5 (db.InitDB keying); go.mod remains untouched.
- **bbolt question answered (ADR Appendix C): not a value add, rejected.** As main store it loses SQL/FK/indexes/migrations AND has no encryption (recreates the exact problem SQLCipher solves); as a side store for wrapped DEKs it duplicates system.db (which must pre-exist login anyway) and adds a second file format to backups/repair/REUSE for zero capability. Future no-CGO scenario points to a CGO-free SQLite port, not a KV store.
- **Step 3 implemented (key hierarchy, binding-independent pure Go):**
  - internal/crypto/keywrap.go: GenerateDEK (32 random bytes), WrapKey/UnwrapKey (base64 over existing Argon2id+AES-256-GCM EncryptBuffer; fresh salt+nonce per wrap), KeyspecHex (64-char uppercase hex for the future PRAGMA raw keyspec). 5 tests.
  - internal/users/dek.go: probe-guarded ALTERs add users.wrapped_dek_pw + recovery_codes.wrapped_dek; UnlockDEK (login-time unwrap; LAZY DEK creation for pre-ADR accounts at the only moment the verified password is in hand).
  - recovery.go: IssueRecoveryCodes(username, dek) wraps the session DEK into each code (nil dek = legacy plain codes); ResetWithRecoveryCode unwraps the matched code's wrap and re-wraps the SAME DEK under the new password atomically with the hash rotation; legacy unwrapped codes generate a fresh DEK (safe only pre-encryption; encryption-enable flow must force re-issue).
  - main.go: App.dek session field (set in Login/CreateAccount via UnlockDEK, zeroed+cleared in Logout); IssueRecoveryCodes passes it. No frontend API changes.
- Tests: dek_test.go covers lazy generation + stability, wrong-password/unknown-user failure, **the data-survival invariant (reset via code -> identical DEK under new password; old password dead)**, legacy fresh-DEK path, migration idempotence. Existing recovery tests updated for the new signature (nil dek).

Verification evidence:

- `go test -race ./cmd/... ./internal/users ./internal/crypto` -> ok (users 17.8 s - Argon2id cost x many wraps, expected)
- `go build ./cmd/...` compiles; release-gate scope check green; go.mod/go.sum untouched

## 2026-06-13 - ADR-001 encryption-at-rest implementation docs updated

- Final docs now match the implemented SQLCipher path: new per-user `.pmforge` project databases are SQLCipher-encrypted with the user's DEK; existing plaintext projects can migrate from Project Settings after recovery-code reissue; `system.db` remains plaintext by design and stores password hashes plus wrapped DEKs, not project records.
- Recovery semantics are explicit in README/ADR/AGENT: active recovery codes wrap the DEK, enabling password reset without orphaning encrypted projects; losing the password and all valid wrapped recovery codes makes encrypted project databases unrecoverable by design.
- Backup semantics are explicit: `.pmba` archives preserve encrypted `project.pmforge` bytes, so backup files inherit project database encryption.
- `docs/design/ADR-001-database-encryption-at-rest.md` moved from accepted design/pending steps to implemented status, with steps 4-7 marked complete and Appendix B summarizing SQLCipher open/migration, Settings opt-in, repair/backup/headless handling, and release gates.
- `AGENT.md` no longer lists per-user database encryption at rest as "implementation not started"; the 2026-06-06 and 2026-06-09 historical notes now say they were superseded by the 2026-06-13 SQLCipher implementation.
- Remaining encryption-at-rest work from the written plan: Task 7 step 2 full verification.

## 2026-06-13 - ADR-001 encryption-at-rest full verification passed

- The encryption-at-rest implementation and verification tasks are complete through Task 7 step 2. Task 0 step 3 remains intentionally unchecked because it is a staging-only instruction and no staging/commit was requested.
- Full verification passed:
  - `npm --prefix frontend run build`
  - `mkdir -p cmd/pmforge/frontend`
  - `rm -rf cmd/pmforge/frontend/dist`
  - `cp -R frontend/dist cmd/pmforge/frontend/dist`
  - `go test -count=1 ./cmd/... ./internal/...`
  - `go test -count=1 -race ./internal/crypto ./internal/users ./internal/db`
  - `make check-encrypted-db`
  - `make license-check`
  - `make release-scope`
  - `make check-release`
- `make check-release` completed all release gates and ended with `PMForge is ready for release.`
- Sequencing note: `make license-check` removes `cmd/pmforge/frontend/dist`. Do not run it in parallel with Go compile gates that import `cmd/pmforge`; recreate the embed dist before standalone Go compile gates, or use `make check-release`, which rebuilds/copies the frontend internally.

## 2026-06-14 - Partial key-hierarchy staging boundary

- Removed a stale empty `.git/index.lock` after confirming no active Git index-writing process was running; the only Git process was the fsmonitor daemon and the lock timestamp was June 10.
- Staged the safe whole-file key-hierarchy slice:
  - `docs/design/ADR-001-database-encryption-at-rest.md`
  - `internal/crypto/keywrap.go`
  - `internal/crypto/keywrap_test.go`
  - `internal/users/dek.go`
  - `internal/users/dek_test.go`
  - `internal/users/recovery.go`
  - `internal/users/recovery_test.go`
  - `internal/users/store.go`
- Left `cmd/pmforge/main.go` and `session-notes.md` unstaged because they contain broad unrelated dirty changes and need deliberate hunk-level staging for only the encryption/session handoff hunks.
- Staged diff hygiene passed with `git diff --cached --check`.

## 2026-06-14 - Dirty hunk classification completed

- Reclassified the broad dirty tree as coherent verified work rather than unrelated noise:
  - scheduling/Gantt/resource/MSPDI roadmap completion,
  - SQLCipher encryption-at-rest completion,
  - release-gate hardening,
  - root project documentation and handoff notes.
- Staged the complete product/docs/handoff set so the index now represents the verified work. The only intentionally unstaged file is `.claude/settings.local.json`, which is local tool configuration.
- Staged index hygiene passed with `git diff --cached --check`; full worktree diff hygiene passed with `git diff --check`.

## 2026-06-14 - Post-commit remaining-work audit

- Committed the broad verified work as `b291b5c Complete scheduling and encryption release work`.
- Remaining current work identified:
  - External PAdES/Acrobat validation with a trusted signing source remains a real release-hardening item.
  - Version string `1.1.0-V1-Expansion` is a release decision; if changed, update `wails.json` and `internal/cli/parser.go` together because `check-release` compares them.
  - ADR-001 Windows packaging validation remains deferred until a Windows target exists.
  - `.claude/settings.local.json` remains local-only and unstaged.
- Completed the safest next item: corrected README's PDF/A-3 TODO text so it no longer says PDF/A still needs release-builder soak before becoming a hard release claim. `make check-pdfa` is already a hard gate in `make check-release`.

## 2026-06-14 - Local Apple Silicon installer package

- Added `make package-macos-installer`, backed by `scripts/package-macos-installer.sh`, to build a local Apple Silicon `.pkg` installer for PMForge.
- The script requires a Darwin/arm64 host, runs the existing production build, assembles a standard `PMForge.app` bundle in `/tmp`, ad-hoc signs it, strips extended attributes/resource-fork sidecars before packaging, and writes `build/packages/PMForge-1.1.0-V1-Expansion-darwin-arm64.pkg`.
- Verification passed:
  - `bash -n scripts/package-macos-installer.sh`
  - `make package-macos-installer`
  - package expansion in `/tmp`, `codesign --verify --deep --strict --verbose=2` on the payload `PMForge.app`
  - `plutil -p` on payload `Info.plist`
  - `file` + `lipo -archs` confirmed the payload executable is `arm64`
  - payload `pmforge --version` reported `PMForge 1.1.0-V1-Expansion`
  - `make release-scope`
  - `go test ./cmd/... ./internal/...`
- Distribution limitation: the package is intentionally local-test only. `spctl -a -vv -t install build/packages/PMForge-1.1.0-V1-Expansion-darwin-arm64.pkg` reports `rejected` with `source=no usable signature` because there is no Developer ID Installer certificate/notarization in this repo.

## 2026-06-15 - Root-caused installed app "never starts / no GUI" + startup hardening

- Reproduced on James's M4 Mac: the installed `/Applications/PMForge.app` launches then dies in <1s with NO window and NO crash report; Console showed only `appDeath for dev.pmforge.PMForge` and the pmforge process emitted zero os_log lines. `~/Documents/PMForge/system.db` existed and its WAL was touched on the failing launch, proving `NewApp()` succeeded and the failure was after it (in `wails.Run`).
- **Root cause (confirmed by running the binary directly):** `build/bin/pmforge --` printed `wails: Wails applications will not build without the correct build tags` (exit 1). The `Makefile` `build` target compiled with a plain `go build` and NO Wails build tags, so the binary linked Wails' stub `Run()` that aborts at launch. Both packaging scripts call `make build`, so the `.pkg` inherited the broken binary. Go's `log.Fatalf` writes to stderr, which a Finder/LaunchServices GUI launch discards -> the failure was completely invisible.
- **Fix 1 (the defect):** added `-tags "desktop,production"` to the `Makefile` `build` target's `go build` line (verified against Wails docs: `wails build` uses exactly these tags). `.gitlab-ci.yml` has the same tag-less `go build` smoke line but is followed by a real `wails build`; left unchanged to avoid guessing the Linux `webkit2_4x` tag from a non-Linux host - flagged as a separate follow-up.
- **Fix 2 (hardening, requested):** new stdlib-only `internal/applog` package. `Init(preferredDir)` tees the standard logger to stderr AND a dated file `~/Documents/PMForge/logs/pmforge-<date>.log` (XDG-aware; home/temp fallback; never fatal). `Fatal(title,msg,logPath,err)` logs the error with a stack trace, shows a native OS error dialog (osascript / PowerShell MessageBox / zenity-kdialog-notify-send via build-tagged `dialog_*.go`), then exits 1. `cmd/pmforge/main.go` initialises it at the top of the GUI path and routes both fatal branches (NewApp + wails.Run) through `applog.Fatal`; CLI maintenance paths unchanged (stderr stays visible in-terminal).
- Tests: `internal/applog/applog_test.go` covers dated-file creation, append-across-calls, `resolveLogDir` (preferred + whitespace/empty fallback), and the `formatFatal`/`dialogMessage` content. Native dialogs are best-effort and not unit-tested (side-effecting OS calls).
- Docs: README gains a "Logs and startup diagnostics" section, a Toolchain build-tags warning, and a first-run-layout mention of `logs/`.
- **Verification still owed (no Go toolchain in this Cowork sandbox):** James must run on the Mac: `make build && ./build/bin/pmforge` (GUI should open), then `go test ./internal/applog/...`, `go test ./cmd/... ./internal/...`, `make check-release`, and `make package-macos-installer` to reship the fixed `.pkg`.

## 2026-06-15 - Second build break: macOS UTType link failure (fixed)

- After the build-tags fix, `make build` got PAST the stub-Run abort and actually compiled the Wails darwin code (good - proves Fix 1 works). It then failed at LINK on James's Mac (Xcode macOS 15 SDK, Go 1.26.4): `Undefined symbols for architecture arm64: "_OBJC_CLASS_$_UTType" ... ld: symbol(s) not found`. The `setShowsBaselineSeparator deprecated in macOS 15` warning confirms the macOS 15 SDK.
- Cause: Wails v2.9.2 file-dialog code references `UTType` (UniformTypeIdentifiers framework, macOS 11+). The `wails` CLI links that framework automatically; a raw `go build` does not, so the framework is missing from the external link line (CoreFoundation/Security/Foundation/Cocoa/WebKit/AppKit were present, UniformTypeIdentifiers was not). Documented in the Wails troubleshooting guide and issues #1140/#3003.
- Fix: `Makefile` now appends `-framework UniformTypeIdentifiers` to `CGO_LDFLAGS`, guarded by `ifeq ($(shell $(GO) env GOOS),darwin)` so only Darwin hosts are affected (Linux/Windows packaging unchanged). README Toolchain note documents the requirement.
- Re-verify on the Mac: `make build && ./build/bin/pmforge` should now link AND open the GUI. Then `make package-macos-installer` for the `.pkg`.

## 2026-06-15 - Switched the build to the Wails CLI (main package moved to repo root)

- Decision (James): stop hand-rolling `go build` and use `wails build` so the CLI owns build tags + framework linking + frontend embed. Confirmed root-cause constraint via Wails docs/issues #2568: **wails build requires the main package at the project root**; PMForge's was in `cmd/pmforge`, which is exactly why it hand-rolled the build.
- **Relocation:** `git mv cmd/pmforge/main.go ./main.go` plus the six `*_test.go` files; removed `cmd/`. The `//go:embed all:frontend/dist` directive is unchanged but now resolves to the real repo-root `frontend/dist` (the Vite output), eliminating the old `cp frontend/dist cmd/pmforge/frontend/dist` embed-copy hack. (Sandbox couldn't delete the generated `cmd/pmforge/frontend/dist` tree - it is gitignored/untracked; James should `rm -rf cmd` locally.)
- **Build pipeline:** `Makefile` `build` target is now `$(WAILS) build $(WAILS_BUILD_FLAGS)` (overridable flags var). Removed the darwin `CGO_LDFLAGS` UniformTypeIdentifiers ifeq and the `-tags`/`-ldflags` raw-build line - wails build handles both. `GO_PACKAGES` -> `. ./internal/...` across Makefile, check-release.sh, memory-safety-scan.sh (also `PMF_DIRS` -> `./main.go ./internal ./scripts`). `release-gate-scope-check.sh` `go list` -> `. ./internal/...`. `validate-encrypted-db.sh` test path `./cmd/pmforge` -> `.`. `clean` target no longer wipes tracked `build/darwin`; `lint-go`/`license-check` paths fixed. `.gitlab-ci.yml` build/release jobs drop the tag-less `go build ./cmd/pmforge` and use the `build/bin/` output.
- **macOS bundle scaffold:** added tracked `build/darwin/Info.plist` + `Info.dev.plist` (CFBundleIdentifier `dev.pmforge.PMForge`, productivity category, NSAllowsLocalNetworking for dev). `.gitignore` rewritten to ignore all of `build/` EXCEPT those two plists; REUSE.toml annotates them. wails build will generate any missing default assets (e.g. appicon) on first run - drop a 1024px `build/appicon.png` for branding.
- **Installer:** `package-macos-installer.sh` rewritten to `make build` then pkgbuild the wails-produced `build/bin/*.app` (ad-hoc signed, xattrs stripped); no longer hand-assembles a bundle. Version read from wails.json. The package-linux/windows/darwin tarball targets and the deterministic-packaging scope gate are untouched (scope gate only forbids `wails build` inside package-{linux,windows,darwin} targets, which still call package.sh).
- **Docs:** README (Toolchain build note, directory tree, applog path, test cmd), ARCHITECTURE, TESTING, STYLE, AGENTS.md, AGENT.md (tree, build-process lesson, gitignore/embed lessons, gate commands, App-method table), and code comments in `internal/templates/jdm.go` + `internal/export/pdf.go` updated to the root layout. Dated historical lessons in AGENT.md (524/743/767) and `.agent_memory/`+`docs/superpowers/` left as-is (records of past state).
- **NOT verified here (no Go/wails toolchain in the Cowork sandbox).** James must run on the Mac:
  - `go install github.com/wailsapp/wails/v2/cmd/wails@latest` (if not present)
  - `rm -rf cmd` (remove the leftover generated tree)
  - `make build` -> expect `build/bin/PMForge.app`; open it (or `open build/bin/*.app`) -> GUI should appear
  - `go test . ./internal/...` and `make check-release`
  - `make package-macos-installer` -> `.pkg`, then install
  - Most-likely-to-need-a-tweak areas after a real wails run: the exact `.app` name under build/bin (installer globs `*.app`), and whether wails wants a `build/appicon.png` (add one if it complains).

## 2026-06-15 - wails build self-sign failure on macOS 15 fixed with -skipbindings

- `wails build` on James's Mac compiled, embedded, and packaged successfully, then FAILED at its own "Self-signing application" step: `codesign failed ... build/bin/pmforge.app/Contents/MacOS/pmforge: resource fork, Finder information, or similar detritus not allowed`.
- Smoking gun: the build output contained `main.go:2962: PMForge exited cleanly` (our applog line) BEFORE packaging - i.e. Wails EXECUTED the freshly linked binary mid-build during binding generation. On macOS 15 (Sequoia) executing a binary stamps it with the `com.apple.provenance` xattr, and Wails' subsequent ad-hoc `codesign` rejects any such "detritus". Same root cause as flutter/flutter#181103 and Apple QA1940.
- Fix: default `WAILS_BUILD_FLAGS ?= -skipbindings` in the Makefile so Wails does not build+run the app for binding generation (no execution -> no provenance -> self-sign succeeds). Safe for PMForge: the frontend uses `window.go.main.App` via the hand-written `frontend/src/wails-window.d.ts` and never imports the generated `frontend/wailsjs`. Also added `rm -rf build/bin` at the top of `package-macos-installer.sh` so a stale, previously-executed binary from an earlier run can't reintroduce the xattr.
- Confirmation the app itself is healthy: the `PMForge exited cleanly` applog line proves the relocated root `main.go` + `wails build` binary launches and runs the full GUI startup path without the old silent-exit failure.
- Re-verify on the Mac: `make build` (now `wails build -skipbindings`) should complete through self-sign; `open build/bin/*.app` shows the GUI; `make package-macos-installer` produces the `.pkg`.

## 2026-06-15 - codesign detritus persisted after -skipbindings: wrap the build to strip+sign

- `-skipbindings` correctly stopped Wails from running the app during build (the `PMForge exited cleanly` line disappeared), but `wails build`'s self-sign STILL failed with `resource fork, Finder information, or similar detritus not allowed`. So the offending extended attributes are NOT execution-provenance - they are `com.apple.FinderInfo` / iCloud file-provider metadata the build artifacts pick up because the repo lives under iCloud-synced `~/Documents/GitLab/...`. Wails signs in-process, so it cannot strip them first.
- Fix: new `scripts/wails-build.sh` wraps `wails build`. It cleans `build/bin`, runs `wails build "$@"` (whose internal self-sign may fail - tolerated), then on darwin locates the produced `.app`, runs `xattr -cr` to strip the detritus, and ad-hoc signs with `codesign --force --deep --sign -` (+ verify). Apple QA1940's documented sequence. On non-darwin it just surfaces Wails' exit status. `make build` now calls this wrapper; `-skipbindings` is retained as the default flag (still correct for PMForge and avoids one provenance source).
- `make build` therefore succeeds even though Wails' own self-sign step prints an error, because the wrapper re-signs after stripping. The installer's later copy+strip+sign remains as belt-and-suspenders.
- If `xattr -cr` + sign still fails (iCloud re-adding attributes mid-sign), the durable cure is to build from a non-synced path (e.g. clone/build under `~/Developer`); the wrapper prints that hint. Pending James's re-run to confirm.

## 2026-06-15 - QA review of test screenshots + P0 fixes implemented

- Reviewed the 11 screenshots in *PM Forge Test 15-June-2026*. Full write-up in `docs/QA-review-2026-06-15.md`. Almost all reported errors reduce to two root causes; both P0 fixes are now implemented.
- **Fix A - empty dates can't unmarshal into time.Time (the pervasive one).** `App.SaveChart`/`SaveStakeholder`/`UpdateProjectMeta`/`SaveDocument` take structs whose `CreatedAt`/`UpdatedAt` were `time.Time`; the frontend sends `created_at:""`/`updated_at:""` for new/edited records (e.g. `Dashboard.svelte` newChart), so Wails rejected the call with `parsing time "" as "2006-01-02T15:04:05Z07:00"` before the Go body ran. This caused the stakeholder-save, project-settings-save (and therefore theme-change), and new-chart failures.
  - Changed `CreatedAt`/`UpdatedAt` from `time.Time` to `string` on `db.Chart`, `db.Document`, `db.Stakeholder`, `db.Project`; the scanners now assign the raw RFC3339Nano strings (the writers already used a server-side `now` string, and the TS `wails-window.d.ts` already typed these as `string`). Fixed the one consumer, the CPM-chart recency sort in `main.go` (`UpdatedAt.After` -> string `>`; RFC3339 sorts lexicographically). `BackupManifest.CreatedAt` is a different (non-bound) struct and stays `time.Time`.
- **Fix B - multi-return methods resolved to null in the bridge.** `CreateProjectFromLaunchpad` (3 values) and `App.EnsureDefaultBoard` (2 values) were destructured as arrays in the frontend; the awaited result came back null, so destructuring threw `null is not an object (evaluating 'window.go.main.App.<m>(...)')` even though the Go call ran (hence projects were still created).
  - Both now return a single struct: `LaunchpadResult{project,seeds,path}` and `BoardWithColumns{board,columns}`. Updated callers `ProjectLaunchpad.svelte`, `KanbanBoard.svelte`, `Backlog.svelte` (read `res.project`/`res.path`, `res.board`/`res.columns`), the `wails-window.d.ts` return types, and the Go test `encryption_project_test.go`.
  - Removed the `-skipbindings` default from the Makefile so Wails generates bindings normally (needed for correct multi-value marshaling). The codesign-detritus issue that `-skipbindings` had masked is handled independently by `scripts/wails-build.sh` (xattr strip + ad-hoc re-sign), so bindings-on is safe.
- **Not verified here (no Go/Node toolchain in the Cowork sandbox).** James to run: `make build` (now `wails build` + wrapper, bindings on), then `go test . ./internal/...`, `npm --prefix frontend run check`, and re-test in the GUI: create project (no error, opens), save stakeholder, save project settings, change theme, create a chart, open Kanban/Backlog.
- Remaining from the review (P1/P2): certificate chooser in the signing modal, native menu bar, project delete/clone, global app settings, clickable chart-template panel.

## 2026-06-15 - P1 work: project Delete/Clone + certificate chooser

- **Project Delete + Clone** (Your Projects screen).
  - Backend: `App.DeleteProject(path)` and `App.CloneProject(path) (ProjectFile, error)` in `main.go`, gated by a new `projectPathFor` helper that rejects any path not directly inside `<user.DataDir>/projects` and not ending in `.pmforge` (no traversal / arbitrary-file deletion). Delete removes the `.pmforge` plus `-wal`/`-shm` sidecars and closes the project first if it is the open one. Clone copies bytes verbatim to `<name>-copy.pmforge` (non-conflicting), so an encrypted project's clone stays encrypted under the same DEK; sidecars copied best-effort. Added a small `copyFile` helper (added `io` import). The picker derives the card name from the filename, so the clone shows as `<name>-copy` with no DB rename needed.
  - Frontend: `ProjectPicker.svelte` rows now have Clone + Delete actions beside the open area (restructured so no nested buttons). Delete uses a two-step confirm (Delete -> Confirm/Cancel) matching the Dashboard pattern; a row is disabled while its op is in flight; deleting the open project clears the session. `wails-window.d.ts` declares `DeleteProject`/`CloneProject`.
- **Certificate chooser in the signing modal.**
  - `SignCertificateModal.svelte` gains a "Choose certificate…" button calling the existing `App.ChooseCertFile()`. A picked path overrides the passed-in `certPath` (`effectivePath = chosenPath || certPath`); Sign & Export is disabled until a certificate is present. The modal now reports the effective path back via `onConfirm(password, certPath)`.
  - Updated the three callers' handlers to the 2-arg form and to sign with the passed path: `CharterEditor.svelte`, `ReportComposer.svelte`, `Dashboard.svelte`. All three already open the modal even when no cert is pre-configured, so the chooser is reachable everywhere.
- P2 (still open): global app-settings screen, clickable chart-template panel.
- **Verification owed (no Go/Node toolchain here):** `make build`, `go test . ./internal/...`, `npm --prefix frontend run check`; then GUI spot-checks: clone a project, delete a project (two-step), and run a signed export choosing a cert from the modal.

## 2026-06-15 - P1: native application menu bar

- Added `options.App.Menu: buildAppMenu(app)` in `main.go` (verified against the official Wails v2 menu reference - `menu.NewMenu()`, `AddSubmenu`, `AddText` + `keys.CmdOrCtrl`, `menu.AppMenu()`/`menu.EditMenu()` on darwin only). Imports: `pkg/menu` and `pkg/menu/keys`.
- Structure: **File** (New Project Cmd+N, Open Project Cmd+O, Project Settings Cmd+comma, Close Project Cmd+W; plus a Quit item on non-macOS) and **Help** (About PMForge -> `wailsruntime.MessageDialog` showing `cli.Version`). On macOS the standard **App** menu (About/Hide/Quit) and **Edit** menu (copy/paste/select-all) are appended so those keep working under a custom menu. Order: App, File, Edit, Help.
- Menu items don't navigate directly (navigation is frontend state): each File item calls `wailsruntime.EventsEmit(app.ctx, "menu:<action>")`. `App.svelte` `onMount` registers `window.runtime.EventsOn` listeners that map `menu:new-project` -> launchpad, `menu:open-project` -> project_picker, `menu:settings` -> project_settings (guarded on an open project), `menu:close-project` -> `CloseProject()` + clear session + project_picker. All navigation guards on `session.user`.
- This addresses the report's "Need Open File / Help / About … with settings in drop down." A global application-settings *screen* (distinct from per-project settings) remains a P2 item; the menu's "Project Settings" routes to the existing per-project settings view.
- **Verification owed (no Go/Node toolchain here):** `make build`, `go test . ./internal/...`, `npm --prefix frontend run check`; then in the GUI check the menu bar shows File/Help (and on macOS the PMForge + Edit menus), Cmd+N opens the launchpad, Cmd+W closes the project, and Help -> About shows the version.

## 2026-06-15 - P2: chart-templates panel is now clickable

- The Dashboard "Available chart templates" panel (grouped by engine) rendered each kind as static `<li>` text, so it read as broken (review item). Each entry is now a button calling the existing `newChart(c.kind, c.name)` - which creates a chart of that kind and routes to its editor. The backend `charts.Definition.Kind` strings (from `ListChartKinds`) exactly match the `chartStarters`/`chartRoutes` keys, so all 21 kinds work with no mapping. Added a "Click a template to create that chart." hint and hover affordance.
- Hardened `newChart` with try/catch + `showToast` error feedback (benefits the existing New-Chart grid too, which previously called it bare).
- Frontend-only change (`Dashboard.svelte`). Remaining P2: a global application-settings screen distinct from per-project settings.

## 2026-06-15 - P2: global Application Settings + Portfolio dashboard + shared toolbar

- **Global Application Settings** (per-user, app-level, distinct from per-project settings).
  - Backend (`main.go`): `AppSettings{DefaultFont, DefaultTheme}` persisted as JSON at `<user.DataDir>/app-settings.json` (independent of any project DB). `App.GetAppInfo()` returns the settings + read-only info (version, data location, username) + the font catalog (`a.ListFonts()`); `App.SaveAppSettings(s)` writes the file. `applyGlobalDefaults(d)` seeds a newly created project's per-project settings (font/theme) from these app defaults - best-effort, called in both `CreateProject` and `CreateProjectFromLaunchpad`. So the app-level defaults are real, not inert.
  - Frontend: `AppSettings.svelte` (edit default font from the catalog + export theme; shows version / signed-in user / data location).
- **Portfolio dashboard** (post-login landing).
  - Backend: `App.ProjectsOverview() []ProjectSummary` opens each project in the user's folder with the session DEK (read-only, leaves the active project untouched), reading status/phase/start/end + chart and document counts; unreadable projects are still listed (`readable=false`).
  - Frontend: `Portfolio.svelte` shows each project as a card with a status badge, phase/dates, and counts; in-progress (planning/active/on_hold/unknown) sorted first, then by last-modified; click opens the project -> per-project dashboard. Shows "N active · M total".
- **Shared toolbar** `AppHeader.svelte` (Dashboard / Projects / App Settings + signed-in-as + Sign out) used by Portfolio, ProjectPicker, and AppSettings. The old bespoke ProjectPicker header (and its duplicate logout) was removed in favour of it.
- **Routing/landing**: added `portfolio` and `app_settings` to the session view union and `App.svelte` route loaders. Post-login now lands on `portfolio` (Login, CreateAccount, and the resume-session path); per-project Close and the menu's Close Project also return to `portfolio`.
- **Native menu**: File menu gains **Dashboard** (Cmd+D -> `menu:dashboard`) and **Application Settings…** (Cmd+, -> `menu:app-settings`); Project Settings lost its accelerator (Cmd+, is now app-level prefs, the macOS convention). `App.svelte` listens for the two new events.
- d.ts: declared `ProjectsOverview`/`GetAppInfo`/`SaveAppSettings` + `ProjectSummary`/`AppInfo`/`AppSettings` interfaces.
- **Verification owed (no Go/Node toolchain here):** `make build`, `go test . ./internal/...`, `npm --prefix frontend run check`; then GUI: log in -> lands on Portfolio with project statuses; toolbar switches Dashboard/Projects/App Settings; App Settings saves a default font/theme and a newly created project inherits them; Cmd+D / Cmd+, work.
- All QA-review items (P0+P1+P2) are now addressed.

## 2026-06-15 - Multi-user isolation investigation + dialog scoping + 4 fonts (batch 1 of a larger request)

- **User-isolation investigation (reported: "adding another user, saw other user's projects in-app").** Audited the session/enumeration paths: `Login` and `CreateAccount` both switch `a.user`+`a.dek` under lock; `Logout` clears them; `requireUser`/`CurrentUser` return `a.user`; `ListProjects`/`ProjectsOverview` read `a.requireUser().DataDir/projects` (per-username `<root>/<username>/projects`); cross-user `OpenProject` is rejected by the wrong DEK. **The current source is correctly per-user isolated** - no in-app enumeration leak is present. Most likely the observation was on an earlier build, or files were seen via a native file dialog / Finder on disk (all users live under one OS home; contents are per-user-DEK encrypted, but the *files* are visible to the same OS user).
  - Added a regression test `user_isolation_test.go::TestProjectsAreIsolatedPerUser` (alice creates a project; bob, a new user, must see 0 via both `ListProjects` and `ProjectsOverview`). Locks the invariant so any future regression fails CI.
  - Defense-in-depth: the three native file pickers (`ImportMSPDIChart`, `ChooseCertFile`, `ImportFont`) now set `DefaultDirectory: a.userDir()` so they open in the signed-in user's own folder. NOTE: this only sets the dialog's initial directory (the user can still navigate up) - it is a nudge, not a hard boundary; real protection remains per-user encryption.
  - **Still owed (if the in-app leak is reproducible):** James to send exact repro steps; I couldn't reproduce it in the current code.
- **Fonts:** added Roboto (Apache-2.0, sans), Arimo (Apache-2.0, Arial-metric sans), Cousine (Apache-2.0, mono), Ledger (OFL-1.1, serif) to `internal/fonts/catalog.go`, with best-effort download URLs in `scripts/fetch-fonts.sh` (NOTEs: google/fonts paths drift; build tolerates missing files), REUSE.toml annotations, and a README catalog row each. They appear in the font picker once fetched.
- **Sequenced (not yet done - large, each its own verifiable pass):**
  1. Per-project unique-ID subfolders (`<projects>/<YYYYMMDD-HHMMSS>-<name>/project.pmforge`) with backward-compat for existing flat `.pmforge` files - touches list/create/open/delete/clone/overview/backup/headless + migration.
  2. Relicense entire repo to **Apache-2.0** + attribution to **"James L. Burns and The PMForge Contributors"** - repo-wide SPDX/header change, preserving third-party licenses (OFL/Bitstream-Vera/CC0-for-ICC); update LICENSES/, REUSE.toml, README; must pass `reuse lint`.
  3. Full **Light/Dark** application theme - the app is currently all-dark hardcoded Tailwind; needs a theme mechanism + re-theming every component.
- **Verification owed (no Go/Node toolchain here):** `go test . ./internal/...` (incl. the new isolation test), `make build`, `npm --prefix frontend run check`.

## 2026-06-15 - Per-project unique-ID subfolders (with legacy compatibility)

- Decision: keep GPLv3 for now (Apache-2.0 relicense task dropped; the attribution-text change to "James L. Burns and The PMForge Contributors" remains a small queued item).
- New on-disk layout: each project lives in its own time-stamped subfolder `…/projects/<YYYYMMDD-HHMMSS>-<safe-name>/project.pmforge` (unique ID = date+time+name, per the request). Implemented in `main.go`:
  - `newProjectPath(dir, safe)` creates the unique subfolder (collision-deduped) and returns the inner `project.pmforge` path. Used by `CreateProject` and `CreateProjectFromLaunchpad`.
  - `enumerateProjects(projectsDir)` lists BOTH the new subfolder layout AND legacy flat `<name>.pmforge` files, so pre-existing projects keep working (no destructive migration). `ListProjects` and `ProjectsOverview` now use it; `projectDisplayName` strips the timestamp prefix for the picker, and `ProjectsOverview` prefers the real DB project name.
  - `projectPathFor` now accepts a file directly in `projects/` (legacy) OR one level deep in `projects/<id>/` (new), still rejecting anything outside the user's projects tree.
  - `DeleteProject` removes the whole project subfolder for the new layout (`os.RemoveAll(parent)`, proven to be an immediate child of the user's projects dir) and falls back to file+sidecar removal for legacy flat files. `CloneProject` creates a fresh unique subfolder named "<source> copy".
  - `inferHeadlessRootDir` steps up out of the per-project subfolder so `--check`/`--repair` on `…/projects/<id>/project.pmforge` still resolves the root.
- Tests: `project_storage_test.go` covers `projectDisplayName`, `enumerateProjects` across both layouts (+ ignoring non-project subfolders), and the full create→clone→delete lifecycle (unique timestamped folder, clone in a new folder, delete removes the folder). The earlier isolation test still applies.
- Frontend needs no change (it's path-based); the picker now shows de-prefixed names and the portfolio shows real DB names.
- **Verification owed (no Go toolchain here):** `go test . ./internal/...` and `make build`.
- Remaining from this batch: Light/Dark full theme; small attribution-text change.

## 2026-06-15 - Light/Dark theme + repo-wide attribution (GPLv3 retained)

- **Attribution:** decided to KEEP GPLv3 (no Apache relicense). Ran a safe repo-wide sed `2026 The PMForge Contributors` -> `2026 James L. Burns and The PMForge Contributors` across 335 files (SPDX-FileCopyrightText + Copyright lines). The full-string match avoided double-prefixing already-correct files and never touched third-party copyrights (Red Hat/Bitstream/Google/JetBrains/Ledger/ICC). Verified: no `James L. Burns and James L. Burns`, zero remaining bare strings.
- **Full Light/Dark theme** via CSS variables + Tailwind, with NO per-component class rewrites:
  - `tailwind.config.js` remaps the `slate` scale and `cyan` accent to `rgb(var(--x) / <alpha-value>)` (channel triplets, so opacity modifiers like `bg-slate-800/40` still work). Tailwind `extend` deep-merges, so untouched shades keep defaults.
  - `src/app.css` defines the variables: `:root` = dark (normal Tailwind values, so dark mode is visually unchanged); `:root[data-theme="light"]` inverts the slate scale (950 = lightest page bg, 50 = darkest text, 500 = mid) and darkens the light cyan text shades for contrast; `color-scheme`/`accent-color` flip too.
  - Emphasis `text-white` (61 occurrences, those NOT on a colored button) converted to `text-slate-50` so they flip; the 58 button labels on colored backgrounds keep `text-white`.
  - Setting + plumbing: `AppSettings.AppTheme` (light|dark, "" => dark) persisted in app-settings.json; `AppSettings.svelte` gains an Appearance -> Application theme selector with instant preview + Save; `frontend/src/lib/theme.ts` `applyTheme()` toggles `data-theme` on `<html>`; `App.svelte` `$effect` applies the user's theme on sign-in and reverts to dark on sign-out (login screen is always dark). d.ts updated.
  - Scope note: accent families other than cyan (red/emerald/amber status colors) stay default - they read acceptably on both themes; their light-shade *text* is slightly lower-contrast in light mode (documented polish item). Chart/SVG renderers keep their own palette by design.
- **Verification owed (no Go/Node toolchain here):** `npm --prefix frontend run check`, `make build`, `go test . ./internal/...`. Then GUI: App Settings -> switch to Light (instant preview) -> Save -> relogin stays Light; toggle back to Dark; confirm contrast across portfolio/picker/dashboard/editors.
- Next: PMForge app icons from the image files in the repo root.

## 2026-06-15 - PMForge app icon for wails build

- Rendered the tracked vector `pmforge-icon-dark.svg` (navy rounded tile + cyan Gantt/anvil symbol, transparent corners, soft shadow) to `build/appicon.png` at 1024x1024 via cairosvg. This is the master icon Wails uses; `wails build` derives the macOS `.icns` (the bundle's `iconfile`, already referenced by build/darwin/Info.plist's CFBundleIconFile) and the Windows `.ico` from it automatically.
- Chose the SVG over the root PNG sources ("PMForge Icon Image ….png" is portrait 784x1168 and gitignored) because the SVG is tracked, square, and renders crisply at any size - reproducible from source.
- Tracking: `.gitignore` now un-ignores `build/appicon.png` (alongside the build/darwin plists); `REUSE.toml` annotates it (GPL-3.0-or-later, James L. Burns and The PMForge Contributors) since it's a binary.
- No HTML favicon added: the running desktop app's window/taskbar icon comes from the generated `.icns`/`.ico`, not a web favicon.
- **Verification owed (no Go/Node/wails toolchain here):** `make build` on the Mac, then confirm `build/bin/<app>.app` shows the PMForge icon in Finder/Dock; `reuse lint` stays clean.

## 2026-06-15 - Theme-aware in-app logo

- New `frontend/src/lib/components/Logo.svelte`: inline SVG of the PMForge wordmark + Gantt/anvil mark. The cyan accent (#00D4FF) is constant; the timeline gridlines and "PMForge" wordmark use `currentColor`, so the logo's ink follows the surrounding text colour and flips automatically with the light/dark theme - one component, no file swapping, transparent background. Sized via the `class` prop (4:1 viewBox, width tracks height).
- Replaced the plain "PMForge" text wordmarks: `AppHeader.svelte` (toolbar; `h-6`, wrapped in a Dashboard "home" link) and `auth/Login.svelte` (`h-10`, centered). Both pass a flipping `text-slate-50/100`, so the logo inherits the theme ink.
- Verified composition by rendering both ink/bg combinations (light ink on dark, navy ink on light) - reads cleanly in both themes.
- The standalone `pmforge-logo-{dark,light}.svg` files are kept as brand assets; the in-app logo intentionally uses a transparent, currentColor variant rather than those (which carry their own background + grid).
- Verification owed: `npm --prefix frontend run check`, then visually confirm the logo in the header + login under both themes.

## 2026-06-15 - Project interchange: CSV/HTML/MS Project XML export + import scoping

- Request: open/use .mpp/.mpx/.xml/.pod; export to MS Project XML, CSV, HTML.
- **Exports** (Project Settings → Schedule Reports). The export engine already had CSV and MSPDI (ToMSPDI); added an HTML renderer `internal/export/html.go` (`FormatHTML` + engine dispatch) - a self-contained, printable schedule table with critical-path highlighting + optional EVM. Wired three new App methods `ExportScheduleReport{CSV,HTML,MSPDI}` and fixed `exportScheduleReportAs` to map extensions via a switch (it previously defaulted PDF to `.docx` - latent bug now fixed: pdf/.pdf, csv/.csv, html/.html, mspdi/.xml, odt/.odt, docx/.docx). d.ts + three new ProjectSettings buttons (Export CSV / HTML / MS Project XML).
- **Imports.** MS Project XML (MSPDI, `.xml`) import already worked; broadened the dialog to also list `.mpp/.mpx/.pod` and added `importScheduleFile(path)` routing: `.xml` → MSPDI parser; `.mpp`/`.pod`/`.mpx` → a precise, actionable error telling the user to re-save as MS Project XML from MS Project/ProjectLibre and import that.
- **Honest scope note:** `.mpp` (OLE binary) and `.pod` (Java-serialized) cannot be parsed in pure Go; the only robust reader is the Java MPXJ library, which would break the dependency-free/local-first design. `.mpx` is a feasible-but-obsolete text format - deferred to the same MSPDI path for now. If true native `.mpp`/`.pod` reading is ever required, the path is bundling/shelling to MPXJ (adds a Java runtime dependency) - flagged, not done.
- Tests: `internal/export/html_test.go` (HTML render + escaping) and `import_formats_test.go` (the .mpp/.mpx/.pod branches return the MSPDI-conversion message). Docs: README "Project interchange" section.
- **Verification owed (no Go/Node toolchain here):** `go test . ./internal/...`, `npm --prefix frontend run check`, `make build`; then GUI: Project Settings → export CSV/HTML/MS Project XML; Dashboard → Import schedule with a .mpp/.pod (expect the guidance message) and with a real MSPDI .xml (expect a CPM chart).

## 2026-06-18 - Manual + timed auto-save with configurable interval

- Request: manual and timed automatic saving, with a settings option to change the auto-save interval and turn it off/on.
- **Setting.** Added `AutoSaveSeconds int` to `AppSettings` (Go) + `auto_save_seconds: number` to the d.ts. `0` disables auto-save. New `defaultAppSettings()` returns `{AutoSaveSeconds: 60}`; `loadGlobalAppSettings()` now falls back to those defaults when no settings file exists (so brand-new users get auto-save on at 60s; an existing file with `0`/absent key reads as off and can be toggled on). `SaveAppSettings` already marshals the whole struct, so it persists automatically.
- **Coordinator.** New `frontend/src/lib/autosave.svelte.ts`: a single 1s heartbeat saves each registered editor whose snapshot string changed since its last save, once per `intervalSeconds`. Snapshot-based = no save when idle (no `updated_at` churn). `autosave.setInterval(s)` (0=off) + `autosave.register(snapshot, save) -> stop()`.
- **Loaded** in `App.svelte` on sign-in (`GetAppInfo().settings.auto_save_seconds`) and reset to 0 on sign-out; applied immediately when saved in `AppSettings.svelte` (new "Saving" section: enable checkbox + interval select 15/30/60/120/300s).
- **Wired into every editor** (register after the doc loads so the baseline snapshot is the saved state; unregister in onDestroy): the two shared shells `_layered_editor_shell` (network/pert/cpm) and `_stats_editor_shell` (8 stats charts), the 10 standalone chart editors (Activity, CauseEffect, Fishbone, Gantt, Matrix, RACI, SWOT, Stakeholder, WBS, Workflow), and `documents/CharterEditor` (all 25 doc kinds; snapshot covers field content + title). Matrix/RACI/SWOT gained an `onDestroy` (+ import) for cleanup.
- Manual save (Ctrl/Cmd+S + Save button) unchanged.
- Tests: `app_settings_test.go` (default = 60; load fallback = 60). Docs: README "Auto-save" subsection.
- **Verification owed (no Go/Node toolchain here):** `go test . ./internal/...`, `npm --prefix frontend run check`, `make build`; then GUI: Application Settings → Saving (toggle off/on, change interval, Save); open a chart and a document editor, make an edit, wait one interval, confirm it persists without pressing save; set interval off and confirm no auto-save; confirm idle editors don't bump the portfolio "modified" order.

## 2026-06-18 - Fast `make verify` gate + CI lockstep (regression hardening)

- Context: this session's two build breakages (a 1-char test username < the 3-32 char `usernameRE`, and a `*/` accidentally embedded in an `app.css` comment that closed the comment early) reached the desk because the manual chain run was a subset. The full `make check-release` would have caught both (frontend-build-budget runs `vite build`; race/test runs `go test`), but it's heavy and wasn't run.
- Added `make verify` = `test` + `frontend-stability` (svelte-check --fail-on-warnings) + `frontend-build-budget` (Vite build). One fast command that catches failing Go tests, type/svelte errors, and frontend/CSS build errors. Registered in `.PHONY`.
- `.gitlab-ci.yml`: the `test` job now runs `make verify` (single source of truth; local + CI can't drift). Fixed a latent `before_script` bug where `cd frontend && npm ci` left CWD in frontend/ so the following `make fonts` would have run from the wrong dir - now a subshell `(cd frontend && npm ci)`.
- Docs: README Quick start now leads with `make verify` as the pre-commit gate and explains verify vs check-release.
- **Deliberately deferred (needs visual QA, not done blind):** Light-theme contrast on the semantic accent colors (emerald/amber/red). Only slate + cyan are remapped to theme CSS vars today; emerald/amber/red text shades (200-400) stay light and read poorly on light surfaces. A correct fix can't be a blanket palette inversion - badge backgrounds use a *mix* of light-shade (`bg-emerald-600/20`) and dark-shade (`bg-emerald-900/50`) tints, and solid action buttons (`bg-red-600 text-white`) would regress if 600 is lightened. This needs a per-pattern pass with the app rendered in light mode to verify contrast. Flagged as the next UI task.
- **Verification owed (no toolchain here):** `make verify` (should pass and now also guards future changes); on tags, `make check-release`.

## 2026-06-19 - Six Sigma launchpad project activation bug fix

- **Bug:** When a project was created via the Project Launchpad (`CreateProjectFromLaunchpad`), all chart/document operations failed immediately ("no project open") until the user manually closed and reopened the project. This was the Six Sigma (and any other launchpad-created) project reported by the user.
- **Root cause:** `CreateProjectFromLaunchpad` closes the project DB at the end (`_ = d.Close()`) and a comment in the code said it "relies on OpenProject to install the project as the app's active one" - but the `onCreated` callback in `App.svelte` never called `OpenProject`. So `a.db` was nil after creation.
- **Fix:** At the end of `CreateProjectFromLaunchpad` (after `d.Close()`), now calls `a.OpenProject(path)` to set the active DB. This also fixes the stale-DB sub-case where creating a launchpad project while another was open would have left the old project active.
- **Test:** Added `TestCreateProjectFromLaunchpadActivatesProject` in `encryption_project_test.go` that calls `ListCharts` without calling `OpenProject` first. Test confirmed: fails (red) without the fix, passes (green) with it. Race detector clean.
- **Verification owed (GUI):** The toolchain here (Vite-only preview, no Wails backend) cannot exercise `window.go.*` calls. Hand-verify: create a new Six Sigma project via Launchpad, immediately click a chart or document without closing+reopening - it should now work. `make verify` passes.
- **Still deferred:** Light-theme contrast on semantic accent colors (emerald/amber/red). See the 2026-06-18 note for the approach and the per-pattern badge co-occurrence caveat that must be resolved before the CSS variable remapping can be called done.

## 2026-06-19 - Light-theme contrast: semantic color CSS variable remapping

- **Problem:** `text-red/emerald/amber/orange/rose/sky-{200-400}` are pastel shades that work on dark surfaces but fail WCAG AA on white (light mode). Additionally, badge patterns like `bg-emerald-900 text-emerald-200` would become dark-on-dark in light mode if only text was remapped.
- **Scope decision:** Added shades 100–400 (text) and 900+950 (badge/alert backgrounds) for red/emerald/amber to CSS variables. Added 300+950 for orange/rose/sky. Shades 500–800 (action buttons: `bg-red-600 text-white`) left as standard Tailwind — they have sufficient contrast and would regress if lightened.
- **Badge co-occurrence:** The common `bg-X-900 text-X-200` pattern (RACI, SprintList, DORADashboard, StakeholderManager, KanbanBoard badges; Toast; TollgateChecklist alerts; CPM warnings) inverts correctly — 900 → 100 (pale tint), 200 → 800 (dark text) — so both bg and text flip together.
- **SWOT editor** uses `bg-{color}-950/30` transparency tints. In light mode these become the 50-shade at /30 opacity, which is a very subtle tint. Acceptable degradation; dark borders remain visible.
- **amber-700 (#b45309) fails WCAG AA (3.5:1 vs white):** amber-300/400 remap to amber-800 (not 700).
- **Files changed:** `frontend/tailwind.config.js` (add 6 color entries with specific shades), `frontend/src/app.css` (add vars to `:root` with dark defaults; add overrides to `:root[data-theme='light']` with remapped values).
- **Visual verification:** Vite preview confirmed dark mode badges (dark bg + light text) unchanged; light mode badges correctly inverted (pale bg + dark text); standalone text readable in both modes.
- **Verification owed:** Full visual QA with the Wails app running in light mode — open Portfolio, a Kanban board, a RACI chart, the DORA dashboard, and the TollgateChecklist. `make verify` passes.

## 2026-06-19 - Portfolio search + filter; Dashboard kind labels

**Portfolio (`Portfolio.svelte`):**
- Added live name search input (bound to `query`, case-insensitive substring match).
- Added three filter tabs — All (N) / Active (N) / Done (N) — where Active = planning/active/on_hold and Done = complete/cancelled. Counts reflect the full unfiltered list so the user can see scope at a glance.
- Empty state distinguishes "no projects yet" (zero collection) from "no projects match your search" (filtered to zero).
- Header stat line switches to use the new `counts` derived object; counts are always against the full project list regardless of current tab/search.

**Dashboard (`Dashboard.svelte`):**
- Added `chartKindLabel` map (from the existing `newChartCards` array) and `docKindLabel` derived map (from `docKinds` loaded on mount).
- Existing-charts list now shows "CPM Chart" instead of "cpm", "Work Breakdown Structure" instead of "wbs", etc.
- Existing-documents list now shows "Project Charter" instead of "charter", "Risk Register" instead of "risk_register", etc.

**Verification:** `make verify` passes. Visual QA in the Vite preview confirmed all Portfolio states (all/active/done tabs, search filter, no-match empty state). Wails-backend verification owed (portfolio with real projects; dashboard chart and document kind labels).

## 2026-06-19 - Window maximize menu + minimum size

- macOS: added `menu.WindowMenu()` after the Edit menu in `buildAppMenu`. This gives Minimize (Cmd+M), Zoom (native fullscreen/maximize), and Bring All to Front with zero custom code.
- Windows/Linux: added a "Window" submenu with "Maximize / Restore" (F11 -> `wailsruntime.WindowToggleMaximise`) and "Minimize".
- Added `MinWidth: 800, MinHeight: 600` to the Wails app options so the window cannot be shrunk to an unusable size.
- **Verification owed:** test macOS Zoom menu item, Windows F11 key, and that the window respects minimum sizes.

## 2026-06-19 - Code quality audit + data management controls

**Go backend audit findings (`main.go`, `internal/db/`, `internal/users/`, `internal/crypto/`, `internal/agile/`):**
- No SQL injection risk (all parameterized queries).
- Mutex discipline correct throughout (`sync.RWMutex`; read ops RLock, writes Lock).
- Path traversal protected via `projectPathFor()` + `sanitizeFilename()`.
- Crypto layer: AES-256-GCM + Argon2id KDF (OWASP 2023 params). Correct.
- User auth: username regex rejects path chars; transparent re-hash on login; user dirs chmod 0700.
- Backup system: integrity-gated, VACUUM INTO snapshot, atomic rename. Correct.
- `internal/db/repair.go`: VACUUM INTO + atomic rename, split to allow caller-controlled close. Correct.

**Fixed — `CloneProject` WAL checkpoint race (`main.go`):**
When the source is the currently-open project, raw `copyFile` + sidecar copy could race against a WAL checkpoint and produce a clone missing committed data. Fixed: when `samePath(a.dbPath, clean)` is true, use `db.CreateSnapshot` (VACUUM INTO) for an atomic consistent copy. Closed projects continue using the existing copy+sidecar path.

**Fixed — audit logging for destructive operations (`main.go`):**
`audit_log` table and `LogAction()` existed but were only called for archive/signature events. Added `LogAction(actor, action, id, "")` calls (best-effort, `_ =`) before the actual delete in:
- `DeleteChart` — action: `"delete_chart"`
- `DeleteDocument` — action: `"delete_document"`
- `DeleteWorkItem` — action: `"delete_work_item"` (adds explicit `requireDB()` guard; `agileStore()` still called for the operation)
Actor is `a.requireUser().Username` or `"unknown"`. `DeleteProject` not logged (audit log is inside the project being deleted).

**Verification:** `go build ./...` clean on all changes.

## 2026-06-20 - Security tests: APFS collision regression + DEK-orphan guard

**`TestCreateAccount_RejectsCaseVariantUsername`** (`internal/users/store_test.go`): regression gate for the `lower(username)` fix. Creates "alice", then asserts that "Alice", "ALICE", and "aLiCe" each return `ErrUserExists`. Passes on case-sensitive CI without depending on APFS — exercises the SQL check directly.

**`TestHasLegacyRecoveryCodeWraps`** (`internal/users/dek_test.go`): covers the three states of the DEK-orphan guard — (1) no codes → false (does not block encryption enablement), (2) nil-DEK legacy codes → true (forces re-issue before enabling encryption), (3) codes re-issued with DEK → false (guard clears correctly). `HasLegacyRecoveryCodeWraps` coverage: 0% → 80%.

**`RemainingRecoveryCodes` skip:** `SELECT COUNT(*)` always returns one row so `sql.ErrNoRows` in that function is unreachable. No test written — defensive guard, not exercisable logic.

**Coverage:** `./internal/users/...` 61.6% → 63.5%. `make check-release` clean.

**"James" account deleted:** Removed the capitalized-"James" row from `~/Documents/PMForge/system.db` (`DELETE FROM users WHERE username = 'James'`). No filesystem changes — the directory is shared with "james" on APFS and deleting it would wipe james's projects.

## 2026-06-20 - Cross-account project name leak fix (APFS case-insensitive collision)

**Root cause:** `CreateAccount` used `WHERE username = ?` (case-sensitive), so accounts "James" and "james" were both created in system.db. On APFS, `~/Documents/PMForge/James` and `~/Documents/PMForge/james` resolve to the same physical directory. Both accounts share one data dir, so `james` can see the names of `James`'s `.pmforge` files in the portfolio.

**Exposure:** Project names only. The "Could not read project details." error confirms the current session's DEK cannot decrypt the other account's files — project contents are not exposed.

**Fix:** `internal/users/store.go` — changed the duplicate check in `CreateAccount` from `WHERE username = ?` to `WHERE lower(username) = lower(?)`. Two accounts whose names differ only by case can no longer be created. `go build ./internal/users/...` confirms clean.

**Existing collision:** The two accounts ("James", "james") shared `~/Documents/PMForge/James` on disk. The "James" account row was later removed from system.db (see session above). No filesystem changes.

## 2026-06-20 - UI/UX polish pass

**Deleted `Settings.svelte`** (orphaned — never routed from `App.svelte`, shadowed by `AppSettings.svelte` and `ProjectSettings.svelte`). 209 → 208 svelte-check files.

**`AppSettings.svelte` — Save button relocated to page footer:**
- Previously the Save button was inside the "Defaults for new projects" section, implying it only saved defaults. It actually saves all three sections (Appearance, Saving, Defaults).
- Moved the Save/status line outside all sections into a dedicated `<div class="flex items-center gap-3 pt-2">` at the end of the form. Button label changed "Save" → "Save settings" for clarity.

**`Dashboard.svelte` — four targeted fixes:**
1. `newCharter()` and `newDocument()` both lacked try/catch. Added error handling with `showToast` so backend failures surface to the user instead of silently rejecting.
2. Document status shown as plain text (`d.status`). Replaced with a styled `docStatusStyles` badge map (draft=slate, review=amber, approved=emerald, archived=muted-slate) matching the existing `statusStyles` pattern for project statuses.
3. Document rows: improved layout — title now truncates cleanly, status badge and version are grouped in a shrink-0 right-side div.
4. Delete buttons ("Del") renamed to "Delete" with proper `aria-label` attributes on both chart and document rows. Confirm/cancel buttons now use consistent colours (`bg-red-700`, `bg-slate-700 text-slate-300`).

**Verification:** `svelte-check` 0 errors/208 files; `vite build` clean; `make frontend-smoke` → App loaded and rendered.

## 2026-06-20 - Wails visual QA (light mode + window mgmt + launchpad)

**Window management (macOS):** Zoom (Window > Zoom) fills the display and restores on second click. Cmd+M minimizes to Dock, clicking restores. Full Screen (Ctrl+F) is in the Window menu. All pass.

**Dashboard returning-user flow (Test Six Sigma project):** PLANNING status badge renders inline with the project title. Subtitle shows "initiation · six_sigma" (phase · methodology joined with ` · `); dates omitted when not set. Existing CHARTS and DOCUMENTS sections appear above the NEW CHART creation grid. All correct.

**Light mode visual QA (app theme = Light):**
- Portfolio: status badges (PLANNING, UNKNOWN), error states, search/filter UI — all legible.
- App Settings: form fields, dropdowns, checkbox, save button — all legible.
- Project Dashboard: header badge, section labels, chart/doc list rows — all legible.
- RACI Matrix editor: role/task inputs, `+ Role`/`+ Task` buttons, instruction text — correct.
- Kanban board: four columns (TO DO/IN PROGRESS/REVIEW/DONE), WIP-limit badges, empty-state "Drag a card here" text — correct.
- DORA Dashboard: four metric cards (Deployment Frequency/Lead Time/CFR/MTTR), 30-day deployment chart, Deployment Log empty state — correct.
- Fishbone Diagram editor: canvas, EFFECT panel, "Apply 6 Ms preset"/Category buttons — correct.

**Six Sigma launchpad bug (fixed):** Created new Engineering > Manufacturing > Six Sigma project. App navigated directly to the project dashboard without requiring a reopen. Immediately clicked "Root cause (Fishbone)" starter chart — editor opened correctly. Bug is confirmed fixed.

**TollgateChecklist — navigation gap found:** `TollgateChecklist.svelte` lives in `frontend/src/lib/components/sigma/TollgateChecklist.svelte` and is rendered by `SigmaProjectView` (`sigma_project` route). However, neither `sigma_dashboard` nor `sigma_project` has any incoming navigation from the Portfolio, main project Dashboard, or top nav. The sigma workspace routes are orphaned — not reachable through the current UI. Flagged for follow-up wiring.

## 2026-06-20 - REUSE compliance restored + audit/WAL tests

**REUSE compliance failure fixed (`make check-release` was blocked):**
- Root cause 1: `cmd/pmforge/frontend/dist/` — stale embed-copy directory from the 2026-06-15 main-package relocation. Not covered by `/frontend/dist/` in .gitignore (different path). Deleted the entire `cmd/` directory (no tracked files remain there after the relocation; session notes from 2026-06-15 said "James should `rm -rf cmd` locally" but it was never done). Added `/cmd/` to `.gitignore` to guard against accidental recreation.
- Root cause 2: `frontend/package.json.md5` — generated npm checksum file, not gitignored. Added to `.gitignore`.
- Root cause 3: `build/bin/**` and `build/packages/**` — gitignored artifacts that `reuse` 6.x scans regardless of `.gitignore`. Added `REUSE.toml` glob annotations for these paths (GPL-3.0-or-later as derived works).
- `reuse lint` now 363/363 files compliant. All 11 `make check-release` gates pass: "PMForge is ready for release."

**New test file `audit_actions_test.go`** (4 tests, race-clean):
- `TestCloneOpenProject_DataSurvivesSnapshot`: Saves a chart to the open project, clones it while open (`a.db != nil` triggers VACUUM INTO), opens the clone, and asserts the chart is present by ID and title. This is the actual WAL-fix invariant — a raw `copyFile` can produce a file that passes `IsEncryptedFile` yet is missing WAL-only data.
- `TestDeleteChart_WritesAuditLog`: Creates a chart, deletes it, queries `audit_log` directly via `app.db.Conn` to assert one `delete_chart` row with the correct `target_id`.
- `TestDeleteDocument_WritesAuditLog`: Same pattern for `delete_document`.
- `TestDeleteWorkItem_WritesAuditLog`: Enables agile, creates a board, saves a work item, deletes it, asserts `delete_work_item` in `audit_log`.

Verification: `go test -count=1 -race -run "TestClone|TestDelete.*Audit" . → ok (all 4 pass)`; `make check-release` re-run on final tree → all 11 gates pass: "PMForge is ready for release." README not updated (no user-facing changes in this session).

## 2026-06-19 - Dashboard UX restructure: return-user flow first

**`Dashboard.svelte`:**
- **Section reorder:** Moved "Existing charts" and "Existing documents" to the top of the main content (immediately after the Stakeholders/Timeline/Budget nav row). Both sections are only rendered when items exist (`{#if charts.length > 0}` / `{#if docs.length > 0}`). The "New chart" grid, "New document" actions, Agile pack, and template reference sections remain below. This serves returning users who want to continue work without scrolling through a 21-card creation grid.
- **Richer project header:** Added a color-coded status badge (same `statusStyles` pattern as Portfolio) inline with the project name. Added a subtitle line: phase, methodology, and date range (`start_date → end_date | ongoing`) — all joining with ` · ` and omitting empty fields. `statusLabel` helper converts `on_hold` -> "on hold".
- **`SignCertificateModal` nesting bug fixed:** The modal was accidentally rendered inside the `<div class="grid">` element due to incorrect indentation/nesting. Moved it outside `<main>` entirely (between `<header>` and `<main>`), which is the correct location for a fixed-position overlay.
- **Template section headings simplified:** "Available document templates (N)" -> "Document templates (N)"; "Available chart templates (N)" -> "Chart templates (N)".
- **Verification:** `svelte-check` 0 errors/209 files; `vite build` clean. Wails-backend verification owed (open a project, confirm status badge, dates, and returning-user section ordering).
