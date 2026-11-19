# ADR-012: Mobile client

Status: **Proposed**
Date: 2026-11-19
Decider: DEV
Supersedes: ADR-001

## Context

The Product Owner replaced the responsive web app with a native, local-first app. Saves must hit on-device storage immediately and work with no network. Android-first is acceptable unless a second platform is in the first release. The money path must not cross a floating-point `number` type. The client also has to hold a GitHub token and an encryption key in real device secure storage, and it has to run a sync that can be interrupted.

## Options

### A. Kotlin, Jetpack Compose, Android

One platform. `Long` for paise. Android Keystore and SQLCipher are ordinary libraries. Background sync can use WorkManager. GitHub is HTTPS calls.

### B. Flutter

One codebase for Android and iOS. Dart `int` is a 64-bit integer on device, so paise fit. SQLCipher exists as a plugin. Keystore access is through plugins. A second platform is real code, not a promise.

### C. Expo / React Native

One JavaScript codebase, native shell. Fast UI iteration. The money path and the crypto path both cross the JavaScript bridge unless every amount stays in a native module.

## Trade-offs

| | A. Kotlin | B. Flutter | C. Expo / React Native |
|---|---|---|---|
| Paise | `Long`. The type system rejects a float if we keep amounts off `Double`. | `int`. Same discipline, easy to slip into `double` at the JSON boundary. | JavaScript `number` is a float. A single `JSON.parse` into a number breaks the ledger. We would have to ban that by convention. |
| Offline save | Room/SQLCipher on the UI thread's dispatcher, no bridge. | Plugin call, still local. | Bridge to a native SQLite module on every save. |
| Keystore and token storage | First-party. | Plugins, mature enough, another supply-chain edge. | Plugins. The token must never be passed through a JS log. |
| Background sync | WorkManager is built for "sync when the radio is free." | Possible, less direct. | Background execution on Android is the weak part of this stack. |
| iOS in v1 | Not covered. A second client later speaks the same snapshot format. | Covered. | Covered, with the float hazard on both. |
| Time to a correct ledger | Shortest on Android. | Longer, and the first release includes two UI shells to get wrong. | Short for screens, longest for a review that proves no float and no token leak. |

## Decision

Option A for the first build: Kotlin, Jetpack Compose, minimum Android SDK 26 (Keystore behaviour we can rely on).

This assumes iOS is not in the first release. If the Product Owner reads "anyone can install" as including iPhone in v1, this ADR flips to Flutter **before any screen is written**. Do not start Kotlin and then add a second client in the same release.

Expo is rejected for the money path, not for lack of UI speed. A banned `number` type that the language wants to use for every JSON value is the wrong default for this ledger.

## Consequences

- The sync format in ADR-014 is the portability boundary. A later Flutter or iOS client must read that format, not a Kotlin object.
- UI tests for money formatting can be JVM tests. They do not need a device.
- No Node server, no Vite app, no shared web session.

## Revisit when

The Product Owner requires iOS in the first release, or Kotlin-on-Android proves unable to store the token and the key the way ADR-015 describes.
