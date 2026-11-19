# User Stories — Personal Expense Tracker

Status: **v0.2 — revised 2026-11-19 (IST)** to the Product Owner's local-first Android direction (see the changelog in `PRD.md`). Priorities: **MVP** (needed for first release) · **P1** (next) · **P2** (later) · **Later** (a deferred phase, only on Product Owner go-ahead) · **Future** (only if validated) · **Retired** (server-era; ID not reused).
Format: *As a … I want … so that …*. Acceptance criteria for every MVP story are in `ACCEPTANCE_CRITERIA.md`.
ID note: IDs of surviving stories are unchanged for traceability. New epics are SET, SYNC and ERS. Retired IDs stay listed so older references still resolve.

## Epic AUTH — Ownership & isolation (retitled; no accounts or logins)
| ID | Story | Priority |
|---|---|---|
| AUTH-1 | ~~Sign up with email and verify it.~~ No accounts exist; replaced by SET-1. | Retired |
| AUTH-2 | ~~Log in and out securely.~~ No server login; replaced by SET-2 (unlock). | Retired |
| AUTH-3 | ~~Reset a forgotten password via email.~~ There is no reset by design; see SET-3. | Retired |
| AUTH-4 | As a person who installs the app, I want my ledger to be mine alone, isolated from every other install, so that my finances stay private. | MVP |
| AUTH-5 | As a user, I want to set my timezone (default Asia/Kolkata) so that "today" and months match my life. | MVP |
| AUTH-6 | As a user, I want to delete my account and all my data (including my GitHub copy) so that I stay in control. See ERS-3. | MVP |
| AUTH-7 | ~~Google sign-in.~~ | Retired |
| AUTH-8 | As a user, I want an optional app lock (PIN/biometric) so that someone holding my unlocked phone can't open my ledger. *(replaces optional 2FA)* | P1 |

## Epic SET — Setup, passphrase and recovery code
| ID | Story | Priority |
|---|---|---|
| SET-1 | As a new user, I want to start using the app with no account and no network — just choosing a passphrase — so that I can log my first expense in under two minutes. | MVP |
| SET-2 | As a user, I want my ledger encrypted on my phone and a wrong passphrase to only make me wait (never wipe or permanently lock my data) so that I'm safe from a thief and from my own typos. | MVP |
| SET-3 | As a user, I want a one-time recovery code and a plain warning that losing both it and my passphrase makes my encrypted GitHub copy unopenable, so that I understand there is no reset before I rely on the app. | MVP |

## Epic TXN — Transactions
| ID | Story | Priority |
|---|---|---|
| TXN-1 | As a user, I want to add an expense (amount, category, date, payment method, note), even offline, so that I record where money went. | MVP |
| TXN-2 | As a user, I want to add income so that I can see my Net. | MVP |
| TXN-3 | As a user, I want to edit a transaction so that I can correct mistakes. | MVP |
| TXN-4 | As a user, I want to delete a transaction and restore it within 30 days so that mistakes are recoverable. | MVP |
| TXN-5 | As a user, I want a warning when I'm about to save a likely duplicate so that totals aren't inflated. | MVP |
| TXN-6 | As a user, I want a double-tap, retry or crash during save to never create two records so that my data stays correct. | MVP |
| TXN-7 | As a user, I want to see the change history of a transaction so that I can trust and audit my data. | MVP |
| TXN-8 | As a user, I want recurring transactions (rent, SIP, subscriptions) so that I don't re-enter them. | P1 |
| TXN-9 | As a user, I want to tag transactions to accounts (HDFC, cash, credit card) and record transfers so that I can see balances per account and avoid double counting. | P1 |
| TXN-10 | As a user, I want to record transactions in other currencies so that travel spending is tracked. | P2 |
| TXN-11 | As a user, I want to split one payment across categories so that a mixed bill is categorised correctly. | P2 |
| TXN-12 | As a user, I want to record a refund against an expense category so that my spending and budget reflect what I actually paid, and my income isn't inflated. | MVP |
| TXN-13 | As a user, I want to repeat a past transaction as a new entry dated today so that rent, SIPs and subscriptions aren't retyped each month. | MVP |
| TXN-14 | As a user, I want a gentle warning when I log a card-bill payment or EMI that may double-count purchases I already recorded, without being blocked. | MVP |

