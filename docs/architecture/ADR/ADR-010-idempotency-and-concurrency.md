# ADR-010: Idempotency and optimistic concurrency

Status: **Proposed**
Date: 2026-11-19
Decider: DEV

## Context

Two different bugs look the same in a ledger: a retried request that creates a second dinner, and two open tabs where the slower save erases the faster one. The PRD separates them. FR-TXN-6 and AC-TXN-6 require an idempotency key and a replay of the original response. AC-TXN-3.3 requires a conflict error, not a silent overwrite. AC-TXN-5 adds a third mechanism, the duplicate warning, which must not be confused with either.

The duplicate warning is a product prompt. Idempotency is a protocol property. The version check is a concurrency property. Collapsing them into one unique index would forbid two legitimate ₹40 chai purchases on the same day.

## Options

### A. Idempotency-Key table, plus a `version` column on transactions and budgets

The key is per user, stored with a hash of the payload and the eventual response, for 24 hours. Updates carry the version the client read. A mismatch returns 409 and the current row, and does not write. The duplicate warning is a separate query, serialized with a transaction-scoped advisory lock so two parallel requests cannot both miss it.

### B. Last write wins

No version. The second tab overwrites the first. Simpler clients. Fails AC-TXN-3.3.

### C. A natural unique key instead of an idempotency key

Uniqueness on `(user, type, amount, category, date)` would stop double-submits and would also stop the second legitimate expense. The acceptance test says Save anyway creates both, and that the system must never auto-merge.

## Trade-offs

| | A. Key + version | B. Last write wins | C. Natural unique key |
|---|---|---|---|
| Double-click and timeout retry | One row, original response replayed | Two rows, unless the client is perfect | One row, and the second real expense is rejected |
| AC-TXN-3.3 two tabs | 409, first write intact | Silent data loss | Unrelated |
| AC-TXN-5.3 Save anyway | New key, `confirm_duplicate`, second row | Easy, and so is the accidental double | Forbidden by the constraint |
| Same key, different body | 409, no write (AC-TXN-6.3) | Not represented | Not represented |
| Client complexity | Must keep the key and the version | Lowest | Lowest, and wrong |
| Storage | A small table, purged after 24 hours | None | None |

The subtle case inside option A is the warning. If the first request stored a `409 POSSIBLE_DUPLICATE` under a key, Save anyway cannot reuse that key: replaying the key would return the warning forever, and changing the body would be `IDEMPOTENCY_KEY_REUSED`. Save anyway is a new user action with a new key. Timeout retries of the *same* click reuse the key. This is specified in the LLD so a later "small fix" does not break it.

## Decision

Option A.

- Header `Idempotency-Key` required on `POST /transactions`.
- Stored per user for 24 hours, then deleted by the daily command.
- Hash covers the transaction fields, not `confirm_duplicate`.
- `in_progress` versus `completed`, so two in-flight retries do not both insert.
- `version` on transactions and budgets, incremented by exactly one on update, matched in the `WHERE` clause.
- Category rename is the documented exception: no version, last write wins, both names audited. Categories are not money. See the API spec.
- Duplicate detection uses an advisory transaction lock and does not create a unique constraint.

## Consequences

- The frontend must not use a generic retry that generates a fresh key for POST.
- A 409 version conflict is a success of the design. Tests should fail if the handler retries the write automatically.
- The 24-hour window is a choice. A retry after that can duplicate. That is acceptable for a flaky network and not a substitute for the duplicate warning.

## Revisit when

We add a write API that is not user-driven (an import). Imports need their own idempotency, probably a hash of the source row, designed in that feature's ADR. Do not overload this key.
