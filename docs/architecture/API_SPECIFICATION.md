# API Specification

Status: **Proposed**
Date: 2026-11-19
Base path: `/api/v1`
Health: `/api/health`
Format: JSON, UTF-8, except the CSV export

The markdown in this file is the contract until application code exists. After that, the OpenAPI document generated from the code and this file must change in the same pull request. If they disagree during a review, this file wins until the pull request merges, and the tests that encode the examples win after that.

Pending product decisions are marked **pending**. The stated behaviour is the proposed default from `docs/discussions/001-mvp-discovery.md`.

## 1. Conventions

### 1.1 Auth

Session cookie `et_session`. Mutating requests, and the CSV export, require header `X-CSRF-Token` equal to cookie `et_csrf`.

Unauthenticated → `401 UNAUTHORIZED`. Authenticated but the resource is not visible, including another user's id → `404 NOT_FOUND`. Email unverified, when verification is required → `403 EMAIL_NOT_VERIFIED`. Account pending deletion → `403 ACCOUNT_PENDING_DELETION` on login, and `401` on other routes because the session was revoked.

### 1.2 Money

Amount fields are JSON strings, two decimal places on output, at most two on input. A JSON number is a `422 VALIDATION_ERROR` even if the mathematical value would have been valid. Currency is `"INR"`. Any other code is `422`.

