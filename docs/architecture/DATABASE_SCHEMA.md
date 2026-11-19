# Database Schema

Status: **Proposed**
Date: 2026-11-19
Engine: PostgreSQL 16
Migrations: Alembic, one head, forward-only once applied

Types below are the contract for the first migration. Names can still change before that migration is written; behaviour should not, without a new ADR.

## 1. Conventions

- Primary keys are UUID v7, generated in the application, so inserts are time-ordered and ids are not guessable counters.
- Every table that holds one person's private data has `user_id uuid not null` referencing `users(id)`.
- Money is `amount_minor bigint` plus `currency char(3)`. Never `real`, `double precision`, or `numeric` for stored amounts. See ADR-007 for why not `numeric`.
- Transaction day is `date`. System times are `timestamptz` default `now()`.
- The application role `expense_app` is not a superuser and does not own tables. Row-level security applies to it. The migration role owns tables and the purge function.
- Extensions: `btree_gist` only, for the budget range exclusion constraint.

## 2. Roles and session context

```text
expense_migrator   owns tables and the purge function, used only by Alembic
expense_app        used by the API, subject to RLS
```

At the start of every authenticated API transaction:

```sql
select set_config('app.user_id', :user_id, true);
```

`true` means the setting dies at commit or rollback. Policies compare `user_id::text` to `current_setting('app.user_id', true)`. When the setting is missing, `current_setting(..., true)` is null and the comparison fails. Fail closed.

Auth routes that do not have a user yet do not use `expense_app` against tenant tables. They call `SECURITY DEFINER` functions owned by `expense_migrator`: `register_user`, `verify_email`, `start_password_reset`, `complete_password_reset`. Those functions do not accept a caller-supplied `user_id` for reads of transactions; they only touch `users`, `email_tokens`, and `sessions`.

## 3. Tables

### 3.1 users

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| email | text not null | Stored lowercase, trimmed |
| password_hash | text not null | Argon2id encoded string |
| timezone | text not null default `Asia/Kolkata` | IANA name. Reject unknown zones in the API. |
| home_currency | char(3) not null default `INR` | |
| email_verified_at | timestamptz null | |
| ai_enabled | boolean not null default false | |
| ai_disclosure_accepted_at | timestamptz null | |
| deletion_requested_at | timestamptz null | |
| created_at | timestamptz not null default now() | |
| updated_at | timestamptz not null default now() | |

Constraints: unique index on `email`. Check `home_currency ~ '^[A-Z]{3}$'`. Check `char_length(timezone) between 1 and 64`.

RLS: a user may select and update only the row with `id = app.user_id`. Inserts go through `register_user`.

### 3.2 sessions

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| user_id | uuid not null references users(id) on delete cascade | |
| token_hash | bytea not null | SHA-256 of the cookie value |
| created_at | timestamptz not null default now() | Absolute lifetime starts here |
| last_seen_at | timestamptz not null default now() | Idle lifetime |
| expires_at | timestamptz not null | Absolute expiry, proposed login + 30 days |
| revoked_at | timestamptz null | |

Unique on `token_hash`. Index `(user_id)`.

RLS: select and update own rows. Insert of a session happens in the login definer or, after the password check, as the user. Revoke is an update of `revoked_at`. The application never needs another user's session.

### 3.3 login_attempts

| Column | Type | Notes |
|---|---|---|
| id | bigint generated always as identity pk | Not a public id |
| email | text not null | |
| ip | inet null | Short-lived, this table only |
| attempted_at | timestamptz not null default now() | |

No RLS. Only the login definer inserts and the maintenance function deletes. The API role has no privileges on this table. Index `(email, attempted_at)`.

### 3.4 email_tokens

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| user_id | uuid not null references users(id) on delete cascade | |
| purpose | text not null | `verify_email` or `password_reset` |
| token_hash | bytea not null | |
| expires_at | timestamptz not null | 24 h verify, 30 min reset |
| used_at | timestamptz null | |

Unique on `token_hash`. Check purpose in the allowed set. No privileges for `expense_app`. Definer functions only.

### 3.5 categories

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| user_id | uuid not null references users(id) on delete cascade | |
| type | text not null | `income` or `expense` |
| name | text not null | Trimmed, 1–40 code points |
| sort_order | integer not null | Seed order, then increasing |
| archived_at | timestamptz null | |
| created_at | timestamptz not null default now() | |
| updated_at | timestamptz not null default now() | |

Checks: type in (`income`, `expense`), `char_length(name) between 1 and 40`.

Unique index: `(user_id, type, lower(btrim(name)))`.

Unique constraint `(id, user_id, type)` to support the composite foreign key from transactions.

