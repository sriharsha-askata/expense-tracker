# Low-Level Design

Status: **Proposed**
Date: 2026-11-19
Depends on: `HLD.md`, `DATABASE_SCHEMA.md`, `API_SPECIFICATION.md`

Algorithms and module rules for the first implementation. Where a product question is still open, the behaviour below is the proposed default and is labelled pending.

## 1. Modules

| Module | Owns | Must not |
|---|---|---|
| `domain.money` | Parse a decimal string to minor units, format minor units to a 2-decimal string, add minor units | Import the ORM, format with locale, call `float` |
| `domain.calendar` | Month bounds, year bounds, "today" from an IANA zone, upcoming predicate | Store anything |
| `domain.budget` | State from two integers: spent and budget | Round the state off a displayed percent |
| `domain.csv` | RFC 4180 encoding and formula-injection prefix | Query the database |
| `services.*` | One use case, one database transaction | Return another user's row, catch and ignore a version conflict |
| `db` | Session, `set_config('app.user_id', ..., true)`, models | Encode product rules that belong in `domain` |
| `ai` | Call the provider, validate the suggestion, execute read tools | Insert into `transactions` |

Routers translate HTTP to a service call and a service result to HTTP. They do not contain arithmetic.

## 2. Money

### 2.1 Representation

INR minor unit is the paisa. `1 INR = 100` minor units. The exponent `2` lives in code next to the currency code until a second currency exists. Unknown currencies are rejected.

Database column: `amount_minor bigint`, check `> 0` and `<= 10000000000` (₹100,000,000.00, which is ₹10,00,00,000.00). The upper bound is inclusive, matching "reject **greater than** 10,00,00,000.00".

API field: string. Canonical form on output is always two decimal places (`"350.00"`, `"0.30"`).

### 2.2 Parsing

Accept, after trimming, a string that matches `^(0|[1-9][0-9]*)(\.[0-9]{1,2})?$`.

- `"350"` → 35000 minor
- `"350.5"` → 35050 minor
- `"350.50"` → 35050 minor
- `"0.10"` → 10 minor
- `"0"`, `"0.00"`, negative, `"100.555"`, `"1e2"`, `"1,23,456.50"`, `"₹350"`, JSON number, JSON null → reject

Commas and the rupee sign are a client concern. The client strips a valid Indian grouping (`1,23,456.50`) or a standard grouping of thousands (`123,456.50`) and sends `123456.50`. Invalid groupings (`1,2,3`) never leave the form. The API's strict pattern is what the server trusts.

Parsing uses a decimal type, then multiplies by 100 and requires an integer result. It does not use binary float.

### 2.3 Arithmetic

Sums are SQL `sum(amount_minor)` returning `bigint`, coalesced to 0. Net for a set of income and expense totals is `income_minor - expense_minor`, which may be negative. Subtraction of two integers cannot drift.

Display percentages, and only display percentages, use a decimal library:

```text
percent_1dp = (Decimal(numerator_minor) / Decimal(denominator_minor) * 100)
              .quantize(Decimal("0.1"), rounding=ROUND_HALF_UP)
```

If the denominator is 0, the display value is null and the UI shows "—" for a savings rate, or "new" for a month-over-month change when the current numerator is non-zero, or "flat" when both are zero. No `float8` in these queries.

Category share percentages are each rounded independently. They are not adjusted so the set sums to 100.

### 2.4 Budget state

```text
if spent_minor * 100 >= budget_minor * 80:
    crossed_80 = true
exceeded   = spent_minor > budget_minor
at_limit   = spent_minor == budget_minor
```

| Condition | State |
|---|---|
| budget missing | `none` |
| not crossed_80 | `ok` |
| exceeded | `exceeded` |
| at_limit | `at_limit` |
| else | `approaching` |

`at_limit` is the proposed label for exactly 100% (pending TQ-8). The overspend amount is `spent_minor - budget_minor` when exceeded, else 0. Remaining is `budget_minor - spent_minor` and may be negative; the UI shows the negative remaining as overspend together with the exceeded state.

