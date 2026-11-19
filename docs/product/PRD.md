# Product Requirements Document — Personal Expense Tracker

Status: **DRAFT v0.1 — [PM] proposal, pending [DEV] evaluation and Product Owner approval**
Owner: PM agent · Product Owner: Sri Harsha · Last updated: 2026-11-08 (IST)

> Labels used: **[ASSUMPTION]** = PM default made to keep moving (reversible); **[PO-Q#]** = depends on an open Product Owner question in `PM_PROPOSAL.md`.

---

## 1. Problem

People in India pay through a mix of UPI, cards, cash and net-banking. Money leaves in many small, fast transactions (₹40 chai, ₹350 lunch, ₹1,299 subscription), so by month-end most people cannot answer three simple questions:

1. **Where did my money go?** (by category, by month)
2. **Am I within what I planned to spend?** (budgets)
3. **Am I actually saving anything?** (income − expenses)

Existing options fail in predictable ways:
- **Spreadsheets** are accurate but recording takes too much effort, so people stop after 2–3 weeks.
- **Bank/UPI app histories** are split across apps, poorly categorised, miss cash, and can't hold budgets.
- **SMS-scraping apps** are convenient but ask for invasive permissions and get categories wrong, so users stop trusting them.

The main reason people quit is **effort of capture**, followed by **lack of trust in the numbers**. The product has to make recording nearly effortless **without giving up correctness or privacy**.

## 2. Vision

> A private, trustworthy ledger of your personal money that takes seconds to update and answers "where did my money go?" instantly. You can update it in plain language, but nothing ever changes without your confirmation.

Product principles (in priority order when they conflict):
1. **Correct over convenient.** Totals must always reconcile to the paisa.
2. **Private by default.** Financial data belongs to the user and never leaves the app's private database except to services the user knowingly enables.
3. **Fast capture.** Logging a typical expense takes ≤ 10 seconds.
4. **AI assists, the user decides.** AI proposes; the backend validates; the user confirms.
5. **Small and finished beats big and half-built.**

## 3. Target users & personas

**Primary (MVP): the individual tracker.** One person tracking their own money. [ASSUMPTION; see PO-Q1]

| Persona | Description | Needs | Not needed in MVP |
|---|---|---|---|
| **P1 "Salaried Saver" (primary; Sri Harsha's own use case)** | 25–40, salaried, urban India, pays mostly via UPI/cards, some cash. Comfortable typing quickly. | Fast entry, monthly category breakdown, monthly budget, see savings | Investments, tax, multi-user |
| **P2 "Budget-Conscious Planner"** | Wants hard category limits (food, shopping) and to be told when close. | Category budgets, utilisation, threshold alerts | Forecasting, goals |
| **P3 "Household Manager" (future)** | Tracks shared family/flat expenses. | Shared ledgers, splits | Deferred (see exclusions) |

Out of target: businesses/freelancer bookkeeping, GST/invoicing, investment portfolio tracking.

## 4. MVP scope (the first usable version)

The MVP is a **single-user, web-based, INR-first ledger** with the following:

| # | Capability | Why it's in |
|---|---|---|
| M1 | **Account & auth**: sign up, log in, log out, password reset; one user's data fully isolated | Prerequisite; privacy |
| M2 | **Transactions**: add/edit/delete income & expense; amount, type, date, category, payment method, optional note | Core value |
| M3 | **Categories**: sensible default set (India-relevant) + create/rename/archive custom categories | Needed for "where did money go" |
| M4 | **Transaction list**: search by note, filter by date range/category/type/payment method, sort | Find and correct entries |
| M5 | **Dashboard (monthly)**: income, expenses, net savings, category breakdown, comparison to previous month | Answers the 3 core questions |
| M6 | **Budgets**: one overall monthly budget + per-category monthly budgets; utilisation %; in-app warning at 80% and 100% | Core planning value; low cost |
| M7 | **Reports**: month view and year view (12-month totals by month and by category) | Monthly/yearly reporting in vision |
| M8 | **Data export (CSV)** of all the user's transactions | Data ownership and trust; cheap; acts as a user-side backup |
| M9 | **Audit trail**: every create/edit/delete of a transaction is recorded (who, when, before/after); deletes are soft deletes | Financial auditability |
| M10 | **AI assistant (narrow)**: (a) natural-language *entry* that produces a **draft** the user must confirm; (b) natural-language *read-only questions* answered from backend-computed figures | The brief's headline differentiator; kept narrow and safe |

**About M10:** AI ships in MVP only as an add-on. Every flow in the MVP must work fully without AI. If the AI provider is down or turned off, the product still works. Per the brief, AI is built in Phase 5 after the core ledger is proven. It counts toward "MVP release" but not toward "core usable".

## 5. Explicit MVP exclusions (with reasons)

| Excluded | Reason | Earliest |
|---|---|---|
| **Bank statement / UPI SMS / account-aggregator import** | High privacy risk (SMS permissions, credentials), messy parsing, dedupe complexity; needs regulatory thinking (RBI AA framework) | Future [PO-Q3] |
| **CSV/Excel import** | Valuable, but brings column-mapping, dedupe and validation work; export comes first | P1 |
| **Receipt OCR** | Low accuracy on Indian thermal receipts, cost, image storage of PII | P2 |
| **Recurring transactions** | Scheduler and "materialise vs. project" semantics add complexity; manual entry is fine at first | P1 |
| **Multi-account / wallet balances** (bank A, card B, cash) | Needs opening balances, transfers and reconciliation; payment method gives about 80% of the insight | P1 |
| **Multi-currency** | INR-only in MVP, **but the data model must store a currency code per amount** so this can be added without a migration of meaning | P2 |
| **Shared/household expenses, splits** | Multi-tenant permissions, invitations, conflict resolution; a different product | Future [PO-Q1] |
| **Email/push notifications** | In-app budget warnings are enough to prove value; delivery infrastructure is extra work | P1 |
| **AI budget recommendations, anomaly detection, conversational multi-turn reports** | Unclear value without months of data; risk of wrong financial "advice" | P2 |
| **AI editing/deleting via chat** | Highest risk of silent alteration; MVP AI can only *create drafts* and *read* | P1 (with confirmation) |
| **Financial goals / savings goals** | Nice to have; savings is shown as income − expense in MVP | P2 |
| **Native mobile apps** | Responsive web first [PO-Q2] | Future |
| **Offline mode** | Sync conflict complexity | Future |
| **PDF reports** | CSV export covers ownership; PDF is polish | P2 |

## 6. Critical user journeys (CUJs)

**CUJ-1 First run (≤ 2 min):** Sign up → land on empty dashboard with default categories already in place → prompt "Add your first expense" → add one → dashboard updates.

**CUJ-2 Log an expense manually (≤ 10 s):** Tap "Add" → amount (focused), category, date (defaults to today in the user's timezone), payment method (defaults to last used) → Save → appears in list and totals immediately.

**CUJ-3 Log via natural language (≤ 10 s):** Type "Spent 350 on lunch today via UPI" → system shows a **draft card** (₹350.00 · Expense · Food & Dining · 8 Nov 2026 · UPI) with fields editable → user taps Confirm → saved. Tap Cancel → nothing saved.

**CUJ-4 Month review:** Open dashboard → see income, expenses, savings for the current month, category breakdown, change vs. last month → tap a category → filtered transaction list.

**CUJ-5 Set and track a budget:** Set monthly overall ₹40,000 and Food ₹8,000 → while adding expenses, see utilisation → at 80%, an in-app warning appears; at 100%+, an "exceeded" state.

**CUJ-6 Correct a mistake:** Find the transaction via search/filter → edit amount or category, or delete → totals and budgets recompute → change visible in history (audit).

**CUJ-7 Ask a question:** "How much did I spend on food this month?" → answer with the figure, the period and category it used, and a link to the matching filtered list so the user can verify it.

**CUJ-8 Year review & export:** Open yearly report → month-by-month totals → export CSV.

## 7. Functional requirements

IDs map to `USER_STORIES.md`.

**FR-AUTH**
- FR-AUTH-1 Email + password sign-up with email verification. [ASSUMPTION; social login P1]
- FR-AUTH-2 Login, logout, session expiry, password reset via email.
- FR-AUTH-3 Every data read/write is scoped to the authenticated user (server-enforced).
- FR-AUTH-4 User can delete their account and all data (hard delete after a grace period, see NFR).

**FR-TXN**
- FR-TXN-1 Create a transaction: type (income|expense), amount (> 0, max 2 decimal places, upper bound ₹10,00,00,000.00 [ASSUMPTION]), currency (default INR), date (calendar date, not instant), category (required, must match type), payment method (UPI, Card, Cash, Net Banking, Other; optional [ASSUMPTION]), note (optional, ≤ 500 chars).
- FR-TXN-2 Edit any field; date changes move the transaction between months/budgets.
- FR-TXN-3 Delete = soft delete. Excluded from all totals. Restorable for 30 days [ASSUMPTION].
- FR-TXN-4 Future-dated transactions are allowed but flagged; dates > 1 year ahead are rejected [ASSUMPTION].
- FR-TXN-5 Possible-duplicate warning: same type, amount, category and date as an existing transaction created within the last 5 minutes → user must confirm. Never auto-block or auto-merge.
- FR-TXN-6 Double-submit protection: a single user action never creates two records (e.g. double-click, retry on flaky network).

**FR-CAT**
- FR-CAT-1 Default categories seeded per user (expense: Food & Dining, Groceries, Transport, Fuel, Rent, Utilities & Bills, Mobile & Internet, Shopping, Health, Entertainment, Subscriptions, Travel, Education, Personal Care, Gifts & Donations, EMI/Loan, Other; income: Salary, Freelance, Interest, Refund, Gift, Other).
- FR-CAT-2 Create, rename, archive custom categories. Names are unique per user and type (case-insensitive).
- FR-CAT-3 Archiving hides the category from new entries but keeps it on history. A category with transactions cannot be hard-deleted.

**FR-LIST**
- FR-LIST-1 Paginated list, newest first; filters: date range, type, category (multi), payment method; text search on note; sort by date/amount.
- FR-LIST-2 Filtered totals shown for the current filter.

**FR-DASH**
- FR-DASH-1 For a selected month (default current, user timezone): total income, total expenses, net savings (income − expenses), savings rate %.
- FR-DASH-2 Category breakdown of expenses (amount and % share).
- FR-DASH-3 Comparison with the previous month (absolute and % change, per total and per category).
- FR-DASH-4 6-month spending trend.

**FR-BUD**
- FR-BUD-1 Set an overall monthly budget and/or per-category monthly budgets. A budget carries forward to following months until changed [ASSUMPTION]; changing it affects the current and future months only.
- FR-BUD-2 Utilisation = spent / budget for that month, shown as amount, %, and remaining.
- FR-BUD-3 In-app states: OK (< 80%), Approaching (≥ 80%), Exceeded (> 100%). Threshold fixed at 80% in MVP.
- FR-BUD-4 Sum of category budgets may exceed the overall budget; show an informational warning, don't block.

**FR-RPT**
- FR-RPT-1 Monthly report = dashboard for any past month.
- FR-RPT-2 Yearly report: calendar year [ASSUMPTION; Indian FY Apr–Mar is P1, see PO-Q6], with totals per month and per category, and yearly income/expense/savings.

**FR-EXP**
- FR-EXP-1 Export all (or filtered) transactions as CSV with UTF-8 encoding, ISO dates, plain decimal amounts (no ₹ symbol or grouping commas in values) and a currency column.

**FR-AUD**
- FR-AUD-1 Immutable audit log for transaction and budget create/update/delete/restore: actor, timestamp (UTC), source (manual | ai_assistant), before/after values.
- FR-AUD-2 User can view a transaction's change history.

**FR-AI** (see §9)
- FR-AI-1 NL entry → structured **draft** → user confirms/edits/cancels → backend validates → save. Single transaction per message in MVP.
- FR-AI-2 NL read-only queries over the user's own data: totals by category/period, comparisons between two periods, budget status.
- FR-AI-3 AI can be disabled per user. The full app works with AI disabled or unavailable.

## 8. Non-functional requirements

| Area | Requirement |
|---|---|
| **Money correctness** | No floating-point anywhere in the money path (storage, API, computation, AI tool outputs). Amounts stored as exact decimal (or integer minor units) **with ISO-4217 currency code**. Aggregates must equal the sum of their line items exactly. Rounding is never applied silently; input with > 2 decimals is rejected, not rounded. |
| **Display** | Indian digit grouping (₹1,23,456.78) for INR; locale-aware formatting kept separate from stored values. |
| **Dates & timezones** | A transaction's *date* is a calendar date in the user's timezone (default Asia/Kolkata), stored as a date, not a UTC timestamp, so it never shifts across midnight. "Today", "this month" and budget periods are resolved in the user's timezone. System timestamps (created/updated/audit) are UTC instants. |
| **Consistency** | Transaction write + audit entry are atomic. Concurrent edits use optimistic concurrency (stale edit → conflict error, not silent overwrite). |
| **Idempotency** | Create endpoints accept an idempotency key. Retried requests do not duplicate. |
| **Performance** | p95 dashboard load < 1 s and list page < 500 ms at 10k transactions per user. Add-transaction round trip < 300 ms p95 (excluding AI). AI draft < 4 s p95. |
| **Availability** | Personal-use target 99.5% monthly [ASSUMPTION]; AI outage must not degrade non-AI flows. |
| **Durability** | Automated daily backups, ≥ 7-day retention, restore tested once per phase-6 release (RPO ≤ 24 h, RTO ≤ 4 h) [ASSUMPTION; depends on PO-Q4]. |
| **Accessibility** | WCAG 2.1 AA for core flows; full keyboard operation of add-transaction. |
| **Responsive** | Usable on 360 px-wide mobile browsers and desktop. |
| **Observability** | Structured logs with request IDs; **no amounts, notes or PII in logs**; error tracking; basic metrics (latency, error rate, AI call count/cost). |
| **Maintainability** | Schema changes via migrations; documented API contract; automated tests gate merges. |

## 9. Security & privacy requirements

1. **Data isolation:** every query is scoped to the authenticated user server-side. Requests for another user's resource ID return 404 (don't reveal that it exists). This is covered by automated tests.
2. **Auth:** passwords hashed with a modern adaptive algorithm; login rate limiting and lockout/backoff; secure, HttpOnly, SameSite session cookies or equivalent short-lived tokens; CSRF protection where applicable.
3. **Transport & storage:** TLS everywhere; database and backups encrypted at rest.
4. **Secrets:** never in the repo; `.env` git-ignored; secrets via environment/secret store; the AI API key lives only on the backend.
5. **GitHub hygiene:** no real financial data, exports or DB dumps in the repo; test fixtures are synthetic only.
6. **AI data minimisation:** send the provider only what is needed. For NL entry, that is the user's message plus the category names and today's date. For queries, the model gets the question plus tool *results* (aggregates), not raw ledger dumps. Notes are not sent unless needed. Prefer a provider and tier that doesn't train on our data and allows a retention opt-out [PO-Q5]. Show a one-time disclosure before first AI use.
7. **Prompt-injection resistance:** text from transaction notes or user content is treated as data. The AI's tools are read-only, plus a single "propose draft" tool that writes nothing. The backend re-validates everything.
8. **User rights:** export all data (CSV), delete account and data (hard purge within 30 days, including removal from backups as they age out), privacy notice. Design in the spirit of India's DPDP Act 2023 (consent, purpose limitation, erasure) [ASSUMPTION: not legal advice; review before any public launch].
9. **Input validation:** all inputs validated server-side regardless of source (UI or AI); output encoding to prevent XSS in notes/category names.
10. **Audit log** is append-only and not editable by the user.

## 10. How AI should — and should not — help

**Should (MVP):**
- **Cut capture effort:** "spent 350 on lunch today via UPI" → a draft with amount, type, category, date, payment method, note. Relative dates ("yesterday", "last Friday") are resolved in the user's timezone. Ambiguity is surfaced, not guessed (e.g. "350 or 3500?" asks; a missing amount → the draft can't be confirmed until filled).
- **Answer read-only questions** with numbers computed by the backend's deterministic queries (tool/function calling). The LLM only phrases the answer and never does the arithmetic. Each answer shows the period/filters used and a "view transactions" link.
- Examples covered: "How much did I spend on food this month?", "Compare my spending in October and November", "Am I exceeding my monthly shopping budget?"

**Should not:**
- Create, edit or delete any transaction without an explicit user confirmation of a displayed draft.
- Do arithmetic for totals, or state figures that weren't returned by a backend tool.
- Give investment, tax or credit advice.
- Edit/delete via chat in MVP, batch-create multiple transactions per message, or remember free-text conversation history beyond the session [ASSUMPTION].
- Be required: every AI feature has a manual equivalent.

**Quality bar to ship AI (gate, Phase 5):** on a labelled test set of ≥ 200 India-style phrases (including Indian shorthand such as "350 swiggy", "2k petrol", "1.5L", "350/-"), amount extraction ≥ 98% exact, type ≥ 98%, category top-1 ≥ 85%, date ≥ 95%; 100% of query answers match the backend tool result; 0 writes without confirmation in tests.

## 11. Success metrics

Personal-use product. Metrics are measured for the Product Owner first and stay valid if more users are added later.

| Metric | Target |
|---|---|
| **Activation:** first transaction within 5 min of sign-up | ≥ 90% |
| **Capture speed:** median time to log an expense (manual / AI) | ≤ 10 s / ≤ 8 s |
| **Habit:** days with ≥ 1 entry, per week, over weeks 2–8 | ≥ 4 of 7 |
| **Retention:** still logging at week 8 | Product Owner yes; ≥ 40% if broader |
| **Trust:** reconciliation defects (totals ≠ sum of line items) | **0** (release blocker) |
| **Correctness:** data-isolation or money-precision bugs in prod | **0** |
| **AI usefulness:** AI drafts confirmed without edits | ≥ 80% |
| **AI safety:** writes without confirmation | **0** |
| **Budget engagement:** months with ≥ 1 budget set | ≥ 70% of active months |

## 12. Risks

| Risk | Impact | Mitigation |
|---|---|---|
| Capture fatigue → user quits | High | ≤ 10 s entry, smart defaults, NL entry |
| Money/date bugs erode trust | High | Decimal-only, date-as-calendar-date, reconciliation tests, property-based tests |
| AI mis-parses amounts ("2k", "1.5 lakh", "350/-") | High | Confirm-before-save, explicit Indian-number test set, never auto-save |
| AI cost/latency | Medium | Small model for parsing; per-user rate limit; AI off-switch; cost metric [PO-Q5] |
| Privacy breach / cross-user leak | Critical | Server-side scoping, isolation tests, minimal data to AI, no PII in logs |
| Prompt injection via notes | Medium | Read-only tools, no write tools, backend validation |
| Scope creep (imports, households, OCR) | Medium | Exclusions table; changes go through PM–DEV protocol and PO approval |
| Hosting/backup neglect for a personal app | Medium | Backup + restore test is a Phase 6 exit criterion |

## 13. Assumptions (all reversible; confirm or override)

- A1 Single user per account; no sharing (PO-Q1).
- A2 Responsive web app first (PO-Q2).
- A3 Manual + AI entry only; no imports in MVP (PO-Q3).
- A4 INR only in MVP; currency stored per amount for later.
- A5 Default timezone Asia/Kolkata, user-changeable.
- A6 Email/password auth with verification.
- A7 Calendar-year yearly report; Indian FY later (PO-Q6).
- A8 Budget threshold fixed at 80%; budgets carry forward monthly.
- A9 Payment method is a fixed list, optional.
- A10 Soft-delete with 30-day restore window.
- A11 AI provider is backend-only and swappable; Grok API is the candidate (DEV to evaluate).
- A12 English (with tolerance for common Hinglish money phrases) only.
