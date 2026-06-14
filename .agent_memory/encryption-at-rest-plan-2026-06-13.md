<!--
SPDX-FileCopyrightText: 2026 The PMForge Contributors
SPDX-License-Identifier: GFDL-1.3-or-later
-->

# Encryption At Rest Planning Handoff - 2026-06-13

## Current status

- Active checkout confirmed as `/Users/jamesburns/Documents/GitLab/PMForge - Go + Typescript`; the older Claude-project path from the environment block was not present.
- Previous developer completed the ADR-001 SQLCipher design/spike and the key-hierarchy implementation slice, but those files are still uncommitted in a dirty worktree.
- Completed encryption-at-rest pieces found:
  - `docs/design/ADR-001-database-encryption-at-rest.md`
  - `docs/design/spike-sqlcipher/`
  - `internal/crypto/keywrap.go`
  - `internal/crypto/keywrap_test.go`
  - `internal/users/dek.go`
  - `internal/users/dek_test.go`
  - DEK-aware `IssueRecoveryCodes` / `ResetWithRecoveryCode`
  - `App.dek` session storage, login/create unlock, logout zeroing, recovery-code wrapping
- Remaining ADR-001 implementation work is now planned in `docs/superpowers/plans/2026-06-13-encryption-at-rest-completion.md`.

## Verification evidence

- `go test -count=1 ./internal/crypto ./internal/users` passed.
- `go test -count=1 -run 'Test(KeyWrap|UnlockDEK|RecoveryReset|DEKMigration|ResetWithRecoveryCode)' ./internal/crypto ./internal/users` passed.
- `go test -count=1 ./cmd/pmforge` did not compile because generated embed files are absent: `cmd/pmforge/main.go:61:12: pattern all:frontend/dist: cannot embed directory frontend/dist: contains no embeddable files`.

## Unresolved work

- SQLCipher dependency has not landed in `go.mod`; repo still uses `github.com/mattn/go-sqlite3 v1.14.22`.
- `internal/db.InitDB` still opens plaintext DBs and does not accept a DEK.
- Project create/open paths still call `db.InitDB(path)` and therefore are not encrypted.
- Plaintext-to-encrypted migration, Settings opt-in UI, `.pre-encryption.bak` retention, and recovery-code reissue enforcement are not implemented.
- Repair/self-heal, backup, and headless CLI flows still need key-aware encrypted handling.
- No `check-encrypted-db` release gate exists yet.
- README/AGENT still need final status updates after implementation.

## Dirty-worktree note

- The worktree includes unrelated scheduling/chart/export changes. When executing the plan, stage encryption files by explicit path or hunk and avoid whole-file staging of `cmd/pmforge/main.go`, `README.md`, `AGENT.md`, or `session-notes.md`.

## 2026-06-13 follow-up

- Performed the next bounded work from `docs/superpowers/plans/2026-06-13-encryption-at-rest-completion.md`: Task 0 Step 1.
- Updated `docs/design/ADR-001-database-encryption-at-rest.md` action item 3 from unchecked to checked so the action list matches Appendix B and the implemented key-hierarchy files.
- Verification: `go test -count=1 ./internal/crypto ./internal/users` passed.
- No staging was performed. `cmd/pmforge/main.go` contains mixed encryption and unrelated scheduling hunks, so staging should remain explicit/hunk-based when preparing a commit.

## 2026-06-13 follow-up 2

- Performed Task 1 from `docs/superpowers/plans/2026-06-13-encryption-at-rest-completion.md`: adopted the SQLCipher-capable driver behind a central package.
- Added `internal/sqlitedriver/driver.go` with the shared driver name and blank import for `github.com/mutecomm/go-sqlcipher/v4`.
- Added `internal/sqlitedriver/driver_test.go`; RED failed with `undefined: Name`, then GREEN passed and confirmed `PRAGMA cipher_version` is non-empty.
- Ran `go get github.com/mutecomm/go-sqlcipher/v4@v4.4.2` and `go mod tidy`; `go.mod` now uses `github.com/mutecomm/go-sqlcipher/v4 v4.4.2` and no longer directly requires `github.com/mattn/go-sqlite3`.
- Switched `internal/db/sqlite.go`, `internal/db/repair.go`, and `internal/users/store.go` from direct `"sqlite3"` opens/blank mattn registration to `sqlitedriver.Name`.
- Verification: `go test -count=1 ./internal/sqlitedriver`, `go test -count=1 ./internal/db ./internal/users`, and `go test -count=1 ./internal/sqlitedriver ./internal/db ./internal/users ./internal/crypto` all passed.
- Remaining next work: Task 2, explicit encrypted project database openers and plaintext-to-encrypted migration.

