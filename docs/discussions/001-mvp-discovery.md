# 001 — MVP discovery: DEV review

Status: **Proposed**
Author: DEV
Date: 2026-11-19 (IST)
Reviews: `PM_PROPOSAL.md`, `docs/product/PRD.md`, `USER_STORIES.md`, `ACCEPTANCE_CRITERIA.md`, `ROADMAP.md` (committed unchanged)

This note is the DEV agent's evaluation. It does not invent a reply from the PM.

**Read the addendum at the end before the sections above.** The Product Owner later chose a native local-first app with GitHub sync and no server. The earlier sections are the withdrawn server review. The addendum is the current decision log.

---

## [DEV]

### Problem, user, and shape

I agree with the problem statement and with the product principles, in the order the PRD gives them. Correctness and privacy outrank capture speed, and capture speed outranks an impressive AI feature. The primary user (one salaried person in India, mixed UPI, card, and cash, INR, Asia/Kolkata) is a coherent v1. The exclusion list is mostly right: bank and SMS import, OCR, households, multi-currency UI, recurring transactions, and AI that edits or advises are each large enough to sink a ledger that does not exist yet.

I also agree with the engineering constraints the PM already set: exact money, calendar dates, server-side isolation, soft delete, audit, idempotency, optimistic concurrency, budgets that warn at 80% and over 100%, CSV that is safe to open, and an AI path that cannot write. Those are designed in the architecture docs, not deferred to "we'll be careful."

The disagreements below are places where a product rule is ambiguous, contradictory, or more expensive than the proposal admits. I am not asking to widen MVP scope.

### Challenges

#### C1. "Savings" is the wrong word for income minus expenses

The dashboard's third headline number is defined as income minus expenses and then called savings (PRD FR-DASH-1, AC-DASH-1.1, CUJ-4). That number is net cash flow for the month. It is not savings. It ignores money the user already held, cash in a drawer, money moved to a fixed deposit, loan principal versus spending, and refunds that are really reversals. The moment the Product Owner compares it with a bank balance, the product looks wrong, which is the failure mode the PRD itself ranks as fatal.

Accounts and transfers are correctly out of MVP (TXN-9, P1). Until they exist, the honest label is **net** or **unspent income**, with the formula shown. I will compute `income - expense` exactly either way. The word on the screen is a product decision. See TQ-5.

#### C2. Email verification is an unbuilt vendor, and it fights the two-minute first run

AUTH-1 and AUTH-3 require a verification mail and a 30-minute reset link before the ledger is usable (AC-AUTH-1.1, CUJ-1 ≤ 2 minutes). That needs an email provider, a secret, deliverability, and a decision about where the app is hosted (PO-Q4). None of those exist. For a single known operator on a private machine, verification email is a ceremony that blocks the only user. For a public URL, it is a real control and I want it.

The architecture puts mail behind an interface (`console` for local, SMTP later) so the ledger code does not depend on a vendor. Whether Phase 3's exit criteria require a live provider before any transaction can be saved is a product and hosting decision. My recommendation: passwords, sessions, and isolation are Phase 3; mandatory email verification is on only when the deployment is reachable from the public internet. See TQ-4.

Related: AC-AUTH-2.4 sets session expiry at 30 days of inactivity. That is long for a ledger. I recommend a 7-day idle timeout and a 30-day absolute cap, both enforced in the database, not only in the cookie. The number is a product assumption, so it stays pending (TQ-3).

#### C3. The phase gates serialize the wrong things, and they invite real data into Git

Three roadmap rules need a cleaner reading.

1. Phase 4's exit includes "the Product Owner uses it for ≥ 2 weeks and logs ≥ 50 real transactions with 0 correctness defects." That is a product dogfood gate. It is valuable. It is not something engineering can satisfy with a pull request, and the evidence must not be a CSV, a database dump, or a screenshot of a real ledger committed to this repository. Defects get filed as issues with synthetic reproductions.
2. A hard dashboard p95 under 1 second, measured on shared CI, will flake. Ten thousand rows is an easy Postgres problem if the queries are set-based. I will treat the number as a design budget (one transaction, no per-category queries, indexes as in the schema doc) and re-measure on the real host after PO-Q4. I do not want a flaky p95 job blocking merges.
3. Phase 5's labelled-set gate (category top-1 ≥ 85%, and the rest of that paragraph) should gate **turning AI on**, not the release of the ledger. The PRD already says every flow works with AI disabled. If the parser misses 85% on categories, the Product Owner still has a usable app. Holding v1.0 until the model cooperates makes the AI vendor part of the critical path for a product whose principle is that AI is optional.

