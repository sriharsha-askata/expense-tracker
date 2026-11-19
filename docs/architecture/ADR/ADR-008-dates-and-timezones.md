# ADR-008: Dates and timezones

Status: **Proposed**
Date: 2026-11-19
Decider: DEV

## Context

A transaction belongs to a calendar day in the user's life: "lunch today," "rent on the 1st." The acceptance tests require that 00:30 IST on 1 December is 1 December, not 30 November, and that changing timezone does not move existing transactions. Budgets and reports are calendar months and calendar years. System events (created, audited, deleted) are real instants and should stay in UTC. The default zone is `Asia/Kolkata` (A5). IST is UTC+5:30 and does not observe daylight saving; the design must still be correct if a user later picks a zone that does.

## Options

### A. `date` for the transaction day, IANA timezone on the user, `timestamptz` for system instants

The user picks a day, or the client defaults it to "today" computed in the user's zone. The column stores that day. Reports filter the day. "Today" is computed when a default is needed, not stored on the row.

### B. Store a `timestamptz` instant and derive the local date on read

"Lunch" is stored as an instant, often midnight UTC or midnight IST. The local date is `instant AT TIME ZONE user.timezone`. Changing the user's timezone recomputes every date.

### C. Store the date and a fixed offset (`+05:30`) on every row

Similar to A, but the offset is not a timezone. It cannot represent a later zone change and it duplicates the same offset onto every row.

## Trade-offs

| | A. Calendar date | B. Instant, derive on read | C. Date plus offset per row |
|---|---|---|---|
| AC-AUTH-5.2, changing timezone does not move old rows | Holds, because the day is the fact | Fails, unless we also store the original zone per row and ignore the user's new zone, which is option A with extra steps | Holds only if we keep the old offset |
| Midnight boundary | Not applicable. There is no midnight inside the column. | The entire design is about not getting this wrong. 00:30 IST must not be stored as the previous UTC date and then re-derived incorrectly. | Same as A if the date is what we filter |
| Month and budget membership | `transaction_date` falls in a month or it does not | Depends on a zone at query time | Depends on which offset we apply |
| "Upcoming" | `date > today` in the user zone, computed | Compare instants, easy to drift from the calendar the user sees | Computed |
| Storage | 4 bytes and an obvious value in a SQL client | An instant that is not what the user meant | Two columns that can disagree |
| Future zones with DST | "Today" uses IANA rules. Stored days stay put. | DST transitions move the derived day of old rows if the zone on the user changes, and even if it does not, a badly chosen instant near the transition is ambiguous | A fixed offset is wrong twice a year |

Option B is the usual choice for events that happened at a moment (a login, a payment captured at an instant from a bank). It is the wrong choice for a date the user typed. The PM's model is the typed date. I am agreeing with it and making it the schema.

## Decision

Option A.

- `transactions.transaction_date` is a `date`.
- `users.timezone` is an IANA name, default `Asia/Kolkata`, validated against the platform zone database. Fixed offsets such as `+05:30` are rejected.
- `created_at`, `updated_at`, `deleted_at`, and audit times are `timestamptz` from the database clock, serialised as UTC.
- Month windows are half-open ranges on the date: `[first of month, first of next month)`.
- The one-year future limit and the `upcoming` flag use `today` in the user zone at request time. `upcoming` is not a column.
- Dates before `2000-01-01` are rejected as junk. This is a technical bound, reversible if the Product Owner has older history to type in.

Reports take a date range. "Calendar year" is a preset. The Indian financial year is a different preset over the same query (PO-Q6), not a different column.

## Consequences

- The client sends `YYYY-MM-DD`, never an instant, for the transaction day.
- Tests that care about 00:30 IST inject a clock. They do not sleep.
- A bank-import feature in a later phase, which does have real instants, must convert to a calendar date once, in the user's zone, at import time, and then store the date. It must not reopen option B for manually typed rows.

## Revisit when

We add a feature that is truly an instant, such as a statement line with a posted timestamp that the user did not choose. Store that instant beside the date if we need it. Do not replace the date.
