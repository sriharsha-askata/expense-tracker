# Low-Level Design

Status: **Proposed**
Date: 2026-11-19
Supersedes: the server LLD. The money, date, and budget-state rules are the same. The place they run is the phone.

## 1. Modules

| Module | Does | Must not |
|---|---|---|
| `domain.money` | Parse a decimal string to a `Long`, format a `Long`, add `Long`s | Touch Android, SQLite, or `Double` |
| `domain.calendar` | Today in an IANA zone, month windows, upcoming | Store anything |
| `domain.budget` | State from two `Long`s | Read the rounded percent |
| `domain.merge` | Pure function from local rows, remote snapshot, and parents to a merge result | Call GitHub |
| `domain.csv` | BOM, quoting, formula prefix | Read the network |
| `db` | SQLCipher, one writer, the save transaction | Encrypt for GitHub (that is the sync module's job, so a bug here cannot skip it) |
| `sync` | Pull, decrypt, call `domain.merge`, encrypt, push | Be on the save path |
| `ai` | Optional, undecided. Draft only. | Import the function that inserts a transaction, except through the same save function the form uses, after confirm |

`domain.merge` and `domain.money` are JVM tests with no device and no GitHub.

## 2. Save transaction

Single SQLite connection, WAL, one transaction at a time. The UI awaits that transaction and nothing else.

`mutation_id` is created when the user commits a gesture, not when the screen opens, so a cancelled form has no id. A retry of a failed disk write reuses it. A second tap while the first transaction is in flight reuses it and the unique index returns the first row.

Revision: `new_rev` is a random UUID. Parent is the row's current `rev`, or none for an insert. `history` stores before/after with amounts as decimal strings.

The duplicate-warning query is inside the same transaction, so two taps cannot both miss it. It does not take a GitHub lock. Two phones can each record a ₹40 chai without a warning; that is a real second purchase unless the user later calls it a conflict, which they will not, because the ids differ. The five-minute warning is per device. Say so in the UI only if we discover it confuses people. Do not block the second phone.

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

`purged_ids`: if the local entity's current rev is listed, delete it locally, including history, and do not put it back in the outbox. If the local entity has a newer rev that is not listed, keep it and let the push offer it; the other phone may conflict, which is safer than dropping an edit the compaction did not know about.

## 4. Dates and budgets

Unchanged from the rules that still stand.

- Today is `LocalDate.now(ZoneId.of(timezone))`.
- A month is `[first, first of next)`.
- Upcoming is `transaction_date > today`. It is not a column.
- Setting this month's budget updates the open row if it already starts this month, or closes the previous open row at this month and inserts a new one.
- A past month is rejected (`BUDGET_PERIOD_LOCKED`) until the product decision changes.
- Archive of a category sets the open budget's `effective_to` to the first day of next month.
- State machine is the integer one in the schema doc. The four names `ok`, `approaching`, `at_limit`, `exceeded` are the ones the PM restated with this direction. The earlier naming question is closed.

## 5. Refunds (Proposed)

If the Product Owner accepts them:

- `spent(category, month) = sum(expense.amount_minor) - sum(refund.amount_minor)` for live rows of that category dated in the month.
- A refund of a category the user did not spend on can make `spent` negative. Show it as negative, do not clamp. Clamping would hide money.
- Income totals ignore refunds.
- `net = income - expense + refund`.
- A refund cannot use an income category.
- Overall budget `spent` uses the same subtraction across all expense categories.

If the Product Owner rejects them, refunds do not exist and a returned amount is an income row, which the app will then mis-state in the category breakdown. That rejection should be explicit, because the analyst's point is that the income treatment is the bug.

## 6. Card and EMI guidance (Proposed)

Not a validator. On save, if the category is `EMI/Loan`, or the payment method is `card` and the note or category looks like a bill payment (a fixed list of hints: "card bill", "credit card payment", "cc payment"), show a dismissible note:

> If you already recorded the original purchase or the individual spends, recording the card payment or the full price as well counts the money twice. Transfers between your own accounts are not expenses.

Do not block the save. Do not try to detect this with a model.

## 7. CSV

`domain.csv` writes the BOM, then the header, then rows from a cursor so a large export is not one giant string. Tests open the bytes, assert the first three are `EF BB BF`, and assert a note of `=1+1` round-trips as a formula-inert cell. This is Proposed until the Product Owner confirms the BOM.

## 8. Repeat (Proposed)

The action is a save with a new id and today's date. It does not copy `rev`, `deleted_at`, or `created_at`. The duplicate warning applies, so repeating within five minutes asks first.

## 9. Passphrase unlock

- Derive KEK with Argon2id using the salt in the manifest (or a new salt on first create).
- Unwrap DEK. On failure, show "That passphrase does not open this ledger." Wait `min(30 seconds, 2^(failures-1) seconds)` before accepting another try. Never delete data because of failures. Never stop accepting tries.
- On success, failures reset, DEK goes to the keystore.
- First create: generate DEK, generate recovery code, wrap both ways, show the recovery code, require an explicit "I have stored this" before continuing. If they decline, record that the recovery wrap is absent and show "If you forget the passphrase, this ledger cannot be recovered."

## 10. Sync worker

WorkManager, network connected, not expedited. Also a manual action. The worker runs the protocol in `API_SPECIFICATION.md`. It does not hold a wake lock beyond one attempt. If the app is killed after the SQLite commit and before the push, the outbox is still there and the next run pushes it.

The worker reads the DEK from the keystore. If the keystore cannot produce it (reboot policy, user locked the device in a way that blocks the key), skip sync. Do not ask for the passphrase from a background task. The next foreground open can ask.

## 11. AI module (undecided)

Same safety rules as ADR-005, on the device:

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
- CSV starts with the BOM once that proposal is accepted; until then the test is marked pending so we do not pretend the Product Owner agreed.

No test in this phase has been written or run.
