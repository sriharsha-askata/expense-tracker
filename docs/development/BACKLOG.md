# Backlog

Status: **Proposed**. Not filed as GitHub issues.
Last updated: 2026-11-19 by DEV.

This is the issue breakdown the roadmap asked for, with complexity instead of dates. Complexity is relative: **S** is a single module with a narrow test, **M** crosses API and database, **L** crosses client, API, and a risky rule. Nothing here is approved to build. Phase 3 does not start until the architecture is approved and the open product questions in `docs/discussions/001-mvp-discovery.md` are answered or explicitly deferred.

Do not create these in GitHub until the PM and the Product Owner say to.

## M1 — Discovery

| Slice | Complexity | Notes |
|---|---|---|
| PM docs committed unchanged | Done in this branch | `docs/product/`, `docs/discussions/PM_PROPOSAL.md` |
| DEV review of scope, A1–A12, roadmap | Done in this branch | `docs/discussions/001-mvp-discovery.md` |
| PM reply to challenges D1–D7 | S, blocked on PM | Especially savings label, email, past budgets, 100% label, audit erasure |
| Product Owner answers PO-Q1–PO-Q6 and TQ-1–TQ-8 | Blocked on PO | Record the answers in `docs/discussions/` |

## M2 — Architecture

| Slice | Complexity | Notes |
|---|---|---|
| HLD, LLD, schema, API, ADRs, traceability | Done in this branch, as proposals | Approval is the remaining exit |
| PM pass over `AC_TRACEABILITY.md` | S, blocked on PM | Pending rows are intentional |
| Product Owner approval of the architecture | Blocked on PO | Gate for every slice below |

## M3 — Foundation

Depends on M2 approval. Suggested issue split:

| Slice | Complexity | Depends on | Exit |
|---|---|---|---|
| Repository tooling: Python and Node projects, formatting, typing, compose services for API and web | M | Approval | One command starts Postgres and the empty app. Record the command actually run. |
| Migrations and RLS bootstrap | M | Tooling | Empty database migrates forward. A query without `app.user_id` returns no tenant rows. |
| Sign up, Argon2id, session login and logout | M | Migrations | AC-AUTH-2.1, AC-AUTH-2.3. Email behaviour follows TQ-4. |
| Login throttling and generic errors | S | Sessions | AC-AUTH-2.2, including a dummy hash when the user does not exist |
| Email verification and password reset | M | TQ-4, a mail interface | AC-AUTH-1 and AC-AUTH-3 if the deploy is public. Console sender in local development. |
| Timezone and profile | S | Sessions | AC-AUTH-5 |
| Account deletion request | S | Sessions | Sets the flag, revokes sessions. The purge job lands with maintenance. AC-AUTH-6. |
| Isolation test suite | M | All auth routes | AC-AUTH-4.1–4.3 against two users. AI case 4.4 waits for M5. |
| CI: tests, Postgres service, secret scan | M | A test exists | Workflow fails the pull request on a red test or a detected secret |
| Logging baseline | S | API skeleton | Request id, no body logging. AC-OPS-2 starts here and finishes in M6. |

## M4 — Core ledger

Depends on M3 auth and CI. Suggested issue split, one pull request each as far as practical:

| Slice | Complexity | Stories | Notes |
|---|---|---|---|
| Money and date domain package, with the precision vectors | M | TXN-1 | No database. This is the first code that can go wrong. |
| Categories, seed on verified user, archive | M | CAT-1–3 | Composite foreign key so a transaction cannot point at another user's category or the wrong type |
| Create income and expense | L | TXN-1, TXN-2, TXN-6 | Idempotency key required. Returns budget warnings. |
| Duplicate warning | M | TXN-5 | Advisory lock. "Save anyway" is a new idempotency key. |
| Edit, optimistic version, audit in the same transaction | L | TXN-3, TXN-7 | 409 returns the current row |
| Soft delete, trash list, restore | M | TXN-4 | Purge command can land here or with foundation maintenance |
| List, filters, filtered totals, cursor pagination | M | LIST-1, LIST-2 | Totals cover the full filter, not the page |
| Dashboard and month report | L | DASH-1–5, RPT-1 | One transaction, set-based SQL. Drill-down is a filtered list URL. |
| Budgets, effective dating, utilisation, warnings | L | BUD-1–4 | Past-month behaviour follows TQ-6 |
| Calendar-year report | M | RPT-2 | Date-range preset |
| CSV export | M | RPT-3 | CSRF header, formula escaping, no BOM |
| Reconciliation property test and timezone boundary tests | M | OPS quality | 10k random 2-decimal amounts. 00:30 IST case. |
| Playwright for CUJ-1, 2, 4, 5, 6, 8 | L | CUJs | CUJ-3 and CUJ-7 are Phase 5 |
| Dogfood | Product | Roadmap Phase 4 | After the tests are green. Real data stays out of Git. |

## M5 — AI

Depends on M4 create and report queries. Can be built against synthetic data before dogfood ends if TQ-1 is accepted. Stays disabled for the Product Owner until the eval bar is met, if TQ-2 is accepted.

| Slice | Complexity | Stories | Notes |
|---|---|---|---|
| Disclosure and off switch | S | AI-6 | Default off |
| Draft endpoint, no write, Indian shorthand for the listed forms | L | AI-1, AI-2 | Provider interface. Grok adapter behind it. |
| Confirm reuses create | M | AI-1 | `source=ai_assistant`. Same duplicate and idempotency rules. |
| Read-only tools and number check | L | AI-3, AI-4, AI-5 | Structured result is the answer. Prose is optional. |
| Eval set of at least 200 synthetic phrases, plus a test that counts writes without confirm | M | Roadmap Phase 5 | Zero unconfirmed writes is a hard fail. Accuracy bar gates the feature, not the ledger, unless TQ-2 says otherwise. |
| Isolation of tool calls | S | AC-AUTH-4.4 | Runs with the M3 suite once tools exist |

## M6 — Production readiness

| Slice | Complexity | Notes |
|---|---|---|
| Threat-model pass against OWASP Top 10 for the routes we actually have | M | Checklist in the PR, not a claim of a third-party audit unless one happens |
| Dependency scan with no high or critical issues left unexplained | S | CI |
| Log scan fixture | S | AC-OPS-2 |
| Encrypted backup and a restore drill into an empty database | M | AC-OPS-1. Record who ran it. Depends on PO-Q4. |
| Deploy and recovery doc | S | The command that was actually used |
| Release notes and Product Owner go/no-go | S | v1.0 only after this is written down |

## Explicitly not in the backlog until re-prioritised

Recurring transactions, CSV import, accounts and transfers, email or push alerts, Indian financial-year preset, AI edit/delete, multi-expense messages, Google sign-in, 2FA, multi-currency, splits, OCR, goals, PDF, sub-categories, household ledgers, bank import, native apps, offline mode, Redis, microservices.

## Ordering constraint

M4's money domain package is the riskiest small slice and should land before screens. Dashboard and budgets should land after create and edit, because they are derived. Export should land after list filters, because it must share them. AI should land after the create service is the single write path, so there is nothing else for a model to call.
