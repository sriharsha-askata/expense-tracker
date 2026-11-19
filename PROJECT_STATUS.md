# Project Status

Last updated: 2026-11-19 (IST) · Updated by: DEV · Phase: **1–2, discovery and architecture, revised**

This file is the shared working memory. The first status in this branch described a server. That plan is withdrawn.

## Snapshot

| Item | State |
|---|---|
| Product docs (`docs/product/`) | Unchanged, on purpose. They still describe a web server. The PM will revise them. |
| Product Owner direction | Recorded in the addendum of `docs/discussions/001-mvp-discovery.md`. Local-first Android, SQLite, GitHub sync, no server, no household sharing. |
| Architecture | Revised and **Proposed**. Server ADRs are marked Superseded. |
| Application code | None. |
| GitHub issues | Not created. |
| CI | No workflow. Nothing has run. |
| Tests | None exist. None were run. |
| Server runtime | Withdrawn. `docker-compose.yml` removed. It was never executed. |

## Phase gate

Do not start implementation until:

1. The Product Owner answers NQ-1 through NQ-6 in the discovery addendum, or explicitly defers each to the recommendation written there.
2. The PM updates `docs/product/` so the acceptance tests match a local-first app. Implementing the current email-and-server tests would rebuild the rejected design.
3. The Product Owner accepts this revision of the architecture.

## Decisions in force (proposed mechanisms)

- Kotlin Android app. SQLite is the ledger. Saves do not wait for the network.
- Integer paise, `YYYY-MM-DD` dates, default timezone `Asia/Kolkata`.
- One encrypted snapshot per sync to a private repo the user owns. Git does not merge the ciphertext.
- Passphrase wraps a random data key. Recovery code shown once. No reset. Lost secrets mean lost data.
- Tombstones for 30 days. Erasure of Git history is a confirmed squash plus a new data key.
- Budget states: `ok`, `approaching`, `at_limit`, `exceeded`.
- AI is optional and undecided.

## Waiting on people

**Product Owner:** NQ-1 (iOS in v1?), NQ-2 (GitHub account required only for sync?), NQ-3 (no passphrase reset), NQ-4 (squash plus key rotation), NQ-5 (analyst items), NQ-6 (AI stays out of the first build). Also still open from the first review: TQ-5 (the word savings), TQ-6 (past budgets), PO-Q6 (financial year).

**PM:** Revise the PRD, stories, acceptance criteria, and roadmap so they stop requiring a server. Reply on the analyst items if you want a recommendation recorded before the Product Owner does. The word "savings" is still unresolved.

## Analyst items (Proposed, not confirmed)

Refunds reduce the expense category. A non-blocking warning about double-counting card bills and EMIs. CSV UTF-8 BOM. Repeat creates a new entry dated today. No hard lockout.

## Not done, and not claimed

- No phone build, no GitHub App registration, no encryption prototype, no merge tests.
- GitHub's rate-limit numbers were checked against GitHub's public docs while revising this design. No request was sent to the GitHub API from this environment.
- License still unchosen.

## Next action

Wait for the Product Owner on NQ-1 through NQ-6 and for the PM's product-doc revision. Do not open implementation issues from the old server backlog.

## Verification log

| Check | Result |
|---|---|
| `docs/product/` and `docs/discussions/PM_PROPOSAL.md` | Unchanged in this revision. Confirmed with `git diff` before commit: no changes in those paths. |
| Application tests | Not run. There is no application. |
| Docker | The compose file was removed. Docker was not installed here, and the old file had never been started. |
| GitHub issue creation | Not done. |