I am not proposing that we skip the dogfood or the eval. I am proposing we stop using them as locks on unrelated work. See TQ-1 and TQ-2.

#### C4. Budget carry-forward is a temporal model, and two acceptance rules disagree with rounded percentages

"A budget carries forward until changed, and a change affects the current and future months only" (A8, AC-BUD-1.1, AC-BUD-1.2) cannot be a single mutable row. A single row either rewrites November when December changes, or forces us to copy a row every month. The schema therefore uses effective-dated, non-overlapping periods. That is a technical decision, recorded in the schema doc and the LLD. I need a product answer on one behaviour the PRD does not state: **can the user correct September's budget in November, or is a past month locked?** I have specified the API as locked to the current month until told otherwise (TQ-6).

Two test phrases will become production bugs if we implement them naively:

- FR-BUD-3 lists three states: OK below 80%, Approaching at or above 80%, Exceeded above 100%. AC-BUD-4.2 then says exactly 100% is "Approaching / At limit", which is two names, and that exactly 100% is not Exceeded. I will compute four derived states — `ok`, `approaching`, `at_limit`, `exceeded` — with integer comparisons (below). The badge text for exactly 100% needs one word from the PM.
- AC-BUD-4.1 talks about 79.9% becoming 80.0%. Display percentages are rounded half-up to one decimal place. State is not. Example: budget ₹100.01, spend ₹80.00 is 79.992% of budget, which rounds to 80.0% on screen and is still under the exact 80% threshold (spend 8000 paise, threshold at 8001). The badge must follow the exact state. The rounded percent is allowed to read 80.0 while the state is still OK. Tests should assert minor units, not the rounded glyph. I will not nudge a category's percentage so that a pie chart sums to 100. Amounts stay authoritative (AC-DASH-2.1).

Comparison, all integers, no division:

- approaching or worse when `spent_minor * 100 >= budget_minor * 80`
- at limit when `spent_minor == budget_minor`
- exceeded when `spent_minor > budget_minor`
- otherwise, if the first inequality holds, approaching
- otherwise ok

Income never counts. A transaction counts in the month of its calendar date, including a later day in the same month. Warnings are derived when a transaction is saved and when the budget screen is read. They are not stored events, and they do not need a queue.

#### C5. An audit log that keeps amounts forever breaks the purge and the deletion promise

FR-AUD-1 asks for an immutable before/after log. AC-TXN-4.3 purges a transaction 30 days after delete. AC-AUTH-6.1 purges the account and its data within 30 days, and the PRD extends that spirit to backups ageing out. If the audit table is truly append-only and keeps amount and note, then purge is cosmetic: the second copy of the ledger remains.

My rule, unless the PM or the Product Owner overturns it:

- While the transaction exists, including during the 30-day restore window, audit rows are immutable. No API updates or deletes them. A trigger rejects `UPDATE` and `DELETE` for the application role.
- The hard purge of an expired transaction deletes the row and its audit rows in one database transaction. A maintenance log line records that a purge happened, with ids and a count, and without the amount or the note.
- Account deletion does the same for every row the user owns, then deletes the user. Backups age out under the 7-day retention already in OPS-1; we do not build selective backup surgery in MVP.
- Category rename is included in the audit stream even though the PRD lists transaction and budget only. Renaming "Food" rewrites the meaning of history (AC-CAT-2.3). This is a small technical addition.

That is a deliberate weakening of "immutable forever" in favour of the erasure promise. I would rather weaken immutability than lie about deletion. Confirm or override it in TQ-7.

#### C6. Several smaller holes that will otherwise be filled in code without a decision

These are specified in the architecture so implementation does not invent them. Push back if the product intent differs.

