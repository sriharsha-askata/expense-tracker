# Acceptance-criteria traceability (v0.2)

Status: **Proposed** for PM sign-off. The design can satisfy every row. Nothing here has been implemented or tested.
Date: 2026-11-19
Source: `docs/product/ACCEPTANCE_CRITERIA.md` v0.2 (191 criteria). Stories: `USER_STORIES.md` v0.2.

Retired and not mapped, because the PM removed them and said the IDs are not reused: AUTH-1, AUTH-2, AUTH-3, AUTH-7. AUTH-8 is P1 (app lock) and has no MVP criteria. AUTH-6 is ERS-3.

Legend: **Satisfiable** means the current architecture has a place to enforce it. **Manual** means the criterion is marked manual and is not a CI gate. **Later** means deferred AI, not in the first build.

## Conflicts found while mapping, and what changed

| v0.2 rule | What the architecture said | Fix |
|---|---|---|
| Income defaults have no "Refund" category (FR-CAT-1, AC-CAT-1.1) | The schema pointed at FR-CAT-1 but an older reading of that list included Refund as income | Schema now names the v0.2 income list and forbids an income category called Refund |
| Future months are editable; a February budget does not change December (AC-BUD-1.4, AC-BUD-1.5) | The LLD described editing the current month and locking the past, and did not say what a future month does to later slices | LLD now rejects only past months, and a change at month M replaces M and later periods only |
| Duplicate warning is per phone (AC-TXN-5.4) | The LLD already said per device, with a hedge about mentioning it in the UI | Hedge removed. The check is specified as this phone only |
| Stale edit after sync on the same phone (AC-TXN-3.3) | The old text was a two-tab HTTP 409 | Save compares the form's revision with the row. If sync moved it, the app shows the current values and asks |
| Spending trend and category comparison include refunds (FR-DASH-2, FR-DASH-4) | The trend was easy to read as expenses only | LLD uses expenses minus refunds for spending, the trend, and category comparison. AC-DASH-3.1 has no refunds, so its numbers still match |
| Other phones are not wiped when one phone deletes the account (AC-ERS-3.4) | Delete-account was specified for the phone that confirmed | LLD: that phone deletes the repo and wipes itself. Others keep the local ledger, are told the backup is gone, and are offered CSV |
| 30-day expiry with a moved clock (AC-TXN-4.5) | Not spelled out | Expiry runs on next launch against the phone's date. A `deleted_at` in the future is kept. A clock set forward can expire early. There is no trusted clock. Called out under pushback, not papered over |
| Two compactions while a phone is offline | The schema said a forgotten row can come back, which fights AC-ERS-1.3 | LLD: one missed compaction applies `purged_ids`. Two or more: do not upload and do not delete; ask. The confirmation screen says to sync other phones first |

## AUTH-4, AUTH-5

| ID | Verdict | Design |
|---|---|---|
| AC-AUTH-4.1 | Satisfiable | No shared database. Two installs do not exchange rows except through a repo the same person connected. |
| AC-AUTH-4.2 | Satisfiable | Sync writes only to the private repo the user picked (ADR-016). |
| AC-AUTH-4.3 | Satisfiable | Each database file is one ledger. Tests use two files, not two server users. |
| AC-AUTH-5.1 | Satisfiable | Calendar date, "today" in Asia/Kolkata. Fixed-clock test. |
| AC-AUTH-5.2 | Satisfiable | Timezone is a setting. Transaction dates are not rewritten. |

## SET-1, SET-2, SET-3

