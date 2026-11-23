---
name: skill-whmcstofossbill
description: Project memory and operating manual for the WHMCStoFossbilling experimental WHMCS 8.x to FOSSBilling 0.8.9 migration module (modules/Whmcstofossbilling/, asset WHMCStoFossbilling-0.2.zip). Use this skill whenever a task involves improving, debugging, reviewing, testing, documenting, packaging, or releasing this module, or whenever it mentions WHMCStoFossbilling, Whmcstofossbilling, WHMCS migration, the 0.2 prerelease, Step 4 drive batches, adaptive batch sizing, imported visibility repair, SQL/GZIP/ZIP backup uploads, a stalled running/paused run, or a session-timeout resume prompt. It carries the settled requirements, hard safety boundaries, architecture, verification commands, and release playbook so a new agent does not re-litigate decisions or perform unauthorized live writes.
license: AGPL-3.0
metadata:
  author: Monlam Makhampa
  version: "1"
---

# WHMCStoFossbilling — project memory and operating manual

**Version:** 1 · **Author:** Monlam Makhampa

## Orientation

- **Migration direction:** WHMCS 8.x → FOSSBilling 0.8.9. The display name does not reverse source and destination.
- **Location:** `D:\Laragon\www\FOSSBill\modules\Whmcstofossbilling/` (installation directory `modules/Whmcstofossbilling/`; capitalization matters on Linux).
- **Identity invariants:** manifest id `whmcstofossbilling`, name `WHMCStoFossbilling`, version `0.2`, author `Monlam Makhampa`, description contains `vibe coded`. Asset: `WHMCStoFossbilling-0.2.zip`. Admin route: `/admin/whmcstofossbilling`.
- **Maturity:** experimental development preview. Version `0.2` is **not** acceptance of the originally requested production-complete migration. Never describe archive-only records as native imports, or financial snapshot totals as final cutover reconciliation.

Before changing code, read the project's own source of truth in that directory: `README.md`, `docs/ARCHITECTURE.md`, `docs/COMPATIBILITY.md`, `docs/OPERATIONS.md`, `docs/TESTING.md`, `docs/REMAINING-WORK.md`, `docs/RELEASE-0.2.md`. This skill adds the decisions and constraints that are easy to miss.

## Non-negotiable boundaries

These are settled requirements, not suggestions. They exist because earlier work established that crossing them risks corrupting a live billing system or invalidating the release.

1. **Change only files under `modules/Whmcstofossbilling/`.** No FOSSBilling core, other modules, `config.php`, Composer files, `vendor/`, or core table structure changes. The extension installs only `whmcstofossbilling_*` tables.
2. **Keep live WHMCS access read-only.** Direct source connections use database-scoped SELECT-only grants and read-only transactions. No source writes, no source write test.
3. **No live mutating operation without explicit authorization and a specific run ID.** This covers import, resume, retry, repair, cancel, overwrite, and destructive cleanup. A previous `resume`/`retry`/`retry` intent and run ID were never confirmed; do not act on them. When the user wants live work, ask for the exact action and run ID first.
4. **No staging database and no create/alter/drop/truncate/delete of FOSSBilling core tables.** Uploaded SQL is parsed as data, never executed.
5. **Never put credentials in logs, AJAX responses, reports, or notes.** Credentials use AES-256-GCM with a module-specific HKDF context and are never returned by an API.
6. **Preserve the reviewed safety features** when making changes: two-source landing UI, direct DB path, SQL/GZIP/ZIP upload path, fresh-install/ownership checks, 32-hex batch idempotency, strict invalid-date default (opt-in substitution only), truthful scanned/expected progress, reduced-motion behavior, destructive-overwrite safeguards, encrypted SQL backup key/offset index handling, and rate-limit-safe run processing.
7. **Do not reset the integrity baseline or clean unrelated differences.** `config.php` is modified and `config.old.php` exists outside the module. Both are left untouched and their origin is unverified. The integrity checker correctly exits nonzero; report that honestly.

## Workflow for improvement or debugging

1. **Locate and read before acting.** Use the reference index below and the project docs. Identify whether the issue is in the source reader, planner, migration engine, destination adapter, API guard, UI, or test harness.
2. **Diagnose with disposable fixtures only.** Use the test suites with `--disposable-databases` / `--disposable-server`; never substitute production WHMCS credentials or the configured FOSSBilling business database.
3. **Change only module code.** Match existing naming, namespace (`Box\Mod\Whmcstofossbilling`), and style. Keep extension-owned tables and module-owned checksums consistent.
4. **Add a regression test for every fix.** Unit/security tests for guards, integration tests for native writes and repair, browser tests for UI behavior. Update the docs when behavior or counts change.
5. **Run the full verification matrix**, not just the test that touched your file. Expected current totals and exact commands are in `references/verification.md`.
6. **Report truthfully.** Say which checks passed, which cannot be established locally, and that no live import/session was used. Do not claim production readiness.
7. **Release only when asked and only after verification.** Follow `references/release-playbook.md` exactly, including the UTF-8 release-note step and asset digest check.

## Settled decisions (details in `references/decisions.md`)

