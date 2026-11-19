# Project Status

Last updated: 2026-11-19 (IST) · Updated by: DEV · Phase: **1–2, discovery and architecture**

This file is the shared working memory. Update it when a phase, decision, blocker, or next step changes. Do not mark work done unless it is actually done.

## Snapshot

| Item | State |
|---|---|
| Product proposal | Written by PM. Committed unchanged in `docs/product/` and `docs/discussions/PM_PROPOSAL.md`. |
| DEV review | Written in `docs/discussions/001-mvp-discovery.md`. PM has not replied. |
| Architecture | Drafted. Every architecture doc is **Status: Proposed**. Not approved. |
| Application code | None. `frontend/`, `backend/`, and `tests/` are placeholders. |
| GitHub issues | Not created, on purpose. Backlog is `docs/development/BACKLOG.md`. |
| CI | Not configured. No workflow has run on this branch. |
| Tests | None exist. None were run. |
| Docker Compose | Sketch only (`postgres` service). Not executed in this phase. |
| Production deploy | Not started. Hosting is an open Product Owner question (PO-Q4). |

## Phase gate

Phase 3 (foundation: auth, migrations, CI) does not start until:

1. The Product Owner answers, or explicitly defers, the questions in `docs/discussions/001-mvp-discovery.md`.
2. The PM responds to the open challenges in that note, especially the five that change scope or the meaning of a number.
3. The Product Owner approves the proposed architecture, or sends it back with changes.

A deferral should be written into `docs/discussions/` so the default in the ADR is the one we build.

## Decisions proposed by DEV

Owned technically, still Proposed until architecture approval. Detail and rejected alternatives are in `docs/architecture/ADR/`.

- Modular monolith. Vite + React SPA, FastAPI, PostgreSQL 16, SQLAlchemy 2 + Alembic. No Redis, no queue, no microservices.
- Money: `bigint` minor units + currency code. API uses decimal strings. JSON numbers are rejected.
- Dates: `date` for the transaction day, IANA timezone on the user (default `Asia/Kolkata`), `timestamptz` for system instants.
- Auth: Argon2id passwords, server-side sessions, HttpOnly cookies. Not JWT in browser storage. Not a managed identity vendor for v1.
- Tenancy: `user_id` on every private row, plus Postgres row-level security as a backstop.
- Writes: idempotency keys on create, optimistic `version` on update, 409 on conflict.
- Budgets: effective-dated rows, not one mutable amount. 80% and 100% states are computed from integer comparisons.
- AI (Phase 5, designed only): stateless draft endpoint, confirm through the normal create path, read-only query tools, prose discarded if it contains a number the tool did not return.

## Waiting on people

**Product Owner:** PO-Q1 through PO-Q6, plus the technical questions TQ-1 through TQ-6 in the discovery note.

**PM:** reply to the challenges in `docs/discussions/001-mvp-discovery.md`. In particular: the word "savings", email as a Phase 3 blocker, budget edits for past months, the exact-100% warning label, audit erasure, and whether the AI accuracy gate can block the ledger.

## Not done, and not claimed

- PM sign-off that every acceptance criterion is satisfiable. A traceability matrix is in `docs/architecture/AC_TRACEABILITY.md` for that review. Three rows are marked pending on purpose.
- Restore drill, security review, performance measurement, AI eval set.
- License choice.

## Next action

Wait for the Product Owner and the PM. Do not open GitHub issues and do not start Phase 3 from this document alone.

## Verification log

| Check | Result |
|---|---|
| PM files byte-identical to the copies supplied for this task | Confirmed with `cmp` on 2026-11-19 before commit. |
| Application tests | Not run. There is no application. |
| `docker compose` | Not run. Docker is not installed in this environment, so the file was not validated either. |
| GitHub issue creation | Not done, by instruction. |
