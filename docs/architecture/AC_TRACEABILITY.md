# Acceptance-criteria traceability

Status: **Proposed**
Date: 2026-11-19

The first version of this matrix mapped the PM's acceptance tests onto a server. That mapping is withdrawn. `docs/product/` is unchanged on purpose; the PM will revise the PRD and the acceptance tests. Until that lands, several criteria (email signup, server 404s, hosted backups) describe a product we are no longer building. They are not "satisfiable" by the local-first design, and they should not be implemented as written.

What the new design is accountable for, assuming the product docs catch up:

| Rule | Where it is enforced | Notes |
|---|---|---|
| Integer paise, no float, reject 3 decimal places | `domain.money`, SQLite `INTEGER` check | Still in force |
| `0.10 + 0.20 = 0.30` | Sum of minor units | Still in force |
| Calendar date, Asia/Kolkata, no shift when the zone changes | `TEXT` date, zone on settings | Still in force |
| Isolation between people | No shared database | The old cross-user HTTP tests do not apply. A test that two installs do not share a file replaces them. |
| Double-save | `mutation_id` unique | Still in force, local |
| Two editors, no silent overwrite | Merge conflicts, push blocked | Replaces the HTTP 409 |
| Duplicate warning | Local five-minute check | Per device, not global |
| Soft delete, 30-day restore | Tombstone in the live snapshot | Still in force |
| Deletion really removes data from GitHub history | Confirmed squash + new data key, or repo deletion | See ADR-017. Not true of tombstones alone. |
| Budget states including exactly 100% as `at_limit` | Integer comparisons | PM restated the four names |
| Change history | `history` rows inside the snapshot | A new phone sees them |
| CSV formula injection and a UTF-8 BOM | `domain.csv` | Accepted |
| AI writes nothing without confirm | No provider client in the first build | Accepted as deferred |
| Refunds reduce the category | Aggregation in the LLD | Accepted. The PRD text is not updated yet. |
| Repeat | New id, today's date | Accepted. The PRD text is not updated yet. |
| Headline Net, calendar year, Indian FY preset | LLD | Accepted |
| Past budgets locked | LLD | Accepted |
| Cloud tests and APK | ADR-018 | Accepted. Not configured yet. |
| No hard lockout | Backoff only | Replaces AC-AUTH-2.2's throttle-as-lockout. GitHub may still lock the GitHub account; that is not our lockout. |

Server-only criteria (email verification, session cookies, RLS, hosted backup drills, p95 on a server) are not scheduled. The PM should retire or rewrite them when the product docs move.