| Area | Decision |
|---|---|
| Batch size | Adaptive 500 → hard max 2000; 5-second local work cap and 8 MiB source-page budget are hard guards; 15-second server drive budget, up to four batches per drive call, ~8-second browser pacing. |
| Rate limits | Each authenticated admin API request counts twice against FOSSBilling's 1000/hour limiter. A 429 stops polling, shows `Retry-After`, and never auto-retries. Core rate-limit settings are not changed. |
| Idempotency | 32-character lowercase hex request IDs; committed request replay returns the original cached result and cannot double-apply work. |
| Imported visibility | Hosting orders set `client_order.group_master=1`; eligible paid/canceled/refunded invoices set `invoice.approved=1`. Canceled/refunded remain admin-visible only. |
| Repair | Separate confirmed, run/source-scoped action with exact phrase `REPAIR IMPORTED VISIBILITY`; changes only those two flags on module-owned rows, refreshes checksums, idempotent. |
| Session timeout | Newest non-dry `running`/`paused` run reopens Step 4 with a passive Resume/Take-over prompt; no batches scheduled automatically. |
| Timezone | Administrator-selected IANA dropdown (default `America/Chicago`); no automatic timezone inspection, UTC assumption, or offset-to-IANA guessing. |
| Dates | Invalid calendar dates fail closed by default; opt-in uses `1900-01-01 00:00:00` with full disclosure. |
| Overwrite | Only the explicit checkbox + `OVERWRITE FOSSBILLING DATA` phrase allows destructive deletion, in the same transaction as the new paused run; table structures and reference records untouched. |

## Verification quick reference

Run from `modules/Whmcstofossbilling/` with `D:\Laragon\bin\php\php-8.4.28-Win32-vs17-x64\php.exe` (the shell's default PHP lacks PDO MySQL):

```powershell
$php = 'D:\Laragon\bin\php\php-8.4.28-Win32-vs17-x64\php.exe'
& $php tests/grants.php
& $php tests/unit.php
& $php tests/security.php
& $php tests/render.php
& $php tests/backup.php
& $php tests/integration.php --disposable-databases
$env:WM_MYSQLD = 'D:\Laragon\bin\mysql\mysql8.0.31\bin\mysqld.exe'
& $php tests/connection.php --disposable-server
$env:WM_PHP = $php
node tests/browser.cjs
& $php tests/integrity.php
```

Latest verified result: **585 PASS assertions** (77 grants, 106 unit, 44 security, 20 render, 68 backup, 246 integration, 20 connection, 4 browser) and syntax checks for 59 PHP + 2 JS files. Integrity reports 8100 files checked with the two known external differences.

## Living safely with unfinished runs

- `running` means explicitly opened/resumed for batch work; it can remain `running` while stalled after logout, a 429, a closed browser, or absolute session expiry. It does not prove another tab is active. Eight-second drive calls keep an idle timer fresh, but `security.session_lifespan` (default 7200) enforces absolute age.
- On page load, passive recovery shows **Unfinished import found**. Paused → **Resume this import** (no confirmation). Running → **Take over processing here** (requires confirmation). **Choose another source** escapes without writes. `requires_action` stays selectable but does not trigger the banner.
- Status watching is read-only. Never auto-resume, auto-retry, or auto-repair. The earlier active report `historical_native · running · 554332/1015997 records scanned 54%` was scanning progress, not an imported/complete run.

## Publication and truthfulness gates

- Publication is an explicitly authorized **experimental prerelease**, not production acceptance.
- Blocked natively: active/suspended/pending hosting, all domain orders, unpaid/draft invoices, compound tax, unknown currencies, unallocated/multi-invoice payments, opaque custom fields, attachments, open tickets. There is no global quarantine: an extension-owned flag is not consulted by core cron/order selectors.
- Not established: real-provider TLS, production-volume throughput, process-kill/concurrent-writer crash safety, authenticated real-admin browser E2E, live cutover reconciliation. Never report these as passing.
- The current GitHub release is `0.2` at https://github.com/thai916/WHMCStoFossbilling/releases/tag/0.2 with asset SHA-256 `BC14A81C9F2C557890467798EB0658ADF361406A22B71FF30F6B82E941C35869` (406688 bytes, 103 entries). Both the repository and the asset carry the AGPL-3.0 license.

## Reference index

Read the relevant file only when needed:

- `references/decisions.md` — full rationale and invariants behind adaptive batching, idempotency, visibility flags, repair, recovery, UI, dates, overwrite, backup index, and rate limiting. Read before changing behavior in any of those areas.
- `references/architecture.md` — layers, file map, extension tables, transaction/lock model, security model, and why `group_master`/`approved` are required by core selectors. Read before touching importers, schema, or API guards.
- `references/verification.md` — exact commands, environment, assertion counts, test boundaries, and the integrity caveat. Read before running or interpreting tests.
- `references/release-playbook.md` — rebuild/packaging exclusions, commit/push/retag/gh steps, UTF-8 release-note handling, and digest verification. Read before any release action.
- `references/limits-and-open-work.md` — production blockers, unestablished claims, and remaining work. Read before promising or implementing new native categories.
