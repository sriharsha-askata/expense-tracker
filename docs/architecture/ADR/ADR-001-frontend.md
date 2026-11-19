# ADR-001: Frontend

Status: **Proposed**
Date: 2026-11-19
Decider: DEV

## Context

The product is a private, authenticated ledger. Pages must work on a 360 px phone browser and on a desktop, including keyboard use for the add-transaction flow. There is no public marketing site and no SEO requirement. The backend is a separate API (ADR-002). The candidate named in the brief is React / Next.js. PO-Q2 (web versus native) is still open; this ADR assumes a web client and would be reopened if the Product Owner requires a native app.

## Options

### A. Vite, React, TypeScript, static build

The browser loads static files and calls the API on the same origin through a reverse proxy. A dev server proxies `/api` so cookies stay first-party.

### B. Next.js (React)

A Node server renders the app and talks to FastAPI, or the Next server becomes a second backend.

### C. Server-rendered HTML (templates, small scripts)

The Python app renders pages. Charts and the draft interaction are added with plain scripts where needed.

## Trade-offs

| | A. Vite SPA | B. Next.js | C. Server-rendered |
|---|---|---|---|
| Runtimes in production | Python API + static files | Python API + Node | Python only |
| Cookie and CSRF model | One origin, straightforward | Easy to get wrong if two origins appear | Simplest |
| Fit for a private app | High. No public pages to render. | SSR and React Server Components buy little when every page requires a session. | High for forms, weaker for a dashboard the PM may want to evolve |
| Phone layout | CSS, same as the others | Same | Same |
| Risk | Client retries and amount types must be disciplined (LLD). | Two deployables, two patch cycles, and a temptation to put money logic in Node. | A second UI technology if a rich client is needed later |

Native Android or iOS was not carried as a live option. It is a second client that must reimplement money parsing and would miss the "small and finished" principle for v1.

## Decision

Option A. The client is a Vite + React + TypeScript single-page app. It sends amounts as decimal strings, mints idempotency keys as the LLD describes, and does not cache `/api` if a home-screen manifest is added later.

## Consequences

- Phase 3 adds a Node toolchain for the client only. Node does not serve production traffic.
- Next.js is rejected for v1, not banned forever. Reopen this ADR if we gain a real need for server rendering, such as a public page that must be fast without a session.
- Option C stays available if the SPA proves too large during foundation. Switching before any screens exist is cheap. Switching after Phase 4 is not.

## Revisit when

The Product Owner answers PO-Q2 with "native," or the PM shows a user-facing requirement that a static client cannot meet.