## 2026-06-13 follow-up 3

- Performed Task 2 from `docs/superpowers/plans/2026-06-13-encryption-at-rest-completion.md`: explicit encrypted project database openers and plaintext-to-encrypted migration.
- TDD RED: added `internal/db/encryption_test.go`; initial run failed because `IsEncryptedFile`, `InitEncryptedDB`, and `MigratePlaintextToEncrypted` were undefined.
- GREEN implementation:
  - `internal/db/encryption.go` adds `InitEncryptedDB(path, dek)`, `IsEncryptedFile(path)`, `MigratePlaintextToEncrypted(path, dek)`, SQLCipher raw-key DSN construction, cipher integrity verification, and `sqlcipher_export` migration.
  - `internal/db/sqlite.go` now routes `InitDB` through shared `initDBWithDSN` and `applyStandardPragmas` so encrypted and plaintext opens share WAL/FK/temp-store setup.
  - Migration validates the DEK first, rejects encrypted or non-regular sources, verifies plaintext source integrity, exports to `<path>.encrypted.tmp`, verifies encrypted integrity and `cipher_integrity_check`, renames the original to `<path>.pre-encryption.bak`, publishes the encrypted file, and tightens file modes.
- Tests cover encrypted creation/header/cipher version, wrong-key rejection, bad-DEK rejection, and migration preserving project and chart data.
- Verification:
  - `go test -count=1 -run 'Test(InitEncryptedDB|MigratePlaintextToEncrypted|OpenEncryptedDB)' ./internal/db`
  - `go test -count=1 ./internal/db`
  - `go test -count=1 ./internal/sqlitedriver ./internal/db ./internal/crypto ./internal/users`
  - `go test -count=1 -race ./internal/db`
  - `git diff --check` over touched encryption files
- Updated plan checkboxes for Task 1 and Task 2. Task 0's staging step remains intentionally unchecked because this checkout has mixed unrelated hunks.
- Remaining next work: Task 3, thread the session DEK through GUI `CreateProject`, `OpenProject`, and `CreateProjectFromLaunchpad` paths and add app-level create/open tests.

## 2026-06-13 follow-up 4

- Performed Task 3 from `docs/superpowers/plans/2026-06-13-encryption-at-rest-completion.md`: thread the session DEK through app project create/open paths.
- Test setup note: `cmd/pmforge/frontend/dist` was empty, so `npm --prefix frontend run build` was run and `frontend/dist` was copied to `cmd/pmforge/frontend/dist` before `go test ./cmd/pmforge`; generated embed output remains ignored by git status.
- TDD RED: added `cmd/pmforge/encryption_project_test.go`; initial targeted app test run failed because `CreateProject` and `CreateProjectFromLaunchpad` produced plaintext files and a different user's session could open the project.
- GREEN implementation:
  - Added `ErrProjectRequiresEncryptionMigration`.
  - Added `App.requireDEKLocked`, which copies the active session DEK while the app lock is held.
  - `CreateProject` and `CreateProjectFromLaunchpad` now call `db.InitEncryptedDB(path, dek)`.
  - `OpenProject` now requires the active session DEK and opens via `db.InitEncryptedDB`; plaintext legacy files return `ErrProjectRequiresEncryptionMigration` for the future Settings migration flow.
