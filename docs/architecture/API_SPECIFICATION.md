# Sync protocol

Status: **Proposed**
Date: 2026-11-19

There is **no application HTTP API**. The server specification that used to live in this file is withdrawn. This document is the protocol between the phone and the user's own private GitHub repository. GitHub's own API is the only network contract, apart from an optional later call to an AI provider.

Local behaviour that never touches the network (save, totals, CSV, conflict resolution) is specified here too, because there is no other contract for it.

## 1. Local save

Not an HTTP call. One SQLite transaction:

1. Reject a second attempt with the same `mutation_id` and return the existing row.
2. Validate money, date, category, note. Rules match the earlier acceptance tests: more than 2 decimal places is an error, not a round; upper bound ₹10,00,00,000.00 inclusive of that value and exclusive above it; date before `2000-01-01` rejected; date more than one year after today in `Asia/Kolkata` (or the user's chosen zone) rejected.
3. Insert or update the row with a new `rev` whose parent is the previous `rev`.
4. Insert `history`.
5. Insert `outbox`.
6. Commit, then update the UI.

Failure rolls the whole transaction back. The UI does not show a row the database does not have.

Duplicate warning: same type, amount, category, and calendar date as a live row created in the last five minutes, by the device clock stored at insert (`created_at`). The user can save anyway, which is a new `mutation_id` and a new entity id. This is separate from `mutation_id`.

## 2. GitHub authentication

Device flow, GitHub App, client id only. No client secret in the app.

1. `POST https://github.com/login/device/code` with the client id and the contents permission.
2. Show the user code and open `https://github.com/login/device`.
3. Poll `POST https://github.com/login/oauth/access_token` at the interval GitHub returns, until the user approves, denies, or the code expires (15 minutes).
4. Store the access token and the refresh token in the Android Keystore.
5. Refresh with `grant_type=refresh_token` when the access token is near expiry. GitHub's documented user-to-server tokens from this flow expire in 8 hours; refresh tokens last on the order of 6 months. Do not send a client secret.
6. On 4xx that means the refresh token is dead, stop syncing and ask the user to connect GitHub again. The local ledger stays.

Logout from GitHub in the app deletes the tokens from the keystore. It does not delete the ledger.

## 3. Repository

- Default name `expense-ledger`, private, user-owned. The user may pick another repo they own.
- Before any write, `GET /repos/{owner}/{repo}`. If `private` is not true, stop and explain. Do not encrypt-and-upload to a public repo.
- First sync writes `README.md`, `manifest.json`, and `snapshot.enc` as described in `DATABASE_SCHEMA.md`.
- README text is fixed and contains no user data: "Encrypted expense ledger. The app that wrote this is local-first. This repository is not readable without the passphrase."

## 4. Pull, merge, push

Preconditions: sync is enabled, a token exists, the passphrase has unwrapped the DEK at least once on this device (or the keystore still has the DEK), and the user is not mid-choice on a conflict unless this pull is only refreshing the conflict list.

1. **Read the tip.** `GET /repos/{owner}/{repo}/contents/manifest.json` and the snapshot it names, using the raw media type. Record the blob sha.
2. If the sha equals `last_pushed_sha` and the outbox is empty, stop.
3. Decrypt. Wrong key or failed authentication tag: stop, tell the user the passphrase does not open this repo, do not overwrite local data.
4. Merge into SQLite using the LLD. If that produces one or more conflicts, commit the conflict rows, update the UI, and **do not push**.
5. If there are already unresolved conflicts, do not push.
6. If the outbox is empty and the merge changed nothing, store the sha and stop.
7. Build the canonical snapshot from SQLite, compress, seal with a new nonce.
8. **Write.** Update `snapshot.enc` and `manifest.json` in one commit. Use the Git Data API (create blob, create tree, create commit, update ref) so the two files land together. Send the current commit sha as the parent. Commit message `Update encrypted ledger`.
9. If the ref update fails because the parent moved, go back to step 1. Do this at most 3 times, then stop and keep the local ledger.
10. On success, store the new sha and clear the outbox entries that were included.

Contents API `PUT` of a single file is an acceptable fallback only if both files can be committed without a window where the manifest and the snapshot disagree. A lone snapshot update with a stale manifest is a bug. Prefer one commit containing both.

Do not call the Contents API once per transaction. Do not issue those writes in parallel.

## 5. Rate limits

Published GitHub limits this design is built around (they can change; re-read them at implementation):

- About 5,000 REST requests per hour for a normal authenticated user.
- Secondary limits, including a content-creation guidance on the order of 80 requests per minute and 500 per hour.
- 403 or 429: if `Retry-After` is present, wait that long. If the rate-limit reset header says the budget is gone, wait until then. Otherwise wait at least one minute, then back off.

Behaviour:

- One commit per successful sync, not one per save.
- Debounce about 30 seconds after the last local save before attempting.
- Also pull on app open, and when the user taps Sync.
- Never a tight retry loop.
- Local saves do not wait for any of this.

`GET /rate_limit` may be used for diagnosis. It does not count against the primary budget. Do not call it on every save.

## 6. Conflict choice

The screen shows both versions, field by field, including a tombstone as "deleted". The user picks a value per field, or picks one whole version. The app writes a new `rev` with **both** previous revs as parents, writes history, and only then may push.

Cancelling the screen leaves the conflict in place and leaves the push blocked. It does not revert the local ledger's other, unrelated rows.

## 7. CSV export

Local file, not a GitHub upload.

- UTF-8 with a leading BOM (`EF BB BF`). **Proposed**, from the product-analyst review, so Excel opens the file. This replaces the earlier no-BOM choice.
- Columns, in order: `date`, `type`, `amount`, `currency`, `category`, `payment_method`, `note`, `created_at`.
- Amounts are plain decimal strings. No rupee sign, no grouping commas.
- `created_at` is UTC with a `Z` suffix.
- A field that starts with `=`, `+`, `-`, `@`, tab, or carriage return is prefixed with a single quote, then quoted with RFC 4180 rules.
- Tombstones are omitted.
- The sum of `amount` for a type, read back as decimals, equals the on-screen total for that filter. Refunds, if accepted, are their own `type` value `refund` and are not folded into the expense column.

## 8. Repeat

**Proposed.** A local action. New entity id, new `mutation_id`, date is today in the user timezone, copied amount, type, category, payment method, and note. It goes through the normal save path, including the duplicate warning.

## 9. AI draft, if the feature is ever enabled

Undecided. Not part of the sync protocol.

- The phone sends the user's sentence, the category names, and today's date to the provider. It does not send the database.
- The response is a draft on screen. SQLite is unchanged.
- Confirm is the local save in section 1, with `source=ai_assistant`.
- Cancel writes nothing.
- More than one expense in the sentence: no draft, a short explanation. This was the PM's preference and it still stands unless they reverse it.
- The provider being unreachable leaves the manual form usable, with the sentence available to paste into the note.

## 10. What this protocol refuses

- A request to our own backend. There isn't one.
- A push while conflicts are open.
- A push to a public repository.
- A commit message that contains user text.
- A silent choice between two versions.
- A sync failure that rolls back a local save.