| ID | Verdict | Design |
|---|---|---|
| AC-SET-1.1 | Satisfiable | First run needs no network. Seeds categories, `Asia/Kolkata`, INR. |
| AC-SET-1.2 | Satisfiable | Passphrase at least 12 characters, common-password list. |
| AC-SET-1.3 | Satisfiable | GitHub is optional (ADR-016). CSV is local. |
| AC-SET-2.1 | Manual | SQLCipher. A copied file is not plaintext SQLite. Checked on a real APK, not in CI. |
| AC-SET-2.2 | Satisfiable | Wrong passphrase message and backoff, cap 30 seconds (LLD). |
| AC-SET-2.3 | Satisfiable | Failures never wipe and never stick. |
| AC-SET-2.4 | Satisfiable | Counter resets on success. |
| AC-SET-2.5 | Satisfiable | Daily open uses the key in secure storage. The passphrase is for restore, a new phone, and key rotation. An OS biometric prompt for the keystore is not our passphrase prompt. |
| AC-SET-3.1 | Satisfiable | Recovery code once, explicit confirmation. |
| AC-SET-3.2 | Satisfiable | The warning text is required before the user depends on the app. |
| AC-SET-3.3 | Satisfiable | Declining the code is stored, and the shorter warning is shown. |
| AC-SET-3.4 | Satisfiable | The recovery code unwraps the same data key (ADR-015). |
| AC-SET-3.5 | Satisfiable | No reset control is part of the UI contract. |

## TXN-1 through TXN-7

| ID | Verdict | Design |
|---|---|---|
| AC-TXN-1.1 | Satisfiable | Local save, integer paise, no network. |
| AC-TXN-1.2 | Satisfiable | `bigint` sums. 10 and 20 paise are 30. Property test of 10k amounts. |
| AC-TXN-1.3 | Satisfiable | Scale over 2 is rejected. |
| AC-TXN-1.4 | Satisfiable | Non-positive and above ₹10,00,00,000.00 rejected. The exact upper bound is accepted. |
| AC-TXN-1.5 | Satisfiable | The form strips Indian grouping. Storage is `123456.50`. Display uses Indian grouping. |
| AC-TXN-1.6 | Satisfiable | Category type must match. Refunds are the exception and must be expense categories (AC-TXN-12.2). |
| AC-TXN-1.7 | Satisfiable | Category required. |
| AC-TXN-1.8 | Satisfiable | More than one year ahead rejected. `upcoming` is derived. |
| AC-TXN-1.9 | Satisfiable | 500 code points. Notes render as text. |
| AC-TXN-1.10 | Satisfiable | Currency INR in v1. The column still stores the code. |
| AC-TXN-2.1 | Satisfiable | Income adds to income. Net = income − expenses + refunds. |
| AC-TXN-3.1 | Satisfiable | Aggregates read the committed row. |
| AC-TXN-3.2 | Satisfiable | Month membership is the calendar date. |
| AC-TXN-3.3 | Satisfiable | Form revision must match the row. Otherwise show current values and ask. |
| AC-TXN-3.4 | Satisfiable | One validator for create and edit. |
| AC-TXN-3.5 | Satisfiable | Row and history in one SQLite transaction. |
| AC-TXN-4.1 | Satisfiable | Tombstone excluded from list, totals, budgets, reports, export. |
| AC-TXN-4.2 | Satisfiable | Restore clears the tombstone inside 30 days. |
| AC-TXN-4.3 | Satisfiable | Next launch drops tombstones whose `deleted_at` is at least 30 days before today. No server job. |
| AC-TXN-4.4 | Satisfiable | Delete and restore each write history. |
| AC-TXN-4.5 | Satisfiable, with the clock limit in the LLD | A future `deleted_at` is kept. A forward clock can expire early. |
| AC-TXN-5.1 | Satisfiable | Same type, amount, category, date, created on this phone within 5 minutes. |
| AC-TXN-5.2 | Satisfiable | Outside that window, or a different field, no warning. |
| AC-TXN-5.3 | Satisfiable | Save anyway inserts a second id. No merge. |
| AC-TXN-5.4 | Satisfiable | The check does not cross phones. |
| AC-TXN-5.5 | Satisfiable | Repeat is a normal save. |
| AC-TXN-6.1 | Satisfiable | One `mutation_id` per gesture. |
| AC-TXN-6.2 | Satisfiable | Retry reuses that id. |
| AC-TXN-6.3 | Satisfiable | The save is one SQLite transaction. A kill commits all of it or none. |
| AC-TXN-6.4 | Satisfiable | A new gesture is a new id, then the duplicate warning may speak. |
| AC-TXN-7.1 | Satisfiable | History rows, UTC stored, shown in the user zone. |
| AC-TXN-7.2 | Satisfiable | No edit or delete control for history. |
| AC-TXN-7.3 | Satisfiable | Permanent removal deletes history rows with the transaction, including from the next snapshot. |

