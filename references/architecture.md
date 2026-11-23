# Architecture and safety model

## Target and namespace

FOSSBilling 0.8.9, PHP 8.3+ (local runtime 8.4.28), mixed Doctrine ORM/DBAL and legacy RedBean. Namespace `Box\Mod\Whmcstofossbilling`. Composer already maps `Box\Mod\` to `modules/`; no autoloader or dependency changes. Installation directory must be `modules/Whmcstofossbilling/` because FOSSBilling lowercases module IDs and applies `ucfirst()` for directory/class lookup.

## Layers

| Directory | Responsibility |
|---|---|
| `Controller/` | Authorized admin page and allowlisted JS/CSS delivery |
| `Api/` | Identity, endpoint permissions, POST, CSRF, origin checks, sanitized errors |
| `Source/` | Connection-security policy, sanitized diagnostics/TLS preflight, selected timezone policy, privilege verification, fixed reads, keyset extraction |
| `Mapping/` | Versioned field-disposition registry and allowlisted text transforms |
| `Security/` | Safe errors, conservative sensitive-data policy, AES-256-GCM credential vault |
| `Installation/` | Idempotent extension-only DDL |
| `Destination/` | Validated aggregates, products/pricing/servers, inactive hosting, eligible invoices/payments/credits, closed history, scoped visibility repair |
| `Archive/` | Redacted structured evidence, historical dates, relationship hints, prior revisions |
| `Migration/` | Plan, durable cursors, serialization, adaptive batch sizing, bounded batches, retries |
| `Finance/` | Decimal arithmetic and resumable extracted-baseline/native assessment (not debt/cutover reconciliation) |
| `Report/` | Streaming safe count/disposition exports |

Key entry points: `Service.php`, `Api/Admin.php` (all endpoints guarded), `Migration/Engine.php` (`batch()`/`result()`), `Migration/BatchSizer.php`, `Migration/Planner.php`, `Migration/Dependencies.php`, `Source/Reader.php`, `Source/DumpReader.php`, `Destination/HostingImporter.php`, `Destination/InvoiceImporter.php`, `Destination/VisibilityRepair.php`, `assets/wizard.js`, `templates/admin/mod_whmcstofossbilling_index.html.twig`. See `docs/FILES.md` for the complete file list.

## Extension-owned tables

All names begin `whmcstofossbilling_`: `source`, `analysis`, `run`, `cursor`, `identity`, `archive`, `archive_revision`, `attempt`, `batch`, `price`, `archive_link`, `legacy_reference`, `snapshot`, `snapshot_link`, `credit_state`, `credit_result`, `assessment`, `reconcile_state`, `reconcile_total`. Installation renames legacy `whmcsmigrator_*` tables without touching core tables. `uninstall()` intentionally retains records. No core table definition is ever changed.

## Transactions and recovery

- Source reads happen before destination write transactions. Each logical record/aggregate commits archive, native aggregate (if supported), identity, attempt, and cursor together.
- A native savepoint preserves a successfully extracted archive when the native adapter fails; native children created by the failed aggregate roll back and the record stays failed.
- Source-level `GET_LOCK` serializes work per source. Same lock for batch and control operations; separate sources progress independently.
- Committed per-record transactions survive a killed request; a retry resumes from the durable cursor. Replay may process subsequent records if the final marker was not committed, but cannot duplicate committed identities.
- Pause/cancel cannot undo committed work; an in-flight request may finish before the control gets the lock. Failed-record retries: five per request, at most five attempts per record.
- One unfinished non-preview run per source. Per-run snapshots protect projection/reconciliation inputs from later runs overwriting the latest searchable archive.
- Historical mode publishes at most five credit aggregates/request, then assesses at most 50 financial records/request. Invoices/credit aggregates cap at 200 lines/movements; larger cases fail explicitly.

## Why the native visibility flags exist

- Core `Order\Service::getSearchQuery()` applies `co.group_master = 1` when `hide_addons` is set (the default), so imported hosting orders need `group_master = 1` to appear.
- Core `Invoice\Api\Client::get_list()` forces `approved = true`, so eligible historical invoices need `approved = 1` to appear in client/admin lists.
- `VisibilityRepair` fixes older imports by setting exactly those two flags on verified module-owned rows, scoped by run/source, and refreshing checksums. It is not a general data repair tool.

## Security model

- Session-bound POST/CSRF is required for every endpoint, even when global CSRF protection is off. Origin/cross-origin redirects are refused. API URLs are origin-relative and preserve installation subdirectories.
- Endpoint-level permissions: view/configure/execute/archive/export. API token possession alone is insufficient.
- Credentials are encrypted with AES-256-GCM derived from the installation secret via a module-specific HKDF context; never returned by an API. No plaintext credential files.
- Identifiers are validated separately from prepared values; identifiers come from allowlists/patterns, never raw input. Views are excluded. Only the WHMCS version setting is queried wholesale; generic configuration is excluded.
- Source extraction probes lengths without selecting secret columns, enforces oversized-row/field guards in the SELECT, and never substitutes fabricated null archive data. Oversized/binary/invalid UTF-8 records fail explicitly.
- InnoDB/repeatable-read permits a stable length/value probe; READ COMMITTED/non-snapshot engines use one bounded row per page; other isolation levels are refused. Destination strict SQL mode is required but never changed.
- Archive free text uses `textContent`; heuristic redaction is not a guarantee that every secret is removed. Archives are confidential.

## Automation blocker (do not "fix" with a flag)

Core `modules/Cron/Service.php` runs credit application, paid-item activation, invoice generation/reminders, suspension/cancellation, ticket auto-close, and email. `modules/Order/Repository/OrderRepository.php` selects expired active orders without a migration exclusion. `modules/Servicedomain/Service.php::onBeforeAdminCronRun()` attempts registrar sync. An extension-owned quarantine flag is not consulted by these selectors, and a before-cron listener cannot be assumed to run first. Do not claim hook-based suppression.

The safe reviewed subset relies on records being non-operational by construction: inactive hosting becomes native canceled orders, eligible invoices are already paid/canceled/refunded with custom/void/executed/charged items, native historical transactions are processed with no gateway, credit aggregates require exact source reconciliation with no actionable destination unpaid invoice, and only closed tickets/email history are published. Unpaid invoices remain blocked because `findUnpaidOlderThan()` ignores approval. Historical publication requires a private, otherwise idle destination; enabling products, entering credentials, or reactivating orders is a separate administrator decision.
