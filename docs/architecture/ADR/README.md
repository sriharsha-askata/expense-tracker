# Architecture decision records

Status: **Proposed** until the Product Owner approves the architecture.
Date: 2026-11-19

Each record compares at least two options. "Proposed" means DEV has chosen and the choice is not yet approved. Superseding a record means a new ADR, not an edit that hides the old one.

| ADR | Decision |
|---|---|
| [ADR-001](ADR-001-frontend.md) | Vite + React + TypeScript SPA |
| [ADR-002](ADR-002-backend.md) | FastAPI monolith, SQLAlchemy 2, Alembic |
| [ADR-003](ADR-003-database.md) | PostgreSQL 16 |
| [ADR-004](ADR-004-authentication.md) | Server-side sessions, Argon2id |
| [ADR-005](ADR-005-ai-integration.md) | Backend draft and read-only tools; confirm is a normal create |
| [ADR-006](ADR-006-hosting-and-deployment.md) | One Docker Compose host. Where it runs is open. |
| [ADR-007](ADR-007-money-representation.md) | Integer minor units and a currency code |
| [ADR-008](ADR-008-dates-and-timezones.md) | Calendar `date` plus an IANA timezone on the user |
| [ADR-009](ADR-009-audit-and-soft-delete.md) | Soft delete, audit while the row lives, purge removes both |
| [ADR-010](ADR-010-idempotency-and-concurrency.md) | Idempotency keys and optimistic versions |
| [ADR-011](ADR-011-modular-monolith.md) | One service, one database, a daily command. No Redis. |
