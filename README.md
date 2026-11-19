# Expense Tracker

A private personal ledger for one person: record income and expenses exactly, see where the money went, and stay inside a monthly budget. Natural-language entry is planned as a later add-on that can only propose a draft. Nothing is saved until the user confirms it.

**Status: Phase 1–2, discovery and architecture. Proposed, not approved.** There is no application yet. Do not treat this repository as a running product.

Product Owner: Sri Harsha. Product decisions belong to the PM agent (Track). Technical decisions belong to the DEV agent. The Product Owner has the final say. This repository is the source of truth for requirements, decisions, and (later) code. It must never hold personal financial records.

## Read this first

| Document | What it is |
|---|---|
| [`PROJECT_STATUS.md`](PROJECT_STATUS.md) | Where the work stands, what is blocked, and what happens next |
| [`docs/discussions/PM_PROPOSAL.md`](docs/discussions/PM_PROPOSAL.md) | PM's discovery proposal, unchanged |
| [`docs/discussions/001-mvp-discovery.md`](docs/discussions/001-mvp-discovery.md) | DEV review, challenges, technical decisions, open questions |
| [`docs/product/PRD.md`](docs/product/PRD.md) | Product requirements (PM draft) |
| [`docs/architecture/HLD.md`](docs/architecture/HLD.md) | Proposed system shape |
| [`docs/development/DEVELOPMENT_GUIDE.md`](docs/development/DEVELOPMENT_GUIDE.md) | Engineering rules and the planned workflow |
| [`docs/development/BACKLOG.md`](docs/development/BACKLOG.md) | Proposed issue breakdown (not filed in GitHub yet) |

## Rules that already apply

- Money is never a floating-point number. The proposal stores integer minor units (paise for INR) plus an ISO-4217 currency code. APIs exchange decimal strings such as `"350.00"`, never JSON numbers.
- A transaction date is a calendar date. It is not a UTC timestamp, so it does not move when the timezone changes.
- One user's data is never returned to another user. A foreign id is a 404.
- Deletes are soft for 30 days. The purge and account-deletion design has an open question on audit retention; see the discovery note.
- The AI assistant is not in this phase. When it is built, it may draft and it may answer from figures the backend computed. It has no tool that writes.
- No microservices, and no Redis, unless a later ADR shows a concrete need.
- No secrets, `.env` files, database dumps, or real ledgers in Git.

## Repository layout

```text
expense-tracker/
├── .github/
│   ├── ISSUE_TEMPLATE/
│   ├── workflows/          # empty until Phase 3; see the README there
│   └── pull_request_template.md
├── docs/
│   ├── product/            # PRD, stories, acceptance criteria, roadmap
│   ├── discussions/        # PM proposal and DEV review
│   ├── architecture/       # HLD, LLD, schema, API, ADRs (all Proposed)
│   └── development/        # guide and backlog
├── frontend/               # placeholder
├── backend/                # placeholder
├── tests/                  # placeholder
├── docker-compose.yml      # proposed Postgres sketch, not a running app
├── DEVELOPMENT_GUIDE.md    # pointer; canonical guide is under docs/development
├── PROJECT_STATUS.md
└── README.md
```

## Local run

Not available. Architecture has not been approved, and application services have not been written. `docker-compose.yml` only describes a future local Postgres. It was not started as part of this work.

## License

Not chosen. Do not assume one.