## TXN-12, TXN-13, TXN-14

| ID | Verdict | Design |
|---|---|---|
| AC-TXN-12.1 | Satisfiable | Food spent is expenses minus refunds. Refunds are not income. |
| AC-TXN-12.2 | Satisfiable | Refund requires an expense category. No default income category named Refund. |
| AC-TXN-12.3 | Satisfiable | Negative spent is shown. Not clamped. |
| AC-TXN-12.4 | Satisfiable | The refund's own calendar date picks the month. |
| AC-TXN-12.5 | Satisfiable | Positive minor units. Same parser as other amounts. |
| AC-TXN-12.6 | Satisfiable | Spent is computed on read. |
| AC-TXN-13.1 | Satisfiable | New id, today, copied fields, old row untouched. |
| AC-TXN-13.2 | Satisfiable | Does not copy `deleted_at` or history. |
| AC-TXN-13.3 | Satisfiable | Duplicate warning on the second repeat inside 5 minutes. |
| AC-TXN-13.4 | Satisfiable | Nothing is written until Save. No scheduler. |
| AC-TXN-14.1 | Satisfiable | EMI/Loan shows the note. Save is not blocked. |
| AC-TXN-14.2 | Satisfiable | Fixed hint list: "card bill", "credit card payment", "cc payment". |
| AC-TXN-14.3 | Satisfiable | The note does not change totals. |
| AC-TXN-14.4 | Satisfiable | Ordinary card spend does not match the hint list. |

## CAT, LIST

| ID | Verdict | Design |
|---|---|---|
| AC-CAT-1.1 | Satisfiable | v0.2 seed list. |
| AC-CAT-1.2 | Satisfiable | UUIDv5 of type and name. |
| AC-CAT-2.1 | Satisfiable | Create category. |
| AC-CAT-2.2 | Satisfiable | Unique on trimmed case-folded name per type. |
| AC-CAT-2.3 | Satisfiable | Transactions store the category id. Rename does not touch amounts. |
| AC-CAT-3.1 | Satisfiable | `archived_at` hides the picker only. |
| AC-CAT-3.2 | Satisfiable | Hard delete only when no transaction row, including tombstones, references it. |
| AC-CAT-3.3 | Satisfiable | Open budget `effective_to` becomes the first of next month. |
| AC-LIST-1.1 | Satisfiable | Order is date, then `created_at`, then id. Virtualised list or pages both meet "scrolls/pages". |
| AC-LIST-1.2 | Satisfiable | Empty state when the count is 0. |
| AC-LIST-2.1 | Satisfiable | One filter for rows and for the total. |
| AC-LIST-2.2 | Satisfiable | Inclusive dates. |
| AC-LIST-2.3 | Satisfiable | Case-insensitive note search. Wildcards escaped. |
| AC-LIST-2.4 | Satisfiable | Tombstones omitted. |

## DASH, BUD