Worked edge: budget 10001 minor (₹100.01), spent 8000 (₹80.00). `8000 * 100 = 800000`, `10001 * 80 = 800080`. Not crossed. Displayed ratio half-up is 80.0. State stays `ok`. This is intentional.

Income rows are excluded from `spent_minor`. Soft-deleted rows are excluded. The month is the calendar month of `transaction_date`.

Warnings are not rows. Create, edit, delete, and restore responses include the states that changed for the affected month, so the UI can show a notice without a second conceptual model.

### 2.5 Which budget applies

For a user, a scope (one category, or overall), and a month start `M`:

```text
select the unique budget where
  effective_from <= M
  and (effective_to is null or effective_to > M)
```

The exclusion constraint makes "unique" a database guarantee. Carry-forward is the absence of a later row, not a copied row.

Setting a budget for the current month `C`:

- If the open period has `effective_from = C`, update its amount and increment `version`.
- If the open period starts before `C`, set its `effective_to = C` and insert a new row from `C`.
- If a past month is requested, return `422 BUDGET_PERIOD_LOCKED` (pending TQ-6).

Clearing a budget for the current month:

- If the open period starts at `C`, delete that row (it never applied to a completed month) and audit the delete.
- Otherwise set `effective_to = C` on the open period. Earlier months still resolve to it. Later months resolve to nothing.

Archiving a category sets `effective_to` to the first day of next month on its open budget, so the current month keeps the budget and the next month does not (AC-CAT-3.3). Unarchive is not in the contract.

The informational warning "category budgets exceed the overall budget" sums the resolved category amounts for the month and compares to the resolved overall amount. It does not block the save (AC-BUD-2.1).

## 3. Dates

```text
today = datetime.now(ZoneInfo(user.timezone)).date()
month window = [month_start, next_month_start)
```

All report filters are `transaction_date >= start AND transaction_date < end`. November 2026 includes 1 Nov and 30 Nov. The 6-month trend is the selected month and the five before it; a month with no rows is `0`, not omitted.

`upcoming` is `transaction_date > today`. It is computed in the response. Future dates more than one year after `today` are rejected. Dates before `2000-01-01` are rejected. Both comparisons use the user's timezone for "today" and the stored calendar date for the transaction.

Changing timezone updates `users.timezone` only.

Audit timestamps and `created_at` are `timestamptz`, written by the database clock (`now()`), and serialised as ISO-8601 UTC.

## 4. Request lifecycle

1. Read or generate the request id (UUID).
2. Resolve the session cookie to a user, unless the route is public. Reject expired, revoked, idle, or absolute-expired sessions. Proposed limits, pending TQ-3: idle 7 days, absolute 30 days from login. Update `last_seen_at` at most once a minute.
3. On unsafe methods and on export, compare `X-CSRF-Token` to the double-submit cookie with a constant-time compare.
4. Open a transaction, `set_config('app.user_id', user_id, true)`.
5. Call the service.
6. Commit, or roll back on any exception including a failed audit insert.
7. Log the template and the status. Do not log the body.

Public routes (signup, login, verify, reset request, health) use a database role that may call the `SECURITY DEFINER` auth functions and may not select from `transactions`.

### 4.1 Idempotency

Required on `POST /transactions`. Header `Idempotency-Key`: 16 to 64 characters, `[A-Za-z0-9_-]`.

Hash is SHA-256 of the canonical JSON of the transaction fields: type, amount string as sent after parse-and-reformat, currency, date, category id, payment method, note. The hash **includes** those fields and **excludes** `confirm_duplicate`, which is not a transaction field. If two requests share a key and differ in the hash, the second gets `409 IDEMPOTENCY_KEY_REUSED` and writes nothing.

States of the key row: `in_progress`, `completed`.

- Insert the key as `in_progress` at the start of the transaction.
- Unique violation plus `in_progress`: `409 IDEMPOTENCY_IN_PROGRESS`.
- Unique violation plus `completed` and same hash: return the stored status and body, set `Idempotency-Replayed: true`. Do not write again.
- On success, store status and body and mark `completed` in the same commit as the transaction.
- On `409 POSSIBLE_DUPLICATE`, store that response as `completed` for this key. A retry of the same save returns the same 409. **Save anyway is a new user action and must use a new key**, with `confirm_duplicate: true`. Reusing the first key for save-anyway is rejected because the stored outcome is the warning, and a different hash is also a conflict. The client mints the second key when the user chooses Save anyway.

