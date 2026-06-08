<!--
SPDX-FileCopyrightText: 2026 The PMForge Contributors
SPDX-License-Identifier: CC0-1.0
-->

# DSS Validation Tool Install - 2026-06-07

- Installed the official DSS 6.4 standalone minimal archive from the European Commission DSS demo download endpoint.
- Local install path: `/Users/jamesburns/.local/opt/dss-6.4`.
- Command wrapper: `/Users/jamesburns/.local/bin/dss-validation-tool`.
- The wrapper uses the official `dss-app.jar` and a small Java CLI around DSS `SignedDocumentValidator`.
- Archive SHA-256: `75e628b3e0947940a53408efc601f95c895bc47854bc3210f6d7e3286efb0723`.
- JAR SHA-256: `22ac8b095a00f3a5e9ae3184a1916dc38162db4c55a5e5ee53c13682facb8eea`.
- Verification: `dss-validation-tool --version`, `dss-validation-tool --help`, `make check-pades`, `dss-validation-tool validate .tmp/pmforge-pades-test/signed-sample.pdf`, and `make check-pades-external`.
- `scripts/validate-pades-external.sh` now invokes `dss-validation-tool validate "$PDF_PATH"` when the wrapper is installed, writes `.tmp/pmforge-pades-test/dss-validation-output.txt`, and fails if DSS reports a PAdES baseline requirements warning or a non-`PAdES-BASELINE-B` `signature.format`.
- PMForge's CMS signer now omits CMS `signing-time` from signed attributes for PAdES baseline-B, while preserving detached CMS verification and `signingCertificateV2`.
- `pdfmeta.InjectPAdESSignature` now writes a signed PDF signature dictionary `/M (D:YYYYMMDDHHmmSSZ)` timestamp so DSS no longer warns that `/M` is missing.
- DSS parses the PMForge signed sample as `signature.format=PAdES-BASELINE-B`. It still reports `valid_signatures=0`, `INDETERMINATE`, and `NO_CERTIFICATE_CHAIN_FOUND` because the deterministic gate sample is self-signed and no trusted source is configured.
- Follow-up verification passed: `go test -count=1 ./internal/crypto ./internal/pdfmeta ./internal/export`, `bash scripts/validate-pades-external_test.sh`, `bash scripts/validate-pades-parallel_test.sh`, `make check-pades`, `make check-pades-external`, `make license-check`, `git diff --check`, `git diff --cached --check`, and `make check-release`.
- README and AGENT now describe DSS as executed coverage rather than a remaining TODO: `make check-pades-external` can run DSS, DSS classifies the sample as `PAdES-BASELINE-B`, and the remaining external gap is Acrobat plus trusted-chain validation with a real trusted signing source.
- `scripts/release-gate-scope-check.sh` now fails if README/AGENT stop mentioning the DSS `PAdES-BASELINE-B` result or regain stale wording that treats DSS as unrun.
- Documentation-guard verification passed: `make release-scope`, `bash -n scripts/release-gate-scope-check.sh`, `make license-check`, `git diff --check`, `git diff --cached --check`, and `make check-release`.
