# Project Status

Last updated: 2026-11-19 (IST) · Updated by: DEV · Phase: **1–2, decisions recorded, waiting for a Phase 3 go-ahead**

## Snapshot

| Item | State |
|---|---|
| Product docs (`docs/product/`) | Unchanged, on purpose. They still describe a web server. The PM will revise them. |
| Product Owner decisions | Recorded in the addenda of `docs/discussions/001-mvp-discovery.md`. The questions from the last two rounds are answered. |
| Architecture | Expo / React Native, on-device SQLite, optional GitHub sync. ADRs the Product Owner decided are marked **Accepted**. ADR-014 (one snapshot file) is still **Proposed**. |
| Application code | None. No Expo project, no workflow file. |
| GitHub issues | Not created. |
| CI | Designed in ADR-018. Not configured, because there is nothing to test yet. |
| Tests | None exist. None were run. |
| Installable build | Not produced. EAS has not been run. No Expo token is stored here. |

## Phase gate

Phase 3 does not start until the Product Owner says it may start. The design questions he was asked are closed. Starting now would still be ahead of that word.

When he says start:

1. Foundation first: Expo project, money and merge tests, GitHub Actions, then an on-demand EAS APK.
2. Do not implement the server-shaped acceptance tests still sitting in `docs/product/`.
3. Treat ADR-014 as accepted unless he rejects the one-snapshot layout in the same breath.

## Decisions in force

- Expo, React Native, TypeScript. Android first. iOS later is the same project, not a Kotlin rewrite.
- SQLite on the phone. Saves do not wait for the network. GitHub is optional.
- Integer paise as `bigint`. No JavaScript `number` for money. Dates are `YYYY-MM-DD`. Default timezone `Asia/Kolkata`.
- Dashboard headline is **Net**. Calendar year by default, plus an Indian FY preset. Past budget months are locked.
- Refunds reduce the expense category. Card/EMI warning does not block. CSV has a UTF-8 BOM. Repeat is a new entry dated today. Passphrase failures back off and do not hard-lock.
- No passphrase reset. One-time recovery code. Lost secrets mean the encrypted copy is gone.
- Tombstones for 30 days. Forgetting history is a confirmed data-branch squash plus a new data key. Deleting the repo wipes the account. GitHub may keep copies.
- AI is out of the first build.
- Tests: GitHub Actions, Node only, every pull request. APK: EAS internal distribution, on demand, not every push. Free tier is 15 Android builds a month (published figure, 2026-11-19).

## Waiting on people

**Product Owner:** the go-ahead to start Phase 3, and later an Expo account plus an `EXPO_TOKEN` GitHub secret when the first APK is wanted. Not before.

**PM:** revise `docs/product/` so stories and acceptance tests match this app. That should land before feature work, not before the Expo shell.

## Not done, and not claimed

- No phone build, no EAS build, no Expo account created by this agent.
- No GitHub Actions run. The workflow file is intentionally absent.
- Pricing and internal-distribution behaviour were read from Expo's public docs. They were not exercised.
- License still unchosen.

## Next action

Wait for the Product Owner to approve starting Phase 3. Do not open implementation issues and do not add a workflow that would show a green check over an empty project.

## Verification log

| Check | Result |
|---|---|
| `docs/product/` and `docs/discussions/PM_PROPOSAL.md` | Must stay unchanged in this commit. Confirm with `git diff` before committing. |
| Application tests | Not run. There is no application. |
| EAS / Expo | Not called. |
| GitHub issue creation | Not done. |
