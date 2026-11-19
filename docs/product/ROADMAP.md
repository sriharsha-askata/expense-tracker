# Roadmap — Personal Expense Tracker

Status: DRAFT v0.1 ([PM] proposal, pending [DEV] review and PO approval). Aligned to the brief's 6 phases.
Rule: **we don't start a phase while the previous phase's critical exit criteria are unmet.** No dates are committed until DEV estimates; durations are PM placeholders.

| Phase | Goal | Key stories | Exit criteria (all measurable) |
|---|---|---|---|
| **1. Product Discovery** | Agree what we build and why | — | PRD, USER_STORIES, ACCEPTANCE_CRITERIA, ROADMAP merged to `main` via PR; DEV has reviewed them in writing ([DEV] evaluation + [DECISION] log); all `[PRODUCT OWNER INPUT REQUIRED]` questions answered or explicitly deferred; `PROJECT_STATUS.md` created |
| **2. Architecture** | Decide how | — | HLD, LLD, DATABASE_SCHEMA, API_SPECIFICATION, DEVELOPMENT_GUIDE drafted; ≥ 1 ADR per material choice (frontend, backend, DB, auth, AI integration, hosting/deploy, money representation, date/timezone model), each comparing ≥ 2 options; PM sign-off that every MVP AC is satisfiable by the design; PO approval recorded |
| **3. Foundation** | A deployable, tested skeleton | AUTH-1–6, OPS-2 (basic) | Repo structure per brief; one-command local run; migrations working; CI runs lint + tests on every PR and blocks merge on failure; auth ACs (AC-AUTH-1…6) passing in CI, including the cross-user isolation suite (AC-AUTH-4.x); secrets scan in CI shows 0 findings |
| **4. Core Development** | Usable ledger without AI ("core usable") | TXN-1–7, CAT-1–3, LIST-1–2, DASH-1–5, BUD-1–4, RPT-1–3 | 100% of ACs for these stories automated and passing; reconciliation property test (totals = exact sum of line items, 10k random decimals) passing; timezone boundary tests passing; E2E for CUJ-1, 2, 4, 5, 6, 8; perf: dashboard p95 < 1 s at 10k txns; **PO uses it for ≥ 2 weeks** and logs ≥ 50 real transactions with 0 correctness defects |
| **5. AI Integration** | Faster capture + verifiable answers | AI-1–6 | All AI ACs passing; labelled India-phrase test set (≥ 200 items) meets: amount ≥ 98%, type ≥ 98%, date ≥ 95%, category ≥ 85%; 0 writes without confirmation; 100% of answer figures match tool outputs; AI draft p95 < 4 s; AI disable switch verified; cost per active user per month measured and within PO's ceiling (PO-Q5) |
| **6. Production Readiness** | Safe to rely on daily | OPS-1, OPS-2 | Security review done (OWASP Top-10 checklist, dependency scan with 0 high/critical); backups + **one successful restore drill**; monitoring/alerting on errors and latency; no financial data in logs (scan); deployment and recovery docs; release notes; PO go/no-go recorded → **MVP v1.0** |

## Post-MVP (re-prioritised with PO after 4–6 weeks of real use)
- **P1 (v1.1–1.2):** recurring transactions (TXN-8), CSV/Excel import with dedupe (IMP-1), accounts & transfers (TXN-9), email/push budget alerts (BUD-5), Indian FY reports (RPT-4), AI edit/delete with confirmation (AI-7), multi-expense messages (AI-8), monthly AI summary (AI-9), Google sign-in/2FA (AUTH-7/8).
- **P2:** multi-currency (TXN-10), splits (TXN-11), receipt OCR (IMP-4), savings goals (GOAL-1), budget recommendations and anomaly detection (AI-10/11), PDF reports, sub-categories.
- **Future (only with validated demand + privacy review):** household/shared ledgers (HH-1), bank statement/SMS/Account Aggregator import (IMP-2/3), native mobile, offline.

## Initial milestone & issue breakdown (proposal for GitHub)
- **M1 Discovery**: Issue — PM docs review; Issue — PO open questions.
- **M2 Architecture**: one issue per ADR; schema; API spec.
- **M3 Foundation**: repo/CI; auth (AUTH-1…3, 5, 6); isolation test suite (AUTH-4); logging baseline.
- **M4 Core**: one issue per epic slice — TXN-1/2, TXN-3/4/7, TXN-5/6, CAT, LIST, DASH, BUD, RPT.
- **M5 AI**: AI-1/2 (entry), AI-3/4/5 (queries), AI-6, AI eval set.
- **M6 Release**: OPS-1 backups and restore drill, security review, deploy docs.
