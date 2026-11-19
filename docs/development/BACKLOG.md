# Backlog

Status: **Accepted** as the order of work once Phase 3 is explicitly started. Not filed as GitHub issues.
Last updated: 2026-11-19 by DEV, after the Product Owner's answers.

Do not file these, and do not start M3, until the Product Owner says Phase 3 may start. The PM's product docs still describe a server; feature slices that follow those docs are the wrong product. Foundation slices (app shell, money, SQLite, CI) do not depend on that rewrite.

## M1 — Discovery

| Slice | State |
|---|---|
| PM docs, first draft | In the repo, unchanged. They still describe a server. |
| Product Owner decisions | Recorded in the addenda of `docs/discussions/001-mvp-discovery.md`. NQ-1 through NQ-6, TQ-5, TQ-6, and PO-Q6 are answered. |
| PM revision of the PRD and acceptance tests | Not done. Should land before feature slices, not before the Expo shell. |

## M2 — Architecture

| Slice | State |
|---|---|
| Local-first design, Expo, encryption, sync, erasure | In this branch. Accepted except ADR-014 (snapshot layout), which is the recommended format to approve with Phase 3. |
| Cloud CI and EAS | Specified in ADR-018 and the development guide. The workflow file does not exist yet. |

## M3 — Phone foundation and cloud loop

Android only. Expo, not Kotlin. No Android Studio on the Product Owner's machine.

| Slice | Complexity | Notes |
|---|---|---|
| Expo project, TypeScript strict, development build config (not Expo Go) | M | Prebuild so SQLCipher and libsodium can link. |
| GitHub Actions: typecheck, lint, tests on every PR | M | Node only. No emulator. |
| Money module and tests (`bigint`, `0.10 + 0.20`) | M | |
| Merge and conflict tests | L | No GitHub network. |
| Crypto round-trip tests via `libsodium-wrappers` | M | Same parameters as the device library. |
| SQLite migrations tested with Node SQLite | M | Same SQL files. SQLCipher itself is not what this proves. |
| SQLCipher, save path, `mutation_id`, history | L | |
| Passphrase, recovery code, secure-store cache of the DEK | L | The "lost key means lost data" screen is part of the slice. |
| CSV export with BOM | S | Works with no GitHub account. |
| EAS internal APK, `workflow_dispatch` only | M | Needs the Product Owner's Expo account and `EXPO_TOKEN` secret. Not on every push. Free tier is 15 Android builds a month. |

## M4 — Ledger behaviour

| Slice | Complexity | Notes |
|---|---|---|
| Categories, stable UUIDv5 seeds | M | |
| Transactions, edit, tombstone, 30-day restore | L | |
| Refunds subtract from the expense category | S | Accepted. |
| Card / EMI double-count warning, non-blocking | S | Accepted. |
| Repeat as a new entry dated today | S | Accepted. |
| Budgets, four states, past months locked | L | Current and future months editable. |
| Dashboard headline **Net**, calendar year, Indian FY preset | M | FY is 1 Apr–31 Mar. |

## M5 — GitHub sync

Optional for the user. Still required for a second phone.

| Slice | Complexity | Notes |
|---|---|---|
| Device flow, token in secure store, refuse public repos | L | |
| Pull, decrypt, merge, one-commit push, backoff on 429 | L | |
| Conflict screen blocks the push | M | |
| Confirmed data-branch squash plus a new data key | L | GitHub may retain copies. Say so. |
| Delete the repo and wipe local keys | S | Full wipe. Same residual. |

## M6 — AI

Not in the first build. Do not schedule it. The boundary is ADR-005.

## M7 — Device end-to-end, later

Maestro or similar, in CI or a device farm, not on the Product Owner's computer. Not a Phase 3 gate.

## Not in this backlog

Kotlin, a server, Postgres, email login, Expo Go as the real app, an APK build on every pull request, Android Studio, a passphrase reset, household sharing.
