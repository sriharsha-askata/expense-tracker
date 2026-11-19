# User Stories — Personal Expense Tracker

Status: DRAFT v0.1 ([PM] proposal, pending [DEV] review). Priorities: **MVP** (needed for first release) · **P1** (next) · **P2** (later) · **Future** (only if validated).
Format: *As a … I want … so that …*. Acceptance criteria for every MVP story are in `ACCEPTANCE_CRITERIA.md`.

## Epic AUTH — Account & access
| ID | Story | Priority |
|---|---|---|
| AUTH-1 | As a new user, I want to sign up with email and password and verify my email so that my ledger is private to me. | MVP |
| AUTH-2 | As a user, I want to log in and log out securely so that only I can see my finances. | MVP |
| AUTH-3 | As a user, I want to reset a forgotten password via email so that I don't lose access. | MVP |
| AUTH-4 | As a user, I want my data to be completely invisible to every other user so that my finances stay private. | MVP |
| AUTH-5 | As a user, I want to set my timezone (default Asia/Kolkata) so that "today" and months match my life. | MVP |
| AUTH-6 | As a user, I want to delete my account and all my data so that I stay in control. | MVP |
| AUTH-7 | As a user, I want Google sign-in so that I don't manage another password. | P1 |
| AUTH-8 | As a user, I want optional 2FA so that my account is harder to compromise. | P1 |

## Epic TXN — Transactions
| ID | Story | Priority |
|---|---|---|
| TXN-1 | As a user, I want to add an expense (amount, category, date, payment method, note) so that I record where money went. | MVP |
| TXN-2 | As a user, I want to add income so that I can see savings. | MVP |
| TXN-3 | As a user, I want to edit a transaction so that I can correct mistakes. | MVP |
| TXN-4 | As a user, I want to delete a transaction and restore it within 30 days so that mistakes are recoverable. | MVP |
| TXN-5 | As a user, I want a warning when I'm about to save a likely duplicate so that totals aren't inflated. | MVP |
| TXN-6 | As a user, I want a double-tap or network retry to never create two records so that my data stays correct. | MVP |
| TXN-7 | As a user, I want to see the change history of a transaction so that I can trust and audit my data. | MVP |
| TXN-8 | As a user, I want recurring transactions (rent, SIP, subscriptions) so that I don't re-enter them. | P1 |
| TXN-9 | As a user, I want to tag transactions to accounts (HDFC, cash, credit card) and record transfers so that I can see balances per account. | P1 |
| TXN-10 | As a user, I want to record transactions in other currencies so that travel spending is tracked. | P2 |
| TXN-11 | As a user, I want to split one payment across categories so that a mixed bill is categorised correctly. | P2 |

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
| LIST-1 | As a user, I want a paginated list of my transactions, newest first, so that I can review them. | MVP |
| LIST-2 | As a user, I want to filter by date range, type, category and payment method, search notes, and see filtered totals so that I can find and verify spending. | MVP |
| LIST-3 | As a user, I want saved filters so that I can return to common views. | P2 |

## Epic DASH — Dashboard
| ID | Story | Priority |
|---|---|---|
| DASH-1 | As a user, I want this month's income, expenses, savings and savings rate so that I know where I stand. | MVP |
| DASH-2 | As a user, I want an expense breakdown by category so that I see where money goes. | MVP |
| DASH-3 | As a user, I want comparison with last month so that I see whether I'm improving. | MVP |
| DASH-4 | As a user, I want a 6-month spending trend so that I spot patterns. | MVP |
| DASH-5 | As a user, I want to click a dashboard figure to see its transactions so that I can verify it. | MVP |

## Epic BUD — Budgets
| ID | Story | Priority |
|---|---|---|
| BUD-1 | As a user, I want an overall monthly budget so that I cap total spending. | MVP |
| BUD-2 | As a user, I want per-category monthly budgets so that I control specific areas. | MVP |
| BUD-3 | As a user, I want to see utilisation (spent, remaining, %) so that I can adjust behaviour. | MVP |
| BUD-4 | As a user, I want an in-app warning at 80% and when exceeded so that I'm alerted in time. | MVP |
| BUD-5 | As a user, I want email/push alerts for budget thresholds so that I'm told outside the app. | P1 |
| BUD-6 | As a user, I want configurable thresholds and rollover of unused budget. | P2 |

## Epic RPT — Reports & export
| ID | Story | Priority |
|---|---|---|
| RPT-1 | As a user, I want a report for any past month so that I can review history. | MVP |
| RPT-2 | As a user, I want a yearly report (by month and category) so that I understand my year. | MVP |
| RPT-3 | As a user, I want to export my transactions as CSV so that I own my data. | MVP |
| RPT-4 | As a user, I want Indian financial-year (Apr–Mar) reports so that they match tax season. | P1 |
| RPT-5 | As a user, I want PDF reports so that I can share them. | P2 |

## Epic AI — AI assistant
| ID | Story | Priority |
|---|---|---|
| AI-1 | As a user, I want to type "spent 350 on lunch today" and get a pre-filled draft I confirm so that logging is faster. | MVP |
| AI-2 | As a user, I want the assistant to ask, or leave a field blank, when my message is ambiguous so that nothing is guessed silently. | MVP |
| AI-3 | As a user, I want to ask questions like "how much on food this month?" and get answers from my real data with a link to verify them. | MVP |
| AI-4 | As a user, I want to compare two periods ("October vs November") in natural language. | MVP |
| AI-5 | As a user, I want to ask about budget status ("am I exceeding shopping?"). | MVP |
| AI-6 | As a user, I want to turn AI off and still use everything. | MVP |
| AI-7 | As a user, I want to edit or delete a transaction via chat, with confirmation. | P1 |
| AI-8 | As a user, I want to log several expenses in one message. | P1 |
| AI-9 | As a user, I want monthly spending summaries in plain language. | P1 |
| AI-10 | As a user, I want budget recommendations based on my history. | P2 |
| AI-11 | As a user, I want unusual spending flagged (anomaly detection). | P2 |

## Epic IMP — Import & capture (post-MVP)
| ID | Story | Priority |
|---|---|---|
| IMP-1 | As a user, I want to import a CSV/Excel file with column mapping and duplicate detection. | P1 |
| IMP-2 | As a user, I want to import bank statement PDFs. | Future |
| IMP-3 | As a user, I want UPI/bank SMS or Account Aggregator import. | Future |
| IMP-4 | As a user, I want to scan receipts (OCR) into drafts. | P2 |

## Epic HH / GOAL — Future
| ID | Story | Priority |
|---|---|---|
| HH-1 | As a household member, I want shared ledgers and splits. | Future |
| GOAL-1 | As a user, I want savings goals with progress. | P2 |

## Epic OPS — Platform (non-functional, MVP)
| ID | Story | Priority |
|---|---|---|
| OPS-1 | As the Product Owner, I want automated daily backups with a tested restore so that data is never lost. | MVP |
| OPS-2 | As the Product Owner, I want logs and error tracking that contain no financial data so that issues are debuggable but private. | MVP |
