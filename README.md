# skill-whmcstofossbill

**Version:** 1  
**Author:** Monlam Makhampa  
**License:** [AGPL-3.0](https://www.gnu.org/licenses/agpl-3.0.html)

An AI agent skill for understanding, improving, and debugging the **WHMCStoFossbilling** experimental WHMCS 8.x → FOSSBilling 0.8.9 migration module.

I am publishing this so that others who are interested in improving the WHMCStoFossbilling migration tool — or making a new one with AI tools — can hand their agent the settled requirements, safety boundaries, architecture, verification commands, and release playbook in one place.

**Migration tool repository:** https://github.com/thai916/WHMCStoFossbilling

## What it covers

- Hard scope: modify only `modules/Whmcstofossbilling/`; no core, Composer, vendor, config, or core-table changes.
- Read-only live WHMCS access; no live import, resume, retry, repair, or destructive action without explicit authorization and a specific run ID.
- Adaptive batching (500 → 2000 rows), rate-limit-safe drive calls, 32-hex idempotency, and truthful progress.
- Native visibility flags (`group_master`, `approved`), the confirmed `REPAIR IMPORTED VISIBILITY` action, and passive session-timeout recovery.
- Two-source UI, direct DB path, SQL/GZIP/ZIP upload path, strict invalid-date default, fresh-install and overwrite safeguards.
- Exact verification commands, the 604-assertion baseline, known integrity differences, and the `0.2` release playbook.
- Nullable historical hosting references and the explicit, disclosed linked-client currency policy for NULL invoice currency.
- Production blockers and claims that must never be reported as passing.

## Install

Copy `SKILL.md`, `LICENSE`, and `references/` into your agent's skills directory as `skill-whmcstofossbill/`, or install the ready-made `skill-whmcstofossbill.skill` package with your skill tooling.

## Contents

| Path | Purpose |
|---|---|
| `SKILL.md` | Orientation, boundaries, workflow, decision table, verification quick reference, publication gates, and reference index. |
| `LICENSE` | Full GNU Affero General Public License v3.0 text. |
| `references/decisions.md` | Rationale behind every settled decision. |
| `references/architecture.md` | Layers, file map, extension tables, transactions, security, and core-selector constraints. |
| `references/verification.md` | Commands, environment, assertion counts, test boundaries, integrity caveat. |
| `references/release-playbook.md` | Packaging, commit/push/retag/`gh` steps, UTF-8 note handling, digest verification. |
| `references/limits-and-open-work.md` | Blocked categories, unestablished claims, remaining implementation work. |
| `skill-whmcstofossbill.skill` | Portable packaged copy of the skill. |

## Important

The migration tool is an experimental development preview, not a production-complete migration. This skill deliberately refuses unauthorized live writes and does not claim that blocked or unverified behavior works. See the [migration tool repository](https://github.com/thai916/WHMCStoFossbilling) for the implementation and release notes.