| ID | Verdict | Design |
|---|---|---|
| AC-DASH-1.1 | Satisfiable | Net ₹32,659.25, formula visible, 38.4% display, the word savings is not the headline. |
| AC-DASH-1.2 | Satisfiable | Refunds ₹1,200 shown apart. Net ₹33,859.25. |
| AC-DASH-1.3 | Satisfiable | Rate is null when income is 0. The UI shows "—". |
| AC-DASH-1.4 | Satisfiable | Negative Net when expenses exceed income plus refunds. |
| AC-DASH-1.5 | Satisfiable | Month window on the calendar date. |
| AC-DASH-2.1 | Satisfiable | Category spending sums to total spending in paise. Percents are not forced to 100. |
| AC-DASH-2.2 | Satisfiable | Archived categories with spend stay in the breakdown. |
| AC-DASH-3.1 | Satisfiable | Absolute delta from integer paise. The example is expenses with no refunds. |
| AC-DASH-3.2 | Satisfiable | Previous 0 and current non-zero is "new". |
| AC-DASH-4.1 | Satisfiable | Six months of spending, missing months are 0. |
| AC-DASH-5.1 | Satisfiable | Drill-down is the filtered list. Its total uses the same spending figure. |
| AC-BUD-1.1 | Satisfiable | Open period carries forward. |
| AC-BUD-1.2 | Satisfiable | A December change does not rewrite November's amount. |
| AC-BUD-1.3 | Satisfiable | Same money parser. ₹0 is rejected, not stored. |
| AC-BUD-1.4 | Satisfiable | Past months rejected. Current and future accepted. |
| AC-BUD-1.5 | Satisfiable | February starts at 1 February and does not change December. |
| AC-BUD-2.1 | Satisfiable | Informational warning. Save still commits. |
| AC-BUD-2.2 | Satisfiable | Budget changes write history. |
| AC-BUD-3.1 | Satisfiable | 6000 / 8000 is 75% exactly. |
| AC-BUD-3.2 | Satisfiable | Income ignored. Refunds reduce spent. |
| AC-BUD-3.3 | Satisfiable | Spent is computed, not stored. |
| AC-BUD-4.1 | Satisfiable | ₹7,999.99 ok, ₹8,000.00 approaching, on a ₹10,000 budget. |
| AC-BUD-4.2 | Satisfiable | Exactly the budget is only `at_limit`. One paisa over is `exceeded`. |
| AC-BUD-4.3 | Satisfiable | ₹3.33 with ₹2.66 is ok (26600 < 26640). ₹2.67 is approaching. Displayed 80.0 does not flip the state. The parenthetical "80.18%" in the criterion is the exact ratio, not the 1-decimal display. |
| AC-BUD-4.4 | Satisfiable | The save response includes the new state. |
| AC-BUD-4.5 | Satisfiable | State is recomputed after delete, re-date, or refund. |
| AC-BUD-4.6 | Satisfiable | No budget means no state and no division. A ₹0 budget cannot be saved (AC-BUD-1.3). |

## RPT

| ID | Verdict | Design |
|---|---|---|
| AC-RPT-1.1 | Satisfiable | A past month uses the dashboard computation. |
| AC-RPT-2.1 | Satisfiable | 1 Jan through 31 Dec. Sums in paise. |
| AC-RPT-2.2 | Satisfiable | The year is the calendar date, not `created_at`. |
| AC-RPT-4.1 | Satisfiable | FY 2026-27 is 1 Apr 2026 through 31 Mar 2027 inclusive. |
| AC-RPT-4.2 | Satisfiable | 31 Mar 2026 is FY 2025-26. 1 Apr 2026 is FY 2026-27. |
| AC-RPT-4.3 | Satisfiable | The preset only changes the window. |
| AC-RPT-3.1 | Satisfiable | Columns include type `income` / `expense` / `refund`. |
| AC-RPT-3.2 | Satisfiable, plus one manual Excel open | First three bytes `EF BB BF`. The byte check is CI. Excel is the manual half. |
| AC-RPT-3.3 | Satisfiable | Formula prefix, then RFC 4180 quoting. |
| AC-RPT-3.4 | Satisfiable | Same filter as the list. Refunds stay positive with their own type. |
| AC-RPT-3.5 | Satisfiable | Tombstones out. No network required. |

## SYNC

