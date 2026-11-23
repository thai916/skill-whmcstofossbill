# Verification reference

## Environment

- Run from `D:\Laragon\www\FOSSBill\modules\Whmcstofossbilling\`.
- PHP: `D:\Laragon\bin\php\php-8.4.28-Win32-vs17-x64\php.exe`. The shell default PHP is a different install without PDO MySQL.
- MySQL fixture server: `D:\Laragon\bin\mysql\mysql8.0.31\bin\mysqld.exe` via `$env:WM_MYSQLD`.
- Browser: `WM_PHP` points at the PHP executable; Playwright is development-only under `.local/browser/`. `WM_PLAYWRIGHT_PATH` can override the module path.

## Commands and latest counts

```powershell
$php = 'D:\Laragon\bin\php\php-8.4.28-Win32-vs17-x64\php.exe'
& $php tests/grants.php                              # 77 PASS
& $php tests/unit.php                                # 106 PASS
& $php tests/security.php                            # 44 PASS
& $php tests/render.php                              # 20 PASS
& $php tests/backup.php                              # 68 PASS
& $php tests/integration.php --disposable-databases  # 265 PASS
$env:WM_MYSQLD = 'D:\Laragon\bin\mysql\mysql8.0.31\bin\mysqld.exe'
& $php tests/connection.php --disposable-server      # 20 PASS
$env:WM_PHP = $php
node tests/browser.cjs                               # 4 PASS lines (multi-assertion)
& $php tests/integrity.php                           # nonzero expected: 2 external diffs
```

Total: **604 PASS lines** (600 PHP + 4 browser). Syntax: 59 PHP files and 2 JS files (`assets/wizard.js`, `tests/browser.cjs`) pass lint. Run syntax checks with `php -l` over all non-`.local` PHP files and `node --check` for both JS files.

## What the suites actually cover

- `grants.php`: cPanel escaped underscores, quoted/doubled identifiers, host forms, MySQL/ANSI/MariaDB syntax, scope/partial revoke, forbidden privileges, privacy of diagnostics; controlled PDO fixtures for `SHOW GRANTS FOR CURRENT_USER()` preference.
- `unit.php`: validation, allowlists, encrypted credential tamper rejection, decimal precision, historical dates, sensitive-data classification, CSV safety, mapping restrictions, `BatchSizer` bounds/start/growth/backoff/inheritance and reader caps.
- `security.php`: every endpoint requires POST, CSRF, correct identity, and its permission; cross-origin refusal; sanitized exceptions; `run_repair_visibility` guard coverage.
- `render.php`: Twig/native loader discovery, renamed branding, author/vibe disclosure, two-source landing, fresh-install fields, reduced-motion and connection controls.
- `backup.php`: real SQL/GZIP/ZIP/ZIP64 bytes, encrypted replay, CRC, RocksDB probe comments, out-of-order INSERT external sort, 131,075-row two-run merge, compressed multi-block preparation, sparse cursor seek, traversal/corruption rejection.
- `integration.php --disposable-databases`: installation/rename, connection policy, dry-run isolation, replay/pause/resume, native catalog/clients/history, historical invoices/payments/credits, nullable missing hosting plan/server references, opt-in linked-client invoice currency with original-NULL retention, actual core selectors exercised on synthetic records, native visibility flags and repair, destructive-overwrite safeguards, adaptive sizing end to end. Report regressions verify every inference appears in the full JSON/CSV report while exception-only exports remain failed-record-only.
- `connection.php --disposable-server`: its own fixture datadir/CA/cert; standard vs verified/unverified TLS, trust/name failures without downgrade, sanitized auth/access errors, query-log read-only audit (`SELECT`/`SHOW`, `READ ONLY` start, rollback).
- `browser.cjs`: synthetic API plus real Twig/JS/CSS; security modes, timezone, IP controls, mapping, preview/start, 32-hex fallback, invalid-date and missing-invoice-currency policies, progress, responsive layouts, repair guards and success, startup recovery for paused/running runs, take-over confirmation, source-choice escape, and the real unauthenticated local login gate.

## Test boundaries — never overstate

- Integration creates only randomly named `wm_test_source_*` / `wm_test_destination_*` databases and a scoped fixture account; it refuses a non-local destination and never writes business records into the configured FOSSBilling database. If DBA privileges are unavailable, fail rather than substituting credentials.
- Browser responses are synthetic. No authenticated real-admin activation/menu/API browser test has been done.
- No live WHMCS credentials, login, extraction, or import is used. Administrator-requested TCP/TLS diagnostics sent no username/password/SQL.
- Not established: real-provider TLS, firewall outages, process-kill/concurrent-writer crash tests, production-volume load/memory/request-time, MariaDB compatibility, real attachment transfer, arbitrary third-party hook safety, live cutover reconciliation.

## Integrity caveat

`tests/integrity.php` checks 8100 baseline files, distinguishes approved `data/cache/` and `data/log/` runtime changes, and reports unauthorized changes. It currently returns nonzero because `config.php` is modified and `config.old.php` exists outside the module. Leave both untouched; do not reset the baseline, delete them, or claim a clean result. Without the private `.local/core-baseline.json`, the checker refuses to claim unchanged files. FOSSBilling itself is not a Git repository, so Git-based verification is unavailable.
