# ADR-013: On-device SQLite, no application server

Status: **Proposed**
Date: 2026-11-19
Decider: DEV. The "no server" part is a Product Owner decision, not a DEV preference.
Supersedes: ADR-002, ADR-003, ADR-011

## Context

The Product Owner decided that the phone holds the ledger, saves are instant and offline, and no server of ours stores transactions or tokens. Many people may install the app. Each install keeps its own data. It is not a household product. Sync, if the user turns it on, copies an encrypted form of that data to a private GitHub repository the user owns (ADR-014, ADR-016).

The correctness rules that did not change: integer paise, currency code, calendar dates, no binary floats, budget states from integer comparisons, change history, no silent overwrite.

## Options

### A. SQLite on the device, SQLCipher, logical rows as the model

The UI reads and writes SQLite in the same action as the user's tap. A sync outbox row is written in that same database transaction. The network is not on the save path.

### B. Hosted Postgres, which ADR-003 chose

Rejected by the Product Owner. Recorded so the previous recommendation stays visible. It was the better operational database. It is not the product.

### C. Files on disk (one JSON file per entry) as the local source of truth

No SQL. Aggregates are written by hand. Crashes can tear a file. We would rebuild the budget and report queries that SQLite already does with `INTEGER` sums.

## Trade-offs

| | A. SQLite | B. Postgres server | C. Local JSON files |
|---|---|---|---|
| Instant offline save | Yes | No. The save waits on the network, or we build a local queue anyway. | Yes, if the write is atomic. |
| Exact sums | `INTEGER` sum. No `REAL` column is allowed in the schema. | `bigint` sum. | Application loops. Easy to use a float by mistake. |
| Crash safety | WAL mode, one transaction for the row and the outbox. | Strong | We would have to invent it. |
| Cross-user isolation | There is no shared database. Isolation is "this phone, this OS user, this app sandbox," plus encryption of anything that leaves. | RLS and a server. | Same as A, with weaker queries. |
| Who can read the data if they steal the phone | SQLCipher key in the Android Keystore. A powered-off or locked phone is the threat model we can actually meet. A phone that is unlocked and compromised is not. | The server operator, unless we had end-to-end encryption anyway. | Same problem, easier to copy. |

## Decision

Option A. SQLite is the source of truth on the device. Schema in `DATABASE_SCHEMA.md`. No `REAL` or `FLOAT` affinity for money. WAL mode. Foreign keys on. The GitHub copy is a replica for sync and backup, never a place the UI reads from directly.

There is no application server, no hosted database, no Redis, and no Docker Compose topology for user data. `docker-compose.yml` from the earlier draft is withdrawn.

## Consequences

- Two phones do not share a transaction until sync merges them. "Multi-user" means many isolated installs, not many people in one ledger.
- A test device and a second install are two ledgers unless both sync to the same repo and the same passphrase.
- Backups that do not go through GitHub are the CSV export and, later, a user-copied encrypted snapshot. The app must be useful with GitHub never configured.
- Category seed ids are UUIDv5 of `(type, name)` under a fixed namespace, so two fresh installs generate the same default categories and do not fork the ledger on first sync.

## Revisit when

The Product Owner wants a shared household ledger, or GitHub sync proves unworkable and they accept a server. Either change is a new product decision, not a patch on this ADR.