- Tests cover encrypted normal project creation/open, encrypted Launchpad creation, wrong-user DEK rejection, and plaintext legacy migration-required error.
- Verification:
  - `go test -count=1 -run 'Test(CreateProjectEncryptsAndReopensWithSessionDEK|CreateProjectFromLaunchpadEncryptsProject|OpenProjectRejectsDifferentUsersDEK)' ./cmd/pmforge`
  - `go test -count=1 ./cmd/pmforge`
  - `go test -count=1 ./internal/db ./internal/users ./internal/crypto ./internal/sqlitedriver`
  - `go test -count=1 -race ./cmd/pmforge`
  - `git diff --check` over touched Task 3 files
- Updated plan checkboxes for Task 3.
- Remaining next work: Task 4, add Wails methods and Project Settings opt-in flow for migrating existing plaintext `.pmforge` files, including recovery-code reissue enforcement.

## 2026-06-13 follow-up 5

- Performed the backend half of Task 4 from `docs/superpowers/plans/2026-06-13-encryption-at-rest-completion.md`: Wails migration methods and recovery-code reissue enforcement.
- TDD RED: added `cmd/pmforge/encryption_migration_test.go`; initial run failed because `App.EncryptProjectAtRest`, `App.IsProjectEncrypted`, and `ErrRecoveryCodesRequireReissue` did not exist.
- GREEN implementation:
  - Added `users.Store.HasLegacyRecoveryCodeWraps(username)` to detect active recovery codes whose `wrapped_dek` is still empty.
  - Added `App.IsProjectEncrypted(path)` for the Settings UI state check.
  - Added `App.EncryptProjectAtRest(path)` to require an active session, block legacy recovery codes with the exact user-facing reissue error, close the active handle when migrating the currently open path, and call `db.MigratePlaintextToEncrypted(path, dek)`.
  - Added Wails bridge declarations for `IsProjectEncrypted` and `EncryptProjectAtRest` in `frontend/src/wails-window.d.ts`.
- Tests cover: blocked migration when active recovery codes are legacy/unwrapped; successful migration after `IssueRecoveryCodes()` reissues wrapped codes; `.pre-encryption.bak` retention; encrypted state before/after; preserving project, chart, and document rows; reopening migrated project with the session DEK.
- Verification:
  - `go test -count=1 -run 'TestEncryptProjectAtRest' ./cmd/pmforge`
  - `go test -count=1 ./cmd/pmforge`
  - `go test -count=1 ./internal/users ./internal/db ./internal/crypto ./internal/sqlitedriver`
  - `npm --prefix frontend run check`
  - `go test -count=1 -race ./cmd/pmforge ./internal/users`
- Updated plan checkboxes for Task 4 steps 1, 2, and 4. Remaining next work: Task 4 step 3, add the Project Settings opt-in UI.

## 2026-06-13 follow-up 6

- Performed Task 4 step 3 from `docs/superpowers/plans/2026-06-13-encryption-at-rest-completion.md`: Project Settings opt-in UI for plaintext-to-encrypted project migration.
- Added `frontend/scripts/project-settings-encryption-check.mjs` and `npm --prefix frontend run test:project-settings-encryption` as a narrow regression gate for the Settings encryption surface.
- TDD RED: the new frontend check failed until the UI tracked `session.projectPath`, loaded encryption state through `IsProjectEncrypted`, called `EncryptProjectAtRest`, displayed the returned `.pre-encryption.bak` path, and exposed recovery-code reissue handling.
- GREEN implementation:
  - `frontend/src/lib/session.svelte.ts` now tracks `projectPath`.
  - `ProjectPicker`, `ProjectLaunchpad`, `App`, and `Dashboard` set or clear `session.projectPath` as projects open, launchpad projects are created, projects close, or users sign out.
  - `ProjectSettings.svelte` now shows database state (`Plaintext`, `Encrypted`, or `Unknown`), offers `Encrypt database` for plaintext files, displays the returned backup path, and handles the legacy recovery-code error by calling `IssueRecoveryCodes()` and showing the replacement codes inline for saving before retry.
- Verification:
  - `npm --prefix frontend run test:project-settings-encryption`
  - `npm --prefix frontend run check`
- Updated the plan checkbox for Task 4 step 3. Remaining next work: Task 5/secondary encrypted handling in repair, backup, and headless CLI flows, plus an encrypted database release gate.

## 2026-06-13 follow-up 7

