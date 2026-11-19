# ADR-011: Modular monolith, and no queue

Status: **Superseded**
Superseded: 2026-11-19 by [ADR-013](ADR-013-local-sqlite.md). "No microservices and no Redis" still holds, because there is no server at all.
Date: 2026-11-19
Decider: DEV

The text below is historical.

## Context

The brief says not to introduce microservices without a demonstrated need. The PM asked us not to add Redis, a queue, or a second service unless an acceptance criterion needs it. The background work actually implied by the criteria is: purge soft-deleted rows after 30 days, finish account deletion, drop old idempotency keys, and take a backup. Budget warnings are computed on read and on write. AI calls are synchronous request/response. None of these need a broker.

## Options

### A. One API process, one database, one daily command

Modules inside the process (`domain`, `services`, `db`, `ai`) keep the boundaries that a premature split would fake. The daily command is `python -m app.maintenance`, the same code, run by the host scheduler once. Backups are `pg_dump` on the same clock, not an application feature.

### B. Microservices

Separate services for auth, transactions, reports, and AI, with their own databases or a shared one. Network calls where function calls would do.

### C. The monolith plus Redis and a worker queue

Even one worker implies a broker, a retry policy, and a dead-letter path. Justified when a job is slow, bursty, or must survive process death independently of the request.

## Trade-offs

| | A. Monolith + daily command | B. Microservices | C. Monolith + Redis |
|---|---|---|---|
| Correctness of a write plus its audit | One database transaction | A distributed transaction, or a bug | One transaction, plus machinery we do not use for that write |
| Ways the AI can write by accident | One create function to audit | A second service that might grow its own insert | Unchanged, plus a job that could retry a write |
| Operational surface for one operator | Compose, a clock, a dump | Several deployables, or a pretence of them | Another container to patch and to fail |
| Budget warning latency | In the request | A hop | A hop for no benefit |
| Purge reliability | If the command misses a day, rows stay hidden and are purged tomorrow | Same problem, more places | A queue does not make a daily delete safer |
| When it breaks down | More than one API replica must not each run the command. A job that takes minutes or calls a flaky third party in volume. | It fits a larger team and an independent scaling story we do not have. | Email bursts, import of large files, OCR |

The demonstrated need does not exist. A queue would be a place for a retried AI confirm to hide, which is the failure mode the product is built to avoid.

## Decision

Option A.

Module rules are in the LLD: domain code does not import the database, the AI module does not import the insert function, routers do not contain arithmetic.

The maintenance command is a separate compose service with a daily schedule, not a thread inside the web workers. With one replica those are equivalent in effect; the separate service is the one that stays correct when a second replica is added later.

Redis is not a dependency. Login throttling is a table. Sessions are a table. Warnings are computed.

## Consequences

- Phase 3's dependency list does not include Redis, Celery, RQ, or a message broker.
- A second API replica is allowed later only together with "the scheduler runs once." That is a hosting change, not a new service.
- If email sending ever blocks requests past a few seconds, the first revision is a short timeout and a user-visible error, not a queue. Revisit C only when mail volume or an import pipeline makes that dishonest.

## Revisit when

Any one of these is true: more than one replica is about to run the daily command; a single job regularly runs longer than a request should; CSV import of large files is in scope and needs durable chunking. Bring evidence, not a diagram.
