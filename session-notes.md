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
