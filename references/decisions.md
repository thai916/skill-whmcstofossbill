# Settled decisions and invariants

Read this before changing behavior in any area listed here. Each decision was made deliberately; re-opening one requires explicit user agreement, not a local judgement call.

## Adaptive batching (`Migration/BatchSizer.php`, `Migration/Engine.php`, `Service.php`, `Source/Reader.php`, `Source/DumpReader.php`, `assets/wizard.js`)

- Hard bounds: `MIN_ROWS = 1`, `MAX_ROWS = 2000`, `START_ROWS = 500`. Requests outside 1–2000 are rejected.
- `initial()` inherits a durable decision from the previous result using `next_limit`, `batch_limit`, `requested_limit` (in that order) so sizing survives a drive call boundary.
- `next()` grows toward 2000 when elapsed time is fast (< 2.5 s) and source-page bytes are small (< 4 MiB); backs off 25% near the 4.5 s warning or 6 MiB sizing threshold; halves on an empty batch; stays stable on mixed measurements.
- Non-negotiable guards that sizing must not bypass: 5-second local work cap per batch, 8 MiB hard source-page byte budget, source-level advisory lock, durable cursor, 32-hex batch idempotency, and ~8-second browser pacing.
- A drive request runs up to four idempotent batches within a **15-second server budget** (raised from 9000 ms). Drive results expose `batch_limit` and `next_limit`. Browser sends `limit = 500` and `budget_ms = 15000` by default.
- Readers return `bytes` metrics; both live and SQL readers enforce the 8 MiB page cap. Earlier hard caps of 100 rows were raised to 2000 only behind the adaptive policy.
- No production throughput promise: these are local, disposable-fixture measurements.

## Rate limiting

- FOSSBilling counts each authenticated admin API request **twice** against the shared 1000/hour limiter — roughly 500 calls/hour. This is why Step 4 drives several batches per call and paces calls ~8 seconds apart, instead of polling after every batch.
- A 429 stops polling, displays the server's `Retry-After`, and never retries automatically. Core rate-limit configuration is never changed.
- Status watching uses read-only requests about every 10 seconds; hidden tabs skip them.

## Idempotency and replay

- Batch and drive request IDs must be 32 lowercase hex characters. A Web Crypto fallback covers environments without `crypto.randomUUID` (for example `http://fossbill.test`).
- Replaying the same request returns the original committed result from extension-owned storage. It cannot double-apply work. This guarantees record-level idempotency, not exactly-once HTTP delivery.
- Cross-run identity mappings and unchanged-record skipping are preserved. Changed native records are **not** automatically updated; there is no selective native-update/link-existing UI.

## Imported record visibility

- `Destination/HostingImporter` sets `client_order.group_master = 1`. Core `Order\Service::getSearchQuery()` defaults to `hide_addons = 1` and filters `co.group_master = 1`; without the flag imported services are hidden from core lists.
- `Destination/InvoiceImporter` sets `invoice.approved = 1` for eligible `paid`/`canceled`/`refunded` invoices. Core `Invoice\Api\Client::get_list()` forces `approved = true`. Only non-unpaid statuses are eligible, so approval grants visibility without creating a payable state. Unpaid/draft invoices remain blocked.
- Canceled/refunded invoices are administrator-visible only: the core client portal has Paid/Unpaid tabs, so no portal status is fabricated.

## Visibility repair (`Destination/VisibilityRepair.php`, `Service::repairVisibility()`, `Api\Admin::run_repair_visibility()`)

- Confirmation phrase must be exactly `REPAIR IMPORTED VISIBILITY`; wrong phrase changes nothing.
- Scoped to a selected run and optional source; uses the same admin POST/CSRF/execute checks as other mutating endpoints.
- Verifies module-owned identity rows, then updates only `client_order.group_master` and `invoice.approved` in a destination transaction.
- Refreshes `destination_checksum` so incremental runs do not see false drift. Idempotent: a repeat reports zero changes.
- Never re-imports, deletes, renumbers, changes statuses, sends notifications, or touches core schema.

