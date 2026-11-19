# Development Guide

Canonical copy: [`docs/development/DEVELOPMENT_GUIDE.md`](docs/development/DEVELOPMENT_GUIDE.md).

The brief places this guide under `docs/development/`. The Phase 1–2 task also asks for `DEVELOPMENT_GUIDE.md` at the repository root. The long form lives with the other development docs so there is one copy to edit. If a root file and that copy ever disagree, `docs/development/DEVELOPMENT_GUIDE.md` wins.

Until the Product Owner approves the architecture:

- Do not add application code, migrations, or CI workflows that pretend the app exists.
- Do not store secrets or personal financial records in the repo.
- Do not use floating-point money, and do not let AI write a transaction without a confirmation.