- Performed Task 5 step 1 from `docs/superpowers/plans/2026-06-13-encryption-at-rest-completion.md`: key-aware repair snapshot integrity and swap.
- TDD RED: added `internal/db/repair_encryption_test.go`; initial focused run failed because `CheckEncryptedSnapshotIntegrity` and `Database.SwapInEncryptedSnapshot` did not exist.
- GREEN implementation:
  - `internal/db/repair.go` now exposes `CheckEncryptedSnapshotIntegrity(path, dek)` for SQLCipher snapshots. It rejects plaintext snapshots and runs SQLite plus SQLCipher integrity checks with the supplied DEK.
  - `Database.SwapInEncryptedSnapshot(livePath, dek)` mirrors the plaintext swap flow but verifies the `.bak` with the DEK before closing the live handle and reopens the published file through `InitEncryptedDB`.
  - `App.RepairAndSwap` now detects encrypted project files and uses the session DEK with `SwapInEncryptedSnapshot`; plaintext projects still use `SwapInSnapshot`.
- Verification:
  - `go test -count=1 -run TestSwapInEncryptedSnapshotPreservesEncryptionAndReopensWithDEK ./internal/db`
  - `go test -count=1 ./internal/db`
  - `go test -count=1 ./cmd/pmforge`
- Updated the plan checkbox for Task 5 step 1. Remaining next work: Task 5 step 2, prove `.pmba` backup archives preserve encrypted project bytes.

## 2026-06-13 follow-up 8

- Performed Task 5 step 2 from `docs/superpowers/plans/2026-06-13-encryption-at-rest-completion.md`: prove `.pmba` backup archives preserve encrypted project bytes.
- Added `TestCreateArchivalBundlePreservesEncryptedProjectBytes` in `internal/db/backup_test.go`.
- The new test creates an encrypted project DB, writes project metadata, runs `CreateArchivalBundle`, opens the resulting `.pmba`, reads the `project.pmforge` entry header, and fails if the archived bytes expose SQLite's plaintext `SQLite format 3\x00` header.
- The regression passed immediately, confirming the existing `CreateSnapshot` plus zip bundle path already preserves SQLCipher-encrypted bytes. No production backup code change was needed for this step.
- Verification:
  - `go test -count=1 -run TestCreateArchivalBundlePreservesEncryptedProjectBytes ./internal/db`
  - `go test -count=1 ./internal/db`
  - `go test -count=1 -race ./internal/db`
- Updated the plan checkbox for Task 5 step 2. Remaining next work: Task 5 step 3, add headless encrypted maintenance password input for `--check`, `--repair`, `--vacuum`, and `--export-audit`.

## 2026-06-13 follow-up 9

- Performed Task 5 step 3 from `docs/superpowers/plans/2026-06-13-encryption-at-rest-completion.md`: headless encrypted maintenance password input.
- TDD RED:
  - Added `cmd/pmforge/headless_encryption_test.go`; initial focused run failed because `openHeadlessDB`, `cli.Config.Username`, and `cli.Config.PasswordEnv` did not exist.
  - Extended `internal/cli/parser_test.go` default-value coverage; initial run failed because the new fields did not exist.
- GREEN implementation:
  - `internal/cli/parser.go` now accepts `--username` and `--password-env`.
  - `cmd/pmforge/main.go` now routes headless maintenance through `openHeadlessDB`.
  - Plaintext `.pmforge` files still open with `db.InitDB`.
  - Encrypted `.pmforge` files require `--username` and `--password-env`; the helper infers the PMForge root from `<root>/<username>/projects/<file>`, opens `system.db`, authenticates the user, unlocks the DEK, and opens with `db.InitEncryptedDB`.
  - Missing credentials, missing password env, wrong password, or a path outside that user layout fail before maintenance operations run.
- Verification:
  - `go test -count=1 -run TestOpenHeadlessDB ./cmd/pmforge`
  - `go test -count=1 ./internal/cli`
  - `go test -count=1 ./cmd/pmforge ./internal/cli ./internal/db ./internal/users ./internal/crypto ./internal/sqlitedriver`
  - `go test -count=1 -race ./cmd/pmforge ./internal/cli`
