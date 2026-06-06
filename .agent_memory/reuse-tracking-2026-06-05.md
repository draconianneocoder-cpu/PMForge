<!--
SPDX-FileCopyrightText: 2026 The PMForge Contributors
SPDX-License-Identifier: CC0-1.0
-->

# REUSE Tracking Pass - 2026-06-05

- Installed and verified `reuse` 6.2.0 via `pipx`; the executable is at `/Users/jamesburns/.local/bin/reuse`.
- Moved `LICENSES/README.md` to `LICENSES.md` because REUSE treats every file in `LICENSES/` as a license text.
- Tracked required license texts for committed assets: `CC0-1.0`, `GFDL-1.3-or-later`, and `GPL-3.0-or-later`.
- Kept fetched font binaries and root PNG exports ignored as local/generated artifacts; tracked the small SVG branding sources instead.
- Replaced the stale PDF/A ICC download source with Compact ICC Profiles `sRGB-v2-magic.icc`, committed the 736-byte CC0 profile, and updated `scripts/fetch-icc.sh`.
- Current verification passed: `make license-check`, clean-font REUSE simulation, focused PDF/Sigma tests, `git diff --check`, and `make check-release`.

## Follow-up PAdES Hardening - 2026-06-05

- `pdfmeta.InjectPAdESSignature` now merges the new invisible signature widget into an existing direct `/AcroForm` field array instead of replacing or ignoring the form tree.
- Existing indirect `/AcroForm` objects are rewritten in the same incremental update so the new signature field is reachable from the document form tree.
- Unsupported indirect `/Fields` arrays now fail before invoking the signing callback, avoiding a signed PDF with an orphan widget.
- Regression coverage was added for direct AcroForm merge, indirect AcroForm object rewrite, and unsupported field-array failure.
- Current verification passed: `go test -count=1 ./internal/pdfmeta ./internal/export`, `go test -count=1 ./internal/crypto`, `make license-check`, `git diff --check`, and `make check-release`.

## Follow-up CMS Signing Regression - 2026-06-05

- Added focused `internal/crypto` coverage for `Signer.SignPDFCMS`, which previously had no direct regression tests.
- The CMS test now parses the returned PKCS#7 SignedData, verifies it is detached, checks SHA-256 signer digest selection, confirms signer and extra certificates are embedded, verifies the exact original PDF bytes, and rejects tampered bytes.
- Missing private-key and missing-certificate error paths are also covered.
- Current verification passed: `go test -count=1 ./internal/crypto`, `go test -count=1 ./internal/crypto ./internal/pdfmeta ./internal/export`, `make license-check`, `git diff --check && git diff --cached --check`, and `make check-release`.

## Follow-up PAdES Embedded CMS Verification - 2026-06-05

- Added an end-to-end `internal/pdfmeta` regression that signs a PDF with PMForge's real CMS signer, extracts the padded `/Contents` DER, parses the embedded PKCS#7 SignedData, and verifies it against the declared `/ByteRange`.
- The same test mutates a signed byte and confirms CMS verification fails, giving local coverage for the tamper-evident contract without depending on external Acrobat/DSS validators.
- The fixture includes pre-existing `/Contents` and `/ByteRange` tokens so tests resolve the appended signature's ByteRange instead of unrelated earlier PDF objects.
- Current verification passed: `go test -count=1 ./internal/pdfmeta`, `go test -count=1 ./internal/crypto ./internal/pdfmeta ./internal/export`, `make license-check`, `git diff --check && git diff --cached --check`, and `make check-release`.

## Follow-up PAdES Local Sample Gate - 2026-06-05

- Added `scripts/validate-pades.sh`, a local gate that generates `.tmp/pmforge-pades-test/signed-sample.pdf` with PMForge's real CMS signer and `pdfmeta.InjectPAdESSignature`.
- The gate checks required PAdES markers, rejects fallback comment-marker signing, parses the padded `/Contents` DER, verifies the CMS against the declared `/ByteRange`, and confirms signed-byte tampering fails verification.
- Added `make check-pades` and wired the local PAdES gate into `scripts/check-release.sh` as a hard local release check. External Acrobat/DSS/veraPDF interoperability remains the separate remaining validation milestone.
- Current verification passed: `make check-pades`, `bash -n scripts/validate-pades.sh scripts/check-release.sh`, `go test -count=1 ./internal/crypto ./internal/pdfmeta ./internal/export`, `make license-check`, `git diff --check && git diff --cached --check`, and `make check-release`.

## Staging Boundary Normalization - 2026-06-05

- After reading `session-notes.md`, staged the coherent PAdES/release-gate files: `Makefile`, `scripts/check-release.sh`, `README.md`, `internal/pdfmeta/pdfmeta.go`, and `internal/pdfmeta/pdfmeta_test.go`.
- Staged only the PAdES local-gate status lines from `AGENT.md`; unrelated unstaged `AGENT.md` edits remain out of the index for separate review.
- Updated `session-notes.md` to reflect the new staged boundary and remaining next steps.
- Resolved the leftover `AGENT.md` working-tree hunks by keeping the current `LICENSES.md` documentation path and preserving the secure-archive fail-closed lesson; `AGENT.md` now has no remaining unstaged diff.

## External PAdES Harness - 2026-06-05

- Added `scripts/validate-pades-external.sh` and `make check-pades-external`.
- The harness extracts the signed sample's padded `/Contents` CMS DER and declared `/ByteRange` bytes, verifies ASN.1 parsing and detached CMS verification with OpenSSL, and writes `.tmp/pmforge-pades-test/external-validation-report.txt`.
- Current local result: OpenSSL 3.6.2 ASN.1 parse passed and detached CMS verification passed. `qpdf`, `pdfsig`, veraPDF CLI, and DSS CLI were not installed locally, so the harness recorded them as skips.