| ID | Verdict | Design |
|---|---|---|
| AC-SYNC-1.1 | Satisfiable | No token means no sync requests. |
| AC-SYNC-1.2 | Satisfiable | `private` must be true or sync stops. |
| AC-SYNC-1.3 | Satisfiable | Snapshot is ciphertext. Commit message is constant. A test can scan the plaintext of the repo fixture. |
| AC-SYNC-1.4 | Satisfiable | Disconnect deletes the token from secure storage and leaves SQLite. |
| AC-SYNC-1.5 | Satisfiable | Token only in `expo-secure-store`. Log denylist. |
| AC-SYNC-2.1 | Satisfiable | Different ids both survive. `domain.merge` test, no network. |
| AC-SYNC-2.2 | Satisfiable | Pull, merge, push. If the ref moved, repeat, at most 3 times. |
| AC-SYNC-2.3 | Satisfiable | Fast-forward when one side's revision is an ancestor. |
| AC-SYNC-2.4 | Satisfiable | A revision already in history is ignored. |
| AC-SYNC-2.5 | Satisfiable | One snapshot upload however large the outbox (ADR-014). |
| AC-SYNC-2.6 | Satisfiable | The merge tests are Node tests (ADR-018). |
| AC-SYNC-3.1 | Satisfiable | Concurrent revisions open a conflict and block the push. |
| AC-SYNC-3.2 | Satisfiable | Field-by-field result is a new revision with both parents. |
| AC-SYNC-3.3 | Satisfiable | The other phone fast-forwards onto that revision. |
| AC-SYNC-3.4 | Satisfiable | No timestamp winner. |
| AC-SYNC-3.5 | Satisfiable | Local saves continue. Only the push waits. |
| AC-SYNC-3.6 | Satisfiable | Delete versus edit is a conflict, not a silent winner. |
| AC-SYNC-4.1 | Satisfiable | Tombstone replicates. |
| AC-SYNC-4.2 | Satisfiable | Restore replicates. |
| AC-SYNC-4.3 | Satisfiable | An aged tombstone is absent from the next snapshot both phones pull. |
| AC-SYNC-4.4 | Satisfiable | Unedited local copy is dropped. A newer local edit is a choice. |
| AC-SYNC-5.1 | Manual | Restore from the repo with passphrase or recovery code. Once per release, on a phone. |
| AC-SYNC-5.2 | Satisfiable | Wrong passphrase does not write, and backoff applies. |
| AC-SYNC-5.3 | Satisfiable | No key and no recovery code: the message says the copy cannot be opened, and names CSV on a phone that still has the ledger. |
| AC-SYNC-6.1 | Satisfiable as a product target | The save returns before sync. The 300 ms figure is measured on a phone during dogfood, not in CI. |
| AC-SYNC-6.2 | Satisfiable | Outbox, backoff on 403/429, status line. |
| AC-SYNC-6.3 | Satisfiable | The outbox is committed with the save. A killed sync retries it. |
| AC-SYNC-6.4 | Satisfiable | Background sync skips if secure storage cannot yield the key. No background passphrase prompt. |

## ERS, OPS

| ID | Verdict | Design |
|---|---|---|
| AC-ERS-1.1 | Satisfiable | Confirmation copy is in the LLD, including the two-compaction warning. Never silent. |
| AC-ERS-1.2 | Manual plus tests | Orphan commit, new data key, old key wiped on that phone. A real repo check is once per release. |
| AC-ERS-1.3 | Satisfiable | One missed generation applies `purged_ids`. Two or more asks, and does not resurrect. Passphrase is required because the data key rotated. |
| AC-ERS-2.1 | Satisfiable | Live snapshot omits the row and its history. |
| AC-ERS-2.2 | Satisfiable | Privacy screen states that older commits remain until forget-history or repo deletion. |
| AC-ERS-3.1 | Satisfiable | CSV offered, then an explicit confirm. |
| AC-ERS-3.2 | Satisfiable | Delete the repo, drop the token, wipe local keys and ledger, return to first run. If the repo delete fails, do not wipe the only copy without saying the backup is still there. |
| AC-ERS-3.3 | Satisfiable | No repo: wipe local only. |
| AC-ERS-3.4 | Satisfiable | Other phones keep their ledger and offer CSV or a new connection. |
| AC-ERS-4.1 | Satisfiable | Privacy text covers GitHub retention, old versions, timing, and size. |
| AC-ERS-4.2 | Satisfiable | Privacy text says we run no server and hold none of those secrets. |
| AC-OPS-1.1 | Satisfiable | CSV always, from the local database. |
| AC-OPS-1.2 | Manual | Same drill as AC-SYNC-5.1. |
| AC-OPS-1.3 | Manual | Confirm the unopenable-copy message matches reality. |
| AC-OPS-2.1 | Satisfiable | Log denylist test. |
| AC-OPS-3.1 | Satisfiable in Phase 3 | The workflow is specified (ADR-018) and is not in the repo yet, because there is no project. |
| AC-OPS-3.2 | Satisfiable in Phase 3 | EAS internal APK, on demand, 15 Android builds a month, no silent upgrade. |
| AC-OPS-3.3 | Satisfiable | `.gitignore` and review rule. The token is a GitHub secret when it exists. |
| AC-OPS-3.4 | Manual | The APK is the acceptance build. Expo Go is not. |