| Topic | What the PRD says | What I will do unless told otherwise |
|---|---|---|
| Recently deleted | Restore is required (TXN-4). No screen lists deleted rows. | `GET /transactions?deleted=only`, owner-scoped, with days remaining. |
| Multi-expense chat (AC-AI-2.4) | PM prefers a clear message. DEV was asked to propose. | Draft nothing. 422 `MULTIPLE_TRANSACTIONS_UNSUPPORTED`. Drafting "the first one" is how a lunch gets saved as the rent. |
| Unrecognised amounts ("1.2cr", litres versus lakh) | A12 names `2k`, `1.5L`, `350/-` only. | Parser accepts plain numbers, `k`, `L`/`lakh`, and `/-`. Anything else leaves amount empty and confirm disabled. Confirmation still shows the rupee figure for `1.5L` because "L" is ambiguous. |
| Far-past dates | Only the future bound is specified (≤ 1 year). | Reject dates before 2000-01-01 as junk input. Reversible. |
| Upcoming flag | Stored-looking word in AC-TXN-1.8. | Derived: `date > today` in the user timezone. Not a column. It goes stale if stored. |
| Category name length | Unspecified. | 1–40 code points after trim. Uniqueness is case-insensitive on the trimmed name, per user and type, including archived rows. |
| Password length | At least 10, not a common password. | Also reject above 128 characters so hashing cannot be used as a denial of service. |
| Session fixation and reset | AC-AUTH-3.2 revokes other sessions on reset. | Also rotate the session id on login. |
| CSV and Excel | UTF-8, formula injection guarded. | No BOM. A BOM breaks a strict read of the first header cell. Document that Excel must import as UTF-8. |
| Pie charts | Not required. | Do not ship a chart library in the core milestone unless the PM asks. The breakdown table is the source of truth. A chart can sit on top later. |
| PWA (PO-Q2) | PM recommends an installable web app. | A home-screen manifest is fine. A service worker must not cache API responses or transaction data. Offline remains out of scope. |
| Download CSRF | Export is a GET. | Require the CSRF header anyway, so another site cannot top-level-navigate the browser into downloading the ledger. |
| Negative savings rate | AC specifies a deficit amount, and "—" only when income is zero. | If income is positive, show the rate even when negative, half-up to 1 decimal. If income is zero, show "—". |

#### C7. What I am not challenging

Excluded on purpose, and I agree: bank/SMS/Account Aggregator import, CSV import in MVP, OCR, recurring transactions, multi-account balances, household splits, email/push budget alerts, AI recommendations and anomaly detection, AI edit/delete, goals, PDF, native apps, offline mode, Redis, microservices. Payment method as a fixed optional enum is the right size. Seeding India-relevant categories per user is the right size. Read-only AI answers that quote backend aggregates, with a link to the filtered list, is the right shape for Phase 5.

PO-Q6 can still pull CSV import into MVP if the Product Owner has a spreadsheet they need on day one. That is a scope change, not a configuration flag. It adds column mapping, validation, and dedupe. I would rather the Product Owner type the opening balances, or wait for P1, than fold import into the first release. The data model does not need to change to keep the option open.

### Assumptions A1–A12

| ID | PM default | DEV position | Status |
|---|---|---|---|
| A1 | One person per account | Agree for the product. Schema and authorization are still per `user_id`, with two-user isolation tests, so a later household feature is not a rewrite of every table. No sharing UI, no invitations. | Product pending PO-Q1. Technical shape decided. |
| A2 | Responsive web, PWA suggested | Agree on responsive web. Reject native. PWA only as a manifest, no cached ledger. Client is a Vite SPA, not Next.js (ADR-001). | Product pending PO-Q2. Client decided. |
| A3 | Manual entry plus confirmed AI drafts. No import. | Agree. | Product pending PO-Q3 and PO-Q6. |
| A4 | INR in the product, currency code stored | Agree. API rejects any currency other than INR. Database does not hard-code a check constraint, so a later currency is a product change rather than a meaning change. Aggregations refuse to add mixed currencies. | Technical decision. |
| A5 | Default timezone Asia/Kolkata, changeable | Agree. IANA name on the user. Transaction `date` does not move when the timezone changes (AC-AUTH-5.2). | Technical decision. |
| A6 | Email and password, with verification | Password auth: agree. Verification mail: see C2. | Product pending PO-Q4 / TQ-4. |
| A7 | Calendar-year reports | Agree as the only preset in MVP. Reports are date-range queries, so an April–March preset does not need a new table. | Product pending PO-Q6. |
| A8 | Fixed 80% threshold, budgets carry forward | Threshold fixed: agree. Carry-forward: effective-dated rows. Past months locked: pending. State names: pending. Math: decided (C4). | Split. See TQ-6. |
| A9 | Fixed payment-method list, optional | Agree. Last-used default is the latest non-deleted transaction's method, computed, not stored. | Technical decision. |
| A10 | Soft delete, 30-day restore | Agree on the window. Purge is a daily command, not a queue. Audit interaction: see C5. | Window agreed. Erasure pending TQ-7. |
| A11 | Backend-only, swappable AI. Grok is a candidate. | Agree. Provider sits behind an interface. Grok is the default implementation to write in Phase 5, not a unique capability we design around. No model id is pinned here; pinning before the eval set exists is guesswork. | Provider pending PO-Q5. Shape decided (ADR-005). |
| A12 | English UI, Indian money shorthand in the parser | Agree. No i18n framework. Parser coverage is the shorthand in the acceptance criteria, nothing broader. | Technical decision. |

### Roadmap