## External PAdES Validator Hardening - 2026-06-06

- Installed Homebrew `qpdf` 12.3.2, `poppler` 26.04.0 (`pdfsig`), and `verapdf` 1.30.2 for local validator coverage.
- Initial `qpdf --check` exposed that the gate sample's Pages tree pointed at a stream instead of a Page dictionary; `scripts/validate-pades.sh` now generates a syntactically valid one-page PDF before signing.
- Initial `pdfsig` validation exposed a CAdES interoperability gap: OpenSSL verified the detached CMS, but Poppler reported the signature invalid until `Signer.SignPDFCMS` added the RFC 5035 `SigningCertificateV2` signed attribute with the signer certificate's SHA-256 hash plus issuer/serial.
- `scripts/validate-pades-external.sh` now uses `pipefail`, initializes a temporary NSS database for `pdfsig` when `certutil` is available, and fails if `qpdf` rejects the PDF or `pdfsig` does not report `Signature Validation: Signature is Valid`.
- Current local result: `make check-pades-external` passes with OpenSSL ASN.1 parse, OpenSSL detached CMS verification, `qpdf` syntax validation, and `pdfsig` signature validation. veraPDF CLI is detected but remains a manual interoperability TODO for this PAdES sample; DSS CLI is still not installed locally.

## PDF/A-3b Schedule Gate Hardening - 2026-06-06

- `scripts/validate-pdfa.sh` now prefers an installed `verapdf` on `PATH` before attempting the older `/tmp/verapdf-1.28.1` auto-download path, and validates with `-f 3b` instead of veraPDF's default PDF/A-1b profile.
- `scripts/validate-pdfa-lib.sh` now accepts veraPDF's XML attribute form `isCompliant="true"` in addition to the older element and JSON forms; `scripts/validate-pdfa-lib_test.sh` covers both true and false attribute outputs and uses a fake `verapdf` injected through `PATH`.
- `pdfmeta.MakePDFA3` now adds the PDF/A binary header comment before incremental updates and adjusts classic xref offsets/startxref, writes a trailer `/ID`, keeps stream `/Length` values aligned with the EOL before `endstream`, and uses the latest incremental Catalog revision so OutputIntent injection preserves `/Metadata`.
- Schedule PDF exports register bundled Source Sans 3 as the Helvetica alias when those assets are present, avoiding non-embedded core-font failures in the representative schedule sample.
- Current local result: `make check-pdfa` passes with veraPDF 1.30.2 against `.tmp/pmforge-pdfa-test/schedule.pdf`, and `qpdf --check` reports no syntax or stream encoding errors. Remaining PDF/A work is to add document and combined-report samples before promoting the gate from soft to hard.

## PDF/A-3b Document Sample Expansion - 2026-06-06

- `scripts/validate-pdfa.sh` now also generates `.tmp/pmforge-pdfa-test/document-charter.pdf` via `documents.Render` and `.tmp/pmforge-pdfa-test/combined-report.pdf` via `documents.BuildCombinedReport`, with bundled Source Sans 3 registered for document renderers.
- `scripts/validate-pdfa-lib_test.sh` now fails unless the fake-veraPDF gate run sees `schedule.pdf`, `document-charter.pdf`, and `combined-report.pdf`, so future simplifications cannot silently reduce the representative sample set.
- Current local result: `bash scripts/validate-pdfa-lib_test.sh`, `make check-pdfa`, `qpdf --check` for all three generated samples, and `go test -count=1 ./internal/documents ./internal/pdfmeta ./internal/export` pass. Remaining PDF/A work is release-builder soak before promoting `make check-pdfa` from soft to hard.

## V2 Encryption-at-Rest Stopgap - 2026-06-06

- The per-user encryption-at-rest decision is now explicit for V2: PMForge does not claim native `.pmforge` database encryption yet; README documents private per-user data directories plus OS-level disk encryption (FileVault / BitLocker / LUKS) as the supported V2 at-rest protection path.
- SQLCipher/native database encryption remains deferred to V3 because it brings native packaging complexity, while whole-file AES-at-rest would require careful crash-recovery and migration semantics.
- `scripts/release-gate-scope-check.sh` now fails if README stops documenting both the OS-level disk-encryption stopgap and the SQLCipher/V3 deferral, so release docs cannot drift into an unsupported encryption claim.
- TDD note: `make release-scope` first failed on the missing README guidance, then passed after README and the line-wrapping-safe guard were updated.

## Commit Boundary Audit - 2026-06-06

- After reading `session-notes.md`, the staged index was audited with `git diff --cached --name-status` and `git diff --cached --stat`.
- The current index is a broad release bundle, not a signing/PDF-only commit boundary: it includes 94 staged files and roughly 17k inserted lines across Sigma/frontend/test/release/REUSE/PDF validation work.
- No staging or commit was performed. A future commit pass should either intentionally commit the broad staged bundle or reset/restage into smaller coherent slices; it should not use a narrow signing/PDF commit message for the current index.

## Validation Staging Boundary Normalization - 2026-06-06

- After re-reading `session-notes.md`, the PAdES/PDF validation surface was staged into the current release bundle: validation scripts, PDF/A helpers, `internal/export/pdf.go`, `internal/pdfmeta/*`, `internal/crypto/pdf_sign_test.go`, README/AGENT/session notes, and project memory.
- `internal/crypto/pdf_sign.go` was staged with a cached-only patch containing only the RFC 5035 `SigningCertificateV2` CMS attribute support. The pre-existing PKCS#12 loader rewrite remains unstaged.
- The index still contains broader Sigma/frontend/release work, so the next commit decision is still whether to commit the broad staged release bundle or split it. Do not describe the staged index as a narrow signing/PDF-only commit.