## Epic CAT — Categories
| ID | Story | Priority |
|---|---|---|
| CAT-1 | As a new user, I want India-relevant default categories so that I can start immediately. | MVP |
| CAT-2 | As a user, I want to create and rename my own categories so that they fit my life. | MVP |
| CAT-3 | As a user, I want to archive a category without losing history so that old data remains accurate. | MVP |
| CAT-4 | As a user, I want sub-categories so that I can analyse in finer detail. | P2 |

## Epic LIST — Find transactions
| ID | Story | Priority |
|---|---|---|
| LIST-1 | As a user, I want a list of my transactions, newest first, so that I can review them. | MVP |
| LIST-2 | As a user, I want to filter by date range, type, category and payment method, search notes, and see filtered totals so that I can find and verify spending. | MVP |
| LIST-3 | As a user, I want saved filters so that I can return to common views. | P2 |

## Epic DASH — Dashboard
| ID | Story | Priority |
|---|---|---|
| DASH-1 | As a user, I want this month's income, expenses, refunds and **Net** (income − expenses + refunds), with the formula visible, so that I know where I stand without mistaking it for savings. | MVP |
| DASH-2 | As a user, I want a spending breakdown by category so that I see where money goes. | MVP |
| DASH-3 | As a user, I want comparison with last month so that I see whether I'm improving. | MVP |
| DASH-4 | As a user, I want a 6-month spending trend so that I spot patterns. | MVP |
| DASH-5 | As a user, I want to tap a dashboard figure to see its transactions so that I can verify it. | MVP |

## Epic BUD — Budgets
| ID | Story | Priority |
|---|---|---|
| BUD-1 | As a user, I want an overall monthly budget that carries forward, with past months locked, so that I cap total spending and history never changes. | MVP |
| BUD-2 | As a user, I want per-category monthly budgets so that I control specific areas. | MVP |
| BUD-3 | As a user, I want to see utilisation (spent, remaining, %) so that I can adjust behaviour; refunds reduce what counts as spent. | MVP |
| BUD-4 | As a user, I want clear in-app states — ok, approaching (≥ 80%), at limit (exactly 100%), exceeded (over, with the overspend) — so that I'm alerted in time. | MVP |
| BUD-5 | As a user, I want notifications for budget thresholds so that I'm told outside the app. | P1 |
| BUD-6 | As a user, I want configurable thresholds and rollover of unused budget. | P2 |

## Epic RPT — Reports & export
| ID | Story | Priority |
|---|---|---|
| RPT-1 | As a user, I want a report for any past month so that I can review history. | MVP |
| RPT-2 | As a user, I want a yearly report (by month and category), by default for the calendar year, so that I understand my year. | MVP |
| RPT-3 | As a user, I want to export my transactions as a CSV that opens correctly in Excel, with no GitHub needed, so that I own my data. | MVP |
| RPT-4 | As a user, I want an Indian financial-year (Apr–Mar) preset on the yearly report so that it matches tax season. *(moved from P1 into MVP)* | MVP |
| RPT-5 | As a user, I want PDF reports so that I can share them. | P2 |

## Epic SYNC — Optional GitHub sync and backup
| ID | Story | Priority |
|---|---|---|
| SYNC-1 | As a user, I want to optionally connect my own GitHub account and a private repository for encrypted backup and sync, and have the app work fully without it, so that I'm never forced onto a service. | MVP |
| SYNC-2 | As a user with two phones, I want entries I created on each to merge automatically (pull, then merge, then push) so that I don't have to think about it. | MVP |
| SYNC-3 | As a user, I want to be shown both versions and choose when the *same* entry was changed on two phones, before anything is pushed, so that nothing is silently overwritten. | MVP |
| SYNC-4 | As a user, I want deletes to sync as tombstones (restorable for 30 days) so that a delete on one phone applies on the other without losing the safety net. | MVP |
| SYNC-5 | As a user with a new or replacement phone, I want to restore my ledger from my GitHub repo with my passphrase or recovery code so that I don't lose my history. | MVP |
| SYNC-6 | As a user, I want sync to never slow or block saving, and to tell me plainly when it is paused or unavailable, so that the app always feels fast and honest. | MVP |