The six phases and the rule "do not start a phase while critical exit criteria are unmet" are sound if "critical" means correctness, isolation, and an approved design. They are not sound if "critical" includes a calendar dogfood, a flaky latency check, or an AI accuracy number. Proposed reading:

| Phase | DEV reading of "done" |
|---|---|
| 1. Discovery | This review is the DEV half. Done only when the PM has answered the challenges and the Product Owner has answered or deferred the questions. |
| 2. Architecture | Docs in `docs/architecture/` are the proposal. Done when the Product Owner approves them and the PM confirms the traceability matrix, including the rows marked pending. |
| 3. Foundation | One-command local run, migrations, session auth, isolation tests, CI that runs those tests and a secret scan. Email provider only if TQ-4 says the deploy is public. |
| 4. Core ledger | Acceptance tests for AUTH (as scoped), TXN, CAT, LIST, DASH, BUD, RPT, plus the reconciliation property test. Dogfood starts after that and gates calling the build safe for daily use. It does not gate writing the Phase 5 harness against synthetic phrases. |
| 5. AI | Eval harness and the confirm-only path. Failure of the accuracy bar disables AI. It does not roll back the ledger. |
| 6. Production | Security review, one restore drill, log scan, deploy and recovery notes, Product Owner go/no-go. |

Complexity for each slice is in `docs/development/BACKLOG.md`. I am not estimating weeks.

### Security and privacy, beyond the PRD list

The PRD's security section is the right list. A few consequences for the design:

- Isolation is enforced twice: every query is scoped in the application, and Postgres row-level security denies rows when `app.user_id` is unset or different. A missing `WHERE` should return nothing, not everything.
- Login and signup take the same password-hashing work on the "user missing" path so response time is a worse enumeration oracle. The HTTP body is generic either way (AC-AUTH-1.2).
- Notes are untrusted. They are stored and rendered as text. They are not passed to the AI on the query path. On the draft path, the user's own message is the text they just typed; the disclosure has to say that, because the message usually contains the amount. We do not upload the ledger.
- There is no write tool in the model tool list. The confirm button calls `POST /transactions`, which runs the same validation, idempotency, duplicate warning, and audit path as a manual save. A prompt cannot reach that code.
- Logs store route template, status, latency, request id, and AI token counts. They do not store amounts, notes, emails, passwords, cookies, or raw model payloads.
- Backups are encrypted dumps with a 7-day retention. Who operates them is PO-Q4. RPO of 24 hours and RTO of 4 hours are acceptable for this product only after someone actually owns the host.

### AI, when we get there

Designed now so Phase 5 does not invent a side door. Not built now.

- `POST /ai/drafts` returns a draft and writes nothing. Closing the page leaves no row (AC-AI-1.3). I rejected a server-side draft table for MVP because a stored draft is a save the acceptance test does not allow.
- Confirm is the ordinary create endpoint with `source = ai_assistant`. The client sends the fields the user saw, after edits. The model output is never persisted as a transaction by itself.
- Ambiguous or invalid model output becomes an editable draft with confirm disabled, or a manual form pre-filled with the original text in the note if the provider errors (AC-AI-1.6, AC-AI-1.7).
- Queries call a fixed set of read functions: period totals, category breakdown, compare two periods, budget status. The functions run as the authenticated user. The UI treats the structured tool result as the answer. Model prose is shown only when every number in it appears in the tool payload; otherwise the prose is dropped.
- AI defaults off until the user accepts a disclosure that names the provider and states that the typed message is sent, and the stored ledger is not.
- A per-user daily cap (proposed: 30 draft calls and 60 queries) fails closed when the provider is down, the cap is hit, or the key is missing. The rest of the app keeps working. The cap is a safety default, not the Product Owner's budget. The real ceiling is PO-Q5.

---

## [DISCUSSION]

These points are unresolved. I am not writing the PM's half.

**D1. Label for income minus expenses.** DEV will not call the API field `savings` while the challenge is open. The proposed field is `net`. The UI string can still be whatever the PM and Product Owner decide. Changing the label does not change the formula the acceptance test checks (₹32,659.25 in AC-DASH-1.1).

**D2. Email verification versus CUJ-1.** DEV's design supports both "verify before any financial route" and "verify only on a public deploy." Choosing the first without a mail provider means Phase 3 cannot finish. I want the PM to treat mail delivery as a dependency of hosting, not as a property of the ledger.

**D3. What a phase exit is allowed to block.** DEV's reading is in the roadmap table above. If the PM wants the 2-week dogfood to block all coding, including the synthetic AI eval, say so. I think that wastes the wait and does not make the ledger safer.