RLS: all commands restricted to `user_id`. Hard delete is a real `DELETE`, allowed for this table only when the service has checked references. Transactions reference categories with `ON DELETE RESTRICT`, so a category that is in use cannot disappear.

Seed lists, created once in the verify transaction, expense then income, `sort_order` from 10 in steps of 10:

Expense: Food & Dining, Groceries, Transport, Fuel, Rent, Utilities & Bills, Mobile & Internet, Shopping, Health, Entertainment, Subscriptions, Travel, Education, Personal Care, Gifts & Donations, EMI/Loan, Other.

Income: Salary, Freelance, Interest, Refund, Gift, Other.

### 3.6 transactions

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| user_id | uuid not null | |
| type | text not null | `income` or `expense` |
| amount_minor | bigint not null | |
| currency | char(3) not null | |
| transaction_date | date not null | The calendar day |
| category_id | uuid not null | |
| payment_method | text null | `upi`, `card`, `cash`, `net_banking`, `other` |
| note | text null | At most 500 code points, no NUL |
| source | text not null | `manual` or `ai_assistant` |
| version | integer not null default 1 | |
| created_at | timestamptz not null default now() | |
| updated_at | timestamptz not null default now() | |
| deleted_at | timestamptz null | Null means live |

Checks:

- type in (`income`, `expense`)
- source in (`manual`, `ai_assistant`)
- `amount_minor > 0 and amount_minor <= 10000000000`
- `currency ~ '^[A-Z]{3}$'`
- payment method null or in the enum
- `version >= 1`
- note length ≤ 500
- `transaction_date >= date '2000-01-01'`

There is no check that the date is within one year of today, because that bound moves and is enforced in the service against the user's timezone.

Foreign keys:

```text
(category_id, user_id, type) → categories(id, user_id, type)
(user_id) → users(id)
```

The composite key rejects another user's category, and an expense filed against an income category, inside the database (AC-TXN-1.6, AC-AUTH-4.3).

RLS on all commands by `user_id`.

Grants: `expense_app` may `SELECT`, `INSERT`, `UPDATE`. It may not `DELETE`. Soft delete is an update of `deleted_at`.

Trigger `transactions_version`: before update, require that `NEW.version = OLD.version + 1`, and set `updated_at = now()`. Reject updates that change `id`, `user_id`, `created_at`, or `source`.

### 3.7 budgets

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| user_id | uuid not null references users(id) on delete cascade | |
| category_id | uuid null | Null means the overall budget |
| scope_key | text generated always as (`coalesce(category_id::text, '__overall__')`) stored | |
| amount_minor | bigint not null | Same bounds as transactions |
| currency | char(3) not null | |
| effective_from | date not null | First day of a month |
| effective_to | date null | First day of a later month, exclusive. Null means open. |
| version | integer not null default 1 | |
| created_at | timestamptz not null default now() | |
| updated_at | timestamptz not null default now() | |

Checks:

- `amount_minor > 0 and amount_minor <= 10000000000`
- `extract(day from effective_from) = 1`
- `effective_to is null or (extract(day from effective_to) = 1 and effective_to > effective_from)`
- currency pattern as above
- `version >= 1`
- category, when set, belongs to the same user: composite foreign key `(category_id, user_id) → categories(id, user_id)`

Exclusion constraint, using `btree_gist`:

```text
exclude using gist (
  user_id with =,
  scope_key with =,
  daterange(effective_from, coalesce(effective_to, 'infinity'::date), '[)') with &&
)
```

Two overall budgets, or two budgets for the same category, cannot overlap. Carry-forward is represented by an open `effective_to`.

RLS by `user_id`. Update and delete are both needed: clearing a budget that was created this month deletes that row; closing an older period updates `effective_to`. Audit rows record both.

Version trigger: same rule as transactions, and reject changes to `user_id`, `category_id`, and `effective_from` after insert. A new period is a new row, not an edited start date. Amount edits on the current period are allowed.

### 3.8 audit_events

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| user_id | uuid not null | |
| actor_user_id | uuid not null | Equal to `user_id` in v1 |
| entity_type | text not null | `transaction`, `budget`, `category` |
| entity_id | uuid not null | Not a foreign key. The subject may be purged. |
| action | text not null | For example `transaction.updated` |
| before | jsonb null | Decimal strings for amounts |
| after | jsonb null | |
| source | text not null | `manual`, `ai_assistant`, `system` |
| request_id | uuid null | |
| created_at | timestamptz not null default now() | |

Checks: entity type, source, and action drawn from fixed lists in the migration so a typo cannot land.

RLS: select and insert own rows. No update policy, and no delete policy, for `expense_app`.

