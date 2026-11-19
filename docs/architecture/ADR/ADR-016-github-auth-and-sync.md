# ADR-016: GitHub authentication and sync transport

Status: **Proposed**
Date: 2026-11-19
Decider: DEV
Supersedes: ADR-004 (sessions), ADR-006 (hosting)

## Context

Each user connects a GitHub account they already have. The app syncs to a private repository that user owns. We do not receive the token and we do not run a server that could. The token lives in the Android Keystore. Scopes stay as small as GitHub allows. Sync has to survive GitHub's rate limits, including the content-creation guidance (on the order of 80 content-generating requests per minute and 500 per hour, plus a primary budget of 5,000 REST requests per hour for a normal authenticated user). Those numbers are GitHub's published limits as of this writing and they can change.

Saves never wait for this network path.

## Options

### A. GitHub App, device flow, Contents permission only

We register one GitHub App. The phone requests a device code with the public client id. The user opens `https://github.com/login/device` and enters the code. The phone polls for a user-to-server token. Device flow does not put a client secret in the app. User access tokens issued this way expire (8 hours) and refresh (refresh token about 6 months) without a client secret. The app asks for Contents read and write, and the mandatory Metadata read. Nothing else. The user grants it on one repository.

### B. OAuth app with the classic `repo` scope

Device flow works. `repo` can read and write every private repository the user has, not just the ledger. That is more power than a ledger needs. Classic OAuth tokens of this kind are harder to limit to one repo.

### C. A personal access token the user pastes in

No app registration. Fine-grained tokens can be limited to one repo. The user has to find the GitHub settings screen, which fails the "install and keep your own data" audience. Easy to paste a token that is far too broad.

## Trade-offs

| | A. GitHub App, device flow | B. OAuth app, `repo` scope | C. Pasted PAT |
|---|---|---|---|
| Secret in the binary | Client id only. | Client id only, if device flow. | None. The user supplies a secret, which is worse. |
| Blast radius of a stolen token | One repo, contents. Still serious: it is the ciphertext and the ability to delete it. | Every private repo. | Whatever the user clicked. |
| UX | A code, a browser, done. | Same, wider consent screen. | Poor. |
| Expiry | 8-hour access token, refresh in the keystore. | Often non-expiring classic tokens, which sit longer if stolen. | User-defined. |
| Our server | None. | None. | None. |

## Decision

Option A. If GitHub App device flow cannot request a single-repo contents permission in the way we expect at implementation time, stop and write a note. Do not silently fall back to `repo` scope.

The repository is created by the user or by the app under the user's account, private, named by a fixed default the user may change (`expense-ledger`). The app checks `private: true` before the first push and on every sync. Public means stop.

Token storage: Android Keystore / encrypted storage. Not SQLite plaintext, not the snapshot, not logs.

Sync frequency:

- Not a timer that pushes every minute whether or not anything changed.
- After a local save, debounce about 30 seconds, then try once if the network is up.
- On app open, pull then push.
- A manual "Sync now".
- If the push is rejected because the remote moved, pull, merge, push again. Cap at 3 cycles, then show "couldn't sync" and keep the local ledger.
- If GitHub returns 403 or 429, honour `Retry-After`, otherwise wait at least a minute, and do not loop.

Batching: one snapshot upload per successful sync (ADR-014), not one request per entry. Prefer a single Git Data API sequence (blob, tree, commit, update ref) or a single Contents API update of `snapshot.enc`. Both are a handful of calls. The Contents API must be used serially; GitHub documents that parallel content writes conflict.

Commit identity will be the user's GitHub account. That does not reveal amounts. It does reveal that this GitHub user runs the app. Say that in the setup screen.

## Consequences

- The app works with no GitHub account: local ledger, local export. Sync is a feature the user turns on, not the save path.
- A revoked token stops sync. It does not lock the local ledger.
- We do not force-push the application source repository. Force-push of the **data** branch exists only as the compaction action in ADR-017, and only after a confirmation.
- Rate-limit behaviour has to be tested against GitHub's headers during implementation. This ADR does not claim those tests have been run.

## Revisit when

GitHub denies the app, changes device flow, or the content limits make even one snapshot an hour unreliable. The fallback is local-only plus a user-managed encrypted file, not a server we suddenly introduce.