**D4. Past budget months, and the word for exactly 100%.** DEV has specified a lock and four state codes so the code has one behaviour. Both are easy to change before any data exists and annoying after. This is the highest-leverage product reply on the data model.

**D5. Audit versus erasure.** DEV has chosen erasure on hard purge. If the PM wants amounts retained indefinitely for audit, the privacy section of the PRD should say that purge removes the transaction from the product but not from an internal log, and the Product Owner should agree in writing. I recommend against that for a personal app.

**D6. AC-AI-2.4.** DEV proposes drafting nothing. The PM's stated preference matches this. I will treat silence as agreement when implementation starts, and a disagreement as a reason to stop and rewrite the endpoint.

**D7. Recently deleted, and category unarchive.** Both are small and, in my view, required to operate the features already in scope. Unarchive is the one I will drop first if the PM calls it scope creep: the user can create a new category, and history stays on the archived one. A trash list is not optional if restore exists.

**D8. Stack.** Documented in the ADRs. The PM asked us to avoid Redis, queues, and microservices unless an acceptance criterion needs them. None does. Daily purge and backup are a single command on a timer. I am rejecting Next.js as the client because it would put a Node server beside FastAPI for an app with no public pages to render. If the PM has a user-facing reason for a Node server, I want to hear it before Phase 3.

**D9. Performance numbers.** I accept them as targets on a single small host. I do not accept them as a CI merge gate until we have measured the real environment. Query shape is the part we can enforce on every pull request.

