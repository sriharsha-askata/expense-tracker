# ADR-005: AI integration

Status: **Accepted as deferred.** The Product Owner removed AI from the first build on 2026-11-19. The module boundary stays. Do not implement it in Phase 3 or Phase 4.
Date: 2026-11-19
Decider: Product Owner for "not in the first build." DEV for the confirm-only boundary, which still applies when someone later opens the feature.

There is no backend. If this is built later, the device calls the provider directly, shows a draft, and writes to SQLite only after the user confirms. The call needs a network. The rest of the app must not. Tokens and prompts stay on the device. Re-read the "backend" wording below as "the on-device AI module." Do not introduce a server to hold the key.

## Context

Phase 5 adds two behaviours: a natural-language draft the user confirms, and read-only questions whose figures come from the database. The brief's candidate provider is the Grok API, called from the backend. The product rules that bind this ADR:

- The model never inserts, updates, or deletes.
- The model does not do the arithmetic that the UI presents as fact.
- The app works with the feature off or the provider down.
- Notes and the stored ledger are not uploaded for a question. The sentence the user just typed is uploaded for a draft, and the disclosure says so.

This ADR decides the shape. It does not add the feature in this phase, and it does not pin a model id. Pinning a model before the labelled phrase set exists is a guess.

## Options

### A. Backend interface, stateless draft, confirm through the normal create endpoint, read-only tools

`POST /ai/drafts` returns a suggestion and writes no row. The user edits it and confirms. Confirm is `POST /transactions`. Questions call four read tools that ignore any user id the model supplies. Prose is dropped when it contains a number the tool payload does not.

### B. Provider SDK in the browser

The SPA holds the API key, or a proxy still forwards free-form tool calls that the model chooses, including writes wrapped in a prompt that says "only if the user confirmed."

### C. Conversational agent with write tools and an in-chat confirmation

The model can call `create_transaction` after it decides the user said yes. Multi-turn memory is part of the design.

## Trade-offs

| | A. Confirm is a normal create | B. Browser or free-form tools | C. Write tools |
|---|---|---|---|
| Unconfirmed writes | A test can count rows. The model has no code path that inserts. | The key leaks, or a tool list is hard to keep read-only once the client assembles it. | "The model thought the user confirmed" is not a confirmation. AC-AI-1.9 becomes a hope. |
| Arithmetic | Backend sums. The UI shows those sums. | The model will answer with its own arithmetic unless we forbid it and check. | Same failure mode |
| Provider swap | One interface, Grok is the first implementation | SDK lock-in | SDK lock-in plus tool schema lock-in |
| AC-AI-1.3, close the page, nothing saved | True, because nothing was stored | True only if the client did not already call a write | False if a tool already fired |
| Cost control | Server-side daily cap, feature flag | Harder. The client can loop. | Harder |
| UX the PM asked for | A draft card, then Confirm. A structured answer with a link. | Similar UI, worse trust boundary | A chat that feels more capable and fails the safety bar |

## Decision

Option A. The provider interface has two methods: propose a draft, and answer with a tool executor the server owns. `GrokProvider` is the implementation to write in Phase 5, using `GROK_API_KEY` and `GROK_API_BASE_URL` from the environment. If those are absent, the process still starts and AI routes return `AI_DISABLED` or `AI_UNAVAILABLE`.

There is no server-side draft table. A stored draft would be a write the acceptance test does not allow, and it is unnecessary because the confirmed fields travel in the create request.

The tool list is exactly: period totals, category breakdown, compare two periods, budget status. Adding a tool is a code change and a test, not a prompt change.

Daily caps of 30 drafts and 60 queries per user are safety defaults until PO-Q5 sets a ceiling. They are not the product's budget.

## Consequences

- The create service remains the only insert path. Phase 5 must not grow a second one "for convenience."
- Eval phrases are synthetic and live in the test tree, not in production logs.
- A failure of the accuracy targets disables the feature. It does not roll back the ledger (TQ-2, pending).
- Multi-expense messages draft nothing (pending D6).

## Revisit when

PO-Q5 forbids any text from leaving the machine (then Phase 5 is not built), or a measured need appears for a server-side draft, for example a review queue. A write tool requires a new ADR and a product change, not a prompt tweak.
