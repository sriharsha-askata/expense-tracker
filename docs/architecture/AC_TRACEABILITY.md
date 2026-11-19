# Acceptance-criteria traceability

Status: **Proposed**
Date: 2026-11-19
Purpose: give the PM a concrete pass over "can this design satisfy each MVP criterion?" A row marked **Satisfiable** means the proposed design has a place to enforce it. It does not mean the test exists or has run. Rows marked **Pending** depend on an open product decision; the design can go either way, and the default is named.

Source criteria: `docs/product/ACCEPTANCE_CRITERIA.md`.

| Criterion | Design home | Verdict |
|---|---|---|
| AC-AUTH-1.1 | Signup, verification token, financial routes check `email_verified_at` when required | **Pending TQ-4** for whether verification is on. The mechanism is in the API spec. |
| AC-AUTH-1.2 | Generic signup response; hash work on both branches | Satisfiable |
| AC-AUTH-1.3 | Password policy in the service, before insert | Satisfiable |
| AC-AUTH-1.4 | Seed categories and defaults in the verify transaction | Satisfiable. On a private deploy, the same seed runs at the end of signup. |
| AC-AUTH-2.1 | Login service, generic failure | Satisfiable |
| AC-AUTH-2.2 | `login_attempts` rolling window, 5 / 15 min | Satisfiable |
| AC-AUTH-2.3 | Server-side revoke, cookie cleared, RLS returns nothing | Satisfiable |
| AC-AUTH-2.4 | Server-side idle and absolute expiry | **Pending TQ-3.** A 7-day idle timeout still makes a session dead at 30 days of inactivity. It does **not** keep the session alive for 29 days. If the criterion is later read as "valid until day 30," the default changes. |
| AC-AUTH-3.1 | Hashed single-use reset token, 30 min | Satisfiable, needs a mail path when verification is on |
| AC-AUTH-3.2 | Reset revokes all sessions | Satisfiable |
| AC-AUTH-4.1 | RLS plus 404 when the select is empty | Satisfiable |
| AC-AUTH-4.2 | Same policies on dashboard, list, reports, budgets, export, and AI tools | Satisfiable. AI tool case lands in Phase 5. |
| AC-AUTH-4.3 | Composite foreign key `(category_id, user_id, type)` | Satisfiable |
| AC-AUTH-4.4 | Tools take the user id from the server context only | Satisfiable in the Phase 5 module. No write tool exists. |
| AC-AUTH-5.1 | `date` column; today computed in Asia/Kolkata | Satisfiable. The 00:30 IST case is a fixed-clock test. |
| AC-AUTH-5.2 | Timezone is a user column. Transaction dates are not rewritten. | Satisfiable |
| AC-AUTH-6.1 | Password re-entry, session revoke, purge function within 30 days, export is a separate call the UI offers first | Satisfiable. Backup ageing is operational, Phase 6. |
| AC-AUTH-6.2 | Purge deletes by `user_id` only | Satisfiable |
| AC-TXN-1.1 | Create service, minor units, response includes the row | Satisfiable. "Without reload" is a client cache update from the response. |
| AC-TXN-1.2 | Integer sums. `10 + 20 = 30` minor units. Property test over minor units. | Satisfiable |
| AC-TXN-1.3 | Parser rejects scale greater than 2. No rounding path. | Satisfiable |
| AC-TXN-1.4 | Checks for non-positive and for `> 100000000.00` | Satisfiable. Exactly `100000000.00` is accepted. |
| AC-TXN-1.5 | Client strips Indian or western grouping; API accepts `123456.50`; formatter emits ₹1,23,456.50 | Satisfiable. The API itself rejects commas, on purpose. |
| AC-TXN-1.6 | Composite foreign key includes type | Satisfiable |
| AC-TXN-1.7 | `category_id` required | Satisfiable |
| AC-TXN-1.8 | Service rejects dates more than 1 year ahead. `upcoming` is derived, not stored. | Satisfiable |
| AC-TXN-1.9 | Length check on code points. React text rendering, no HTML insertion. | Satisfiable |
| AC-TXN-1.10 | API rejects currency other than INR. Column still stores the code. | Satisfiable |
| AC-TXN-2.1 | `type=income` excluded from expense sums. `net = income - expense`. | Satisfiable. The field is named `net` (**pending TQ-5** for the UI word). The figure matches AC-DASH-1.1. |
| AC-TXN-3.1 | Update plus aggregates read committed rows | Satisfiable |
| AC-TXN-3.2 | Aggregates filter on `transaction_date` | Satisfiable |
| AC-TXN-3.3 | `version` predicate, 409 returns `current`, no write | Satisfiable |
| AC-TXN-3.4 | Shared validator for create and update | Satisfiable |
| AC-TXN-3.5 | One database transaction for the update and the audit insert | Satisfiable |
| AC-TXN-4.1 | `deleted_at` excluded from every read model listed, including AI tools and export | Satisfiable |
| AC-TXN-4.2 | Restore clears `deleted_at` inside the window | Satisfiable. Trash list is the way the user finds the row. |
| AC-TXN-4.3 | Daily definer function, 30 days | Satisfiable. Not a queue. |
| AC-TXN-4.4 | Audit actions `transaction.deleted` and `transaction.restored` | Satisfiable |
| AC-TXN-5.1 | Match on type, amount, category, date, created within 5 minutes by the database clock | Satisfiable |
| AC-TXN-5.2 | Outside the window, or any matched field different, no warning | Satisfiable. Payment method is intentionally not a match field. |
| AC-TXN-5.3 | `confirm_duplicate` with a new idempotency key inserts a second row. No merge. | Satisfiable |
| AC-TXN-5.4 | Confirm uses the same create service | Satisfiable |
| AC-TXN-6.1 | One key per submit, unique constraint, no automatic POST retry | Satisfiable |
| AC-TXN-6.2 | Completed key replays the stored response | Satisfiable |
| AC-TXN-6.3 | Hash mismatch returns `IDEMPOTENCY_KEY_REUSED` | Satisfiable |
| AC-TXN-7.1 | Audit list ordered by `created_at`, UTC stored, displayed in the user zone by the client | Satisfiable. Create, two edits, and delete are four events. |
| AC-TXN-7.2 | No route, no grant, trigger rejects update and delete for the app role | Satisfiable |
| AC-TXN-7.3 | RLS on `audit_events` | Satisfiable |
| AC-CAT-1.1 | Seed list copied verbatim from FR-CAT-1, per user | Satisfiable |
| AC-CAT-2.1 | Create category | Satisfiable |
| AC-CAT-2.2 | Unique index on lower trimmed name per user and type | Satisfiable |
| AC-CAT-2.3 | Transactions store `category_id`. Rename does not touch amounts. | Satisfiable |
| AC-CAT-3.1 | `archived_at` hides the picker and not the history | Satisfiable |
| AC-CAT-3.2 | `ON DELETE RESTRICT` from transactions, including soft-deleted | Satisfiable |
| AC-CAT-3.3 | Archive closes the budget period at next month start | Satisfiable |
| AC-LIST-1.1 | Keyset pagination, 50 default, order date then created_at then id | Satisfiable |
| AC-LIST-1.2 | Empty state is a client concern when `items` is empty and totals count is 0 | Satisfiable |
| AC-LIST-2.1 | Same filter for rows and for the aggregate | Satisfiable |
| AC-LIST-2.2 | Inclusive dates via half-open `[from, to + 1 day)` | Satisfiable |
| AC-LIST-2.3 | Case-insensitive note match, wildcards escaped | Satisfiable |
| AC-LIST-2.4 | Partial index and default filter `deleted_at is null` | Satisfiable |
| AC-DASH-1.1 | Integer subtraction. Display rate half-up to 1 dp. 32659.25 / 85000 → 38.4 | Satisfiable for the numbers. Label **pending TQ-5**. |
| AC-DASH-1.2 | `net_rate_display` null when income is 0 | Satisfiable |
| AC-DASH-1.3 | Negative `net` decimal string | Satisfiable |
| AC-DASH-1.4 | Month window on `transaction_date` | Satisfiable |
| AC-DASH-2.1 | Sum of category minor units equals expense minor units. Shares not fudged. | Satisfiable |
| AC-DASH-2.2 | Breakdown query does not filter out archived categories | Satisfiable |
| AC-DASH-3.1 | Previous month aggregate, absolute delta as a decimal string | Satisfiable |
| AC-DASH-3.2 | `expense_change = new` when previous is 0 and current is not | Satisfiable |
| AC-DASH-4.1 | Six months generated in the calendar domain, left join to sums, missing is 0 | Satisfiable |
| AC-DASH-5.1 | Client links to the filtered list. Same aggregate as the breakdown amount. | Satisfiable |
| AC-BUD-1.1 | Open period with `effective_from` in November resolves in December | Satisfiable |
| AC-BUD-1.2 | December change inserts a new period and closes the old one at 1 Dec. November's amount is untouched. | Satisfiable |
| AC-BUD-1.3 | Same money parser and bounds | Satisfiable |
| AC-BUD-2.1 | `category_sum_exceeds_overall` is informational. PUT still saves. | Satisfiable |
| AC-BUD-2.2 | Budget audit actions in the same commit | Satisfiable |
| AC-BUD-3.1 | Spent, remaining, and a display percent from minor units. 6000/8000 → 75 exactly. | Satisfiable |
| AC-BUD-3.2 | Expense filter on the spent query | Satisfiable |
| AC-BUD-3.3 | Spent is computed on read, not stored, so edit, delete, and re-date are visible on the next read and in the write response | Satisfiable |
| AC-BUD-4.1 | Integer comparison `spent * 100 >= budget * 80`. The badge follows the state, not the rounded percent. | Satisfiable, with the documented edge where the rounded glyph says 80.0 and the state is still `ok` |
| AC-BUD-4.2 | Exactly equal → `at_limit`. Greater → `exceeded` with overspend. | **Pending TQ-8** for the word. The inequalities match the criterion. |
| AC-BUD-4.3 | `budget_warnings` on the write response | Satisfiable. Not a stored notification. |
| AC-BUD-4.4 | State recomputed after delete | Satisfiable |
| AC-RPT-1.1 | Past month uses `GET /dashboard?month=` | Satisfiable. One formula. |
| AC-RPT-2.1 | Year query, twelve months, sums in minor units | Satisfiable |
| AC-RPT-2.2 | Year bounds on `transaction_date`, not on `created_at` | Satisfiable |
| AC-RPT-3.1 | CSV columns and formats as specified. UTF-8, no BOM. | Satisfiable |
| AC-RPT-3.2 | Prefix formula-like cells, then RFC 4180 quoting | Satisfiable |
| AC-RPT-3.3 | Export uses the list filter. Sum of the amount column matches `totals`. | Satisfiable |
| AC-RPT-3.4 | `deleted_at is null`, RLS | Satisfiable |
| AC-AI-1.1 | `POST /ai/drafts` writes nothing and returns the draft | Satisfiable in Phase 5 |
| AC-AI-1.2 | Confirm is `POST /transactions` with `source=ai_assistant` | Satisfiable |
| AC-AI-1.3 | No draft table. Cancel does not call create. | Satisfiable |
| AC-AI-1.4 | Confirm body is the edited fields, not the model payload | Satisfiable |
| AC-AI-1.5 | Shorthand list in the LLD: `2k`, `1.5L`, `350/-` | Satisfiable. The draft shows the resolved amount. |
| AC-AI-1.6 | Invalid suggestion becomes `missing_fields` or a rejected confirm. Confirm runs the normal validator. | Satisfiable |
| AC-AI-1.7 | `503` with `fallback_note`, no insert | Satisfiable |
| AC-AI-1.8 | No write tool. Draft endpoint does not insert. | Satisfiable |
| AC-AI-1.9 | Test counts transactions before and after a suite that never calls confirm | Satisfiable |
| AC-AI-1.10 | Same create service | Satisfiable |
| AC-AI-2.1 | Null amount, `confirm_enabled` false | Satisfiable |
| AC-AI-2.2 | `category_uncertain`, category null, not a silent Food guess | Satisfiable. The criterion also allows "Other"; the design chooses null so the user must pick. |
| AC-AI-2.3 | Type returned on the draft, user confirms. UI shows the type. | Satisfiable |
| AC-AI-2.4 | 422, no draft | **Pending D6.** Matches the PM's stated preference. |
| AC-AI-3.1 | `category_total` payload plus `transactions_query` | Satisfiable |
| AC-AI-3.2 | Number check. UI treats `structured` as the answer. | Satisfiable |
| AC-AI-3.3 | `kind=unsupported`, no invented amount | Satisfiable |
| AC-AI-3.4 | Tool returns `"0.00"` when the sum is null | Satisfiable |
| AC-AI-3.5 | Tool registry is the four read functions | Satisfiable |
| AC-AI-4.1 | `compare_periods` uses the same aggregate as the dashboard | Satisfiable |
| AC-AI-4.2 | Year resolution rule in the API spec, `assumed_year` returned | Satisfiable |
| AC-AI-5.1 | `budget_status` uses `domain.budget` | Satisfiable |
| AC-AI-5.2 | `state=none`, `set_budget: true`, no insert | Satisfiable |
| AC-AI-6.1 | `ai_enabled` false short-circuits before any provider client is built | Satisfiable |
| AC-AI-6.2 | Disclosure timestamp required before enable | Satisfiable |
| AC-AI-6.3 | Log fields listed in the LLD. Test scans the log fixture. | Satisfiable |
| AC-OPS-1.1 | Daily encrypted `pg_dump`, 7 copies | **Pending PO-Q4** for who runs it. The design is one database, which is dumpable. |
| AC-OPS-1.2 | Restore drill is a Phase 6 procedure, not a schema feature | Pending the drill itself. Not claimed done. |
| AC-OPS-2.1 | Log denylist and a fixture test | Satisfiable |
| AC-OPS-2.2 | Error handler without the request body | Satisfiable. The vendor for error tracking is not chosen; logs are enough until Phase 6. |

## Counts

Every criterion in `ACCEPTANCE_CRITERIA.md` is in the table above. None are marked done. Pending rows are: AC-AUTH-1.1, AC-AUTH-2.4, AC-BUD-4.2, AC-AI-2.4, AC-OPS-1.1, AC-OPS-1.2, plus the label note on AC-TXN-2.1 and AC-DASH-1.1.

The PM's phase-2 exit, "every MVP AC is satisfiable," is ready to review. I am not signing it on the PM's behalf.
