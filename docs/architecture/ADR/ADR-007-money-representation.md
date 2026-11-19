# ADR-007: Money representation

Status: **Accepted** — integer paise remain the rule on the Expo client. `number` is not a money type (ADR-012).
Date: 2026-11-19
Decider: DEV

Storage is now a SQLite `INTEGER` minor-unit column, not a Postgres `bigint`. The representation is unchanged: paise as integers, currency code beside them, decimal strings at the edges, no binary floats. See [ADR-013](ADR-013-local-sqlite.md).

## Context

The product's first principle is that totals reconcile to the paisa. The acceptance tests reject `100.555` instead of rounding it, require `0.10 + 0.20` to display as `0.30`, and require aggregates to equal the sum of the line items. The API is JSON, which has no decimal type. The product is INR-only in v1, and the schema must still store a currency code so a later currency is not a change of meaning (A4). Floating point is forbidden by the brief and by the PRD.

## Options

### A. Integer minor units (`bigint` paise) plus `char(3)` currency

`₹350.00` is stored as `35000` and `INR`. The API accepts and returns decimal strings. Parsing uses a decimal type, then scales by 100. Sums are integer sums. Display formatting, including Indian grouping, happens at the edge.

### B. `numeric` / `Decimal` in the database and the application

Postgres `numeric(20, 2)` is exact. SQLAlchemy maps it to Python `Decimal`. JSON still cannot carry it, so the API would use strings anyway.

### C. IEEE-754 floating point

Named only to record the rejection. It fails `0.10 + 0.20` and is not a candidate.

## Trade-offs

| | A. Minor units | B. Numeric | C. Float |
|---|---|---|---|
| Exact sums | Yes, in the type the CPU adds | Yes, inside Postgres and `Decimal` | No |
| The failure we are most likely to ship | Someone treats the integer as rupees and is off by 100. Tests on `0.10` catch this. | Someone writes `float(amount)` or a JSON number at the boundary. The type does not scream. | The bug is the representation |
| JSON boundary | Strings, and a type that rejects numbers. One conversion, at the edge. | Same string boundary, plus `Decimal` discipline everywhere in between | JSON numbers look convenient and are wrong |
| More than 2 decimal places | Rejected before scaling, because the scaled value must be an integer | A `numeric(20, 2)` column may round, which the acceptance test forbids. We would still need a reject-before-write check. | Rounds constantly |
| Later currencies with 0 or 3 decimals | Needs an exponent table. Until then, non-INR is rejected and the exponent for INR is the constant 2. | `numeric(20, 4)` could store them, and every format call must remember the currency's scale anyway | Worse |
| Aggregates and the 80% test | `spent * 100 >= budget * 80` is exact | Possible with `numeric`, and easier to write a division by accident | Unusable |
| Readability in a SQL client | `35000` needs interpretation | `350.00` is obvious | `349.999999` is obvious in a bad way |

Option B is a legitimate design. I am not choosing A because `numeric` is inexact. I am choosing A because this codebase will be written and reviewed by agents, the dangerous mistake is a float at the boundary, and an integer column makes that mistake a type error more often. The API is strings in both designs, so the client does not get simpler with `numeric`.

## Decision

Option A. Columns are `amount_minor bigint` and `currency char(3)`. Bounds: `amount_minor > 0` and `amount_minor <= 10000000000` (₹100,000,000.00, written ₹10,00,00,000.00). The API rejects JSON numbers, rejects scale above 2, and emits two decimal places. SQL money expressions use `bigint` only. Display percentages use `Decimal` from those integers, half-up to one decimal place, and are never stored.

The database does not `CHECK (currency = 'INR')`. The API rejects anything else. Aggregates group by currency and fail closed if a user ever has two.

## Consequences

- Formatting `₹1,23,456.78` is a presentation function with tests. It is not the storage format.
- CSV amounts are the same decimal strings, without a rupee sign and without grouping commas.
- Audit JSON stores `"350.00"`, not `35000` and not `350.00` as a JSON number, so a reader of the audit does not need this ADR to interpret a diff.
- A future currency with a non-2 exponent requires a currency table and a migration of the parser. The stored INR minor units stay valid.

## Revisit when

We add a currency whose minor unit is not 1/100, or a measured performance problem appears (it will not be the `bigint`). Do not revisit in favour of float.
