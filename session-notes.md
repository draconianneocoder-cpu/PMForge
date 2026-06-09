<!--
SPDX-FileCopyrightText: 2026 The PMForge Contributors
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