Grants: `SELECT`, `INSERT` only.

Trigger: `BEFORE UPDATE OR DELETE` raises unless `current_user` is `expense_migrator`. The purge function is `SECURITY DEFINER` with owner `expense_migrator`, so the body runs as that user and the trigger allows the delete. The API cannot impersonate the owner.

JSON shape for a transaction `before` / `after` object:

```json
{
  "type": "expense",
  "amount": "350.00",
  "currency": "INR",
  "date": "2026-11-19",
  "category_id": "uuid",
  "payment_method": "upi",
  "note": "lunch",
  "deleted": false
}
```

Amounts in this JSON are strings. A check constraint cannot fully enforce that; the service test does.

### 3.9 idempotency_keys

| Column | Type | Notes |
|---|---|---|
| id | uuid pk | |
| user_id | uuid not null references users(id) on delete cascade | |
| idempotency_key | text not null | 16–64 chars, the header value |
| request_hash | bytea not null | |
| state | text not null | `in_progress` or `completed` |
| response_status | integer null | |
| response_body | jsonb null | The HTTP body that will be replayed |
| created_at | timestamptz not null default now() | |
| completed_at | timestamptz null | |

Unique `(user_id, idempotency_key)`. Check key charset, state, and that a completed row has a status. RLS by `user_id`.

The response body is the API's own response, which contains amounts. It is covered by the same purge as the user and by the 24-hour cleanup. It is not an audit trail. Do not log it.

## 4. Indexes

```text
transactions (user_id, transaction_date desc, created_at desc, id desc)
  where deleted_at is null

transactions (user_id, category_id, transaction_date)
  where deleted_at is null

transactions (user_id, type, amount_minor, category_id, transaction_date, created_at)
  where deleted_at is null
  -- duplicate warning lookup

categories (user_id, type, sort_order)

budgets (user_id, scope_key, effective_from desc)

audit_events (user_id, entity_type, entity_id, created_at)

sessions (token_hash)

idempotency_keys (created_at)
```

The partial indexes keep soft-deleted rows out of the hot paths. Trash view uses a second index:

```text
transactions (user_id, deleted_at desc) where deleted_at is not null
```

No GIN trigram index in the first migration. Note search at the stated volume is a sequential filter inside the user's rows.

## 5. Row-level security policies

Do not `FORCE ROW LEVEL SECURITY`. The API role is not the table owner, so policies still apply to every request the application makes. The owner bypasses RLS, which is what lets the purge function (security definer, owned by the migrator) delete rows. The application connection string must not be the migrator. A test connects as `expense_app` with `app.user_id` unset and asserts zero tenant rows.

For each of `categories`, `transactions`, `budgets`, `audit_events`, `idempotency_keys`, `sessions`:

```text
using (
  user_id::text = current_setting('app.user_id', true)
)
with check (
  user_id::text = current_setting('app.user_id', true)
)
```

`users` uses `id` instead of `user_id`.

A second test asserts `expense_app` has no `DELETE` grant on `audit_events` or `transactions`.

## 6. Purge function

`purge_expired(now timestamptz) returns void`, security definer, owner `expense_migrator`, granted to `expense_app` so the maintenance command can call it without owning tables.

Inside, for each transaction with `deleted_at` older than 30 days: delete the `audit_events` rows whose `entity_type` is `transaction` and whose `entity_id` is that transaction, then delete the transaction. Do this in one database transaction per batch so a crash cannot leave an audit row whose transaction is already gone, or the reverse.

The same function then deletes idempotency keys older than 24 hours, login attempts older than 1 day, used or expired email tokens, and users with `deletion_requested_at` older than 30 days along with every row they still own.

The function does not take a user-supplied filter that the API can aim at another person, other than "everything that is due." The maintenance command calls it with the database clock.

## 7. What is deliberately absent

- No `float` or `double precision` columns.
- No stored `upcoming` flag.
- No stored budget utilisation or percentage.
- No `savings` column. Net is computed.
- No draft table. Phase 5 drafts are not rows.
- No notification table. Warnings are computed.
- No accounts, transfers, currencies-with-rates, or household membership tables.
- No database enum types for payment method and transaction type. Text plus a check constraint is easier to extend in a later migration without the operational pain of altering a Postgres enum. The allowed values are still closed.

## 8. Backup

The schema is covered by a full `pg_dump` of the database. There is no file store beside Postgres in MVP. Restoring the dump restores the ledger. The dump is encrypted by the operator's tool (age or gpg) before it hits disk that is not the database volume. Keys are not in the repo. Retention is 7 daily dumps. This procedure is Phase 6 work; the schema only has to remain dump-friendly, which a single database is.
