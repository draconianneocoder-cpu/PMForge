<!--
SPDX-FileCopyrightText: 2026 The PMForge Contributors
SPDX-License-Identifier: GFDL-1.3-or-later
-->

# Post-Commit Audit - 2026-06-14

- Broad verified work was committed as `b291b5c Complete scheduling and encryption release work`.
- Remaining current work:
  - External PAdES/Acrobat validation with a trusted signing source.
  - Version string decision; if changed, keep `wails.json` and `internal/cli/parser.go` in sync because `check-release` compares them.
  - ADR-001 Windows packaging validation when a Windows target exists.
  - `.claude/settings.local.json` remains local-only and unstaged.
- Completed next safe item: README's PDF/A-3 TODO now states that `make check-pdfa` is a hard release blocker wired into `make check-release`, matching the implemented release gate.
