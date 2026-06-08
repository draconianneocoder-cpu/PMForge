<!--
SPDX-FileCopyrightText: 2026 The PMForge Contributors
SPDX-License-Identifier: CC0-1.0
-->

# PAdES External veraPDF Extraction - 2026-06-07

- `scripts/validate-pades-external.sh` now runs `verapdf --off --extract signature --format xml` when `verapdf` is installed.
- The harness parses the XML with Python's standard `xml.etree.ElementTree` and fails unless veraPDF extracts a signature with `filter=Adobe.PPKLite` and `subFilter=ETSI.CAdES.detached`.
- The full veraPDF feature output is written to `.tmp/pmforge-pades-test/verapdf-signature-features.xml`; stderr warnings are kept in `.tmp/pmforge-pades-test/verapdf-signature-features.stderr`.
- `scripts/validate-pades-external_test.sh` injects a fake veraPDF CLI and proved the old TODO-only branch failed before the harness change.
- `scripts/validate-pades.sh` and `scripts/validate-pades-external.sh` share `.tmp/pmforge-pades-test.lock`; the external harness holds the lock while generating and extracting the sample so parallel validators cannot remove the sample directory mid-read.
- `scripts/validate-pades-parallel_test.sh` reproduced the shared-temp-directory race by running the local gate, external gate, and external fake-veraPDF regression at the same time, then passed after the lock was added.
- Verification run: `bash scripts/validate-pades-external_test.sh`, `bash scripts/validate-pades-parallel_test.sh`, `bash -n scripts/validate-pades.sh scripts/validate-pades-external.sh scripts/validate-pades-external_test.sh scripts/validate-pades-parallel_test.sh`, `make check-pades`, `make check-pades-external`, `git diff --check && git diff --cached --check`, `make license-check`, and `make check-release`.
- Current local external coverage: OpenSSL ASN.1 parse, OpenSSL detached CMS verification, `qpdf --check`, `pdfsig` signature validation, veraPDF signature feature extraction, and DSS validation pass when the tools are installed. DSS currently classifies the sample as `PAdES-BASELINE-B`; trusted-chain validity remains indeterminate because the deterministic gate sample is self-signed.
