# ADR-009: Audit and soft delete

Status: **Proposed**
Date: 2026-11-19
Decider: DEV. Whether purge also deletes audit amounts is pending TQ-7. The recommendation below is what the schema specifies.

## Context

The user must be able to undo a delete for 30 days (A10, TXN-4), see who changed a transaction and what it was before (TXN-7, FR-AUD), and also delete the account and have the data gone within 30 days (AUTH-6). Deletes must disappear from totals immediately. The audit is not editable by the user. The PM also asked that we not run a queue for this; the only job is the daily purge.

These goals conflict if "immutable audit" means "keep the amount forever."

## Options

### A. Soft delete on the row, an audit table beside it, purge deletes both

`deleted_at` hides the row from every read model. Restore clears it inside 30 days. A daily definer function deletes expired rows and their audit events. The application role cannot update or delete audit events. Account deletion uses the same function.

### B. Event sourcing

The ledger is the fold of immutable events. Delete is an event. Restore is an event. "Purge" is a rewrite of history, which fights the model, or a cryptographic erasure problem.

### C. Hard delete, no history

Simplest, and it fails TXN-7 and the restore criterion.

## Trade-offs

| | A. Soft delete + audit, purged together | B. Event sourcing | C. Hard delete |
|---|---|---|---|
| Restore for 30 days | The row is still there | Possible, by folding | Impossible |
| Totals exclude the delete immediately | Filter `deleted_at is null` | Fold must understand the event | Gone |
| History of edits | `before` / `after` JSON in the audit table | The event stream is the history | None |
| User edits the audit | No grant, trigger rejects it | No, if we do it right | Not applicable |
| Account erasure and the 30-day purge | Delete the rows. Backups age out in 7 days. | Hard. The stream is the database. | Easy, and incomplete as a product |
| Complexity | One extra table and a daily command | A new way to write every feature, unjustified at this size | Lowest |
| Conflict with "immutable forever" | We give up forever, on purpose, at purge time | We pretend we did not | No audit to protect |

I will not recommend event sourcing for a single-user ledger with a handful of entity types. It makes every correction harder and does not answer erasure.

Keeping audit rows after the transaction is purged (a variant of A) was considered and not chosen. It makes "we deleted your data" false, because the amount and the note are still in `audit_events`. A maintenance log line that says an id was purged, without the amount, is enough to operate the system.

## Decision

Option A, with purge removing financial audit rows.

- Soft delete is an update of `deleted_at` and `version`. It is audited.
- Restore is the reverse, inside 30 days, and is audited.
- The trash list is `GET /transactions?deleted=only`. Restore is undiscoverable without it.
- While the row exists, audit events are insert-only for the application role.
- The purge function, owned by the migration role, deletes expired transactions and the audit events whose `entity_id` matches, in one transaction.
- Category renames, archives, and budget changes are audited too. The PRD required transaction and budget. Category is included because a rename changes how history reads (AC-CAT-2.3).
- Audit amounts are decimal strings inside `jsonb`.
- There is no user-facing edit of an audit row, and no admin UI over it.

Pending TQ-7: if the Product Owner wants the amounts kept after purge, the alternative is to null the `before` and `after` payloads and keep the action and the timestamp. That is a small change to the purge function. Keeping the payloads is the option I recommend against.

## Consequences

- "Immutable" in the API means "no endpoint and no application grant," not "retained for the life of the company."
- The daily command is the one in ADR-011. Missing a day delays purge; it does not corrupt totals, because soft-deleted rows are already excluded.
- A backup taken before purge still contains the row until that backup ages out (7 days). The privacy notice should say that. We will not build surgical backup editing.

## Revisit when

A legal review for a public launch says retention must be longer or shorter than 30 days, or TQ-7 comes back as "keep the amounts."