Client rule: mint the key when the user first submits that form, and reuse it for a timeout retry of that same submit. Do not let a generic HTTP client retry library mint a new key, and do not enable automatic POST retry.

Keys older than 24 hours are deleted by maintenance. A retry after that window can create a second row. 24 hours covers double-clicks and flaky mobile networks. It is not a forever dedupe.

### 4.2 Duplicate warning

Separate from idempotency. Inside the same transaction, take `pg_advisory_xact_lock` on a hash of `(user_id, type, amount_minor, category_id, date)` so two parallel saves cannot both miss the warning.

Then look for a non-deleted transaction with the same type, amount, category, and date, `created_at > now() - interval '5 minutes'`. The clock is the database clock.

- If found and `confirm_duplicate` is not true: `409 POSSIBLE_DUPLICATE` with a summary of the existing row. No insert.
- If found and `confirm_duplicate` is true: insert.
- Payment method and note are not part of the match (the acceptance criteria do not include them).

The lock makes the check reliable under concurrency. It does not turn the warning into a hard unique constraint. Two chai purchases a minute apart are allowed once the user confirms, and two purchases six minutes apart do not warn.

### 4.3 Optimistic concurrency

`version` starts at 1. A trigger increments it on every successful update of the transaction or the budget row. The client sends the version it read.

```text
update ... set ..., version = version + 1
where id = :id and version = :version
returning *
```

The trigger's increment and the statement's increment must not both apply. Implementation choice: the application writes the business columns and sets `version = :version + 1` in the `UPDATE`, and the trigger only rejects a statement that does not increase `version` by 1. One of those two mechanisms owns the increment; they must not stack. The LLD requirement is a single increment and a failed match when the caller's version is stale.

If the `UPDATE` returns no row:

- `SELECT` visible to RLS returns nothing → 404.
- `SELECT` returns a row → 409 `VERSION_CONFLICT` with that row. The client shows it and does not retry blindly.

Delete and restore are updates of `deleted_at` and also require `version`.

## 5. Categories

Seeded in the same transaction that marks the user verified, once, per user. Names and types are exactly the PRD FR-CAT-1 lists, in that order, stored with a `sort_order`.

Uniqueness: `(user_id, type, lower(btrim(name)))`. The API trims before insert. `"Pets"` and `"pets"` and `"Pets "` collide. The same name as an income category does not collide.

Archive sets `archived_at`. Archived categories stay on old transactions and in reports, and are rejected on create or on edit when the category id changes to an archived one. Hard delete is allowed only when no transaction row references the category, including soft-deleted transactions. If budget rows exist and no transactions do, they are removed in the same transaction and audited, then the category is deleted.

Rename updates the category row. History shows the new name because transactions store the category id, not a copied name (AC-CAT-2.3). The rename is an audit event.

## 6. Lists, dashboard, reports, export

List default order: `transaction_date DESC, created_at DESC, id DESC`. Cursor is an opaque encoding of the last tuple for that sort. Invalid cursor → 400. Default limit 50, maximum 100. Stable for a fixed dataset. New inserts appear on the first page; they do not shift a cursor the client already holds, because the cursor is a keyset, not an offset.

Filtered totals are aggregates of the full filter, not of the page, returned beside the page: `count`, `income`, `expense`, `net`, `currency`.

Search is case-insensitive substring on `note` only. `%` and `_` in the user query are escaped. At 10k rows a `user_id` index plus a filter is enough. A trigram index is a later change if a measurement says so, not part of the first migration.

Dashboard for `YYYY-MM` returns income, expense, net, the display rate, category breakdown of expenses (amount and display percent), comparison with the previous month (absolute minor-unit difference, display percent or `new` / `flat`), the six expense totals, and resolved budgets with state. One database transaction, at most four aggregate statements, no per-category round trip.