### 1.3 Errors

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Amount must have at most 2 decimal places.",
    "fields": {"amount": "max 2 decimal places"},
    "request_id": "5b1c0f16-2d3e-4a5b-8c7d-9e0f1a2b3c4d"
  }
}
```

| HTTP | Code | When |
|---|---|---|
| 400 | `BAD_REQUEST` | Malformed JSON, bad cursor |
| 401 | `UNAUTHORIZED` | No session or dead session |
| 403 | `EMAIL_NOT_VERIFIED` | Financial route, verification required, not verified |
| 403 | `ACCOUNT_PENDING_DELETION` | Login during the deletion window |
| 403 | `AI_DISABLED` | AI route while AI is off |
| 403 | `AI_DISCLOSURE_REQUIRED` | Enabling AI before disclosure |
| 403 | `CSRF_FAILED` | Missing or mismatched CSRF token |
| 404 | `NOT_FOUND` | Missing or not owned |
| 409 | `VERSION_CONFLICT` | Stale `version`. Body includes `current`. |
| 409 | `POSSIBLE_DUPLICATE` | Duplicate warning. Nothing was written. |
| 409 | `IDEMPOTENCY_KEY_REUSED` | Same key, different transaction payload |
| 409 | `IDEMPOTENCY_IN_PROGRESS` | Same key is currently being processed |
| 409 | `CATEGORY_IN_USE` | Hard-delete of a category that still has transactions |
| 422 | `VALIDATION_ERROR` | Field rules, including money and dates |
| 422 | `BUDGET_PERIOD_LOCKED` | **pending TQ-6.** Past month. |
| 422 | `MULTIPLE_TRANSACTIONS_UNSUPPORTED` | **pending D6.** More than one expense in a message. |
| 422 | `NOT_IN_MVP` | Reserved. Not used by the calendar-year report. |
| 429 | `RATE_LIMITED` | Login window or AI daily cap. `Retry-After` set. |
| 500 | `INTERNAL` | Unexpected. No SQL in the message. |
| 503 | `AI_UNAVAILABLE` | Provider error or timeout. Ledger routes stay up. |

### 1.4 Idempotency

`POST /transactions` requires `Idempotency-Key`. Other creates do not, in MVP. Replay of a completed key returns the original status and body plus `Idempotency-Replayed: true`.

## 2. Auth and profile

### POST /auth/signup

Request: `{ "email": "person@example.com", "password": "correct horse battery" }`

Response: `202` and `{ "message": "Check your email." }` whether or not the email is registered.

Creates an unverified user when the email is new. When `REQUIRE_EMAIL_VERIFICATION` is false (**pending TQ-4**, the private-deploy default proposed), the user is created already verified and the message is instead `{ "message": "Account created. You can log in." }` only on the new-email branch. That variant reveals that a fresh signup worked; it is acceptable only when there is no public signup. On a public deploy the generic body is mandatory.

### POST /auth/verify-email

Request: `{ "token": "..." }`. `204` on success. `400` if the token is unknown, used, or expired. Seeds default categories in the same transaction the first time verification succeeds.

### POST /auth/verify-email/resend

`202` generic, if verification is enabled.

### POST /auth/login

Request: `{ "email", "password" }`.

Success: `200` and the profile (section 2.1), `Set-Cookie` for session and CSRF.

Failure: `401` generic, or `429` after 5 failures in 15 minutes for that email. `403 ACCOUNT_PENDING_DELETION` if deletion was requested. `403 EMAIL_NOT_VERIFIED` if verification is required and incomplete. Those two 403s reveal account state after a correct password only; a wrong password is still a generic 401.

### POST /auth/logout

`204`. Revokes the current session. Requires CSRF.

### POST /auth/password-reset/request

`202` generic always.

### POST /auth/password-reset/confirm

Request: `{ "token", "password" }`. `204` on success and all sessions for that user revoked. `400` on a bad token.

### GET /me

Profile:

```json
{
  "id": "018f1a2b-3c4d-7e5f-8a9b-0c1d2e3f4a5b",
  "email": "person@example.com",
  "timezone": "Asia/Kolkata",
  "home_currency": "INR",
  "email_verified": true,
  "ai_enabled": false,
  "ai_disclosure_accepted": false,
  "deletion_requested_at": null
}
```

### PATCH /me

Request, any subset: `{ "timezone": "Asia/Kolkata", "ai_enabled": false }`.

Unknown timezone → 422. `ai_enabled: true` without disclosure → `403 AI_DISCLOSURE_REQUIRED`. Does not change transactions.

### POST /me/ai-disclosure

Request: `{ "accepted": true }`. `204`. Records the timestamp. Does not by itself turn AI on.

### POST /me/deletion

Request: `{ "password": "...", "confirm": "DELETE" }`. Requires the current password. `202`. Revokes all sessions. Does not wait for the purge. The UI should call the export endpoint first; this endpoint does not embed a file.

## 3. Categories

### GET /categories

Query: `type=expense|income` optional, `include_archived=false` default.

```json
{
  "items": [
    {
      "id": "uuid",
      "type": "expense",
      "name": "Food & Dining",
      "archived": false,
      "sort_order": 10
    }
  ]
}
```

### POST /categories

Request: `{ "type": "expense", "name": "Pets" }`. `201` returns the category. Duplicate after trim and case-fold → `422` on `name`.

### PATCH /categories/{id}

Request: `{ "name": "Eating Out" }`. Category rename and archive do not take a `version`. They are not money, and no acceptance criterion requires a conflict error. Last write wins. Each attempt still writes an audit row with the previous name and the new name, in commit order. This is an explicit, narrow exception to optimistic locking. Transactions and budgets are not exempt.

### POST /categories/{id}/archive

`200` returns the category with `archived: true`. Closes the category budget at the next month boundary (LLD). Idempotent if already archived.

### DELETE /categories/{id}

`204` if no transaction, including soft-deleted, references it. Budget rows for it are deleted in the same transaction and audited. `409 CATEGORY_IN_USE` otherwise.

## 4. Transactions

### Transaction object

```json
{
  "id": "uuid",
  "type": "expense",
  "amount": "350.00",
  "currency": "INR",
  "date": "2026-11-19",
  "category": {
    "id": "uuid",
    "name": "Food & Dining",
    "type": "expense",
    "archived": false
  },
  "payment_method": "upi",
  "note": "lunch",
  "upcoming": false,
  "source": "manual",
  "version": 1,
  "created_at": "2026-11-19T06:10:00Z",
  "updated_at": "2026-11-19T06:10:00Z",
  "deleted_at": null,
  "budget_warnings": []
}
```

`upcoming` is derived. `budget_warnings` is present on write responses and omitted on list items. Each warning:

```json
{
  "scope": "category",
  "category_id": "uuid",
  "month": "2026-11",
  "state": "approaching",
  "spent": "6400.00",
  "budget": "8000.00",
  "remaining": "1600.00",
  "overspend": "0.00"
}
```

`state` is `ok`, `approaching`, `at_limit`, or `exceeded`. The name `at_limit` is **pending TQ-8**. `scope` is `category` or `overall`.

### POST /transactions

Headers: `Idempotency-Key`, `X-CSRF-Token`.

```json
{
  "type": "expense",
  "amount": "350.00",
  "currency": "INR",
  "date": "2026-11-19",
  "category_id": "uuid",
  "payment_method": "upi",
  "note": "lunch",
  "confirm_duplicate": false,
  "source": "manual"
}
```

`source` may be `manual` (default) or `ai_assistant`. The client is trusted to label the source for audit. A false label does not skip validation. There is no way to supply a source of `system`.

`payment_method` may be null. `note` may be null or omitted.

`201` returns the transaction object. The create is not repeated if the key is replayed.

Validation highlights, all `422`:

- amount not a string, not positive, more than 2 decimal places, or above `100000000.00`
- date before `2000-01-01` or more than 1 year after today in the user timezone
- category missing, archived, wrong type, or not owned
- note longer than 500 code points
- currency not `INR`

`409 POSSIBLE_DUPLICATE` body:

```json
{
  "error": {
    "code": "POSSIBLE_DUPLICATE",
    "message": "Possible duplicate.",
    "request_id": "uuid"
  },
  "existing": {
    "id": "uuid",
    "amount": "350.00",
    "date": "2026-11-08",
    "category_name": "Food & Dining",
    "created_at": "2026-11-08T08:00:00Z"
  }
}
```

No row is created. The next step, Save anyway, is a **new** `Idempotency-Key` and `"confirm_duplicate": true`.

### GET /transactions

Query parameters:

| Name | Meaning |
|---|---|
| `from`, `to` | Inclusive calendar dates |
| `type` | `income` or `expense` |
| `category_id` | Repeatable |
| `payment_method` | One of the enum, or `none` |
| `q` | Substring of `note`, case-insensitive |
| `sort` | `date_desc` (default), `date_asc`, `amount_desc`, `amount_asc` |
| `limit` | Default 50, max 100 |
| `cursor` | Opaque |
| `deleted` | `only` for the trash list. Default is live rows. There is no `include`. |

```json
{
  "items": [],
  "next_cursor": null,
  "totals": {
    "count": 2,
    "income": "0.00",
    "expense": "0.30",
    "net": "-0.30",
    "currency": "INR"
  }
}
```

`totals` ignore the page and honour the filter. For `deleted=only`, totals are omitted and each item includes `restore_until` (deleted_at plus 30 days) and `days_remaining`.

### GET /transactions/{id}

`200` with the object, including soft-deleted rows so the trash screen can open one. `404` if it is not owned or already purged.

### PATCH /transactions/{id}

```json
{
  "version": 1,
  "amount": "450.00",
  "date": "2026-11-01",
  "category_id": "uuid",
  "payment_method": "card",
  "note": "dinner",
  "type": "expense"
}
```

All mutable fields optional except `version`, which is required. At least one mutable field required. Same validation as create. `200` with the updated object and `budget_warnings` for the old month and the new month when the date or the amount moved. `409 VERSION_CONFLICT`:

```json
{
  "error": {"code": "VERSION_CONFLICT", "message": "This transaction was changed.", "request_id": "uuid"},
  "current": { }
}
```

`current` is the transaction object. The stored row is unchanged by the failed request.

### DELETE /transactions/{id}

Request body: `{ "version": 1 }`. `200` returns the object with `deleted_at` set. The row leaves lists, totals, budgets, reports, export, and AI answers. `409` on a stale version.

### POST /transactions/{id}/restore

Request body: `{ "version": 2 }`. `200` when `deleted_at` is within 30 days. `409` when the window has passed but the row is still physically there (purge has not run). `404` when it is gone. Restore writes an audit event and returns fresh `budget_warnings`.

### GET /transactions/{id}/history

```json
{
  "items": [
    {
      "id": "uuid",
      "action": "transaction.updated",
      "source": "manual",
      "at": "2026-11-19T07:00:00Z",
      "before": {"amount": "350.00"},
      "after": {"amount": "450.00"}
    }
  ]
}
```

Ordered oldest first. `before` and `after` contain only changed fields plus identifiers needed to understand them, using decimal strings. No route accepts a method that modifies this collection.

## 5. Dashboard and reports

### GET /dashboard?month=2026-11

`month` is required, `YYYY-MM`, interpreted in the user's timezone only as "which calendar month," not as an instant. The default the UI asks for is the month that contains `today`.

```json
{
  "month": "2026-11",
  "currency": "INR",
  "income": "85000.00",
  "expense": "52340.75",
  "net": "32659.25",
  "net_rate_display": "38.4",
  "previous_month": {
    "month": "2026-10",
    "income_delta": "0.00",
    "expense_delta": "10000.00",
    "expense_rate_display": "25.0",
    "expense_change": "up"
  },
  "categories": [
    {
      "category_id": "uuid",
      "name": "Food & Dining",
      "amount": "8200.00",
      "share_display": "15.7",
      "archived": false
    }
  ],
  "trend": [
    {"month": "2026-06", "expense": "0.00"}
  ],
  "budgets": []
}
```

`net_rate_display` is display-only: `net / income`, half-up to one decimal place (32659.25 / 85000 is `38.4`). It is null when income is `0.00`. The UI shows "—" in that case. When net is negative the amount is a signed decimal string and the UI is expected to call it a deficit; the field is still `net`, not `savings` (**pending TQ-5**). The acceptance figure ₹32,659.25 is `net`.

`expense_change` is `up`, `down`, `flat`, or `new`. `new` is used when the previous month's amount is zero and this month's is not. Deltas are decimal strings and may be negative.

`categories` includes archived categories that have spend in the month. Amounts sum to `expense` in minor units. `share_display` values need not sum to 100.

`trend` has six months ending at `month`, expenses only.

`budgets` is the same shape as `GET /budgets` for that month, so the dashboard does not invent a second formula.

Drill-down is not a separate call. The client opens `GET /transactions` with `from`, `to`, `type=expense`, and `category_id`.

### GET /reports/year?year=2026

Calendar year of `transaction_date`. Twelve month objects, including empty ones, plus category totals for expenses and for income separately.

```json
{
  "year": 2026,
  "year_type": "calendar",
  "currency": "INR",
  "income": "0.00",
  "expense": "0.00",
  "net": "0.00",
  "months": [
    {"month": "2026-01", "income": "0.00", "expense": "0.00", "net": "0.00"}
  ],
  "expense_categories": [],
  "income_categories": []
}
```

The sum of `months[].expense` equals `expense`. The sum of `expense_categories[].amount` equals `expense`. Same for income. A transaction on 31 Dec 2026 is inside 2026 regardless of the UTC time of `created_at`.

Indian financial year is not a query parameter in this version. Adding it later means accepting `from` and `to`, or `year_type=financial_in`, on this route. No stored column changes.

## 6. Budgets

### GET /budgets?month=2026-11

Returns the resolved budget for that month after carry-forward, not the raw history.

```json
{
  "month": "2026-11",
  "currency": "INR",
  "overall": {
    "id": "uuid",
    "amount": "40000.00",
    "spent": "0.00",
    "remaining": "40000.00",
    "state": "ok",
    "version": 1,
    "effective_from": "2026-11-01"
  },
  "categories": [],
  "category_sum_exceeds_overall": false
}
```

`overall` is null when none resolves. Category entries that are archived and whose budget does not apply this month are absent.

### PUT /budgets

```json
{
  "month": "2026-11",
  "category_id": null,
  "amount": "45000.00",
  "version": 1
}
```

`category_id` null means overall. `version` is required when a row already exists for this change, and omitted when this call will insert the first period. The server returns `409 VERSION_CONFLICT` with `current` when the open period's version does not match.

`month` must equal the user's current month (**pending TQ-6**). Otherwise `422 BUDGET_PERIOD_LOCKED`.

`200` returns the `GET /budgets` document for that month, including `category_sum_exceeds_overall`.

### DELETE /budgets/{id}

Body: `{ "version": 1 }`. Closes or deletes the period as in the LLD. `200` returns the month document. `409` on a stale version. Deleting a period that only affects the past is rejected with `BUDGET_PERIOD_LOCKED`; the client deletes the current view, not an old slice.

## 7. Export

### GET /exports/transactions.csv

Same query parameters as `GET /transactions`, except `cursor`, `limit`, and `deleted`. Deleted rows are never exported. Requires the session and `X-CSRF-Token`.

Response: `200`, `Content-Type: text/csv; charset=utf-8`, `Content-Disposition: attachment; filename="transactions.csv"`.

Header row exactly:

```text
date,type,amount,currency,category,payment_method,note,created_at
```

No BOM. Amounts like `123456.50`. Dates like `2026-11-19`. `created_at` like `2026-11-19T06:10:00Z`. A note that starts with `=`, `+`, `-`, `@`, tab, or carriage return is prefixed with `'`. Commas, quotes, and newlines are quoted by RFC 4180 rules. The sum of the amount column for a given type, read back as decimals, equals the filtered total for that type.

