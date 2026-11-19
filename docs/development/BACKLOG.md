# Backlog

Status: **Proposed**. Not filed as GitHub issues.
Last updated: 2026-11-19 by DEV, after the local-first decision.

The server-shaped milestone list that used to be here is withdrawn. Do not file those issues. Complexity is S/M/L, not dates. Nothing below starts until the Product Owner answers NQ-1 through NQ-6 (or defers them in writing) and the PM updates `docs/product/`.

## M1 — Discovery

| Slice | State |
|---|---|
| PM docs, first draft | In the repo, unchanged. They still describe a server. |
| DEV review, first draft | In `docs/discussions/001-mvp-discovery.md`, above the addendum. Historical. |
| Product Owner local-first direction | Recorded in the addendum. |
| PM revision of the PRD and acceptance tests | Not done. Blocked on the PM. |
| NQ-1 through NQ-6 | Blocked on the Product Owner. |

## M2 — Architecture revision

| Slice | State |
|---|---|
| HLD, LLD, SQLite schema, sync protocol, new ADRs | In this branch, Proposed. |
| Server ADRs marked Superseded | Done in this branch. |
| Product Owner acceptance of this revision | Not done. |

## M3 — Phone foundation

Depends on M2 acceptance. Android, unless NQ-1 flips the client to Flutter first.

| Slice | Complexity | Notes |
|---|---|---|
| Kotlin project, no product screens beyond a shell | M | Min SDK 26. |
| SQLCipher, migrations, schema with no `REAL` column | M | The money and date tests run on the JVM. |
| Save path: one transaction, `mutation_id`, history | M | Includes the five-minute duplicate prompt. |
| Passphrase, recovery code, keystore cache of the DEK | L | Copy for "lost key means lost data" is part of this slice, not polish. |
| CSV export, including the BOM if NQ-5 accepts it | S | Local file only. |

## M4 — Ledger behaviour

| Slice | Complexity | Notes |
|---|---|---|
| Categories, stable UUIDv5 seeds | M | Two fresh installs must not fork on first sync. |
| Transactions, edit, tombstone, restore | L | Totals ignore tombstones. |
| Budgets and the four states | L | Integer comparisons. Past months locked unless TQ-6 changes. |
| Reports and dashboard | M | `net` is the field. The on-screen word waits on TQ-5. |
| Refunds, card/EMI hint, repeat | S each | Only if NQ-5 says yes. Do not build them on a shrug. |

## M5 — GitHub sync

Depends on M4 save and merge being testable without a phone UI. This is the risky milestone.

| Slice | Complexity | Notes |
|---|---|---|
| Pure merge tests: two creates, conflicting edits, tombstone versus edit, resolution with two parents | L | No GitHub. These tests are the release bar. |
| GitHub App device flow, token in the keystore | L | Contents permission, one private repo. Stop if the scope cannot be limited. |
| Pull, decrypt, merge, one-commit push, retry on moved ref, backoff on 429 | L | Saves must already have returned. |
| Conflict screen blocks the push | M | |
| Compaction: orphan commit, new DEK, confirmed force-push of the data branch only | L | Not silent. |
| Delete repository and wipe local keys | S | Copy states the residual about GitHub backups. |

## M6 — AI

Not scheduled. NQ-6's recommendation is to leave it out of the first implementation. When it is explicitly wanted: on-device draft, confirm through the normal save, no write tool, provider outage does not affect the ledger.

## Not in this backlog

A server, Postgres, email login, Redis, household sharing, bank import, one GitHub request per transaction, silent merge, passphrase reset.

## Ordering

Money and merge tests before any sync UI. Sync before compaction. Compaction before anyone is told that delete removes Git history. The PM's product-doc update can happen in parallel with nothing else; it blocks code.
