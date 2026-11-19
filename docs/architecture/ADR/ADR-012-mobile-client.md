# ADR-012: Mobile client

Status: **Accepted**
Date: 2026-11-19
Decider: Product Owner, recorded by DEV
Supersedes: ADR-001

## Context

The app is local-first. The first release is Android only. A later iOS build must not require a rewrite, so the Product Owner chose React Native with Expo and TypeScript, not Kotlin. Saves still hit on-device storage immediately and work offline. GitHub is optional.

DEV had recommended Kotlin because JavaScript's `number` type is a binary float. That risk is real and is not wished away by this decision. The mitigations below are part of the acceptance, not optional polish.

## Options

### A. Kotlin and Jetpack Compose

Best fit for `Long`, the Android Keystore, and WorkManager. An iOS client would be a second codebase. The Product Owner rejected that for the reason above.

### B. Flutter

One codebase for Android and iOS. Dart integers are exact. Rejected by the Product Owner in favour of the React Native ecosystem.

### C. Expo, React Native, TypeScript

One TypeScript codebase. The first store listing is Android. iOS is a later build of the same project, not a port. Native modules are required for SQLCipher and for a fast Argon2id.

## Trade-offs

| | A. Kotlin | B. Flutter | C. Expo / React Native |
|---|---|---|---|
| Paise | `Long` | `int` | `number` is a float. Money must be `bigint` or a decimal string, converted at the SQLite boundary. |
| iOS later | A new app | Same app | Same app, which is why this option won |
| Offline save | Direct | Plugin | Native module through the bridge. The save still commits before any network call. |
| Secure storage and crypto | First-party | Plugins | `expo-secure-store` plus a native libsodium build. Expo Go cannot load those. |
| Background sync | WorkManager | Adequate | Weaker. Foreground sync is the path we rely on. |

## Decision

Option C. Expo and React Native, TypeScript, Android first.

### Libraries

Pin versions when the Expo project is created, not in this note. Use the current stable Expo SDK at that moment.

| Job | Library | Why |
|---|---|---|
| Language | TypeScript, `strict` | Amounts are a branded `bigint` or a decimal string, not `number`. |
| Local database | `op-sqlite` with SQLCipher | Expo's bundled SQLite is not encrypted. SQLCipher matches ADR-013. |
| Snapshot encryption | `react-native-libsodium`: `crypto_aead_xchacha20poly1305_ietf` | 24-byte random nonce. Safer to misuse than AES-GCM's 12-byte nonce. |
| Passphrase KDF | libsodium `crypto_pwhash` (Argon2id) | Same native library. Memory 19 MiB, opslimit and parallelism matching the OWASP baseline (iterations 2, parallelism 1) unless libsodium's parameter API forces the nearest stronger interactive limit. Record the actual opslimit in the manifest. |
| Key and token storage | `expo-secure-store` | Android Keystore now, iOS Keychain later. Values must stay under its size limit (about 2 KB). A token and a wrapped key fit. The database does not. |
| GitHub device flow | `fetch` plus `expo-web-browser` | Device flow has no redirect URI and no client secret. `expo-auth-session` is the wrong shape. |
| Percent display | `decimal.js`, constructed from strings only | Half-up to one decimal. Never from a `number`. |

AES-256-GCM was the other cipher the Product Owner named. It is acceptable if the libsodium build cannot ship. Switching cipher is a `format` bump in the manifest, not a quiet change. XChaCha20-Poly1305 is the one to implement.

### Expo Go, managed workflow, and a development build

Expo Go cannot load SQLCipher or libsodium. The project uses Expo's continuous native generation (prebuild) and a development build. That is still the Expo workflow. It is not a hand-maintained bare React Native repo, and it is not Expo Go.

Argon2id at 19 MiB on the JavaScript thread can freeze a cheap phone for seconds. The KDF runs inside libsodium, off that thread. Do not drop the memory setting to make a pure-JS Argon2 (`@noble/hashes`) feel fast.

Background sync on Android through Expo is best-effort (`expo-background-task` / task manager). The reliable sync points are: app open, a debounced attempt after a save while the app is open, and a Sync button. A kill between the SQLite commit and the push leaves the outbox. The next foreground run pushes it. Saves do not wait.

### Money rule for this stack

Our largest single amount, ₹10,00,00,000.00, is 10^10 paise. A realistic sum of those amounts stays under 2^53, so a SQLite driver that returns an INTEGER as a JavaScript `number` has not yet rounded it. That is not permission to do arithmetic in `number`. The database boundary converts to `bigint` immediately. Addition is `bigint`. Percentages go through `decimal.js` from strings. A test covers `0.10 + 0.20` as 30 paise. ESLint forbids `Number(` in the money module.

## Consequences

- The first release binary is Android. The TypeScript and the snapshot format are what make iOS a later build rather than a rewrite. No iOS project is created in Phase 3.
- `android/` in this repo is a placeholder note, not a Kotlin tree. Phase 3 creates the Expo project. Do not start that project until the Product Owner says Phase 3 may start.
- A reviewer rejects a pull request that parses an amount with `JSON.parse` into a `number` or that logs a token.

## Revisit when

The libsodium or SQLCipher config plugin cannot be built with current Expo prebuild. The fallback is a development build with those modules linked explicitly, not a return to Kotlin, unless the Product Owner reopens the framework.
