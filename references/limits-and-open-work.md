# Production limits, unestablished claims, and remaining work

Use this before implementing a new native category or telling the user something is supported. Version `0.2` is an experimental preview; the originally requested complete migration has **not** been accepted.

## Hard blockers (still blocked natively)

- Active, suspended, or pending hosting services. Only terminated/cancelled/fraud records become native canceled orders, with exact source status retained in metadata.
- All native domain orders.
- Unpaid/draft invoices: core `findUnpaidOlderThan()` does not check approval, so approval would not isolate them from expiry deletion.
- Compound taxes, discount variants, unknown/ambiguous historical currencies without explicit source currency (except the explicit, disclosed linked-client policy for NULL/empty invoice currency), unallocated/multi-invoice payments, simultaneous in/out credit movements, complete outstanding-debt reconstruction, and historical FX/income semantics.
- Opaque custom-field values, opaque product/module configuration slots (may contain secrets), unknown/custom module tables, binary/oversized/invalid-UTF-8 values, attachments (metadata only; no verified file transfer).
- Open-ticket workflows, configurable status/author mapping, chronological staging of out-of-order replies. Only closed tickets and redacted plain-text email history are published.
- Active configurable options, independently fixed non-base-currency native prices, reviewed module-specific package-slot mappings, ambiguous server groups, contact/shared-user ownership. Missing historical plan/server references are nullable by design; never invent them.

## Structural reasons

- No global quarantine exists. Core cron/order/domain selectors do not consult an extension-owned flag, and a before-cron listener cannot be assumed to run first. Do not invent a flag or claim hook-based suppression.
- A multi-request live scan is not a consistent source snapshot; row deletions, count mismatches, and updates behind a completed cursor need a new reviewed run.
- Destination strict SQL mode and UTC configuration are required and checked, never changed by the extension.
- Native client credentials are random, discarded, and hashed; no WHMCS password is extracted and no welcome/reset email is sent. Reactivation is a separate administrator task.
- Products stay disabled/manual after import. Historical publication requires a private, otherwise idle destination.

## Not established locally (do not report as passing)

- Credentialed live WHMCS connection or production migration.
- Successful verified real-provider remote TLS, real firewall outage, MariaDB compatibility, actual query/network deadlines.
- Production-volume load, memory, request-time, or multi-gigabyte upload/throughput benchmarks. The 64 GiB / 128 GiB figures are ceilings.
- Process kill, database disconnect mid-transaction, concurrent out-of-band writers, simultaneous-process crash behavior, frozen source snapshots at cutover.
- Authenticated real-admin activation/menu/API end-to-end browser test. Anonymous local API correctly returns `Authentication Failed`/201 before source access, but the reason a logged-in session was rejected remains unverified.
- Real attachment transfer, arbitrary third-party hook safety, live cutover reconciliation, complete legal billing snapshots.
- Origin of the two external integrity differences (`config.php` modified, `config.old.php` created). Leave them intact; do not claim a clean baseline.

## Remaining implementation work

1. Extend the tested supported-subset isolation to the blocked categories above; if safe isolation requires changes outside the extension, stop and get explicit approval.
2. Decide a stable source snapshot/cutover strategy.
3. Build side-by-side mapping for every implemented native category, secure synthetic previews, selective native update/link-existing conflict workflow, gateway/hosting module mapping, post-migration activation checklist, complete archive search/revisions, and credential rotation workflow.
4. Reconcile outstanding debt, credit movements, paid/unpaid/canceled/refunded totals, and taxes without manufacturing ledger entries.
5. Add authenticated installation/browser tests, MariaDB and production-volume tests, crash/concurrency tests, real attachment transfer, and resolver for the integrity differences.

## Publication gate

Public repository `thai916/Whmcstofossbilling` and tag/release `0.2` are explicitly authorized as experimental prereleases with incomplete-scope warnings. That authorization does not grant production acceptance, live WHMCS access, or live mutating operations. Any live resume/retry/repair still needs explicit authorization and a specific run ID.