## 8. AI (Phase 5 — specified now, not enabled)

All routes in this section return `403 AI_DISABLED` unless `ai_enabled` is true, and they are not registered as something the dashboard needs.

### POST /ai/drafts

```json
{ "text": "I spent 350 rupees on lunch today" }
```

`200`:

```json
{
  "draft": {
    "type": "expense",
    "amount": "350.00",
    "currency": "INR",
    "date": "2026-11-19",
    "category_id": "uuid",
    "payment_method": null,
    "note": "lunch",
    "missing_fields": [],
    "ambiguities": []
  },
  "confirm_enabled": true
}
```

No transaction is created. `confirm_enabled` is false when `amount` or `category_id` or `date` or `type` is null. Ambiguity codes include `amount_missing`, `category_uncertain`, `type_assumed_income`. For `category_uncertain` the category may be null; the UI must not pretend a guess. For a message the server classifies as more than one expense, `422 MULTIPLE_TRANSACTIONS_UNSUPPORTED` and no `draft` (**pending D6**).

Provider failure: `503 AI_UNAVAILABLE` with

```json
{
  "error": {"code": "AI_UNAVAILABLE", "message": "Couldn't understand — add manually.", "request_id": "uuid"},
  "fallback_note": "I spent 350 rupees on lunch today"
}
```