## Session-timeout recovery (`assets/wizard.js`, template recovery banner)

- On page load, the newest non-dry `running`/`paused` run from dashboard data opens Step 4 directly with **Unfinished import found**. It hides the source choice and shows navigation/limits.
- Passive only: no resume, retry, repair, or drive request is scheduled until the user acts. Status watching remains read-only.
- `paused` → **Resume this import**, no confirmation. `running` → **Take over processing here**, with a browser confirmation explaining another tab may still be driving; declining leaves the banner and run untouched.
- **Choose another source** clears the banner, returns to the two-source landing page, and writes nothing.
- `requires_action` runs remain selectable in Step 4 but do not trigger the banner.
- `running` is ambiguous by design: it can persist after logout, a 429, a closed browser, or absolute session expiry. Do not treat it as proof of an active worker.

## Source security and timezone

- Direct source account: database-wide SELECT only; global SELECT, write/EXECUTE/FILE/grant-option/role privileges are refused. No source write is attempted to test permissions.
- Three source modes: standard MySQL without TLS (default, explicitly acknowledged), verified TLS, explicitly acknowledged unverified TLS. No automatic downgrade; credential-free TLS preflight and actual PDO cipher checks prevent silent downgrade.
- WHMCS timezone is administrator-selected from an IANA dropdown defaulting to `America/Chicago`. No MySQL timezone/offset SELECT, remote PHP execution, or offset-to-IANA inference. Empty selection stays Unverified and blocks timezone-sensitive native runs. Run plans freeze security mode and timezone.
- Invalid calendar dates (for example `0000-00-00`) fail closed by default. Opt-in uses `1900-01-01 00:00:00`, with the exact source value archived and every substitution disclosed in Date substitutions, JSON/CSV reports. DST gaps/ambiguous times are never substituted.

## UI and workflow

- Two-source landing: **Connect to WHMCS database** or **Upload WHMCS backup**. Fields appear only after selection; both paths share mapping, preview, authorization, archive, and native adapters.
- Fresh-install/ownership checks replace mandatory destination backup references. Unrelated data blocks authorization by default.
- Destructive overwrite requires a per-table checkbox plus `OVERWRITE FOSSBILLING DATA`; deletion of checked business rows happens in the same transaction as the new paused run, with stale identity mappings cleared and table structures/reference records untouched.
- Truthful progress: the top bar measures scanned/expected, never migration completion. Preparation shows real completed-table percentages; no invented time-based progress.
- Smooth scrolling respects `prefers-reduced-motion`; reduced motion disables smooth scroll, stripes, and transitions.
- Archive text renders with `textContent`, never HTML. No browser storage for upload tokens.

## SQL/GZIP/ZIP backup path

- One `.sql`, `.sql.gz`/`.gz`, or `.zip`/`.zip64` with one SQL member. Internal authenticated blocks provide in-page pause/continue and idempotent upload retries.
- SQL is tokenized as data, never executed. No staging database, shell command, Composer dependency, or core schema change.
- A bounded, checkpointed external sort builds an encrypted `(key, offset)` index so INSERT rows need not be in primary-key order. Duplicate/invalid keys fail closed. The exact MySQL 5.7 RocksDB capability-probe comments are accepted; altered probes and other executable comments are rejected.
- Backups/checkpoints are authenticated and encrypted at rest with ownership, path traversal, CRC, size, and syntax checks. Files prepared by earlier 0.2 builds must be re-uploaded to gain the final index.
- Limits 64 GiB upload / 128 GiB expansion are ceilings, not measured throughput.

## Publication decision

Committing, pushing, force-retagging `0.2`, replacing the GitHub asset, and publishing notes are explicitly authorized **for an experimental prerelease only**. Production acceptance criteria remain unmet. See `release-playbook.md`.
