# ADR-003: Database

Status: **Superseded**
Superseded: 2026-11-19 by [ADR-013](ADR-013-local-sqlite.md). The ledger is on-device SQLite, not hosted PostgreSQL.
Date: 2026-11-19
Decider: DEV

The text below is historical. Do not provision Postgres for the product.

## Context

The store has to give exact integer sums, real transactions, constraints that reject a bad category, a calendar date type, and row-level security or something equally hard to forget. The product is one person, with tests that use two users. The brief's candidate is PostgreSQL. Backups have to be a dump we can restore in Phase 6.

## Options

### A. PostgreSQL 16

One database. `bigint`, `date`, `timestamptz`, exclusion constraints, row-level security, `jsonb` for audit payloads.

### B. SQLite

One file, trivial to copy as a backup. Sufficient for a single writer on a laptop. Decimal extension or integer columns both possible.

### C. A hosted document database

Rejected as a serious store for a ledger that needs constraints and atomic audit writes. Named only so the trade is explicit: flexible documents, no multi-row transactions we can trust the way we trust Postgres, and money integrity left entirely to application code.

## Trade-offs

| | A. PostgreSQL | B. SQLite | C. Document store |
|---|---|---|---|
| Exact sums and constraints | Strong, including exclusion constraints for budget periods | Integers and checks work. Exclusion constraints and RLS do not. | Weak. Integrity is application code. |
| Concurrent requests | Many readers and writers | One writer. Two overlapping requests can lock or fail. | Vendor-specific |
| Two-user isolation tests | RLS is a backstop when a `WHERE` is missing | Easy to forget a `WHERE`, no RLS | Same, with a less strict schema |
| Backup | `pg_dump`, one artifact | Copy the file, if no connection is writing | Vendor export |
| Operational cost | A container | Almost none | A network dependency and a bill |
| Test fidelity | CI runs the same engine as production | Tests would miss Postgres-only constraints | Tests would mock the thing most likely to lose data |

SQLite is attractive because PO-Q4 might be "my laptop." It stops being attractive the moment two requests overlap (idempotency insert plus the duplicate-warning lock) or we want the database to refuse a cross-user category. Those are v1 requirements, not scale fantasies.

## Decision

Option A. PostgreSQL 16 in Docker for local development, CI, and the eventual host. Tests do not use SQLite. The application role is not a superuser.

## Consequences

- Phase 3 CI starts a Postgres 16 service. That is slightly slower than SQLite and much more honest.
- `btree_gist` is required for the budget exclusion constraint. It ships with the official Postgres image.
- A laptop deploy is still just this database in Compose. We do not keep a second schema dialect.

## Revisit when

The Product Owner forbids running Postgres, or the restore drill shows operational cost they will not carry. The fallback is SQLite with the RLS and exclusion constraints reimplemented as application locks, and a new ADR that lists what we lost.
