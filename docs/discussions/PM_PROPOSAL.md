# [PM] Initial Product Proposal — Personal Expense Tracker

Date: 2026-11-08 (IST) · Phase 1 (Product Discovery) · For [DEV] evaluation and Product Owner review.
Full detail: `docs/product/PRD.md`, `USER_STORIES.md`, `ACCEPTANCE_CRITERIA.md`, `ROADMAP.md`.

[PM]

**1. What problem are we solving?**
People can't answer "where did my money go, am I within plan, am I saving?" Recording takes effort (UPI, cards and cash spread across apps), so people stop tracking; and when the numbers are wrong, people stop trusting the tool. We need near-effortless capture with exact, private, auditable numbers.

**2. Who is the intended user?**
MVP: one individual in India tracking their own money (the Product Owner is user #1). INR by default, IST by default, mixed UPI/card/cash payments. Households, freelancers/businesses and investors are out of scope.

**3. What should the first usable version contain?**
A single-user responsive web ledger: auth with strict data isolation; add/edit/delete (soft, restorable) income and expenses with category, date, payment method and note; default and custom categories; list with search, filters and filtered totals; monthly dashboard (income, expenses, savings, category breakdown, vs. last month, 6-month trend); overall and per-category monthly budgets with 80%/100% in-app warnings; monthly and yearly reports; CSV export; an audit history of changes. After that, a narrow AI layer: natural-language entry that creates a draft you **confirm**, and read-only questions answered from numbers the backend computes.

**4. Which features should be excluded from the MVP?**
Bank/UPI-SMS/statement import, CSV import, OCR, recurring transactions, multiple accounts and transfers, multi-currency (but the schema stores a currency code), households and splits, email/push alerts, AI recommendations/anomaly detection/edit-via-chat, goals, PDF, native apps, offline. Reasons are in PRD §5: mostly privacy risk, dedupe complexity, or unproven value relative to cost.

**5. What are the main user journeys?**
First run → first expense; manual add in ≤ 10 s; NL add with confirm; monthly review with drill-down; set/track budget; correct a mistake (with history); ask a question and verify the answer; yearly review and export (PRD §6, CUJ-1…8).

**6. Which architectural decisions need to be made?** (DEV owns these. PM lists the product constraints each one must meet.)
- Client: responsive web vs. PWA vs. native (PO-Q2).
- Backend and API style; DB choice (must support exact decimals, transactions, constraints).
- **Money representation** (decimal vs. integer minor units, plus currency code) and the **date model** (calendar date + user TZ).
- Auth approach (own vs. managed provider), session model.
- AI integration: provider (Grok candidate), backend-only tool/function calling, draft/confirm flow, eval harness, cost controls.
- Audit log and soft-delete design; idempotency; optimistic concurrency.
- Hosting/deployment, backups/restore (PO-Q4); whether background jobs are needed in MVP (PM view: only purge of soft-deleted records, so keep it minimal).
- PM challenge to DEV: please avoid Redis, queues or microservices in MVP unless a specific AC needs them.

**7. What security and privacy requirements apply?**
Server-enforced per-user isolation (404 on other users' IDs, test suite); strong password hashing, rate-limited login, secure sessions; TLS and encryption at rest including backups; no secrets or financial data in GitHub; no amounts, notes or PII in logs; minimal data to the AI provider, with a disclosure and an off-switch; prompt-injection resistance (read-only tools plus a non-writing "propose draft" tool); CSV-injection-safe export; account deletion and data export, designed in the spirit of India's DPDP Act 2023 (PRD §9).

**8. How should AI improve the expense-tracking experience?**
Two things only in MVP: (a) **faster capture**: "350 lunch today UPI" becomes a pre-filled draft, and nothing is saved until the user confirms; (b) **instant verifiable answers**: "food this month?", "Oct vs Nov?", "over shopping budget?" are answered using backend-computed figures with a link to the underlying transactions. AI never does arithmetic, never writes without confirmation, never gives financial advice, and the app works fully without it. Recommendations and anomaly detection wait until there are months of real data showing they're worth it.

**9. What is the development roadmap?**
The brief's six phases, each gated by measurable exit criteria (ROADMAP.md): Discovery → Architecture (ADRs comparing ≥ 2 options each) → Foundation (auth + isolation tests + CI) → Core ledger, then the PO dogfoods it for 2 weeks with 0 correctness defects → AI with an India-phrase accuracy gate and 0 unconfirmed writes → Production readiness (security review, restore drill) → v1.0. P1 next: recurring transactions, CSV import, accounts, alerts, Indian FY reports.

---

## Key assumptions (reversible; PM defaults unless overridden)
A1 single user · A2 responsive web · A3 manual + AI entry only · A4 INR only, currency stored per amount · A5 default TZ Asia/Kolkata · A6 email/password auth · A7 calendar-year reports · A8 80% fixed threshold, budgets carry forward · A9 fixed payment-method list · A10 30-day soft-delete restore · A11 backend-only, swappable AI provider · A12 English with common Indian shorthand ("2k", "1.5L", "350/-").

## [PRODUCT OWNER INPUT REQUIRED]
1. **PO-Q1 — Single user or household?** Is v1 only for you, or must your spouse or family log into a shared ledger? (Shared roughly doubles auth and permissions scope; PM recommends single-user now, household later.)
2. **PO-Q2 — Web or mobile?** Is a responsive web app (installable as a PWA on your phone) acceptable for v1, or do you need a native Android/iOS app? (PM recommends responsive web/PWA.)
3. **PO-Q3 — Bank/UPI SMS or statement import?** Is automatic import from bank SMS, statements or Account Aggregator a must-have for you to keep using the app, or are manual + AI entry enough for v1? (PM recommends excluding it; CSV import in P1.)
4. **PO-Q4 — Hosting:** Should this run on a managed cloud you pay for (roughly ₹500–2,000/month), self-hosted on your own machine or server, or both? Who is responsible for backups? (This drives deployment, backup and availability targets.)
5. **PO-Q5 — AI provider, cost and privacy:** Is Grok API the provider, what monthly AI spend ceiling is acceptable (e.g. ≤ ₹500/month), and are you comfortable sending transaction text (not your full ledger) to an external AI provider?
6. **PO-Q6 — Reporting year and existing data:** Should yearly reports follow the calendar year or the Indian financial year (Apr–Mar)? And do you have past expense data (e.g. a spreadsheet) you'd need imported on day one? (A yes moves CSV import into MVP.)