Year report uses 1 January through 1 January next year on `transaction_date`. Twelve months are always present. Category totals and month totals each sum to the year total in minor units. `year_type=financial_in` is not accepted in MVP.

CSV columns, in order: `date`, `type`, `amount`, `currency`, `category`, `payment_method`, `note`, `created_at`. Amount is the canonical decimal string. `created_at` is UTC with a `Z` suffix. Encoding is UTF-8 without a BOM. A field that starts with `=`, `+`, `-`, `@`, tab, or CR is prefixed with a single quote before quoting. Deleted rows and other users are absent. The filter query is the list query. Streaming response, so a large export does not build one giant string in memory. Same-origin fetch with the CSRF header; the SPA then saves the blob. A top-level navigation from another site does not have the header.

## 7. Audit

| Event | When |
|---|---|
| `transaction.created` / `.updated` / `.deleted` / `.restored` | Same commit as the change |
| `budget.created` / `.updated` / `.closed` | Same commit |
| `category.created` / `.renamed` / `.archived` / `.deleted` | Same commit. Category coverage is a DEV addition; see the discovery note. |

Row contents: id, user id, actor user id (the same, in v1), entity type, entity id, action, `before` JSON, `after` JSON, source (`manual` or `ai_assistant` or `system` for purge-adjacent category work), request id, `created_at`.

JSON amounts are decimal strings, never numbers. The application role has `INSERT` and `SELECT` on `audit_events`, not `UPDATE` or `DELETE`. A trigger rejects both. The purge function is `SECURITY DEFINER` and deletes audit rows for a transaction it is deleting. Maintenance logs outside the audit table: entity id, action `purged`, no amount, no note.

There is no HTTP route that edits an audit row.

## 8. Auth details

- Email is trimmed and lowercased. Unique on the lowercase value.
- Signup response is always generic. The existing-account branch still runs a hash so the timing is closer to the new-account branch. If the email is new, store the user unverified and send a verification token (hashed at rest, 24-hour expiry) when mail is enabled.
- Pending TQ-4: if the deployment is private, the verification function may mark the user verified inside signup and send nothing. The financial routes require `email_verified_at` when the setting `REQUIRE_EMAIL_VERIFICATION` is true.
- Password reset token: 32 random bytes, stored as SHA-256, 30-minute expiry, single use. Success revokes every session.
- Session token: 32 random bytes, cookie value, database stores SHA-256 only. Cookie name `et_session`, `HttpOnly`, `SameSite=Lax`, `Path=/`, `Secure` whenever `APP_ENV` is not `local`.
- CSRF cookie `et_csrf` is readable by the SPA, regenerated on login, mirrored in `X-CSRF-Token`.
- Login failures: count rows in `login_attempts` for that email in the last 15 minutes. At 5, reject until the window rolls off. Also cap by source IP at 50 failures per 15 minutes. Store the IP only in this short-lived table, not on the session.
- Account deletion sets `deletion_requested_at`, revokes sessions, and offers the client a chance to export first by being a separate call the UI makes before this one. The purge command deletes owned rows after 30 days.

Default timezone `Asia/Kolkata`. Default currency `INR`. `ai_enabled` defaults false. Enabling AI without `ai_disclosure_accepted_at` returns `403 AI_DISCLOSURE_REQUIRED`.

## 9. AI module (Phase 5)

Interface:

```text
propose_draft(message, categories, today, timezone) -> suggestion
answer(message, tool_executor) -> model_text, tool_calls
```

`GrokProvider` is the first implementation, using `GROK_API_BASE_URL`, `GROK_API_KEY`, and `GROK_MODEL` from the environment. The model id is chosen when the eval set exists, not in this document. A second provider implements the same interface or it does not ship.

Draft suggestion schema, validated before it is returned to the client: type, amount string or null, date or null, category id or null, payment method or null, note, list of ambiguity codes. The service then runs the same field validators as create, but turns validation failures into `missing_fields` rather than a stored row. A suggestion whose category id is not the caller's is treated as missing, not as a 500.