## Epic ERS — Erasure and privacy
| ID | Story | Priority |
|---|---|---|
| ERS-1 | As a user, I want to "forget old versions" of my data on GitHub (a confirmed history squash plus a new data key) so that deleted things are really gone from my repo's history. | MVP |
| ERS-2 | As a user, I want permanently deleted transactions to take their amounts and notes out of my change history too, so that "deleted" means deleted. | MVP |
| ERS-3 | As a user, I want to delete my account — removing my GitHub repo and wiping this phone's keys and ledger — after a confirmation and an export offer, so that I can leave cleanly. | MVP |
| ERS-4 | As a user, I want the privacy text to tell me honestly that GitHub may retain copies I and the app can't remove, so that I can decide with eyes open. | MVP |

## Epic AI — AI assistant (**Later**: not in the first build; confirm-only when built)
| ID | Story | Priority |
|---|---|---|
| AI-1 | As a user, I want to type "spent 350 on lunch today" and get a pre-filled draft I confirm so that logging is faster. | Later |
| AI-2 | As a user, I want the assistant to ask, or leave a field blank, when my message is ambiguous so that nothing is guessed silently. | Later |
| AI-3 | As a user, I want to ask questions like "how much on food this month?" and get answers from my real data with a link to verify them. | Later |
| AI-4 | As a user, I want to compare two periods ("October vs November") in natural language. | Later |
| AI-5 | As a user, I want to ask about budget status ("am I exceeding shopping?"). | Later |
| AI-6 | As a user, I want AI off by default, with a disclosure of what is sent before I turn it on, and to use everything without it. | Later |
| AI-7 | As a user, I want to edit or delete a transaction via chat, with confirmation. | P1 after AI |
| AI-8 | As a user, I want to log several expenses in one message. | P1 after AI |
| AI-9 | As a user, I want monthly spending summaries in plain language. | P1 after AI |
| AI-10 | As a user, I want budget recommendations based on my history. | P2 |
| AI-11 | As a user, I want unusual spending flagged (anomaly detection). | P2 |

## Epic IMP — Import & capture (post-MVP)
| ID | Story | Priority |
|---|---|---|
| IMP-1 | As a user, I want to import a CSV/Excel file with column mapping and duplicate detection. | P1 |
| IMP-2 | As a user, I want to import bank statement PDFs. | Future |
| IMP-3 | As a user, I want UPI/bank SMS or Account Aggregator import. | Future |
| IMP-4 | As a user, I want to scan receipts (OCR) into drafts. | P2 |

## Epic HH / GOAL / PLAT — Future
| ID | Story | Priority |
|---|---|---|
| HH-1 | As a household member, I want shared ledgers and splits. | Future |
| GOAL-1 | As a user, I want savings goals with progress. | P2 |
| PLAT-1 | As an iPhone user, I want the app on iOS (same project). | Later |

## Epic OPS — Platform (non-functional, MVP)
| ID | Story | Priority |
|---|---|---|
| OPS-1 | As the Product Owner, I want my data never to be lost: CSV export always works, optional encrypted GitHub backup, and a verified restore onto a clean phone. *(replaces server backups)* | MVP |
| OPS-2 | As the Product Owner, I want crash/error reports and logs that contain no financial data, passphrase, recovery code or tokens so that issues are debuggable but private. | MVP |
| OPS-3 | As the Product Owner, I want automated tests on every pull request and an installable Android build I can request on demand and install from a link, with no Android Studio or local toolchain, so that I can test everything from chat. | MVP |
