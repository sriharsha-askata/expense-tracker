# Architecture decision records

Date: 2026-11-19

Records marked Superseded are the withdrawn server design. Do not build them.

**Accepted** means the Product Owner decided it, or adopted a proposal by name. **Proposed** means it is still DEV's recommendation inside that decision.

| ADR | Status | Decision |
|---|---|---|
| [ADR-001](ADR-001-frontend.md) | Superseded by ADR-012 | Web SPA |
| [ADR-002](ADR-002-backend.md) | Superseded by ADR-013 | FastAPI server |
| [ADR-003](ADR-003-database.md) | Superseded by ADR-013 | PostgreSQL |
| [ADR-004](ADR-004-authentication.md) | Superseded by ADR-015 and ADR-016 | Server sessions |
| [ADR-005](ADR-005-ai-integration.md) | Accepted as deferred | Confirm-only, not in the first build |
| [ADR-006](ADR-006-hosting-and-deployment.md) | Superseded by ADR-016 and ADR-018 | Hosted Compose |
| [ADR-007](ADR-007-money-representation.md) | Accepted | Integer minor units; `bigint` on the Expo client |
| [ADR-008](ADR-008-dates-and-timezones.md) | Accepted | Calendar date, Asia/Kolkata, calendar year plus Indian FY preset |
| [ADR-009](ADR-009-audit-and-soft-delete.md) | Superseded by ADR-017 | Server-side purge |
| [ADR-010](ADR-010-idempotency-and-concurrency.md) | Accepted | No silent overwrite, no double-save |
| [ADR-011](ADR-011-modular-monolith.md) | Superseded by ADR-013 | Server monolith |
| [ADR-012](ADR-012-mobile-client.md) | Accepted | Expo, React Native, TypeScript |
| [ADR-013](ADR-013-local-sqlite.md) | Accepted | On-device SQLite, no server |
| [ADR-014](ADR-014-sync-format.md) | Proposed | One encrypted snapshot per sync |
| [ADR-015](ADR-015-encryption-and-keys.md) | Accepted | Passphrase and recovery code, no reset |
| [ADR-016](ADR-016-github-auth-and-sync.md) | Accepted | GitHub optional; device flow when used |
| [ADR-017](ADR-017-tombstones-and-git-history.md) | Accepted | Tombstones, then squash and key rotation |
| [ADR-018](ADR-018-cloud-ci-and-builds.md) | Accepted | Actions for tests, EAS for the APK |

Build from the Accepted rows plus ADR-014 unless the Product Owner rejects that file layout when he starts Phase 3.