Nothing here is a reason to delay the architecture review. The pending rows change labels, gates, and one API rule (whether a past month's budget can be edited). They do not change the choice of a modular monolith, Postgres, or integer money.

---

## [DECISION]

The table below is the **first-draft** decision log from earlier on 2026-11-19. The Product Owner then changed the architecture. Those rows are **not** the plan to build. The decision that replaces them is the addendum at the end of this file. It is kept here so the earlier reasoning is not silently rewritten.

| ID | Decision | Status | Where |
|---|---|---|---|
| DEC-T1 | Client is a Vite + React + TypeScript SPA, one origin with the API. Not Next.js. Not native. | Proposed, DEV-owned | ADR-001 |
| DEC-T2 | One Python FastAPI service, Pydantic v2, SQLAlchemy 2, Alembic. Not Django. Not a second backend. | Proposed, DEV-owned | ADR-002 |
| DEC-T3 | PostgreSQL 16 is the only database. Tests use Postgres, not SQLite. | Proposed, DEV-owned | ADR-003 |
| DEC-T4 | Argon2id, server-side sessions, HttpOnly `SameSite=Lax` cookies, CSRF header on mutating requests and on export. Not localStorage JWT. Not a managed login vendor for v1. | Proposed, DEV-owned | ADR-004 |
| DEC-T5 | AI provider is a backend interface. MVP behaviour is stateless draft plus read-only tools plus confirm via `POST /transactions`. Grok is the Phase 5 implementation candidate. Model id is not pinned. | Proposed, DEV-owned. Provider and spend pending PO-Q5. | ADR-005 |
| DEC-T6 | Ship as one Docker Compose application on one host. No microservices. The host (laptop, VM, or a managed service) is not chosen. | Packaging proposed. Location pending PO-Q4. | ADR-006 |
| DEC-T7 | Persist money as `bigint` minor units and a currency code. Accept and return decimal strings. Reject JSON numbers and more than 2 decimal places. INR only in the API. | Proposed, DEV-owned | ADR-007 |
| DEC-T8 | Transaction day is a `date`. User holds an IANA timezone, default `Asia/Kolkata`. System timestamps are `timestamptz`. | Proposed, DEV-owned | ADR-008 |
| DEC-T9 | Soft delete with `deleted_at`. Hard purge at 30 days removes the financial audit rows with the transaction. Daily command, no Redis. | Proposed, DEV-owned, erasure pending TQ-7 | ADR-009 |
| DEC-T10 | `Idempotency-Key` on create, stored per user for 24 hours. Optimistic `version` on transaction and budget updates; mismatch is 409 and does not write. Duplicate warning is a separate check, serialized with a transaction advisory lock. "Save anyway" uses a new key. | Proposed, DEV-owned | ADR-010 |
| DEC-T11 | One process, one database. Background work is `python -m app.maintenance`, run once a day. Revisit a queue only if we run more than one API replica or a job outgrows a few seconds. | Proposed, DEV-owned | ADR-011 |
| DEC-T12 | Row-level security on tenant tables, fail closed when `app.user_id` is unset, plus explicit `user_id` filters in the application. | Proposed, DEV-owned | HLD, schema |
| DEC-T13 | Budget rows are non-overlapping month periods. Resolved budget for a month is the period containing that month. Threshold math is the integer comparison in C4. Warnings are computed, not stored. | Proposed, DEV-owned. Past-month edit and the 100% label pending. | Schema, LLD |
| DEC-T14 | Reports take a date range. "This month", "calendar year" are presets. Indian financial year is not a schema change. | Proposed, DEV-owned. Preset pending PO-Q6. | API spec |
| DEC-T15 | Percentage display uses decimal half-up to 1 decimal place, in application code, from integer sums. SQL does not cast money to `float8`. Category percents are not adjusted to sum to 100. | Proposed, DEV-owned | LLD |
| DEC-P1 | Single user versus household | **Pending PO-Q1** | |
| DEC-P2 | Web versus native, and whether a manifest is wanted | **Pending PO-Q2** | |
| DEC-P3 | Imports in v1 | **Pending PO-Q3 and PO-Q6** | |
| DEC-P4 | Where it runs, and who restores backups | **Pending PO-Q4** | |
| DEC-P5 | AI vendor, monthly ceiling, and permission to send typed messages | **Pending PO-Q5** | |
| DEC-P6 | Calendar year versus Indian financial year | **Pending PO-Q6** | |
| DEC-P7 | UI label for income minus expenses | **Pending** (C1, TQ-5) | |
| DEC-P8 | Mandatory email verification before the first transaction | **Pending** (C2, TQ-4) | |
| DEC-P9 | Idle session length | **Pending** (TQ-3). DEV recommendation: 7 days idle, 30 days absolute. | |
| DEC-P10 | Editing a budget for a month that has already ended | **Pending** (TQ-6). DEV recommendation: locked. | |
| DEC-P11 | Badge text at exactly 100% | **Pending** (C4). DEV recommendation: `at_limit`. | |
| DEC-P12 | Whether hard purge deletes audit amounts | **Pending** (TQ-7). DEV recommendation: yes. | |

---

## [PRODUCT OWNER INPUT REQUIRED]

The first six are the PM's questions. I have added the technical consequence of each answer so a "yes" has a known cost. The rest are questions the PM's list does not cover and that I should not guess.

### PO-Q1 — Single user or household?

Is v1 only for you, or must your spouse or family log into a shared ledger?

DEV note: I recommend one person per account, as the PM does. Sharing is a second permission model (invitations, who can delete, what happens when someone leaves) and it is the kind of change that makes audit and isolation much harder. The schema is already per user, so choosing "just me" does not throw away the multi-user tests. Choosing "household" stops Phase 3 until we redesign authorization. I would not start that in v1.

### PO-Q2 — Web or mobile?

Is a responsive web app, usable on a phone browser, acceptable for v1? Do you want an installable home-screen icon? Do you need a native Android or iOS app?

DEV note: I recommend a responsive web app and, if you want the icon, a manifest only. I recommend against a native app and against any offline cache of transactions. A native app is a second client we would have to keep correct on money and dates. Say if a phone browser is not acceptable; that changes DEC-T1.

### PO-Q3 — Bank, UPI SMS, or statement import?

Is automatic import a requirement for you to keep using the app, or are typing and (later) confirmed natural-language entry enough for v1?

DEV note: I recommend leaving import out. It is a privacy problem and a deduplication problem, and it dominates the schedule. If you know you will not type transactions, say so now, because the rest of the plan assumes you will.

### PO-Q4 — Hosting and backups

Should this run on a managed cloud you pay for, on a machine you control, or only on your own computer? Who is responsible for backups and for trying a restore?

DEV note: the deployable unit I can own without this answer is Docker Compose on one host: the API, the static client, and Postgres. I am not picking a cloud vendor in this proposal. A laptop-only setup can use a local mail sink and still meet the ledger tests. A public URL needs TLS, a real SMTP provider, encrypted backups, and someone who will run the restore drill in Phase 6. The RPO of 24 hours means a day of entries can be lost if the disk dies. Say if that is unacceptable.

### PO-Q5 — AI provider, cost, and privacy

Is Grok the provider you want? What monthly spend ceiling is acceptable? Are you willing to send the sentence you typed (which usually contains the amount and the description) to that provider? We would not send the stored ledger.

DEV note: the app is specified to work with AI switched off, and AI stays off until you accept a disclosure. If you do not want any transaction text to leave the machine, we should leave Phase 5 unbuilt rather than pick a different model and pretend the text stayed local. A ceiling of ₹500/month, if that is the example you accept, becomes a hard stop in configuration, not a hope. I also recommend that missing the Phase 5 accuracy bar turns AI off and does not block the ledger (TQ-2).

### PO-Q6 — Reporting year and existing data

Should yearly reports use the calendar year or the Indian financial year (April–March)? Do you have a spreadsheet you need loaded before the app is useful?

DEV note: I recommend calendar year for v1, with the financial year as a later preset on the same date-range report. I recommend against pulling CSV import into MVP. If the spreadsheet is a must on day one, treat that as a scope change and re-plan Phase 4 before coding; do not add it quietly in the middle of transactions.

### TQ-1 — Dogfood versus engineering exit

May Phase 4 be called engineering-complete when the automated acceptance tests pass, with the 2-week, 50-transaction dogfood gating "safe for daily use" rather than gating all further coding? Real transactions would stay in the private database. GitHub would see only synthetic bug reports.

DEV recommendation: yes.

### TQ-2 — Does AI accuracy block v1.0?

If the ledger is solid and the phrase tests miss the category target, do we ship the ledger with AI disabled, or do we hold the whole release?

DEV recommendation: ship the ledger, keep AI off until the bar is met.

### TQ-3 — Session length

The draft acceptance criteria say 30 days of inactivity. I recommend 7 days idle and 30 days absolute, both enforced on the server.

### TQ-4 — Email verification

For v1, is a public sign-up page in scope? If the only account is yours and the app is not on the public internet, may we skip email verification until a public deploy exists? Password reset mail has the same dependency.

DEV recommendation: skip mandatory verification on a private deploy; require it on a public one.

### TQ-5 — Label

May the dashboard call income minus expenses **net** (or "unspent income") instead of **savings**? The number does not change.

DEV recommendation: yes, call it net.

### TQ-6 — Past budgets

In November, can you change September's budget, or only the current month and the future?

DEV recommendation: current month and future only. Past months stay as they were.

### TQ-7 — Audit and deletion

When a soft-deleted transaction is finally purged, and when you delete the account, must the amounts and notes disappear from the audit history as well?

DEV recommendation: yes. Keep the audit trail only while the transaction is still restorable. The maintenance log may record that a row was purged, without the amount or the note.

### TQ-8 — Exactly 100% of a budget

When spend equals the budget, should the product say "at limit", "approaching", or both? Over 100% would still say "exceeded" and show the overspend.

DEV recommendation: a distinct "at limit" state, and "exceeded" only when spend is greater than the budget.

---

## What I need back from the PM

The paragraph that used to end here asked for a reply on the server-shaped design. That design is withdrawn. The open points are in the addendum.

---

## Addendum — Product Owner direction (2026-11-19)

The Product Owner changed the architecture after the first draft. This addendum is the decision record. The sections above stay as the history of the server proposal. Do not implement them.

### [DECISION]

Confirmed by the Product Owner, and accepted as product decisions:

| ID | Decision |
|---|---|
| DEC-PO-1 | Native mobile app, local-first. On-device SQLite is the source of truth. Saves are instant and work offline. |
| DEC-PO-2 | No backend and no hosted Postgres. We do not hold ledger data or GitHub tokens. |
| DEC-PO-3 | Anyone can install the app and keep an isolated ledger. This closes PO-Q1 and PO-Q2. It is not a household or shared ledger. |
| DEC-PO-4 | Sync and backup go through a private GitHub repository the user owns. Pull, then merge, then push. |
| DEC-PO-5 | Entries created independently merge because their ids differ. Two edits of the same entry since the last sync are shown to the user, who chooses, before any push. Never silently overwrite. |
| DEC-PO-6 | Deletes sync as tombstones. |

DEV decisions for how to meet that, all still **Proposed** until the Product Owner accepts this revision of the architecture:

| ID | Decision | Where |
|---|---|---|
| DEC-T16 | Android client in Kotlin and Jetpack Compose. iOS is not in the first build unless NQ-1 says otherwise. | ADR-012 |
| DEC-T17 | SQLCipher SQLite, integer paise, calendar dates as `YYYY-MM-DD` text. No server, no Postgres, no Redis. | ADR-013, ADR-007, ADR-008 |
| DEC-T18 | One encrypted logical snapshot per sync, not one GitHub file per entry and not a git merge of ciphertext. | ADR-014 |
| DEC-T19 | Random data key wrapped by a passphrase and by a one-time recovery code. Device keystore caches the unwrapped key. Lost passphrase and lost recovery code, with no device still holding the key, means the data is lost. There is no reset. | ADR-015 |
| DEC-T20 | GitHub App device flow, contents permission on one private repo, tokens only in the keystore. One commit per sync. Back off on rate limits. | ADR-016 |
| DEC-T21 | 30-day restore is the tombstone in the live snapshot. Real erasure is a confirmed history squash of the **data** branch plus data-key rotation, or deletion of the repo for a full wipe. GitHub may keep internal copies. | ADR-017 |
| DEC-T22 | Budget states are `ok`, `approaching`, `at_limit`, `exceeded`, from integer comparisons. The PM restated these names with this direction, so TQ-8 is closed. | Schema, LLD |
| DEC-T23 | AI stays an optional, undecided module. If it is built, the phone calls the provider and writes SQLite only after confirm. | ADR-005 |

Closed by this direction: PO-Q1, PO-Q2, PO-Q4, TQ-3 (server sessions), TQ-4 (email verification), TQ-7 as a server purge (replaced by DEC-T21), TQ-8.

Not closed: the word "savings" versus net (TQ-5), whether a past month's budget can be edited (TQ-6, still proposed as locked), calendar year versus Indian financial year (PO-Q6), bank import (PO-Q3, still out), and whether AI is built (PO-Q5, explicitly undecided).

Withdrawn as the thing to build: Vite, FastAPI, Postgres, server sessions, Docker Compose as the product runtime, row-level security, and the HTTP API. The ADRs are marked Superseded rather than deleted.

### [DEV]

I will build this if the Product Owner keeps it. I do not think it is the low-risk design. The candid version:

GitHub is a reasonable place for a person to keep an encrypted backup, and a bad database. The rate limits, the history that never forgets, the account requirement, and the metadata (when you sync, how large the blob is, which GitHub user you are) are structural. The protocol in this revision stays inside those limits by uploading **one** snapshot per sync and by never putting a save on the network path. That makes the idea workable. It does not make GitHub a normal sync service with a contract. If the OAuth app is suspended, or the user has no GitHub account, the phone still has the ledger and the CSV. That local path is mandatory, not a grace case.

A second bad idea, which I am not doing: letting git merge the encrypted file, or writing one API object per transaction. The first cannot see a conflict. The second will hit GitHub's content-creation limits as soon as the phone has been offline for a while.

Deletion is the place the new direction can quietly lie. Tombstones give a real 30-day restore. They do not erase Git history. I am recommending a confirmed squash of the data branch plus a new data key, and repo deletion for a full wipe, and a sentence in the product that GitHub's own backups are outside our reach. Key rotation alone does not do it. Squash alone does not do it if the old key still opens whatever blobs remain.

The passphrase cannot be reset. If a later review asks for "forgot passphrase," the honest answer is that it reintroduces a server or a stored key, which DEC-PO-2 forbids.

Android-only is a product hole if "anyone can install" includes an iPhone in the first release. I am not guessing that it does. NQ-1 is the question. Flutter is the alternative, and it should be chosen before code, not after.

### [DISCUSSION]

Analyst findings, folded in as **Proposed**. The Product Owner has not confirmed them. They are in the schema and the LLD so the next edit does not assume them away.

- Refunds reduce the expense category. They are not income. The amount stays a positive integer.
- The app warns, and does not block, when a card-bill payment or an EMI is likely to double-count a purchase already logged.
- CSV is UTF-8 with a BOM.
- Repeat copies a past transaction into a new id dated today. It is not a recurring scheduler.
- A wrong passphrase backs off and never wipes or permanently locks.

I am not treating silence as acceptance. NQ-5 asks for a yes or no on each.

### [PRODUCT OWNER INPUT REQUIRED]

**NQ-1.** Does the first release have to run on iPhone, or is Android enough until a later client? Android keeps Kotlin. Both platforms in the first release should switch this design to Flutter before any code.

**NQ-2.** Is it acceptable that sync requires a GitHub account, while a person with no GitHub account can still keep a ledger on one phone and export CSV?

**NQ-3.** Please confirm there is no passphrase reset, and that setup shows a one-time recovery code. Losing both, with no phone that still has the key, means the GitHub copy cannot be opened.

**NQ-4.** Please confirm erasure: day-to-day deletes are tombstones for 30 days; forgetting old versions is a confirmed history squash of the data repo plus a new data key; deleting the account deletes the repo and the local keys. GitHub may keep internal copies we cannot remove.

**NQ-5.** Yes or no on each analyst item: refunds as negative expenses, the card/EMI warning, the CSV BOM, repeat-as-a-new-entry, and no hard lockout.

**NQ-6.** AI stays out of the build until you explicitly want it. Confirm that "undecided" means we do not start it in the first implementation.

**Still open from the first review, and not answered by this direction:** TQ-5 (call the headline number net, or savings), TQ-6 (lock past budget months), PO-Q6 (calendar year or Indian financial year). Bank and SMS import stays out unless you reopen PO-Q3.
