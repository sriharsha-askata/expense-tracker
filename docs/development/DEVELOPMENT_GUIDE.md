# Development Guide

Status: **Proposed**
Audience: PM agent, DEV agent, Product Owner
Last updated: 2026-11-19

This is the canonical guide. A short pointer lives at `/DEVELOPMENT_GUIDE.md`.

## Current phase

Discovery and architecture only. The Product Owner has not approved the design. Until that approval is recorded in `docs/discussions/` and `PROJECT_STATUS.md`:

- Do not add application code under `frontend/`, `backend/`, or `tests/`.
- Do not add a GitHub Actions workflow that implies the app is tested.
- Do not create GitHub issues unless the Product Owner or the PM asks for them. The proposed breakdown is `docs/development/BACKLOG.md`.
- Do not merge this work to `main` without a review. Do not force-push or delete branches.

## How the two agents use the repo

GitHub is the source of truth. Personal transactions are not. They belong in the private database once one exists.

| Kind of change | Who owns it | Where it lands |
|---|---|---|
| What the product does | PM, with Product Owner authority | `docs/product/`, then a discussion note if it is contested |
| Whether a design is acceptable technically | DEV | `docs/architecture/`, ADRs |
| Disagreement | Both, in writing | `docs/discussions/NNN-*.md` with `[PM]`, `[DEV]`, `[DISCUSSION]`, `[DECISION]`, `[PRODUCT OWNER INPUT REQUIRED]` |
| Implementation, later | DEV | `feature/<issue>-<slug>` branch, pull request, tests |
| Acceptance, later | PM | Comment on the PR against the acceptance criteria |

Do not invent the other agent's reply. If the PM has not written, the decision is open.

Branch roles:

- `main` — accepted work only.
- `docs/<topic>` — discovery and design, as in this branch.
- `feature/<issue-id>-<description>` — implementation, after issues exist.
- `bugfix/<issue-id>-<description>` — defects.

Pull requests use `.github/pull_request_template.md`. A merge requires the tests that exist at that time to have been run, and a review. There are no tests yet, so a docs pull request says "not run" and does not pretend otherwise.

## Engineering rules

These apply from the first line of application code.

**Money.** No `float`, no `double`, no JavaScript `number` for amounts, no Postgres `float4` or `float8`. Store integer minor units and an ISO-4217 code. Parse input with a decimal type. Reject more than two decimal places for INR instead of rounding. Sum in integers. Format for humans only at the edge. The vectors in AC-TXN-1.2 and AC-TXN-1.3 are release blockers: `0.10 + 0.20 = 0.30`, and `100.555` is rejected.

**Dates.** The transaction day is a calendar `date`. "Today" and "this month" use the user's IANA timezone, default `Asia/Kolkata`. Changing timezone does not rewrite stored dates. `created_at`, `updated_at`, and audit times are UTC instants.

**Authorization.** Every private read and write is scoped to the authenticated user in the application and by row-level security. Another user's id returns 404. Tests must cover the cross-user cases in AC-AUTH-4 with two seeded users and overlapping amounts.

**Writes.** Creating a transaction requires an idempotency key. Updating a transaction or a budget requires the `version` the client read. A stale version returns 409 and the current row. It does not overwrite. The audit insert commits in the same database transaction as the change.

**AI.** No endpoint may turn model output into a stored transaction. The only write is the normal create endpoint, called after the user confirms the fields on screen. Query tools are read-only. If model prose contains a number the tool result does not, drop the prose. AI off, or AI down, must leave the rest of the app working.

**Logs and errors.** Structured logs with a request id, route template, status, and latency. Do not log amounts, notes, emails, passwords, cookies, authorization headers, or raw prompts. Error tracking, when added, does not receive request bodies.

**Schema.** Changes go through Alembic migrations. Editing a migration that has already been applied on a shared database is not allowed; add a new one. Test fixtures are synthetic.

**Secrets.** `.env` is git-ignored. `.env.example` holds placeholders only. The AI key, if one is ever configured, lives in the backend environment. Do not paste ledgers, dumps, or exports into issues or pull requests.

