# Expense Tracker

A local-first Android ledger built with Expo and React Native. The phone stores the transactions. A save does not wait for the network. GitHub is optional: without an account the app still works on one phone, including CSV export. With an account, an encrypted copy in a private repository you own can reach a second phone. The project does not run a server and does not hold your ledger or your GitHub token.

**Status: product decisions accepted, Phase 3 not started.** The Product Owner rejected the earlier web server, chose Expo, and will install APKs from a cloud build. No application has been built. `docs/product/` still describes the server until the PM updates it.

Product Owner: Sri Harsha. Product decisions belong to the PM. Technical decisions belong to DEV. This repository is the source of truth for the design. It must never hold a personal ledger, a GitHub token, a passphrase, or a recovery code.

## Read this first

| Document | What it is |
|---|---|
| [`PROJECT_STATUS.md`](PROJECT_STATUS.md) | Current state and what is blocked |
| [`docs/discussions/001-mvp-discovery.md`](docs/discussions/001-mvp-discovery.md) | DEV review. The **addendum at the end** is the current decision. The top of the file is the withdrawn server review. |
| [`docs/architecture/HLD.md`](docs/architecture/HLD.md) | Accepted shape. ADR-014 is the one item still Proposed. |
| [`docs/architecture/API_SPECIFICATION.md`](docs/architecture/API_SPECIFICATION.md) | Sync protocol. There is no application HTTP API. |
| [`docs/development/DEVELOPMENT_GUIDE.md`](docs/development/DEVELOPMENT_GUIDE.md) | Engineering rules |

## Rules that already apply

- Money is integer paise plus a currency code. APIs, files, and history use decimal strings. No floating point.
- A transaction date is a calendar day (`YYYY-MM-DD`). The default timezone is `Asia/Kolkata`.
- The app must not silently pick between two edits of the same entry.
- Deletes sync as tombstones. Removing them from Git history is a separate, confirmed action.
- If the passphrase and the recovery code are gone, and no phone still has the key, the encrypted copy cannot be opened. That is lost data, not a reset flow.
- AI is out of the first build. If it is added later, it cannot write a transaction without a confirmation.
- The dashboard word is Net. Reports default to the calendar year and include an Indian financial-year preset.
- Tests will run on GitHub Actions without an emulator. Installable APKs come from EAS Build, not from a local Android Studio.
- No secrets and no real financial records in this repository.

## Layout

```text
docs/product/          PM drafts (not yet revised for local-first)
docs/discussions/      PM proposal and DEV review
docs/architecture/     Current proposal and superseded ADRs
docs/development/      Guide and backlog
android/               Placeholder. No code yet.
```

`frontend/` and `backend/` are leftovers from the withdrawn server layout. Do not put the app there.

## Local run

Not available. There is nothing to install.
