# ADR-015: Encryption and key management

Status: **Proposed**
Date: 2026-11-19
Decider: DEV
Supersedes: the session half of ADR-004

## Context

Anything that reaches GitHub is ciphertext, except a manifest that contains cipher parameters and a wrapped key. The Product Owner also said we never hold tokens or ledger data on a server of ours. There is therefore nowhere to put a spare key. Two devices must still open the same ledger. A lost key must be described to the user as lost data, because that is the truth.

## Options

### A. Passphrase-derived key only, no random data key

Argon2id(passphrase) is the key that encrypts the snapshot. Simple. Changing the passphrase re-encrypts everything. A weak passphrase is a weak key. We cannot rotate the data key without changing the passphrase.

### B. Random data key, wrapped by a passphrase key; device keystore holds the unwrapped data key

Generate a 256-bit data-encryption key (DEK) on first run. Encrypt snapshots with AES-256-GCM under the DEK. Derive a key-encryption key (KEK) with Argon2id from the passphrase and a salt stored in the manifest. Store the DEK wrapped by the KEK in the manifest. After the user unlocks once, put the DEK in the Android Keystore so ordinary saves and syncs do not ask for the passphrase every time.

A second device: sign in to GitHub, enter the passphrase, unwrap the DEK from the manifest, cache it in that phone's keystore.

Also generate a recovery code (high-entropy, shown once) that wraps the same DEK. The user writes it down. It is not sent anywhere.

### C. Device keystore only

The key never leaves the phone. A second device cannot decrypt. This fails the Product Owner's multi-device requirement. It is the right cache (used inside B) and the wrong only key.

## Trade-offs

| | A. Passphrase is the data key | B. Random DEK, passphrase wraps it | C. Keystore only |
|---|---|---|---|
| Second device | Works, if the passphrase matches. | Works the same way. | Does not work. |
| Stolen GitHub repo | Safe only as far as the passphrase holds. | Same, and the DEK is random, so a weak passphrase is the only weak point, not a poorly generated "key" made of the password itself. | Repo is useless without the phone. And so is the user's new phone. |
| Rotate after a history squash | Re-encrypt under a new password, which is a user-visible event. | New DEK, re-wrap with the same passphrase, re-encrypt the snapshot. Devices that still have the passphrase unwrap the new DEK. Devices that only have the old DEK in the keystore cannot, until the user types the passphrase. | Cannot rotate a key that lives in one phone and also delete history on GitHub in a way the next phone can read. |
| Lost secret | Lost data. | Lost data if both the passphrase and the recovery code are gone, and no phone still has the DEK in its keystore. | Lost phone, or uninstalled app, means lost data even if the user remembers everything. |
| Our ability to reset it | None. | None. | None. |

## Decision

Option B.

Parameters to record in the manifest and pin in implementation, not invent per device: Argon2id, memory 19 MiB, iterations 2, parallelism 1, random 16-byte salt. AES-256-GCM, random 12-byte nonce per encryption, nonce stored with the ciphertext. These are the OWASP Argon2id baseline; if the library's documented mobile guidance at implementation time is stricter, use that and write it down in the PR.

Local database: SQLCipher, key generated on device and stored in the Android Keystore, not derived from the passphrase. Losing the passphrase does not, by itself, lock the phone that is already unlocked and still has the DEK. It does stop every other phone, and it stops this phone after the keystore entry is wiped.

**A lost passphrase and a lost recovery code, with no device that still holds the DEK, means the GitHub snapshot cannot be decrypted. The data is gone. The app must say that in the setup screen, in plain language, before the first save that the user cannot afford to lose. There is no support reset. We must not build a "forgot passphrase" flow that pretends otherwise.**

Changing the passphrase re-wraps the same DEK. It does not re-encrypt history. History erasure is ADR-017 and does rotate the DEK.

## Consequences

- The recovery code is shown once. The user has to confirm they stored it. If they skip it, the screen tells them the passphrase is the only copy.
- The DEK, the passphrase, the recovery code, and the GitHub token are never written to logcat, never written into the snapshot plaintext, and never sent to us. There is no us.
- A public GitHub repo is a failed setup. Ciphertext in a public repo is still ciphertext, and it is also an invitation to download the manifest and attack the passphrase offline. The app refuses.
- Offline attack on the wrapped DEK is limited by Argon2id, not prevented. A short passphrase will fall. The UI requires at least 12 characters and checks a small common-password list. That is not a substitute for the recovery code.

## Revisit when

The Product Owner wants account recovery that a human can perform. That requires holding a key or a reset channel, which this decision forbids. Reopen the product decision before reopening the ADR.