## Proposed application layout (not created yet)

```text
backend/app/
  api/            HTTP routers, request and response models
  domain/         money parsing, budget state, date windows (no database)
  services/       transactions, budgets, reports, export
  db/             SQLAlchemy models, session, RLS setup, migrations via Alembic
  maintenance/    daily purge and idempotency cleanup
  ai/             provider interface; implementation arrives in Phase 5

frontend/src/
  pages/          dashboard, transactions, budgets, reports, settings
  api/            client that reuses an idempotency key on retry and never sends amount as a JSON number
```

`domain/` must be importable without a database so the money and budget tests do not need Postgres. Integration tests hit a real Postgres, the same major version as production (16).

## Local environment (Phase 3, not now)

Planned, not verified in this phase:

1. Copy `.env.example` to `.env` and replace the placeholders.
2. `docker compose up` for Postgres. The API and web services are not in the compose file yet.
3. Apply migrations and run the API.
4. Serve the SPA through a dev proxy so the browser sees one origin and the session cookie works.

Do not describe that sequence as working until someone has run it and recorded the result in the pull request.

## Testing strategy (when code exists)

| Layer | What it proves | Tooling to use |
|---|---|---|
| Domain unit | Money parse and reject, minor-unit sums, budget state edges, month windows, CSV formula escaping | pytest |
| Property | Random 2-decimal amounts, dashboard total equals the integer sum, including 10k values | Hypothesis, or a seeded generator if Hypothesis is declined in an ADR |
| API integration | Auth, isolation, idempotency, 409 on stale version, soft delete, RLS with `app.user_id` unset | pytest + Postgres service |
| Contract | Error codes and the decimal-string amount type | Same API tests, against the documented examples |
| UI unit | Indian grouping input normalizes to a decimal string before POST | Vitest |
| End to end | CUJ-1, 2, 4, 5, 6, 8 | Playwright, against the compose stack |
| AI eval (Phase 5) | Labelled phrases, zero unconfirmed writes, answer numbers subset of tool output | Fixture file of synthetic phrases, no live ledger |

A pull request that changes money, dates, authorization, or writes does not merge without the matching tests run and the command pasted into the PR. "Should pass" is not a result.

Performance: review the query count for the dashboard in code review (set-based aggregates, no query per category). Do not add a flaky p95 gate on shared runners until the Product Owner's host is known and a baseline has been recorded. Target remains the PRD figure: dashboard under 1 second and list under 500 milliseconds at 10k transactions for one user, add under 300 milliseconds excluding AI.

## CI (Phase 3, not now)

The first workflow should:

- Run the Python tests against Postgres 16.
- Run the frontend unit tests once a frontend exists.
- Fail on a secret scan.
- Not deploy.

No workflow file is in this branch. Absence of a green check is expected.

## Observability and operations (later phases)

- Health endpoint with no dependency details beyond ok/not ok.
- Request id returned to the client and stored on the audit row.
- Daily maintenance command: purge transactions whose `deleted_at` is older than 30 days, purge idempotency keys older than 24 hours, purge expired login-attempt rows, finish account deletions whose request is older than 30 days.
- One API replica in MVP, so the daily command can be a compose service on a clock. If we ever run two replicas, the command stays a single external timer, not an in-process scheduler on every replica.
- Backups: encrypted `pg_dump`, retain 7 days, restore procedure written in Phase 6 and executed once before v1.0. Not designed as a second application.

## Definition of done, per feature, once implementation starts

1. The story and acceptance criteria are in `docs/product/` or the issue.
2. Schema or API changes are in the same pull request as the docs that describe them.
3. Automated tests for the new criteria were run.
4. Logs from the new path do not contain the fixtures' amounts or notes.
5. `PROJECT_STATUS.md` moves the slice from in progress to done only after the above.

## Working agreement on disagreements

If a product change breaks an ADR, write a new ADR or supersede the old one. Do not quietly diverge in code. If a technical constraint makes an acceptance criterion unsatisfiable, stop and update `docs/discussions/` rather than half-implementing the criterion.
