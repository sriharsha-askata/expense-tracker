# Database schema

Status: **Accepted** for the product columns (refunds included). The snapshot envelope uses XChaCha20-Poly1305 (ADR-012). Phase 3 still waits for a go-ahead.
Date: 2026-11-19
Supersedes: the PostgreSQL schema in the first draft. Do not create that database.

Two shapes are specified: SQLite on the device, which is the source of truth, and the logical snapshot that is encrypted and stored in the user's private GitHub repo.

## 1. SQLite rules

- SQLite with WAL, `foreign_keys=ON`, SQLCipher. The SQLCipher key is random and lives in the Android Keystore. It is not the passphrase.
- Money columns are `INTEGER` minor units. The schema contains no `REAL` column. A review check fails if one appears.
- Calendar days are `TEXT` `YYYY-MM-DD`. Instants are `TEXT` UTC ISO-8601. There is no timestamp affinity that shifts a date across midnight.
- Primary keys are UUID strings. Default category ids are UUIDv5 of the UTF-8 bytes of `type` + `\n` + `name`, namespace `6f1d4c8a-9b3e-4f2a-8c7d-0e1f2a3b4c5d` (a constant chosen for this app, not a registered URN). Every install must produce the same ids. Pin the first id in a test vector when code exists.
- `rev` is a random UUID for that version of the row. `base_rev` is the `rev` this version was edited from, or null for a create. A resolved conflict stores two parents; see `rev_parent`.
- Amounts in JSON, history, and the snapshot are decimal strings, never JSON numbers.

### 1.1 settings

Singleton. Synced as one entity so two phones notice a timezone clash instead of flipping it silently.

| Column | Type | Notes |
|---|---|---|
| id | TEXT pk | Fixed id `settings`. |
| timezone | TEXT not null | IANA, default `Asia/Kolkata`. |
| home_currency | TEXT not null | `INR`. |
| rev | TEXT not null | |
| base_rev | TEXT | |
| updated_at | TEXT not null | |

### 1.2 categories

| Column | Type | Notes |
|---|---|---|
| id | TEXT pk | UUIDv5 for defaults. |
| type | TEXT not null | `income` or `expense`. |
| name | TEXT not null | Trimmed, 1–40 code points. |
| sort_order | INTEGER not null | |
| archived_at | TEXT | |
| rev, base_rev | TEXT | |
| updated_at | TEXT not null | |

Unique on `(type, lower(trim(name)))`. Seed lists are the names in the PRD (FR-CAT-1), unchanged until the PM edits the product docs.

### 1.3 transactions

| Column | Type | Notes |
|---|---|---|
| id | TEXT pk | |
| type | TEXT not null | `income`, `expense`, or `refund`. Accepted. |
| amount_minor | INTEGER not null | `> 0` and `<= 10000000000` (₹10,00,00,000.00). |
| currency | TEXT not null | `INR` in v1. |
| transaction_date | TEXT not null | `YYYY-MM-DD`, not before `2000-01-01`. The one-year future cap is applied in code against "today" in the user timezone. |
| category_id | TEXT not null | References `categories(id)`. |
| payment_method | TEXT | `upi`, `card`, `cash`, `net_banking`, `other`, or null. |
| note | TEXT | At most 500 code points. |
| source | TEXT not null | `manual`, `repeat`, or `ai_assistant`. |
| mutation_id | TEXT | Unique when set. One save gesture, one id. |
| rev | TEXT not null | |
| base_rev | TEXT | |
| deleted_at | TEXT | Tombstone. Null means live. |
| created_at, updated_at | TEXT not null | |

Trigger, accepted: if `type = 'refund'`, the category's type must be `expense`.

A refund's `amount_minor` is still positive. Reports subtract it. Do not store a negative integer.

### 1.4 rev_parent

A normal edit has one parent, equal to `base_rev`, and this table may mirror it. A conflict resolution has two parents: the two heads the user looked at.

| Column | Type |
|---|---|
| rev | TEXT not null |
| parent_rev | TEXT not null |
| entity | TEXT not null |
| entity_id | TEXT not null |

Primary key `(rev, parent_rev)`.

### 1.5 budgets

Effective-dated, as in the earlier design. The product rule did not change. Past months stay locked unless the PM reopens that (still proposed).

| Column | Type | Notes |
|---|---|---|
| id | TEXT pk | |
| category_id | TEXT | Null means the overall budget. |
| amount_minor | INTEGER not null | Same bounds as transactions. |
| currency | TEXT not null | |
| effective_from | TEXT not null | First of a month. |
| effective_to | TEXT | First of a later month, exclusive. Null means open. |
| rev, base_rev | TEXT | |
| deleted_at | TEXT | Closing a current-month budget that never applied to a past month may delete the row locally; the sync form is a tombstone or an `effective_to` update so other devices see it. Prefer an `effective_to` update over a hard delete, so history is a normal merge. |
| updated_at | TEXT not null | |

Overlapping periods for the same scope are rejected in a transaction before commit. SQLite has no exclusion constraint; the LLD holds the lock by doing the check and the write in one transaction on a single writer connection. There is one writer on a phone.

### 1.6 history

Local and synced, so a new phone can show change history.