## AI (Later — not in the first build)

Every AC-AI-* row is **Later**. The confirm-only shape is ADR-005. Nothing is built or tested for v1.0.

| ID | Note for when AI is opened |
|---|---|
| AC-AI-1.1 | Draft only. No row yet. |
| AC-AI-1.2 | Confirm uses the normal save, source `ai_assistant`. |
| AC-AI-1.3 | Cancel writes nothing. |
| AC-AI-1.4 | Edited draft fields are what get saved. |
| AC-AI-1.5 | Shorthand `2k`, `1.5L`, `350/-`. |
| AC-AI-1.6 | Invalid model output is not stored. |
| AC-AI-1.7 | Provider failure opens the manual form. |
| AC-AI-1.8 | Instructions in the text cannot delete. |
| AC-AI-1.9 | Test counts writes without Confirm. Must be 0. |
| AC-AI-1.10 | Duplicate warning, double-save, and the card/EMI note apply. |
| AC-AI-2.1 | Missing amount blocks confirm. |
| AC-AI-2.2 | Missing category is not a silent guess. |
| AC-AI-2.3 | Income stays a visible choice. |
| AC-AI-2.4 | **Pushback.** The criterion still allows drafting the first of several expenses. DEV still wants no draft at all. Not a v1 issue. |
| AC-AI-3.1 | Answer quotes the computed Food total and links to the list. |
| AC-AI-3.2 | Every number in the answer matches a computed figure. |
| AC-AI-3.3 | Unanswerable questions get no invented number. |
| AC-AI-3.4 | No rows means ₹0.00. |
| AC-AI-3.5 | Query tools are read-only. |
| AC-AI-4.1 | Period compare uses the dashboard figures. |
| AC-AI-4.2 | A bare month name states the assumed year. |
| AC-AI-5.1 | Budget answer matches the integer state. |
| AC-AI-5.2 | No budget is created by the question. |
| AC-AI-6.1 | Off by default. No provider call. |
| AC-AI-6.2 | Disclosure names the typed sentence, not the ledger. |
| AC-AI-6.3 | AI logs omit amounts and notes. |
| AC-AI-6.4 | A missed accuracy bar leaves the ledger alone. |

## Count

191 of 191 v0.2 criteria are in the tables above. AUTH-1, AUTH-2, AUTH-3, and AUTH-7 have no rows, on purpose.

## DEV pushback (does not block the v0.2 docs)

1. **A phone clock set forward can expire a delete early** (AC-TXN-4.5). We will not call a time server. The safe half is implemented: a `deleted_at` in the future is kept. The unsafe half is accepted as a limit.
2. **AC-AI-2.4** still permits drafting the first of several expenses. When AI is someday built, DEV wants zero drafts for that message. Not a v1 issue.
3. **Minimum Android version** is still marked open in the PRD. DEV's answer is API 26, because that is the Keystore behaviour `expo-secure-store` should rely on. It can move in Phase 3 if Expo's current SDK requires newer.
4. **ADR-014** is still Proposed. v0.2 already says to approve it with Phase 3. I am not reopening it. One snapshot per sync is what keeps 500 offline saves inside GitHub's content limits and what the compaction screen rewrites.
5. **AC-BUD-4.3's "80.18%"** is the unrounded ratio. The on-screen percent is still one decimal place. Tests should assert the integer state, not that glyph.
