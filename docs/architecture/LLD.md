# Low-Level Design

Status: **Accepted** for the product rules below. Phase 3 waits for an explicit go-ahead.
Date: 2026-11-19
Supersedes: the server LLD. The money, date, and budget-state rules are the same. The place they run is the phone. The language is TypeScript, not Kotlin.

## 1. Modules

| Module | Does | Must not |
|---|---|---|
| `domain.money` | Parse a decimal string to a `bigint`, format it, add `bigint`s | Touch React Native, SQLite, or `number` |
| `domain.calendar` | Today in an IANA zone, month windows, upcoming | Store anything |
| `domain.budget` | State from two `Long`s | Read the rounded percent |
| `domain.merge` | Pure function from local rows, remote snapshot, and parents to a merge result | Call GitHub |
| `domain.csv` | BOM, quoting, formula prefix | Read the network |
| `db` | SQLCipher, one writer, the save transaction | Encrypt for GitHub (that is the sync module's job, so a bug here cannot skip it) |
| `sync` | Pull, decrypt, call `domain.merge`, encrypt, push | Be on the save path |
| `ai` | Not in the first build. The folder stays empty. | Import the save function until a later phase adds a confirm step |

`domain.merge` and `domain.money` are Node tests with no device, no emulator, and no GitHub. See ADR-018.

## 2. Save transaction

Single SQLite connection, WAL, one transaction at a time. The UI awaits that transaction and nothing else.

`mutation_id` is created when the user commits a gesture, not when the screen opens, so a cancelled form has no id. A retry of a failed disk write reuses it. A second tap while the first transaction is in flight reuses it and the unique index returns the first row.

Revision: `new_rev` is a random UUID. Parent is the row's current `rev`, or none for an insert. `history` stores before/after with amounts as decimal strings.

The duplicate-warning query is inside the same transaction, so two taps cannot both miss it. It looks only at rows created on **this phone** in the last five minutes (same type, amount, category, and calendar date). It does not look at the other phone. Two phones can each log a ₹40 chai inside that window with no warning, and after sync both rows exist (AC-TXN-5.4). They are different ids, so they are not a conflict. Repeat uses the same check (AC-TXN-5.5).

If the edit form was opened against revision R and the row on this phone is no longer R (a sync applied a newer revision while the form was open), Save does not write. It shows the current values and asks (AC-TXN-3.3). The same screen is the two-phone conflict screen.

## 3. Merge

Inputs: local rows (including tombstones), remote snapshot rows, and the parent map. Output: rows to upsert, conflicts to store, ids to drop because `purged_ids` names them and the local rev is one of the purged revs.

For each entity id present on either side:

1. If the remote rev equals the local rev, do nothing.
2. If the local rev is an ancestor of the remote rev (walk parents), apply the remote row.
3. If the remote rev is an ancestor of the local rev, keep local. It will be pushed.
4. If either rev is already recorded as applied (history has that rev), ignore it. Replays are normal.
5. Otherwise the revs are concurrent. Store a conflict containing both full rows. Do not modify the local entity. Do not push.

Ancestor walk is bounded by the parent map inside the union of local `rev_parent` and the snapshot's `rev_parents`. A rev with no path is concurrent, not "newer." There is no timestamp winner.

Tombstone versus a live concurrent edit is step 5, not a special overwrite.

Resolution: the user builds the chosen field set. The new rev has both conflicting revs as parents. That row is a normal edit and can be pushed. Both phones, once they have the resolution, see their own rev as an ancestor and fast-forward.

### Purge, one compaction, and two compactions

Each snapshot carries `compaction_generation` (an integer, starting at 0) and `purged_ids` (entity ids removed by the **latest** compaction only). The phone stores the generation it last applied.

**The other phone synced after the previous forget-history** (remote generation is exactly one ahead):

- If this phone's copy of that id is the revision that was purged, or an ancestor of it, delete it locally, including history, and do not put it in the outbox. An unedited copy does not come back (AC-SYNC-4.4, AC-ERS-1.3).
- If this phone edited it after the revision that was purged, the purge did not know about that edit. Show both: "forgotten on the other phone" and "edited here". Do not push until the user chooses (AC-SYNC-4.4).

**The other phone stayed offline across two forget-history actions** (remote generation is more than one ahead). The latest `purged_ids` list no longer names what the first compaction removed. Absence from the new snapshot is not proof the row was forgotten, and it is not proof it should be uploaded again.

The phone must not push those local-only rows. It lists them: "This phone missed a history reset. These entries are on this phone and not in the backup. Keep them (they go back into the backup) or drop them?" Nothing is uploaded and nothing is deleted until the user answers. That is the resurrection edge case. Silent upload would bring a forgotten entry back. Silent delete would destroy an edit made on the offline phone. The screen exists so neither happens by itself.

**Compaction confirmation copy**, shown before the squash runs, and never skipped:

> This rewrites the backup so only the current ledger remains, and creates a new data key. Other phones will need your passphrase the next time they sync. GitHub may still keep private copies we cannot delete.
>
> If another phone has not synced since the last time you forgot old versions, sync it first. Doing this twice while a phone stays offline does not put forgotten entries back by itself: that phone will ask you whether to keep or drop the entries it still has.

Day-to-day 30-day expiry does not increment `compaction_generation`. It only drops the row from the live snapshot. A phone that was offline and still holds an unedited copy of that row drops it on the next sync because the tombstone's age is in the snapshot it pulls (AC-SYNC-4.3). Expiry uses the phone's current calendar date compared with `deleted_at` plus 30 days, on next launch. If `deleted_at` is after that date (the clock moved backwards), the row is kept. A clock set **forward** can expire a row early; there is no trusted clock without a server. The privacy text does not need to dwell on this. The rule is in this section so the test for AC-TXN-4.5 does not invent a network time.

**Delete account on one phone** (AC-ERS-3.4): that phone deletes the repo, drops the token, and wipes its own ledger and keys. Other phones are not wiped remotely. On their next sync the repo is gone. They keep the local ledger, say the backup is gone, and offer CSV export or a new connection. They do not erase themselves.

## 4. Dates and budgets

Unchanged from the rules that still stand.

- Today is the calendar day in the user's IANA zone (`Asia/Kolkata` by default), computed with `Intl` or an explicit timezone library, not by slicing a UTC timestamp.
- A month is `[first, first of next)`.
- Upcoming is `transaction_date > today`. It is not a column.
- A past month is rejected (`BUDGET_PERIOD_LOCKED`). The current month and any future month are editable (AC-BUD-1.4, AC-BUD-1.5).
- Setting month M (current or future): leave every period that ends at or before M untouched. Close the period that covers M at M's first day if it started earlier. Remove periods that start on or after M. Insert one open period from M. That period carries forward until a later edit. Setting February while it is December does not change December. It does replace a March period that had already been set, because a change applies to the chosen month and later months (FR-BUD-1).
- The year report defaults to 1 January through 31 December. The Indian FY preset is 1 April through 31 March, labeled `FY 2026-27` when it starts in April 2026. Both are date windows over the same rows.
- The dashboard title is **Net**, with the formula visible. Spending, the 6-month trend, and month-over-month category comparison use expenses minus refunds. The AC-DASH-3.1 example has no refunds, so its +₹10,000 still holds. Income is compared on its own.
- Archive of a category sets the open budget's `effective_to` to the first day of next month.
- State machine is the integer one in the schema doc: `ok`, `approaching`, `at_limit`, `exceeded`.

## 5. Refunds (Accepted)

- `spent(category, month) = sum(expense.amount_minor) - sum(refund.amount_minor)` for live rows of that category dated in the month.
- A refund of a category the user did not spend on can make `spent` negative. Show it as negative, do not clamp. Clamping would hide money.
- Income totals ignore refunds.
- `net = income - expense + refund`.
- A refund cannot use an income category.
- Overall budget `spent` uses the same subtraction across all expense categories.

## 6. Card and EMI guidance (Accepted)

Not a validator. On save, if the category is `EMI/Loan`, or the payment method is `card` and the note or category looks like a bill payment (a fixed list of hints: "card bill", "credit card payment", "cc payment"), show a dismissible note:

> If you already recorded the original purchase or the individual spends, recording the card payment or the full price as well counts the money twice. Transfers between your own accounts are not expenses.

Do not block the save. Do not try to detect this with a model.

## 7. CSV

`domain.csv` writes the BOM, then the header, then rows. Tests open the bytes, assert the first three are `EF BB BF`, and assert a note of `=1+1` round-trips as a formula-inert cell. Accepted.

## 8. Repeat (Accepted)

The action is a save with a new id and today's date. It does not copy `rev`, `deleted_at`, or `created_at`. The duplicate warning applies, so repeating within five minutes asks first.

## 9. Passphrase unlock

- Derive KEK with Argon2id using the salt in the manifest (or a new salt on first create).
- Unwrap DEK. On failure, show "That passphrase does not open this ledger." Wait `min(30 seconds, 2^(failures-1) seconds)` before accepting another try. Never delete data because of failures. Never stop accepting tries.
- On success, failures reset, DEK goes to the keystore.
- First create: generate DEK, generate recovery code, wrap both ways, show the recovery code, require an explicit "I have stored this" before continuing. If they decline, record that the recovery wrap is absent and show "If you forget the passphrase, this ledger cannot be recovered."

## 10. Sync worker

Sync runs when the app is open: on launch, about 30 seconds after the last save, and when the user taps Sync. A background task (`expo-background-task`) may also run, and it is best-effort. Android can kill it. The outbox survives. If secure storage cannot produce the DEK, skip sync. Do not ask for the passphrase from a background task.

## 11. AI module (not in the first build)

Do not create the provider client in Phase 3 or Phase 4. When the feature is opened later, the same safety rules apply:

- No insert function is reachable from the provider client.
- Confirm builds a normal save.
- Daily cap stored in SQLite (proposed default 30 drafts, 60 questions) so a loop does not spend the user's money. The cap is local and easy to reset by reinstalling; that is acceptable because it is the user's own key and the user's own bill. Say that.
- Logs: latency and a success bit. Not the sentence, not the amount.

## 12. Tests that matter before any UI polish

- `0.10 + 0.20` is 30 minor units, displayed `0.30`.
- `100.555` is rejected.
- Schema dump contains no `REAL`.
- Merge: two creates, different ids, both present.
- Merge: edit and edit from the same parent, conflict, no row change, push function not called.
- Merge: resolution with two parents fast-forwards the other side.
- Tombstone excludes the row from the spent sum; restore includes it.
- Snapshot canonical JSON has sorted keys and string amounts; a test encrypts and decrypts.
- Manifest fixture has no key named amount, note, or category.
- Wrong authentication tag does not modify SQLite.
- CSV starts with the BOM.
- A financial year window includes 1 April and 31 March and excludes 31 March of the previous year and 1 April of the following year.
- A budget write for a past month is rejected.

These tests run under Node on GitHub Actions (ADR-018). No test in this phase has been written or run.
