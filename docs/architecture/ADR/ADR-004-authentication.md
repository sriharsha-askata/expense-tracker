# ADR-004: Authentication

Status: **Superseded**
Superseded: 2026-11-19 by [ADR-015](ADR-015-encryption-and-keys.md) and [ADR-016](ADR-016-github-auth-and-sync.md). There is no server session.
Date: 2026-11-19
Decider: DEV

The text below is historical. Do not build signup, email verification, or server sessions.

## Context

Financial data needs a session that a script cannot read, a password that can be reset, and no dependence on a third party that also holds the ledger's identity. The PM's assumption A6 is email and password. Social login and 2FA are P1 (AUTH-7, AUTH-8). The acceptance tests require generic errors, a lockout window, session revocation, and 404s rather than cross-user reads.

## Options

### A. Argon2id passwords and server-side sessions

The cookie holds a random token. The database stores its SHA-256. The cookie is `HttpOnly`, `SameSite=Lax`, and `Secure` outside local development. CSRF uses a double-submit token on writes and on export.

### B. JWT in browser storage

The API issues a signed token. The SPA stores it and sends `Authorization`. Revocation is a denylist or a short lifetime.

### C. A managed identity vendor (Clerk, Auth0, or similar)

The vendor hosts signup, reset, and sessions. We trust their session and map it to a user id.

## Trade-offs

| | A. Server sessions | B. JWT in the browser | C. Managed vendor |
|---|---|---|---|
| Theft via XSS | Cookie is not readable from script | `localStorage` is readable. A stolen token is the ledger. | Depends on how the vendor's SDK stores the session |
| Revoke on password reset | Delete the rows (AC-AUTH-3.2) | Need a denylist, which is a session store in disguise | Vendor API |
| Privacy | Credentials stay in our database | Same, plus the token is on the client | The vendor sees the email and the login pattern |
| Work | We implement reset, throttling, and cookies | Less server state, more security review | Least code, a bill, a network dependency, and a vendor outage that locks the user out of their own ledger |
| Fit for one operator | High | Acceptable only with an HttpOnly cookie, at which point it is option A with extra signature code | Poor fit before PO-Q4 and PO-Q5 are even answered |

Option B with an HttpOnly cookie is not a different architecture from A; it is A plus a signature we do not need, because the database is the source of revocation. A pure stateless JWT cannot meet "reset kills every session" without storing something.

## Decision

Option A. Parameters for Argon2id follow the OWASP baseline at implementation time: memory 19 MiB, time cost 2, parallelism 1, unless the chosen library's documented recommendation is stricter. Record the actual parameters in the Phase 3 pull request rather than treating this sentence as a pin that drifts.

Proposed session lifetime, pending TQ-3: 7 days idle, 30 days absolute, both enforced from `sessions.last_seen_at` and `sessions.expires_at`. The cookie max-age is not the control.

Login throttling is a table, not Redis: 5 failures in 15 minutes per email, 50 per IP in 15 minutes. Unknown emails still run a verify against a dummy hash.

`REQUIRE_EMAIL_VERIFICATION` is a deployment setting. Proposed default: false for a private deploy, true for a public one (TQ-4). The code path exists either way.

## Consequences

- No third-party script on the login page in v1.
- Google sign-in, if it arrives as P1, is an additional verifier in front of the same session table, not a replacement.
- 2FA (AUTH-8) can be a second factor on this session later. It is not designed in detail here.
- The SPA must send the CSRF header on export as well as on POST, so a foreign page cannot navigate the browser into downloading the CSV.

## Revisit when

The Product Owner wants social login in v1, or a public deploy makes a managed vendor cheaper to operate safely than the mail and session code. That is a new ADR, not a quiet dependency.
