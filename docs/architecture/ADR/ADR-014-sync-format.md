# ADR-014: What we store in the GitHub repo

Status: **Proposed** — the Product Owner accepted pull-then-merge-then-push, unique ids, user choice on conflicting edits, and tombstones. They have not separately signed the one-snapshot file layout. This record is the layout DEV recommends they include in the Phase 3 approval.
Date: 2026-11-19
Decider: DEV
Related: ADR-015, ADR-016, ADR-017

## Context

SQLite on the device is the source of truth (ADR-013). GitHub receives a copy so another device can merge and so the user has a backup. GitHub must not see plaintext amounts, notes, or category names. Sync is pull, merge, then push. New entries with different ids merge by themselves. Two edits of the same entry since the last sync are a conflict the user resolves. Deletes are tombstones. A batch of offline saves must not become hundreds of API calls.

Git's own merge is the wrong tool once the payload is ciphertext. A one-byte change in plaintext changes the whole ciphertext. The app decrypts, merges, and encrypts. Git only moves blobs and detects "the remote moved."

## Options

### A. Append-only encrypted file per change

`ops/<random-uuid>.enc`, one operation each. A new device downloads every file and replays them. Tombstones are just files. Compaction deletes old files and rewrites history.

### B. One encrypted logical snapshot per sync, plus the revision chain inside it

Each successful sync writes a single ciphertext (`snapshot.enc`) of the full logical state: rows, tombstones still inside the 30-day window, history, unresolved conflicts, and each row's revision parents. A small plaintext `manifest.json` holds only cipher parameters and the wrapped key. No amounts. The snapshot is a canonical serialization, not a raw SQLite file (page order would change the ciphertext for no reason and would leak nothing useful to a merger).

### C. Periodic raw SQLite snapshot plus a plaintext changelog

Rejected as stated. A raw database file is a poor canonical form, and a plaintext changelog violates the encryption requirement. Mentioned because "snapshot plus changelog" is easy to misread as this.

## Trade-offs

| | A. One file per change | B. One snapshot per sync | C. Raw SQLite + plaintext log |
|---|---|---|---|
| GitHub sees amounts | No, if each file is encrypted. | No. Manifest is crypto metadata only. File size still leaks roughly how large the ledger is. | Yes. Forbidden. |
| API cost of a sync | One content write per change. GitHub's published guidance is on the order of 80 content-creating requests per minute and 500 per hour. A phone that was offline for a trip, or a repeat of many rows, blows that budget. | One content write per sync, however many local saves are in the batch. A fetch is one read of that file. | Same cost problem as A if the log is one file per row, plus the privacy failure. |
| Merge | Natural per operation. | In-app, using revision parents stored inside the snapshot. Git will not merge the blob. If the remote `sha` moved, pull, decrypt both, merge, push. | Git would conflict on the binary and the log would be readable. |
| New phone | Replay the whole log. Fine at the size of a personal ledger, painful after years if we never compact. | Download one snapshot. | Download one file, and the world can read it. |
| Deletes | Tombstone file stays until compaction. | Tombstone row stays inside the snapshot until it expires. Older commits still hold old ciphertexts until history is squashed (ADR-017). | Plaintext history. |
| Implementation risk | Many files, partial failure, directory listings that reveal how often the user edits. | Must get the merge right. The transport is boring, which is what we want. | High privacy risk. |

## Decision

Option B. One canonical encrypted snapshot per successful sync. The revision DAG inside it is the changelog. Unresolved conflicts are part of the snapshot so the other device cannot push a silent winner.

Transport details and the pull-then-push loop are in `API_SPECIFICATION.md` (that file is now the sync protocol; there is no application HTTP API).

Do not `git merge` the ciphertext. Do not PUT one Contents API file per transaction.

## Consequences

- A sync of 500 offline saves is still one upload.
- Commit messages are a constant string, `Update encrypted ledger`. They must not contain amounts, notes, or category names.
- The repo is refused if it is public.
- History grows by one blob per sync until the user compacts it (ADR-017). That is acceptable at this size and is the reason compaction exists.
- Two devices that both synced from the same snapshot and then both edit different entries produce two snapshots with the same parent revision. The merger unions those entries. Two edits of one entry become a conflict record. The push waits.

## Revisit when

Snapshots get large enough that uploading the full state on every sync is actually slow on a bad network (this is a personal ledger; that is not the first problem), or GitHub refuses the Contents/Git Data pattern and we have to switch to a smart-HTTP push of the same single blob.
