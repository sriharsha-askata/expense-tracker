# Architecture decision records

Status: **Proposed** except where a record says Superseded.
Date: 2026-11-19

The Product Owner rejected the hosted server. Records marked Superseded are kept so the earlier reasoning is visible. Do not build them.

| ADR | Status | Decision |
|---|---|---|
| [ADR-001](ADR-001-frontend.md) | Superseded by ADR-012 | Web SPA |
| [ADR-002](ADR-002-backend.md) | Superseded by ADR-013 | FastAPI server |
| [ADR-003](ADR-003-database.md) | Superseded by ADR-013 | PostgreSQL |
| [ADR-004](ADR-004-authentication.md) | Superseded by ADR-015 and ADR-016 | Server sessions |
| [ADR-005](ADR-005-ai-integration.md) | Proposed, feature undecided | Confirm-only drafts, now on-device if ever built |
| [ADR-006](ADR-006-hosting-and-deployment.md) | Superseded by ADR-016 | Hosted Compose |
| [ADR-007](ADR-007-money-representation.md) | Proposed, still in force | Integer minor units and a currency code |
| [ADR-008](ADR-008-dates-and-timezones.md) | Proposed, still in force | Calendar date, default Asia/Kolkata |
| [ADR-009](ADR-009-audit-and-soft-delete.md) | Superseded by ADR-017 | Server-side purge |
| [ADR-010](ADR-010-idempotency-and-concurrency.md) | Proposed for local double-saves; cross-device rule is ADR-014 | No silent overwrite |
| [ADR-011](ADR-011-modular-monolith.md) | Superseded by ADR-013 | Server monolith |
| [ADR-012](ADR-012-mobile-client.md) | Proposed | Kotlin, Android-first |
| [ADR-013](ADR-013-local-sqlite.md) | Proposed | On-device SQLite, no server |
| [ADR-014](ADR-014-sync-format.md) | Proposed | One encrypted snapshot per sync |
| [ADR-015](ADR-015-encryption-and-keys.md) | Proposed | Passphrase-wrapped data key |
| [ADR-016](ADR-016-github-auth-and-sync.md) | Proposed | GitHub App device flow |
| [ADR-017](ADR-017-tombstones-and-git-history.md) | Proposed | Tombstones, then squash and key rotation |

Current set to build, once approved: ADR-007, ADR-008, ADR-012 through ADR-017, and the safety rules in ADR-005 if AI is later approved. ADR-010's local double-save rule still applies.
