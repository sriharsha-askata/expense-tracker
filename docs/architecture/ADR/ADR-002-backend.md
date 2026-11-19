# ADR-002: Backend

Status: **Proposed**
Date: 2026-11-19
Decider: DEV

## Context

The server must validate money without floating point, enforce authorization, expose a stable HTTP contract, and later call an AI provider without letting it write. The brief's candidate is Python and FastAPI, with SQLAlchemy. The team is two agents and a Product Owner, so framework ceremony is a cost.

## Options

### A. FastAPI, Pydantic v2, SQLAlchemy 2, Alembic

One Python service. Pydantic models reject a JSON number where a decimal string is required. Alembic owns schema changes. SQLAlchemy 2 is the data access layer, with SQL used directly for the aggregates.

### B. Django and Django REST framework

Batteries included: auth, admin, ORM, migrations. Decimal fields exist. The admin is a second UI over the ledger.

### C. Node (Nest or similar) so the client and server share types

One language across the stack. Money handling would be a TypeScript decimal library, and the database access would be a Node ORM or query builder.

## Trade-offs

| | A. FastAPI | B. Django | C. Node |
|---|---|---|---|
| Validation of "string, not number" | Natural in Pydantic | Possible, more custom | Possible, and the language's default number is a float, which is the hazard |
| Auth | We build sessions (ADR-004). More work, fewer surprises. | Mature auth. Also a large surface, including admin. | We build sessions, in the language where `number` is the footgun |
| SQL control for aggregates | High | High, if we drop to SQL | High |
| AI HTTP client | Ordinary Python HTTP | Same | Ordinary |
| Operational weight | Small | Larger than this product needs | A second ecosystem if the rest of the design stayed Python, or a rewrite of the candidate stack |
| Main risk | Hand-rolled auth must be tested, not assumed | Django admin over financial rows is a privacy surface we would have to lock down | Float leakage through JSON |

## Decision

Option A. One FastAPI process. Domain money code does not import FastAPI or SQLAlchemy, so it can be tested alone. Aggregates that must be exact are SQL `bigint` sums, not ORM objects added up in a loop.

Django's admin and auth are the strongest argument for B, and they are real. They are not worth a second conceptual model (ORM plus admin permissions) for a single-operator ledger whose authorization story is "the logged-in user, and row-level security." If Phase 3 auth tests prove we cannot get sessions right, reopening this ADR in favour of Django is preferable to inventing a third design.

## Consequences

- No Django admin. Support queries, if they are ever needed, are SQL run by the operator against a backup, not a live admin app.
- OpenAPI is generated from the code and checked against `API_SPECIFICATION.md`.
- Node is confined to the frontend toolchain (ADR-001).

## Revisit when

The isolation and session tests in Phase 3 cannot be made reliable, or a requirement arrives that Django already solves and FastAPI would make us rebuild (for example a complex permission matrix for households).