Multiple distinct expenses in one message: do not call the provider a second time and do not return a partial draft. `422 MULTIPLE_TRANSACTIONS_UNSUPPORTED`. Detection can be a first-pass heuristic (two amounts, or a list pattern) and may also trust a structured "multiple" flag from the model. Either signal is enough to refuse. Pending the PM's confirmation; this matches the PM's stated preference.

Shorthand the parser must accept, because the acceptance tests name them: `2k` → 2000, `1.5L` and `1.5 lakh` → 150000, `350/-` → 350. Display the resolved rupee amount on the draft so the user sees what `L` became. Other multipliers, including `cr`, leave the amount empty.

Query tools, the entire set:

| Tool | Inputs | Output |
|---|---|---|
| `period_totals` | date range | income, expense, net, currency, count |
| `category_breakdown` | date range, type | list of category id, name, amount |
| `compare_periods` | two date ranges | the two totals and the absolute difference |
| `budget_status` | month, category id or overall | state, spent, budget, remaining |

Each tool takes the user id from the server context, not from the model. The model cannot pass a different user id and have it honoured. AC-AUTH-4.4 tests that.

Number check: extract numeric tokens from the prose, normalise each the same way as money or as a percent, and require each to appear in the tool payload. On failure, `prose` is null and `prose_verified` is false. The UI renders `structured` regardless.

Daily caps, safety defaults pending PO-Q5: 30 draft requests and 60 queries per user. Over the cap → `429`. Provider timeout (proposed 8 seconds, under the 4 second p95 target as a budget, with the timeout slightly above it so a slow success can still return) → `503` and the manual-form fallback body. No retry against the provider that could double-bill without being counted.

Logs for these routes: latency, token usage, cap decision, outcome code. Not the message, not the amount, not the note.

## 10. Maintenance command

`python -m app.maintenance`, one transaction per user or one batched delete with the definer function:

1. For each transaction with `deleted_at < now() - interval '30 days'`, delete its audit rows and the transaction.
2. Delete idempotency keys with `created_at < now() - interval '24 hours'`.
3. Delete login attempts older than 1 day and email tokens that are expired and used.
4. For users with `deletion_requested_at < now() - interval '30 days'`, delete their rows in dependent tables, then the user.
5. Emit one log line per step with counts and no financial fields.

The function is `SECURITY DEFINER`, owned by the migration role, which is also the table owner and therefore the only role that bypasses row-level security. The audit trigger allows `DELETE` only when `current_user` is that owner. It does not trust a session variable, because the application role could set one. The API connection string is the application role and cannot call the delete directly.

## 11. Error model

All errors:

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Human-readable, no internal SQL.",
    "fields": {"amount": "max 2 decimal places"},
    "request_id": "uuid"
  }
}
```

`fields` is omitted when the error is not about a field. Codes are listed in the API spec. Unhandled exceptions become `500 INTERNAL` with no SQL text, and the request id.

## 12. Frontend rules that keep the server honest

- Amount inputs are strings. The JSON body contains a string. A review check rejects `JSON.stringify` of a number for money fields.
- The add form generates one idempotency key per submit attempt and a second only for Save anyway.
- Edit forms send the `version` from the GET. On 409 they replace the form with the server row and ask the user to reapply the change.
- The draft screen does not call create until Confirm. Confirm sends the edited fields.
- Query answers render the structured block (amount, period, link) even if prose is present.
- The service worker, if a manifest is added later, does not cache `/api`.

## 13. Testing hooks

The design is shaped so the risky rules are testable without a browser:

- `domain.money` and `domain.budget` tests for the vectors in section 2.
- API tests that open two users and swap ids.
- An API test that runs the create twice with one key and once with the same key and a different amount.
- An API test that updates with a stale version and asserts the first write is intact.
- A test that the application role's `DELETE` on `audit_events` fails.
- A test that `POST /ai/drafts` leaves the transaction count unchanged, including when the body says "delete all transactions."
- A test that dashboard `net` minor units equal the sum.

Fixtures are synthetic. The 00:30 IST / 19:00 UTC previous-day case is a fixed clock in the calendar test, not a sleep until midnight.
