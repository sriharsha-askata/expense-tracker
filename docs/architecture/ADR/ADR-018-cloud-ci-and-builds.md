# ADR-018: Cloud tests and installable Android builds

Status: **Accepted**
Date: 2026-11-19
Decider: Product Owner. He tests and builds only through chat and cloud agents. He does not install Android Studio or a local Android toolchain.
Related: ADR-012

## Context

Every pull request needs typecheck, lint, and tests without a laptop. The Product Owner needs an APK he can install from a link, with no Play Store and no USB cable. Expo Go cannot load SQLCipher or libsodium (ADR-012), so it is not a stand-in for the real app.

Numbers below are from Expo's published pricing and docs as of 2026-11-19 ([expo.dev/pricing](https://expo.dev/pricing/), [internal distribution](https://docs.expo.dev/build/internal-distribution/)). They can change. Nobody has run a build in this phase, and no Expo account has been created from this repository.

## Options

### A. GitHub Actions for tests, EAS Build for the APK

Actions runs Node on `ubuntu-latest` for every pull request. EAS Build, Expo's cloud builders, compiles the Android binary when we ask it to. Internal distribution yields an APK and a URL. EAS can create and store the Android signing key on first build, so nobody generates a keystore locally.

### B. GitHub Actions with an Android emulator or a Gradle build on the runner

The runner would need the Android SDK. That is a local toolchain moved into CI: slow, large, and easy to get wrong. It still does not give the Product Owner a tap-to-install link as simply as EAS internal distribution.

### C. Expo Go as the thing he installs

Fast, and useless here. The encrypted database and libsodium are not in Expo Go. A UI shell that skips them would not be the product.

## Trade-offs

| | A. Actions + EAS | B. SDK on the runner | C. Expo Go |
|---|---|---|---|
| Product Owner's machine | A phone and a browser | Same, after we maintain the SDK image | A phone, and the wrong binary |
| Tests for money, merge, crypto, SQL | Node, no emulator | Possible, at a high cost | None of the native path |
| APK link | Yes, internal distribution | We would host the file ourselves | No APK of this app |
| Free capacity | Expo Free plan: up to 15 Android builds and 15 iOS builds per calendar month, low-priority queue, 45-minute build timeout. Resets on the first of the month. Unused builds do not roll over. Over the quota, builds stop until next month or a paid plan. | GitHub Actions minutes, which are a separate budget | Free, and not this app |
| Secrets | An Expo access token in GitHub Actions secrets. The keystore stays in EAS, not in git. | Keystore would also be a secret we manage | None |

## Decision

Option A.

**Tests on every pull request.** One workflow, on `ubuntu-latest`, Node only:

- `tsc --noEmit`
- lint
- unit and integration tests: money math, merge and conflict logic, crypto round-trips, SQLite migrations

Crypto round-trips in CI use `libsodium-wrappers` (Node), the same XChaCha20-Poly1305 and Argon2id parameters as `react-native-libsodium` on the device. A shared test vector must pass in both. Do not compile the native module on the runner.

SQLite migration tests use Node's SQLite (`node:sqlite` on a current Node, or `better-sqlite3` if that API is missing). They apply the same SQL files the app ships. They do not cover SQLCipher page encryption. That check is manual, on a real install, once an APK exists: a copied database file is not readable as plaintext SQLite.

No emulator, no Android SDK, and no Expo Go step in this workflow. Cloud agents run the same commands in their VM: install Node if needed, install the npm dependencies, run the test script. That is the whole loop for a code change that does not claim to have been tapped on a phone.

**APKs are not built on every pull request.** Fifteen Android builds a month is enough for a solo Product Owner if a build is requested on purpose: a `workflow_dispatch` run, or a build when a release tag is pushed. It is not enough if every documentation commit or every push produces an APK. The low-priority queue can wait a long time at peak; Expo's own note says 90 minutes or more. The first time the free quota runs out, builds stop until the next month unless the account moves to a paid plan (Starter is the published next step). Say so in the workflow log. Do not silently start paying.

The internal-distribution profile sets `distribution: internal` (or `android.buildType: apk`). The build page returns a URL. Opening that URL on the phone installs the APK after Android's "install unknown apps" prompt. Default internal URLs are unauthenticated if Expo's project setting is left on; anyone with the link can download. Prefer requiring an Expo login for the link once the account exists, so a leaked URL is not a leaked binary. The binary still holds no ledger and no user token.

**Credentials, never committed:**

- An Expo account the Product Owner can sign in to. The cloud agent cannot create this for him; he has to own the login.
- `EXPO_TOKEN` (an Expo access token) stored as a GitHub Actions secret. The workflow passes it to `eas build`. It is not in the repo, not in an issue, and not in the APK.
- The Android keystore, created by EAS on the first cloud build and kept in Expo's credential store. Losing access to that Expo account means we cannot sign an update with the same key without a recovery of those credentials. Export the keystore from EAS once and store it in the Product Owner's password manager. Do not put it in this repository.

**Preview.** Expo Go is not a practical preview of this app. The preview is the internal APK. A second EAS profile can build a development client (also an APK, also one build credit) if a later phase wants fast JavaScript reload. That is optional and still cloud-built. It is not required for the first install.

**End-to-end later.** Maestro (or a similar runner) against an emulator is a Phase 6 candidate, not a Phase 3 gate. A GitHub-hosted emulator is slow and is a toolchain the Product Owner asked to avoid. If we add it, it runs in CI or a device farm, still not on his machine. Unit tests remain the merge bar.

## Consequences

- This phase does not add the workflow file. There is no Expo project to typecheck. Adding a green workflow that runs nothing would be a false test result. Phase 3 adds the workflow in the same change as the project.
- The Product Owner installs from a link. He does not install Android Studio, a JDK, or an emulator.
- A cloud agent that claims "I installed it on a phone" has not done so unless an APK URL exists and a person opened it. Agents run the Node tests.

## Revisit when

The free quota is too small for the way we actually build, or EAS drops internal APK distribution. The alternative is a paid Expo plan, not a return to a local Android SDK.