| Column | Type | Notes |
|---|---|---|
| id | TEXT pk | |
| entity | TEXT not null | `transaction`, `budget`, `category`, `settings`. |
| entity_id | TEXT not null | |
| rev | TEXT not null | |
| action | TEXT not null | `created`, `updated`, `deleted`, `restored`, `purged`. |
| before_json | TEXT | Decimal strings for amounts. |
| after_json | TEXT | |
| source | TEXT not null | |
| device_id | TEXT not null | |
| at | TEXT not null | UTC. |

No API edits this table. A purge removes history rows for that entity from the **live** snapshot only.

### 1.7 conflicts

Rows waiting for a human. While any row exists here, sync may pull and may update this table, and it must not push a snapshot that picks a winner.

| Column | Type | Notes |
|---|---|---|
| id | TEXT pk | |
| entity | TEXT not null | |
| entity_id | TEXT not null | |
| local_json | TEXT not null | Full row, decimal strings. |
| remote_json | TEXT not null | Full row, decimal strings. |
| created_at | TEXT not null | |

### 1.8 outbox

Plaintext, local only, inside SQLCipher. Not copied into the snapshot as a second channel. The snapshot is the state after the outbox has been applied locally, which it already has.

| Column | Type | Notes |
|---|---|---|
| op_id | TEXT pk | |
| entity | TEXT not null | |
| entity_id | TEXT not null | |
| created_at | TEXT not null | |

The outbox means "local state is ahead of the last successful push." The payload is the SQLite row itself, so there is one copy to get wrong, not two.

### 1.9 sync_state

| Key | Value |
|---|---|
| device_id | UUID created on first launch |
| last_pushed_sha | Git blob or commit sha of the snapshot we last successfully pushed or pulled |
| last_pulled_at | UTC instant |
| compacted_through | Id of the last compaction this device has accepted |

## 2. Indexes

```text
transactions (transaction_date DESC, created_at DESC, id DESC) WHERE deleted_at IS NULL
transactions (category_id, transaction_date) WHERE deleted_at IS NULL
transactions (type, amount_minor, category_id, transaction_date, created_at) WHERE deleted_at IS NULL
history (entity, entity_id, at)
```

The duplicate warning (same type, amount, category, and date within five minutes) reads the third index. It is a prompt, not a unique constraint. "Save anyway" inserts a second id.

## 3. Budget state

Unchanged arithmetic, now on SQLite integers:

- approaching or worse when `spent * 100 >= budget * 80`
- `at_limit` when `spent = budget`
- `exceeded` when `spent > budget`
- otherwise, if the first line is true, `approaching`
- otherwise `ok`

`spent` for a category is the sum of live expense rows in that month, minus the sum of live refund rows in that month if refunds are accepted. Income is not spent. Tombstones are not spent. The overall budget uses the same rule across categories.

Display percentages are half-up to one decimal, in application code, from those integers. They are not stored. They do not decide the state. A rounded 80.0 can still be `ok`. Category shares are not nudged to sum to 100.

## 4. Snapshot format (plaintext, before compression and encryption)

Canonical JSON. Object keys sorted. No insignificant whitespace. UTF-8. Amounts are strings. This is the bytes that are then compressed and sealed. It is never written to GitHub and never logged.

```json
{
  "compacted_through": null,
  "conflicts": [],
  "device_id": "uuid-of-writer",
  "format": 1,
  "purged_ids": [],
  "rows": {
    "budgets": [],
    "categories": [],
    "history": [],
    "rev_parents": [],
    "settings": {},
    "transactions": []
  }
}
```

`purged_ids` is the list of entity ids a compaction removed, so a phone that still has them locally does not treat "missing" as "create them again." The list is inside the ciphertext. The next compaction clears it after every device the user still cares about has synced, which we cannot know; clear it on the compaction after the one that introduced it, and accept that a phone offline across two compactions can resurrect a purged row. The compaction screen says that.

`conflicts` in the snapshot are the unresolved ones. A phone that pulls them shows the same choice and does not push a different winner.

Encryption envelope (the bytes of `snapshot.enc`):

```text
version: 1 byte = 1
nonce: 24 bytes
ciphertext: XChaCha20-Poly1305(compress(canonical JSON), DEK, nonce), tag appended
```

Compress with deflate. Compression is for size, not secrecy. The nonce is random per write.

## 5. Repository layout

```text
README.md          plaintext, no financial data, says the repo is an encrypted ledger
manifest.json      plaintext, no amounts, no notes, no names
snapshot.enc       envelope above
```

`manifest.json`:

```json
{
  "format": 1,
  "cipher": "xchacha20poly1305-ietf",
  "kdf": "argon2id",
  "kdf_salt": "base64",
  "kdf_m_kib": 19456,
  "kdf_t": 2,
  "kdf_p": 1,
  "wrapped_dek": "base64",
  "wrapped_dek_recovery": "base64",
  "snapshot": "snapshot.enc"
}
```

`wrapped_dek_recovery` may be absent if the user refused the recovery code. The manifest must not grow an `entry_count` or a note. File size is already visible.

Commit message, always: `Update encrypted ledger`.

## 6. What is deliberately absent

- No `REAL` column and no float JSON.
- No server user table, no session table, no email token table.
- No raw SQLite file in the repo.
- No per-entry file in the repo (ADR-014).
- No plaintext changelog.
- No shared household tables.