- Updated the plan checkbox for Task 5 step 3. Remaining next work: Task 6 step 1, create the deterministic encrypted database release gate script.

## 2026-06-13 follow-up 10

- Performed Task 6 step 1 from `docs/superpowers/plans/2026-06-13-encryption-at-rest-completion.md`: deterministic encrypted database gate script.
- TDD RED: `bash scripts/validate-encrypted-db.sh` failed with `No such file or directory`.
- GREEN implementation:
  - Added executable `scripts/validate-encrypted-db.sh` with SPDX metadata.
  - The script runs targeted deterministic Go tests that create SQLCipher-encrypted `.pmforge` files, verify encrypted headers, reject wrong keys and bad DEKs, migrate plaintext fixtures with integrity plus `cipher_integrity_check`, swap encrypted repair snapshots, confirm `.pmba` archives preserve encrypted `project.pmforge` bytes, and verify app/headless encrypted open flows.
- Verification:
  - `bash scripts/validate-encrypted-db.sh`
- Updated the plan checkbox for Task 6 step 1. Remaining next work: Task 6 step 2, wire `check-encrypted-db` into the Makefile and release gate.

## 2026-06-13 follow-up 11

- Performed Task 6 step 2 from `docs/superpowers/plans/2026-06-13-encryption-at-rest-completion.md`: wired the encrypted database gate into Makefile and `scripts/check-release.sh`.
- RED: `make check-encrypted-db` failed because the target did not exist.
- GREEN implementation:
  - `Makefile` now exposes `check-encrypted-db` in `.PHONY` and help output, calling `scripts/validate-encrypted-db.sh`.
  - `scripts/check-release.sh` now runs the encrypted database validation gate after the production build and before PDF/A/PAdES gates, with failure guidance pointing to `make check-encrypted-db`.
- Verification:
  - `make check-encrypted-db`
  - `bash -n scripts/check-release.sh scripts/validate-encrypted-db.sh`
  - `make help | rg 'check-encrypted-db|check-pades|check-release'`
- Updated the plan checkbox for Task 6 step 2. Remaining next work: Task 6 step 3, update release-scope guard for SQLCipher/native encryption doc drift.

## 2026-06-13 follow-up 12

- Performed Task 6 step 3 from `docs/superpowers/plans/2026-06-13-encryption-at-rest-completion.md`: updated the release-scope guard for SQLCipher/native encryption status drift.
- RED: after adding the guard, `bash scripts/release-gate-scope-check.sh` failed because README still said SQLCipher/native database encryption was deferred and did not document SQLCipher-encrypted per-user `.pmforge` project databases.
- GREEN implementation:
  - `scripts/release-gate-scope-check.sh` now fails if `go.mod` lacks `github.com/mutecomm/go-sqlcipher/v4`.
  - The guard now fails stale README claims that SQLCipher/native database encryption is still deferred.
  - The guard now requires README to document SQLCipher-encrypted per-user `.pmforge` project databases.
  - README's per-user at-rest protection item now states that new `.pmforge` project databases are SQLCipher-encrypted, existing plaintext projects can be migrated from Project Settings after recovery-code reissue, `system.db` remains plaintext by design, and OS-level disk encryption remains whole-device protection.
- Verification:
  - `bash scripts/release-gate-scope-check.sh`
- Updated the plan checkbox for Task 6 step 3. Remaining next work: Task 7 step 1, update final README/AGENT/ADR/session-notes documentation for the completed encryption-at-rest implementation and recovery semantics.

## 2026-06-13 follow-up 13

- Performed Task 7 step 1 from `docs/superpowers/plans/2026-06-13-encryption-at-rest-completion.md`: final documentation updates for completed encryption-at-rest behavior.
- Updated README:
  - New per-user `.pmforge` project databases are SQLCipher-encrypted with the user's DEK.
  - Existing plaintext project databases can be migrated from Project Settings after recovery-code reissue.
  - `system.db` remains plaintext by design and stores password hashes plus wrapped DEKs, not project records.
  - `.pmba` bundles preserve encrypted `project.pmforge` bytes.
  - Losing the password and all valid wrapped recovery codes makes encrypted projects unrecoverable by design.
