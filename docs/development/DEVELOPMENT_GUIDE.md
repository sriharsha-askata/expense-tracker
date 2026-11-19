# Development Guide

Status: **Proposed**
Audience: PM, DEV, Product Owner
Last updated: 2026-11-19

This is the canonical guide. A pointer lives at `/DEVELOPMENT_GUIDE.md`.

The guide that described FastAPI, Postgres, and a server session is withdrawn. Do not follow it if you remember it.

## Current phase

Architecture, revised for a local-first Android app, still Proposed. Until the Product Owner accepts this revision and the PM updates `docs/product/`:

- Do not add application code under `android/` or anywhere else.
- Do not register a GitHub App, create a data repository, or call the GitHub API with a real token from this project.
- Do not add a CI workflow that implies the app is tested.
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

**Money.** No `float`, no `double`, no Kotlin `Double`, no SQLite `REAL`. Store paise as `Long` / `INTEGER` and a currency code. Parse with a decimal type, then scale. Reject more than two decimal places. Sum integers. Format only at the screen and in CSV.

**Dates.** `YYYY-MM-DD` text. "Today" uses the user's IANA timezone, default `Asia/Kolkata`. Instants are UTC text. Changing the zone does not rewrite stored days.

**Saves.** The SQLite transaction completes, then the UI updates, then sync may run. A sync failure does not roll back the save.

**Conflicts.** Different ids both survive. Two heads of one id are a conflict. The push does not choose.

**Encryption.** GitHub receives ciphertext plus a manifest of cipher parameters. The passphrase, recovery code, data key, and GitHub token are not logged and are not written into the snapshot plaintext.

**Deletion.** A tombstone is not erasure of Git history. Say so in the interface. Compaction force-pushes only the user's data branch, only after confirmation, and rotates the data key.

**AI.** Not in the first build unless the Product Owner explicitly asks. A later module writes only through the normal save, after confirm.

**Logs.** No amounts, notes, secrets, or tokens.

**Secrets.** `.env` is git-ignored. This app has no server environment file to commit. A token pasted into an issue is a leaked token.

## Proposed layout (not created)

```text
android/
  app/src/main/        Compose UI, sync worker
  domain/              money, calendar, merge, csv — plain JVM, no Android imports
  db/                  SQLCipher
```

`domain/` tests run on the JVM. Merge tests do not need GitHub. A later test may use a fixture GitHub repo; it must contain only synthetic ciphertext and must not be this source repository.

## Local run

Not available in this phase. When it exists, it will be an Android build, not `docker compose`.

## Definition of done, once implementation starts

1. The story matches the revised product docs, not the server-era acceptance tests.
2. Schema or sync-format changes update the architecture docs in the same pull request.
3. The money and merge tests were run, and the command is in the pull request.
4. Logs from the new path do not contain the fixture amounts.
5. `PROJECT_STATUS.md` moves only after the above.

If a product change breaks an ADR, write a new ADR or mark the old one Superseded. Do not quietly grow a server because sync is hard.
