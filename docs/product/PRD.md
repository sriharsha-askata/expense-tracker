# Product Requirements Document — Personal Expense Tracker

Status: **v0.2 — revised to the Product Owner's local-first Android direction (2026-11-19). Product rules accepted by the Product Owner; Phase 3 starts only on his explicit go-ahead.**
Owner: PM agent · Product Owner: Sri Harsha · Last updated: 2026-11-19 (IST)

> Labels used: **[ASSUMPTION]** = PM default made to keep moving (reversible); **[DECISION DEC-PO-n]** = a Product Owner decision recorded in `docs/discussions/001-mvp-discovery.md`; **[OPEN]** = still needs an answer.

## Changelog

### v0.2 — 2026-11-19 (IST) — revision to the Product Owner's direction

Why: the Product Owner replaced the web-app-plus-server plan with a native, local-first phone app that has no server of ours, and answered the DEV/analyst questions (see `docs/discussions/001-mvp-discovery.md`, DEC-PO-1 to DEC-PO-16). This revision rewrites the product docs to match. It says what and why; how is in `docs/architecture/`.

What changed:
- **Product shape:** responsive web app with accounts → **native Android app (React Native/Expo; iOS possible later)**. The ledger is an encrypted SQLite database on the phone; saves work offline. There is no server, no sign-up and no login of ours.
- **Users:** "multi-user" now means anyone can install the app and gets their own isolated ledger on their own phone. There is no household or shared ledger.
- **GitHub is optional.** It is used only for encrypted sync and backup to the user's own private repository. Sync pulls, then merges, then pushes. Different entries merge automatically; the same entry edited on two phones is shown to the user to choose before anything is pushed; deletes sync as tombstones. One phone plus CSV export is a complete app.
- **Keys:** a passphrase plus a one-time recovery code. There is **no reset**. Losing both means the encrypted GitHub copy cannot be opened, and setup says so in plain language.
- **Erasure:** 30-day restorable deletes; "forget history" = a confirmed squash of the data branch plus a new data key; deleting the repository wipes the account. GitHub may retain copies we cannot remove, and the privacy text says so.
- **Analyst items accepted:** refunds reduce their expense category (they are not income); a non-blocking card-bill/EMI double-count warning; CSV is UTF-8 with a BOM; "repeat" creates a new entry dated today; a wrong passphrase means backoff, never a hard lockout.
- **Dashboard and reports:** the headline is **Net** (income minus expenses, refunds reducing expenses), not "savings". Reports default to the calendar year, with an Indian FY (1 Apr–31 Mar) preset.
- **Budgets:** four states (ok / approaching / at_limit / exceeded) from exact integer comparisons; exactly 100% is at_limit; past months are locked; the current and future months are editable.
- **AI is out of the first build.** It stays in the roadmap as a later, confirm-only phase.
- **Removed server-era items:** email sign-up and verification, password reset, server sessions and login, hosting, server backups and restore drills, per-user server isolation tests, the HTTP API, response-time targets against a server. PO-Q1, PO-Q2, PO-Q4 are resolved (§14).
- **Kept:** integer paise with a currency code (no floats), Asia/Kolkata calendar dates, change history, double-save protection, duplicate warnings.
- **Success gates:** tests passing ends engineering for a phase; a 2-week dogfood gates "safe for daily use"; "correctness defect" is now defined (§11, `ROADMAP.md`). Testing is remote-only: CI on every PR, on-demand EAS APK builds (Expo's free tier allows 15 Android builds a month).
- **Earlier DEV/analyst challenges accepted** (savings→Net, past-budget locking, exact 100% rule, erasure wording, no lockout) are folded in throughout.

### v0.1 — 2026-11-08 (IST) — PM proposal (web app plus server). Superseded by v0.2.

---

## 1. Problem

People in India pay through a mix of UPI, cards, cash and net-banking. Money leaves in many small, fast transactions (₹40 chai, ₹350 lunch, ₹1,299 subscription), so by month-end most people cannot answer three simple questions:

1. **Where did my money go?** (by category, by month)
2. **Am I within what I planned to spend?** (budgets)
3. **What is my net for the month?** (income minus spending). This is net cash flow, not "savings": it does not know about opening balances, money moved to a fixed deposit, or cash in hand.

Existing options fail in predictable ways:
- **Spreadsheets** are accurate but recording takes too much effort, so people stop after 2–3 weeks.
- **Bank/UPI app histories** are split across apps, poorly categorised, miss cash, and can't hold budgets.
- **SMS-scraping apps** are convenient but ask for invasive permissions and get categories wrong, so users stop trusting them.
- **Cloud expense apps** need an account on someone's server that holds your whole financial life.

The main reason people quit is **effort of capture**, followed by **lack of trust in the numbers**. The product has to make recording nearly effortless **without giving up correctness or privacy**.

## 2. Vision

> A private, trustworthy ledger of your personal money that lives on your phone, takes seconds to update, works with no signal, and answers "where did my money go?" instantly. If you want a second phone or a backup, it syncs through a private GitHub repository you own, encrypted so that even GitHub cannot read it.

Product principles (in priority order when they conflict):
1. **Correct over convenient.** Totals must always reconcile to the paisa. Nothing is ever silently overwritten.
2. **Private by default.** Your data lives on your phone. We run no server and never hold your data or your keys. Anything that leaves the phone is encrypted first and goes only to places you chose.
3. **Fast capture.** Logging a typical expense takes ≤ 10 seconds, with no network wait.
4. **Works without anything else.** One phone and no GitHub account is a complete app, not a degraded mode.
5. **Small and finished beats big and half-built.** AI is a later, optional add-on.

## 3. Target users & personas

**Primary (MVP): the individual tracker.** One person tracking their own money on their own Android phone. Anyone can install the app; each install holds an isolated ledger. [DECISION DEC-PO-3]

| Persona | Description | Needs | Not needed in MVP |
|---|---|---|---|
| **P1 "Salaried Saver" (primary; Sri Harsha's own use case)** | 25–40, salaried, urban India, pays mostly via UPI/cards, some cash. Comfortable typing quickly. | Fast entry, monthly category breakdown, monthly budget, see Net | Investments, tax, shared use |
| **P2 "Budget-Conscious Planner"** | Wants hard category limits (food, shopping) and to be told when close. | Category budgets, utilisation, threshold warnings | Forecasting, goals |
| **P3 "Two-phone user"** | Has a phone and a spare/new phone, or is replacing a phone. | Optional encrypted sync/backup via their own GitHub, restore on a new phone | A server account |
| **P4 "Household Manager" (future)** | Tracks shared family/flat expenses. | Shared ledgers, splits | Deferred (see exclusions) |

Out of target: businesses/freelancer bookkeeping, GST/invoicing, investment portfolio tracking. **iPhone users are not served in the first release** (Android only); the same project can produce an iOS build later.

## 4. MVP scope (the first usable version)

The MVP is a **native Android, local-first, INR-first personal ledger**:

| # | Capability | Why it's in |
|---|---|---|
| M1 | **Private ledger on the phone:** first-run setup creates the ledger, a passphrase and a one-time recovery code; the ledger is encrypted on the device; each install is isolated from every other | Privacy and ownership with no server; prerequisite for everything |
| M2 | **Transactions:** add/edit/delete income, expense and refund; amount, type, date, category, payment method, optional note; "repeat" a past entry | Core value |
| M3 | **Categories:** sensible default set (India-relevant) + create/rename/archive custom categories | Needed for "where did money go" |
| M4 | **Transaction list:** search by note, filter by date range/category/type/payment method, sort | Find and correct entries |
| M5 | **Dashboard (monthly):** income, expenses, refunds, **Net**, category breakdown, comparison to previous month | Answers the core questions |
| M6 | **Budgets:** one overall monthly budget + per-category monthly budgets; utilisation; four states (ok / approaching / at limit / exceeded); past months locked | Core planning value; low cost |
| M7 | **Reports:** month view and year view (calendar year by default; Indian FY preset) | Monthly/yearly reporting in vision |
| M8 | **CSV export** (UTF-8 with BOM) of all or filtered transactions, with no GitHub needed | Data ownership; a complete backup path on one phone |
| M9 | **Change history:** every create/edit/delete/restore of a transaction is recorded on the device; deletes are restorable for 30 days | Financial auditability |
| M10 | **Optional GitHub sync and backup:** encrypted, to the user's own private repo; pull → merge → push; conflict screen; tombstones; restore onto a new phone | Second phone and off-device backup, with nothing of ours in the middle |
| M11 | **Erasure controls:** 30-day restore; confirmed "forget history" (squash + new key); delete repo to wipe the account | Honest deletion on top of Git |
| M12 | **Warnings that protect the numbers:** possible-duplicate warning; card-bill/EMI double-count warning (never blocking) | Trust in totals |

**AI is not in the first build** [DECISION DEC-PO-12]. The module boundary is documented (ADR-005) and nothing is implemented. See §10.

## 5. Explicit MVP exclusions (with reasons)

| Excluded | Reason | Earliest |
|---|---|---|
| **A server of ours, user accounts, email sign-up/verification, password reset** | The product has no server; the ledger is on the phone [DEC-PO-2] | Not planned |
| **iPhone / iOS build** | Android first [DEC-PO-7]; same project can produce iOS later | Later |
| **Web app / desktop** | Phone is the product | Not planned |
| **More than one person's data in one ledger (household, shared, splits)** | Different product: sharing, invitations, conflict policy between people | Future |
| **Bank statement / UPI SMS / account-aggregator import** | High privacy risk (SMS permissions, credentials), messy parsing, dedupe complexity [PO-Q3: stays out] | Future |
| **CSV/Excel import** | Valuable, but brings column-mapping, dedupe and validation work; export comes first | P1 |
| **Receipt OCR** | Low accuracy on Indian thermal receipts, cost, image storage of PII | P2 |
| **Recurring transactions** | Scheduler semantics add complexity. The cheap "repeat as a new entry dated today" covers rent/SIP/subscriptions until then | P1 |
| **Multi-account / wallet balances and transfers** | Needs opening balances, transfers and reconciliation. Until then, card bills and EMIs can double-count, so the app warns (FR-TXN-8) | P1 |
| **Multi-currency** | INR-only in MVP, **but every amount stores a currency code** so this can be added without changing meaning | P2 |
| **Push/email notifications** | In-app budget states are enough to prove value | P1 |
| **AI of any kind** (drafts, questions, edits) | Deferred [DEC-PO-12]; see §10 | Later phase, only on PO go-ahead |
| **Financial goals / savings goals** | Nice to have; Net is shown instead | P2 |
| **Passphrase reset / "forgot passphrase"** | Would require us to hold a key or run a server [DEC-PO-9] | Never, by design |
| **PDF reports** | CSV export covers ownership; PDF is polish | P2 |
| **Real-time multi-device sync, background-guaranteed sync** | Sync runs when the app is open (and best-effort in background); GitHub is a backup store, not a sync service with a contract | Not planned |

## 6. Critical user journeys (CUJs)

**CUJ-1 First run (≤ 2 min):** Install the app → choose a passphrase → see and store the recovery code (with the plain-language warning) → land on an empty dashboard with default categories in place → "Add your first expense" → add one → dashboard updates. No account, no network needed.

**CUJ-2 Log an expense manually (≤ 10 s, offline):** Tap "Add" → amount (focused), category, date (defaults to today in the user's timezone), payment method (defaults to last used) → Save → appears in list and totals immediately.

**CUJ-3 Repeat a past entry:** Open an old transaction (e.g. rent) → "Repeat" → a new entry with the same amount/category/payment method/note, dated today → adjust if needed → Save.

**CUJ-4 Month review:** Open dashboard → see income, expenses, refunds and **Net** for the current month, category breakdown, change vs. last month → tap a category → filtered transaction list.

**CUJ-5 Set and track a budget:** Set monthly overall ₹40,000 and Food ₹8,000 → while adding expenses, see utilisation → at 80% "approaching", at exactly 100% "at limit", above 100% "exceeded".

**CUJ-6 Correct a mistake:** Find the transaction via search/filter → edit amount or category, or delete (restorable for 30 days) → totals and budgets recompute → change visible in history.

**CUJ-7 Year review & export:** Open yearly report (calendar year, or switch to FY Apr–Mar) → month-by-month totals → export CSV that opens correctly in Excel.

**CUJ-8 Turn on sync (optional):** Settings → Connect GitHub → sign in to GitHub on the device → pick/create a **private** repo (a public one is refused) → first encrypted backup is pushed. The ledger keeps working throughout.

**CUJ-9 Add a second phone / replace a phone:** Install the app on the new phone → "Restore from GitHub" → sign in to GitHub → enter the passphrase (or the recovery code) → the ledger appears.

**CUJ-10 Same entry edited on two phones:** Phone A changes a transaction's amount; phone B changed the same transaction's category before syncing. On the next sync the app shows both versions side by side; the user chooses (or combines) fields; only then is anything pushed. Entries that were created independently on each phone merge without any question.

**CUJ-11 Forget history / leave:** Settings → Privacy → "Forget old versions" (explains what happens, asks to confirm; other phones will need the passphrase again) — or "Delete my account", which deletes the user's repository and wipes the keys on this phone.

## 7. Functional requirements

IDs map to `USER_STORIES.md`. Retired server-era IDs (FR-AUTH-1/2 sign-up/login/reset) are not reused.

**FR-SET — Setup, keys and unlock**
- FR-SET-1 First run creates a ledger on the device with no account and no network. The user chooses a passphrase (≥ 12 characters, not in a common-password list [ASSUMPTION: per DEV HLD]).
- FR-SET-2 A one-time recovery code is shown at setup; the user must explicitly confirm they have stored it. If they decline, the app records that no recovery code exists and warns that a forgotten passphrase cannot be recovered.
- FR-SET-3 Setup states in plain language, before the user depends on the app: **if you lose both your passphrase and your recovery code, and no phone of yours still holds the key, your encrypted GitHub copy cannot be opened. Nobody can reset it, including us.**
- FR-SET-4 There is no passphrase reset and no "forgot passphrase" link anywhere.
- FR-SET-5 A wrong passphrase causes an increasing wait before the next try (never longer than 30 seconds). It never wipes data and never locks the user out permanently.
- FR-SET-6 The ledger on the phone is encrypted at rest. The user is not asked for the passphrase on every save; it is needed when adding a phone, restoring, or rotating keys.

**FR-ISO — Ownership and isolation**
- FR-ISO-1 Each install holds only its own user's ledger. The app has no way to read or show another person's data, and we hold no copy of anyone's data.
- FR-ISO-2 Deleting the user's data (FR-ERS-4) removes the local ledger and keys and, if sync was on, the user's repo.

**FR-TXN**
- FR-TXN-1 Create a transaction: type (income | expense | refund), amount (> 0, max 2 decimal places, upper bound ₹10,00,00,000.00 [ASSUMPTION]), currency (default INR), date (calendar date, not instant), category (required, must match type; a refund names an **expense** category), payment method (UPI, Card, Cash, Net Banking, Other; optional [ASSUMPTION]), note (optional, ≤ 500 chars). Amounts are always positive; the type carries direction.
- FR-TXN-2 Edit any field; date changes move the transaction between months/budgets.
- FR-TXN-3 Delete is restorable for 30 days. Deleted transactions are excluded from all totals. After 30 days the transaction is permanently removed from the live ledger (see FR-ERS).
- FR-TXN-4 Future-dated transactions are allowed but flagged "upcoming"; dates > 1 year ahead are rejected [ASSUMPTION].
- FR-TXN-5 Possible-duplicate warning: same type, amount, category and date as an existing transaction created on this phone within the last 5 minutes → user must confirm. Never auto-block or auto-merge. The check is per phone: two phones can each log the same ₹40 chai without a warning, because that may be two real purchases.
- FR-TXN-6 Double-save protection: a single user action never creates two records (double-tap, retry after a failed disk write, app killed mid-save).
- FR-TXN-7 **Repeat:** copy a past transaction's amount, type, category, payment method and note into a **new** entry dated today. It is not a recurring scheduler. The duplicate warning applies.
- FR-TXN-8 **Card-bill / EMI double-count warning:** when saving an entry in the EMI/Loan category, or a card payment whose note or category looks like a card-bill payment, show a dismissible note: *if you already recorded the original purchase(s), recording the card payment or the full price as well counts the money twice; transfers between your own accounts are not expenses.* Never blocks the save. Detection is a fixed list of hints, not a model.
- FR-TXN-9 **Refunds:** a refund is not income. It reduces the spending of the expense category it names, and therefore that category's budget use and the dashboard's spending, and it raises Net. A refund of a category with no spend can make that category's spend negative; it is shown as negative, never clamped.

**FR-CAT**
- FR-CAT-1 Default categories seeded on first run (expense: Food & Dining, Groceries, Transport, Fuel, Rent, Utilities & Bills, Mobile & Internet, Shopping, Health, Entertainment, Subscriptions, Travel, Education, Personal Care, Gifts & Donations, EMI/Loan, Other; income: Salary, Freelance, Interest, Gift, Other). **There is no "Refund" income category**: refunds are a transaction type (FR-TXN-9).
- FR-CAT-2 Create, rename, archive custom categories. Names are unique per type (case-insensitive, whitespace-trimmed).
- FR-CAT-3 Archiving hides the category from new entries but keeps it on history. A category with transactions cannot be hard-deleted.

**FR-LIST**
- FR-LIST-1 Paginated/virtualised list, newest first; filters: date range, type, category (multi), payment method; text search on note; sort by date/amount.
- FR-LIST-2 Filtered totals shown for the current filter.

**FR-DASH**
- FR-DASH-1 For a selected month (default current, user timezone): total income, total expenses, total refunds, and the headline **Net**, with the formula visible: **Net = income − expenses + refunds** (that is, income minus spending, where spending is expenses minus refunds). It is net cash flow, not "savings". A "Net as % of income" figure is shown (displayed to 1 decimal place; "—" when income is zero).
- FR-DASH-2 Category breakdown of spending (expenses minus refunds; amount and % share).
- FR-DASH-3 Comparison with the previous month (absolute and % change, per total and per category).
- FR-DASH-4 6-month spending trend.

**FR-BUD**
- FR-BUD-1 Set an overall monthly budget and/or per-category monthly budgets. A budget applies from the month it is set and carries forward to following months until changed. **Past months are locked**: a budget for a past month cannot be created or changed, so history never changes. The current month and future months are editable; changing a budget affects that month and later months only.
- FR-BUD-2 Spent for a budget = sum of expenses − sum of refunds in the budget's scope and month. Utilisation = spent / budget, shown as amount, %, and remaining. Income never counts.
- FR-BUD-3 Four states, decided by exact integer comparisons on paise (never from a rounded percentage): **ok** (spent × 100 < budget × 80), **approaching** (spent × 100 ≥ budget × 80 and spent < budget), **at_limit** (spent = budget exactly), **exceeded** (spent > budget; the overspend amount is shown). The 80% threshold is fixed in MVP. Displayed percentages are rounded for display only (a displayed 80.0% can still be "ok").
- FR-BUD-4 Sum of category budgets may exceed the overall budget; show an informational warning, don't block.

**FR-RPT**
- FR-RPT-1 Monthly report = dashboard for any past month.
- FR-RPT-2 Yearly report defaults to the **calendar year** (1 Jan–31 Dec), with totals per month and per category, and yearly income/expenses/refunds/Net. An **Indian financial-year preset** (1 Apr–31 Mar, labelled by its starting year, e.g. "FY 2026-27") switches the same report to that window. Both are date windows over the same ledger.

**FR-EXP**
- FR-EXP-1 Export all (or filtered) transactions as CSV: **UTF-8 with a byte-order mark (BOM)** so Excel on Windows shows ₹ and non-Latin notes correctly; ISO dates; plain decimal amounts (no ₹ symbol or grouping commas in values); a type column (income/expense/refund) and a currency column. Cells starting with `=`, `+`, `-`, `@` are neutralised so a note can't run as a formula.
- FR-EXP-2 Export works with no GitHub account and no network.

**FR-HIS — Change history**
- FR-HIS-1 Every create/edit/delete/restore of a transaction and budget change is recorded on the device: when, what changed, before/after values. There is no screen to edit or delete a history entry.
- FR-HIS-2 The user can view a transaction's change history.
- FR-HIS-3 History is kept while the transaction exists or can still be restored. When a transaction is permanently removed, its history, including amounts and notes, is removed with it (see FR-ERS-2).

**FR-SYNC — Optional GitHub sync and backup**
- FR-SYNC-1 Sync is off until the user connects their own GitHub account and a **private** repository. A public repository is refused. The app works fully without any of this.
- FR-SYNC-2 What goes to GitHub is encrypted on the phone first. GitHub (and we) see only that the account uses the app, when syncs happen, and roughly how large the data is.
- FR-SYNC-3 A save never waits on the network. Sync runs later (when the app is open, shortly after saves, on demand, and best-effort in the background) and never delays or blocks entry.
- FR-SYNC-4 **Sync order is pull, then merge, then push.** Entries created independently on different phones both survive automatically. An edit that builds on the version the phone already has is simply applied.
- FR-SYNC-5 If the **same existing entry** was changed on two phones since they last agreed, the app shows both versions and lets the user choose (field by field) before anything is pushed. Nothing is silently overwritten, and there is no "latest timestamp wins".
- FR-SYNC-6 A delete on one phone and an edit of the same entry on another is also shown to the user, not silently resolved either way.
- FR-SYNC-7 Deletes sync as tombstones, so a delete on one phone removes the entry on the others (and stays restorable for 30 days).
- FR-SYNC-8 If GitHub is unreachable, rate-limits the app, or the user has no account, the app keeps working and queues changes. It says plainly that sync is paused and why, and that the phone copy and CSV export are unaffected.
- FR-SYNC-9 A new phone can restore from the user's repo using GitHub sign-in plus the passphrase (or recovery code).
- FR-SYNC-10 The app never stores or sends GitHub tokens anywhere except the phone's secure storage and GitHub itself. We hold no tokens.

**FR-ERS — Erasure and honesty about Git**
- FR-ERS-1 Day-to-day delete: restorable for 30 days (FR-TXN-3), across synced phones.
- FR-ERS-2 After 30 days the transaction leaves the live ledger and the next sync, including its change-history rows.
- FR-ERS-3 **Forget history (confirmed):** because Git keeps old versions, the app offers "forget old versions": a confirmed rewrite of the data branch so only the current state remains, plus a new data key. It explains that other phones will need the passphrase again. It is never run silently.
- FR-ERS-4 **Delete my account:** deletes the user's GitHub repository (if sync was on) and wipes the keys and ledger on this phone, after an explicit confirmation and an offer to export CSV first.
- FR-ERS-5 The privacy text states plainly that **GitHub may keep internal copies or backups we cannot remove**, and that old encrypted versions stay on GitHub until history is forgotten or the repo is deleted.

**FR-AI** — *Not in the first build.* See §10 for the later, confirm-only scope.

## 8. Non-functional requirements

| Area | Requirement |
|---|---|
| **Money correctness** | No floating-point anywhere in the money path (storage, merge, computation, export). Amounts are stored as **integer paise with an ISO-4217 currency code** (INR). Decimal strings are used for any interchange (history, sync, CSV). Aggregates equal the sum of their line items exactly. Rounding is never applied silently; input with > 2 decimals is rejected, not rounded. |
| **Display** | Indian digit grouping (₹1,23,456.78) for INR; display formatting kept separate from stored values. |
| **Dates & timezones** | A transaction's *date* is a calendar date in the user's timezone (default Asia/Kolkata), stored as a date, not a UTC instant, so it never shifts across midnight. "Today", "this month" and budget periods are resolved in the user's timezone. System timestamps (created/updated/history) are UTC instants. |
| **Consistency** | A save and its history entry are atomic. Two edits of the same entry never silently overwrite: on one phone a stale edit is rejected; across phones it becomes a user choice (FR-SYNC-5). |
| **Double-save** | A double-tap, retry or crash during save yields exactly one record. |
| **Offline** | Every ledger feature (add, edit, delete, restore, list, dashboard, budgets, reports, CSV export) works with no network. Only sync and restore-from-GitHub need one. |
| **Performance** | On a mid-range Android phone with 10,000 transactions: dashboard < 1 s, list page < 500 ms, a save confirmed on screen < 300 ms (excluding sync). These are **targets measured on a real phone during dogfood**, not CI gates. |
| **Durability** | The phone copy is the primary copy. CSV export is always available. Optional encrypted GitHub backup, and a **restore onto a clean phone** that reproduces identical totals, is verified before v1.0 (a manual, once-per-release check). |
| **Accessibility** | Core flows usable with Android TalkBack and large font sizes; touch targets ≥ 48 dp [ASSUMPTION]. |
| **Platform** | Android only in the first release (minimum supported version to be set by DEV). iOS possible later from the same project. |
| **Observability** | Crash and error reports (if any) contain **no amounts, notes, passphrase, recovery code, tokens or ledger content**. |
| **Maintainability** | Schema changes via versioned migrations on the device; automated tests gate merges (§11). |

## 9. Security & privacy requirements

1. **Isolation by construction:** each install holds only its owner's ledger; there is no shared database to leak from. [DEC-PO-3]
2. **Encrypted at rest:** the ledger on the phone is encrypted; its key is held in the phone's secure storage, not in the passphrase. A stolen, **locked** phone does not expose the ledger. A stolen, **unlocked** phone does; the product does not pretend otherwise. An app-lock/biometric screen is a P1 candidate [ASSUMPTION].
3. **Encrypted before leaving the phone:** anything sent to GitHub is ciphertext. Commit messages and file names carry no amounts, notes or category names. Sync refuses a public repo.
4. **Keys:** the passphrase and recovery code never leave the phone and are never logged. There is no reset, and we cannot recover them. A weak passphrase is the main offline risk, hence the length and common-password checks. Wrong attempts back off; they do not wipe or lock forever.
5. **Tokens:** GitHub tokens live only in the phone's secure storage and are limited to the one chosen repo's contents. We never see them.
6. **GitHub hygiene (development):** no real financial data, exports or DB dumps in the source repo; test fixtures are synthetic only; no secrets (Expo token, signing keys) in the repo.
7. **What GitHub can learn:** that this GitHub account uses the app, when syncs happen, and approximate data size. Stated in the privacy notice.
8. **User rights:** export all data (CSV) any time; delete the account and data (FR-ERS-4); a plain-language privacy notice that includes the GitHub-retention caveat (FR-ERS-5). Designed in the spirit of India's DPDP Act 2023 (consent, purpose limitation, erasure) [ASSUMPTION: not legal advice; review before any public launch].
9. **Input handling:** all inputs validated identically wherever they originate; notes and category names are rendered as plain text; CSV cells are formula-neutralised.
10. **Honest limits:** GitHub is a place to put encrypted files, not a sync service with a contract. If GitHub suspends the app's registration or the user's account, sync stops; the phone copy and CSV export keep working.

## 10. AI — deferred (not in the first build) [DECISION DEC-PO-12]

AI is **out of the first build**. Nothing is implemented, no provider key is needed, and no AI wording appears in the app. It stays in the roadmap as a later, optional phase, and starts only when the Product Owner explicitly asks for it. When it does, these rules apply:

- **Confirm-only:** AI may only *propose a draft* the user reviews, edits and confirms. Nothing is saved without that confirmation. Read-only questions are answered from figures the app computed, with a link to verify.
- **Data minimisation and consent:** off by default; before first use the app discloses that the **typed sentence** is sent to the provider (the stored ledger is not), and the user must accept. Provider and cost ceiling are decided then [PO-Q5, deferred].
- **No side door:** the AI path has no way to write; confirm uses the same save as the manual form (same validation, duplicate warning and double-save protection). Notes are treated as data, never instructions.
- **Quality gate to switch it on:** labelled set of ≥ 200 India-style phrases (e.g. "350 swiggy", "2k petrol", "1.5L", "350/-"): amount ≥ 98%, type ≥ 98%, category ≥ 85%, date ≥ 95%; every number in an answer matches a computed figure; 0 writes without confirmation. **Missing the bar means AI stays off; it never blocks or rolls back the ledger.**
- The phone would call the provider directly; there is still no server of ours.

## 11. Success metrics and "done" definitions

Personal-use product. Metrics are measured for the Product Owner first and stay valid if more users install it.

**Definitions**
- **Correctness defect:** any wrong total (a figure that does not equal the exact sum of its line items), any wrong date or month placement, any lost entry, any duplicated entry (including after sync, restore or a crash), or any silent overwrite of an edit.
- **Tests passing ends engineering for a phase.** A phase's engineering work is done when its automated tests pass in CI. The dogfood is not an engineering gate.
- **2-week dogfood gates "safe for daily use" (v1.0).** It does not block development of later phases.

| Metric | Target |
|---|---|
| **Activation:** first transaction within 5 min of install | ≥ 90% |
| **Capture speed:** median time to log an expense (manual) | ≤ 10 s |
| **Habit:** days with ≥ 1 entry, per week, over weeks 2–8 | ≥ 4 of 7 |
| **Retention:** still logging at week 8 | Product Owner yes; ≥ 40% if broader |
| **Trust:** correctness defects (defined above), in the dogfood and in tests | **0** (release blocker) |
| **Sync safety:** silent overwrites, lost or duplicated entries after sync/restore | **0** |
| **Recoverability:** a restore onto a clean phone reproduces identical totals | Pass (before v1.0) |
| **Budget engagement:** months with ≥ 1 budget set | ≥ 70% of active months |

## 12. Risks

| Risk | Impact | Mitigation |
|---|---|---|
| Capture fatigue → user quits | High | ≤ 10 s entry, smart defaults, repeat-as-new-entry |
| Money/date bugs erode trust | High | Integer paise, date-as-calendar-date, reconciliation and boundary tests, property tests |
| **Lost passphrase and recovery code** | High (GitHub copy unrecoverable) | Plain-language warning at setup, explicit "I have stored this", recovery code, CSV export always available |
| **GitHub dependency** (account, rate limits, app suspension, history that never forgets) | Medium | GitHub is optional; one upload per sync; phone copy + CSV always work; honest privacy text; forget-history and delete-repo actions |
| Sync merges wrongly or silently drops an edit | High | Never silently overwrite; user chooses on same-entry edits; merge logic covered by tests with no network |
| Card-bill/EMI double counting inflates spending | Medium | Non-blocking warning; accounts/transfers in P1 |
| Stolen unlocked phone | Medium | Stated honestly; app lock is a P1 candidate |
| Android-only excludes iPhone users | Medium | Accepted for v1; same React Native/Expo project can add iOS |
| Free-tier build limits (15 Android builds/month, low-priority queue) | Low | Builds on demand only; tests on every PR; see `ROADMAP.md` |
| Scope creep (imports, households, OCR, AI) | Medium | Exclusions table; changes go through PM–DEV protocol and PO approval |

## 13. Assumptions

Resolved ones carry the decision; the rest are reversible defaults.

- A1 **Resolved (DEC-PO-3):** one person's ledger per install; anyone can install; no sharing or household.
- A2 **Resolved (DEC-PO-1, DEC-PO-7):** native Android app (React Native/Expo), iOS later; no web app.
- A3 Manual entry only in the first build; no imports (PO-Q3 stays out); AI deferred (DEC-PO-12).
- A4 INR only in MVP; currency code stored with every amount.
- A5 Default timezone Asia/Kolkata, user-changeable; changing it never moves existing transaction dates.
- A6 **Replaced:** no email/password accounts. Replaced by passphrase + one-time recovery code, no reset (DEC-PO-9).
- A7 **Resolved (DEC-PO-14):** reports default to the calendar year; an Indian FY (Apr–Mar) preset is included.
- A8 **Resolved (DEC-PO-15):** budget threshold fixed at 80%; budgets carry forward; four states; past months locked; current and future months editable.
- A9 Payment method is a fixed list, optional.
- A10 **Resolved (DEC-PO-10):** deletes are restorable for 30 days; forgetting history = confirmed squash + new data key; deleting the repo wipes the account; GitHub may retain copies.
- A11 **Deferred (DEC-PO-12):** AI is out of the first build; if built later, it only drafts, runs from the phone, and is off by default.
- A12 English UI; Indian money shorthand only matters if AI is built.
- A13 **Resolved (DEC-PO-8):** GitHub is optional; one phone plus CSV is a complete app.
- A14 **Resolved (DEC-PO-11):** refunds reduce their expense category; non-blocking card/EMI warning; CSV UTF-8 with BOM; repeat = new entry dated today; backoff, not lockout.
- A15 **Resolved (DEC-PO-13):** the dashboard headline is "Net".
- A16 **Resolved (DEC-PO-16):** development and testing are remote only (cloud CI; installable APKs from cloud builds); the Product Owner installs no Android toolchain.

## 14. Open and resolved questions

| ID | Question | Status |
|---|---|---|
| PO-Q1 | Single user or household? | **Resolved:** anyone can install and keeps an isolated ledger; no household/shared ledger (DEC-PO-3). |
| PO-Q2 | Web or mobile? | **Resolved:** native Android first, React Native/Expo; iOS later (DEC-PO-1, DEC-PO-7). |
| PO-Q3 | Bank / UPI SMS / statement import? | **Out of MVP** (stays out unless reopened). |
| PO-Q4 | Hosting and backups? | **Resolved / dissolved:** no server. Backup = CSV export plus optional encrypted GitHub sync (DEC-PO-2, DEC-PO-4, DEC-PO-8). |
| PO-Q5 | AI provider, cost, privacy? | **Deferred** with AI; decide only when AI is scheduled (DEC-PO-12). |
| PO-Q6 | Reporting year? | **Resolved:** calendar year default; Indian FY preset (DEC-PO-14). |
| TQ-5 | "Savings" or "Net"? | **Resolved:** "Net" (DEC-PO-13). |
| TQ-6 | Past budgets editable? | **Resolved:** past months locked (DEC-PO-15). |
| TQ-7 | Audit vs deletion | **Resolved:** purge removes history with the entry; forget-history for Git (DEC-PO-10). |
| TQ-8 | Exactly 100% of a budget | **Resolved:** `at_limit`. |
| **[OPEN]** | Minimum Android version; whether an app-lock is needed at v1.0; the exact wording of the privacy notice (review before any public release) | Open, low risk; DEV/PO to confirm in Phase 3. |
| **[OPEN]** | ADR-014 (one encrypted snapshot per sync) is **Proposed**; it is the mechanism behind FR-SYNC and should be approved with Phase 3 | DEV/PO |