- Updated AGENT:
  - Per-user database encryption at rest is no longer listed as implementation-not-started.
  - The old V2 stopgap and stale TODO-list notes now explicitly say they were superseded by the 2026-06-13 SQLCipher implementation.
- Updated ADR-001:
  - Status is now Implemented as of 2026-06-13.
  - Decision text reflects default encrypted new project databases plus Settings migration for legacy plaintext projects.
  - Action items 4-7 are checked.
  - Appendix B summarizes SQLCipher open/migration, Settings opt-in, repair/backup/headless handling, and release gates.
- Updated `session-notes.md` with the current encryption-at-rest implementation handoff and remaining full-verification step.
- Verification:
  - Targeted stale-text scan over README/AGENT/ADR/session-notes.
- Updated the plan checkbox for Task 7 step 1. Remaining next work: Task 7 step 2, run the full verification sequence.

## 2026-06-13 follow-up 14

- Performed Task 7 step 2 from `docs/superpowers/plans/2026-06-13-encryption-at-rest-completion.md`: full encryption-at-rest verification.
- Verification passed:
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
- `make check-release` passed through version, REUSE, frontend budget/stability/smoke, release-scope, memory-safety, race, build, encrypted database, PDF/A-3, and PAdES local gates, ending with `PMForge is ready for release.`
- Execution note: do not parallelize `make license-check` with Go compile gates such as `make check-encrypted-db`; `license-check` intentionally removes `cmd/pmforge/frontend/dist`, and the `cmd/pmforge` embed pattern requires that directory for compilation. Recreate the embed dist after `license-check` before standalone Go compile gates, or rely on `make check-release` to rebuild/copy it internally.
- Updated the plan checkbox for Task 7 step 2. Encryption-at-rest implementation and verification tasks are complete; Task 0 step 3 remains intentionally unchecked because it is a staging-only instruction and no staging/commit was requested.

## 2026-06-14 follow-up 15

- Performed the safe whole-file portion of Task 0 step 3 staging.
- Removed a stale empty `.git/index.lock` after confirming no active Git index-writing process was running; the only Git process was the fsmonitor daemon and the lock timestamp was June 10.
- Staged:
  - `docs/design/ADR-001-database-encryption-at-rest.md`
  - `internal/crypto/keywrap.go`
  - `internal/crypto/keywrap_test.go`
  - `internal/users/dek.go`
  - `internal/users/dek_test.go`
  - `internal/users/recovery.go`
  - `internal/users/recovery_test.go`
  - `internal/users/store.go`
- Did not stage `cmd/pmforge/main.go` or `session-notes.md`; both contain broad unrelated dirty changes and require deliberate hunk-level staging for only the `App.dek`, login/create/logout/recovery-code, and current handoff hunks.
- Verification:
  - `git diff --cached --name-status`
  - `git diff --cached --stat`
  - `git diff --cached --check`
- Task 0 step 3 remains unchecked pending hunk-level staging for `cmd/pmforge/main.go` and `session-notes.md`.

## 2026-06-14 follow-up 16

- Analyzed the remaining "unrelated" dirty hunks and classified them as coherent, already verified product/doc work rather than disposable noise:
  - Scheduling core expansion: calendar anchoring, dependency types and lag, constraints, baselines, EVM, resource leveling/histogram, MSPDI import/export, and first-class Gantt.
  - Encryption-at-rest completion: SQLCipher driver, encrypted DB open/migration, Settings opt-in, repair/backup/headless support, release gates, docs.
  - Root project documentation: AGENTS/ARCHITECTURE/TESTING/STYLE/SECURITY/DEPENDENCIES plus handoff notes.
- Staged all product, docs, tests, scripts, and project handoff files for the completed work set.
- Left only `.claude/settings.local.json` unstaged as local tool configuration.
- Verification:
  - `git diff --cached --name-status`
  - `git diff --cached --stat`
  - `git diff --cached --check && git diff --check`
- Remaining local-only dirty file: `.claude/settings.local.json`.