The fallback string is the user's text, truncated to 500 code points, for the manual form's note field. It is not logged.

### POST /ai/queries

```json
{ "text": "How much did I spend on food this month?" }
```

`200`:

```json
{
  "structured": {
    "kind": "category_total",
    "amount": "8240.50",
    "currency": "INR",
    "period": {"from": "2026-11-01", "to": "2026-11-30"},
    "category": {"id": "uuid", "name": "Food & Dining"},
    "transactions_query": {
      "from": "2026-11-01",
      "to": "2026-11-30",
      "type": "expense",
      "category_id": "uuid"
    }
  },
  "prose": "You spent ₹8,240.50 on Food & Dining from 1 to 30 Nov 2026.",
  "prose_verified": true
}
```

`kind` is `category_total`, `period_compare`, `budget_status`, or `unsupported`. For `unsupported` (advice, market questions, empty of meaning), `structured` explains the refusal, amounts are null, and prose does not contain a figure. When no transactions match, `amount` is `"0.00"`.

`period_compare` includes both periods, both expense totals, and the absolute difference as decimal strings. A bare month name uses the year of `today` if that month is not after the current month, otherwise the previous year, and `assumed_year` is set so the UI can say which year was used. "October" asked in November 2026 is October 2026.

`budget_status` includes `state`, `spent`, `budget`, `overspend`. If no budget resolves, `state` is `none` and the payload includes `set_budget: true` and does not create one.

If prose fails the number check, `prose` is null and `prose_verified` is false. Clients must render `structured`.

These routes write nothing. A test asserts the transaction count and the max audit id are unchanged.

## 9. Health

### GET /api/health

No auth. `200` `{"status":"ok"}` or `503` `{"status":"unavailable"}`. No version, no counts.

## 10. Examples that tests should pin

| Case | Expected |
|---|---|
| Two creates, `"0.10"` and `"0.20"` | List `totals.expense` is `"0.30"` |
| Amount `"100.555"` or `100.55` as a JSON number | 422, no row |
| Amount `"100000000.00"` | Accepted |
| Amount `"100000000.01"` | 422 |
| PATCH with a stale version | 409, previous amount unchanged, one failed attempt produces no audit row |
| Double POST, same key, same body | One row, second response replayed |
| Same key, different amount | 409 `IDEMPOTENCY_KEY_REUSED`, one row |
| User B requests user A's transaction id | 404, row unchanged |
| Export note `=1+1` | Cell text starts with `'=1+1` after CSV parse |
| Dashboard income 0 | `net_rate_display` null |
| AI draft then abandon | Transaction count unchanged |
