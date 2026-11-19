# Development Guide

Status: **Accepted** for how Phase 3 will be tested and built, once the Product Owner says Phase 3 may start.
Audience: PM, DEV, Product Owner
Last updated: 2026-11-19

This is the canonical guide. A pointer lives at `/DEVELOPMENT_GUIDE.md`.

The guide that described FastAPI, Postgres, and a server session is withdrawn. Do not follow it if you remember it.

## Current phase

The Product Owner has answered the open product questions. Phase 3 still does not start until he explicitly says to start it. Until that word:

- Do not add application code, an Expo project, or a GitHub Actions workflow. A workflow with nothing to test would be a false green check.
- Do not register a GitHub App, create a data repository, or call the GitHub API with a real token from this project.
- Do not create an Expo account or an `EXPO_TOKEN` from a cloud agent. The Product Owner owns that login.
- Do not file the backlog as GitHub issues until asked.
- Do not merge on the strength of this document.

## How the two agents use the repo

Unchanged in spirit. GitHub is the source of truth for **this source repository**. It is also, in the product, where a user's encrypted ledger may be copied. Those are different repositories. Never put a test ledger, a token, or a passphrase in this one.

| Kind of change | Who | Where |
|---|---|---|
| What the product does | PM, Product Owner has the last word | `docs/product/`, then a discussion note |
| How it is built | DEV | `docs/architecture/`, ADRs |
| Disagreement | Both, in writing | `docs/discussions/` |
| Implementation, later | DEV | `feature/<issue>-<slug>` |

Branch roles: `main` for accepted work, `docs/<topic>` for design (this branch), `feature/` and `bugfix/` once issues exist.

A pull request says which tests were actually run. "Not run" is the honest answer when nothing was run.

## Engineering rules

**Money.** No `float`, no `double`, no JavaScript `number`, no SQLite `REAL`. Store paise as `bigint` in TypeScript and `INTEGER` in SQLite, plus a currency code. Convert driver values to `bigint` at the database boundary. Parse with a decimal type, then scale. Reject more than two decimal places. Sum integers. Format only at the screen and in CSV.

**Dates.** `YYYY-MM-DD` text. "Today" uses the user's IANA timezone, default `Asia/Kolkata`. Instants are UTC text. Changing the zone does not rewrite stored days.

**Saves.** The SQLite transaction completes, then the UI updates, then sync may run. A sync failure does not roll back the save.

**Conflicts.** Different ids both survive. Two heads of one id are a conflict. The push does not choose.

**Encryption.** GitHub receives ciphertext plus a manifest of cipher parameters. The passphrase, recovery code, data key, and GitHub token are not logged and are not written into the snapshot plaintext.

**Deletion.** A tombstone is not erasure of Git history. Say so in the interface. Compaction force-pushes only the user's data branch, only after confirmation, and rotates the data key.

**AI.** Out of the first build. A later module writes only through the normal save, after confirm.

**Logs.** No amounts, notes, secrets, or tokens.

**Secrets.** `.env` is git-ignored. This app has no server environment file to commit. A token pasted into an issue is a leaked token.

## Layout (not created until Phase 3)

```text
app/                 Expo project (created later, not a Kotlin tree)
  src/domain/        money, calendar, merge, csv — TypeScript, no React Native imports
  src/db/            SQLCipher via op-sqlite, same SQL files the Node tests run
.github/workflows/   typecheck, lint, test on every PR; APK build only on demand
```

`android/` in this repo is a placeholder note from an earlier Kotlin sketch. Do not put the Expo project there unless Expo's prebuild does so itself.

## How cloud agents and GitHub Actions run tests

No Android Studio, no emulator, and no phone are required for the merge bar.

1. Use Node on Linux. Install dependencies with the project's package manager.
2. Run typecheck, lint, and the test script.
3. Money tests and merge tests import plain TypeScript. They do not open a device.
4. Crypto round-trips import `libsodium-wrappers` and use the same parameters as the app's `react-native-libsodium` (ADR-012, ADR-018).
5. Migration tests open Node's SQLite and apply the app's SQL files. They do not prove SQLCipher.

A cloud agent that has only run those commands has not verified a tap on a phone. Say that in the pull request.

The workflow file is added in Phase 3 with the Expo project, not before. Design: ADR-018.

## Installable builds

EAS Build, internal distribution, APK. The Product Owner opens the URL on the phone and accepts Android's unknown-apps prompt. Expo Go is not a preview of this app.

`EXPO_TOKEN` is a GitHub Actions secret. The Android keystore is created by EAS and kept there, with a copy in the Product Owner's password manager. Neither is committed.

Do not run EAS on every pull request. The published Free plan (checked 2026-11-19) includes 15 Android builds a month on a low-priority queue. That is enough for on-demand builds and a release tag. It is not enough for a build per push. Details and the "do not start paying by accident" rule are in ADR-018.

## Local run

There is no local Android toolchain on purpose. The Product Owner does not install one. Agents do not install one to claim a device test.

## Definition of done, once implementation starts

1. The story matches the revised product docs, not the server-era acceptance tests.
2. Schema or sync-format changes update the architecture docs in the same pull request.
3. GitHub Actions ran typecheck, lint, and the Node tests on the pull request, and the result is linked. A local agent run of the same commands is extra evidence, not a substitute for the workflow once the workflow exists.
4. Logs from the new path do not contain the fixture amounts.
5. `PROJECT_STATUS.md` moves only after the above.

If a product change breaks an ADR, write a new ADR or mark the old one Superseded. Do not quietly grow a server because sync is hard.
