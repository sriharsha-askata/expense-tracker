# ADR-017: Tombstones, 30-day restore, and Git history

Status: **Accepted**
Date: 2026-11-19
Decider: Product Owner, adopting the proposal in the decision section below.
Supersedes: ADR-009

## Context

Deletes must sync, and a delete must be restorable for 30 days. The user was also promised that deletion really deletes. Those two promises are easy on a server that can `DELETE` a row. They are not easy in Git. Every successful sync creates a commit. Old commits keep old ciphertexts. Anyone who has the data key can decrypt an old commit and read a "deleted" transaction. GitHub can also keep unreachable objects after a history rewrite, and we cannot see those backups.

Encryption without destroying the key does not delete anything. Key rotation without rewriting history leaves the old commits decryptable by the old key. Squashing history without rotation leaves the same key able to read any blob GitHub has not garbage-collected, and any clone the user made.

## Options

### A. Tombstones only, keep Git history forever

Restore is easy. "Really deletes" is false for anyone who has the key and the repo history.

### B. Tombstones for 30 days, then drop them from the live snapshot, and keep history

The current file no longer has the row. `git log` still does. This is what most "sync" apps quietly do. It should not be described as deletion.

### C. Tombstones for 30 days in the live snapshot; erasure is an explicit history squash plus data-key rotation

Day to day, history stays so sync is simple. When the user asks to forget deleted items, or confirms the prompt after tombstones have expired, the app writes a **new orphan commit** whose tree is only the current manifest and the current snapshot, encrypted with a **new** DEK. The new DEK is wrapped with the same passphrase and recovery code. The old DEK is wiped from this device. The branch is force-pushed to that orphan commit. Other devices must enter the passphrase once to unwrap the new DEK (the keystore still has the old one, which no longer opens the new snapshot).

Full "delete everything": delete the GitHub repository through the API, wipe SQLite, wipe the keystore entries. Tell the user GitHub may retain internal copies we cannot reach.

## Trade-offs

| | A. History forever | B. Drop from the tip only | C. Squash and rotate, on purpose |
|---|---|---|---|
| 30-day restore | Yes | Yes, until the tombstone is dropped | Yes, until the tombstone is dropped and, after a squash, only if the row is still inside the new snapshot |
| A person with the passphrase reads old deletes | Yes | Yes | Not from the branch tip. Residual: GitHub's unseen backups, and any phone that has not synced and still holds the old key and the old database. |
| Offline phone during the squash | Unaffected | Unaffected | Must merge its local SQLite with the new snapshot and type the passphrase. If we squashed silently, we would strand it. |
| Force-push | No | No | Yes, of the data branch only, after confirmation. Never of the application source repo. |
| Honesty | Poor if we say "deleted" | Poor | Matches the words, with a stated residual. |

## Decision

Option C for erasure. Option A's silence is how ledgers lie.

Ordinary delete does **not** squash. It writes a tombstone, syncs it, and allows restore for 30 days on every device that has synced. When 30 days have passed, the device drops the row from SQLite and from the next snapshot it writes. The old commits still contain it, under the current DEK, until the user runs compaction.

Compaction is a confirmed action, not a background surprise. The confirmation says: other devices must sync first; you will need the passphrase on those devices after this; GitHub may keep private copies anyway.

Full account erasure deletes the repo and the local keys. The screen says the data will not come back.

## Consequences

- Restore during the window does not need Git history. The tombstone is in the current snapshot.
- Change history for a purged row is removed from the live snapshot with the row. It remains in old commits until compaction.
- A device that missed compaction merges by revision parents against the new snapshot (the local database is still the full pre-compaction ledger) and then drops anything the user already purged on the other device only if the new snapshot's tombstone list or absence says so. Absence after a compaction means "this revision was deliberately left out," which the snapshot marks with a `compacted_at` marker and a list of purged ids so a late device does not resurrect them. That list is ids only, inside the ciphertext, and is itself dropped at the next compaction.
- We do not claim GitHub's operational backups forget on our schedule.

## Revisit when

The Product Owner wants automatic silent compaction. That needs a new ADR. The failure mode is an offline phone and a passphrase prompt the user does not expect.
