# ADR-006: Hosting and deployment

Status: **Superseded**
Superseded: 2026-11-19 by [ADR-016](ADR-016-github-auth-and-sync.md). The Product Owner rejected a hosted ledger. PO-Q4 is closed.
Date: 2026-11-19
Decider: DEV

The text below is historical. Do not deploy a Compose stack for user data. The `docker-compose.yml` sketch was removed when this record was superseded.

## Context

The product is a personal ledger. The availability target in the PRD (99.5%, RPO 24 hours, RTO 4 hours) is only meaningful once someone owns a host. Discovery should still decide the unit we deploy, so Phase 3 has one local story and Phase 6 has one production story. The brief's candidates are Docker and Docker Compose, and GitHub Actions for CI.

## Options

### A. Docker Compose on one host

Postgres, the API, the static client behind one proxy, and a daily maintenance container. The same compose file runs on a laptop and on a small VM. TLS terminates at the proxy when there is a hostname.

### B. A managed platform that runs the API and the client as separate services, with a managed database

Less machine administration. More vendor surface, a monthly bill in the range the PM sketched, and a stronger temptation to split the system early.

### C. Laptop only, no containers

Run Postgres and Python on the host directly. Simplest backups (or none). Hardest to reproduce in CI and impossible to hand to a VM later without a second set of instructions.

## Trade-offs

| | A. Compose on one host | B. Managed split | C. Laptop, no containers |
|---|---|---|---|
| Matches "no microservices" | Yes | The platform will push two or three services. We can resist, but the defaults are against us. | Yes |
| Reproducible for CI and for the Product Owner | One file | Different config for local and prod | "Works on this machine" |
| Backup | `pg_dump` from the database container, encrypt, keep 7 days | Vendor snapshots plus a dump we still want, because a snapshot is not a tested restore | A file copy, easy to forget |
| Public internet | Optional. TLS at the proxy. | Usually public | Usually not, which may be what PO-Q4 says |
| Cost | A VM in the PM's ₹500–2,000 example, or zero on a laptop | Similar or higher, plus managed-database pricing | Zero, and no off-site backup unless we add one |
| Secret handling | Environment file on the host, not in Git | Vendor secret store | A local environment file |

## Decision

Option A is the packaging, whatever PO-Q4 answers. Local development, CI, and the deployed host use the same major Postgres version and the same application image. We do not introduce a second deploy path for "just production."

The proxy is Caddy if the host has a public hostname and we want automatic TLS, and nginx or Caddy with TLS off if it does not. That choice is local to this ADR and can flip without touching the API.

What this ADR does **not** decide: the cloud vendor, the region, whether the URL is public, and who runs the restore drill. Those are PO-Q4. Until they are answered, Phase 6 cannot honestly claim a backup.

GitHub Actions is the CI runner (Phase 3), not the production host. CI does not receive a production database URL and does not deploy until a later ADR says how.

## Consequences

- `docker-compose.yml` in this branch starts Postgres only, as a sketch. It is not a running system and was not executed during discovery.
- One replica. The daily command is a separate compose service so it does not run twice if we later add an API replica. See ADR-011.
- Encrypted dumps are the backup. Retention 7 days. The encryption key is the operator's, stored outside the repo.
- No Kubernetes, no multi-region, no separate worker cluster.

## Revisit when

PO-Q4 picks a platform that cannot run Compose, or the restore drill fails because one host is too fragile for the RPO the Product Owner actually wants. A managed database under the same API is a reasonable revision. Splitting the API into services is not.
