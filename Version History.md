# Version History

Named or automatic point-in-time snapshots of an app's whole record store,
restorable at any moment. Extracted from Kujira Collectibles v3.31
(`app.js`, lines 2817-3152: `sbFetchVersions`/`sbSaveVersion`/`sbDeleteVersion`,
`saveVersion`/`_saveVersionWithName`, `maybeRunDailyAutoVersion`,
`restoreVersion`, `deleteVersion`, `_cacheVersions`/`_evictVersionBlobsFromLS`).
Generalised below: no Supabase URL or keys, no Collectibles table names, a
storage-agnostic interface stands in for the concrete Supabase calls.

Category: Flows. This is a systems pattern (a cloud table, a size-bounded
local cache, and a restore path that touches every table in the app), too
large for a single gallery `<template>` card. The "Version History" card in
`index.html` is a self-contained in-memory visualiser of save, restore and
prune only, no real backend. This file is the full pattern.

## 1. Snapshot shapes

Two kinds of row, same shape, told apart only by `name`:

- **Named (manual)**: the user clicks Save Version, optionally typing a
  name. A blank name falls back to `'Version ' + <formatted date/time>`,
  so a row is never saved nameless.
- **Auto-daily**: once per calendar day (checked on load, and again every
  6 hours so a long-lived tab still triggers), saves one row named
  `'Auto · YYYY-MM-DD'`. A `localStorage` date marker de-dupes so several
  tabs or reloads on the same day never spam the list. Skips entirely
  while the store is empty: an empty-DB snapshot on first boot is noise,
  and could clobber a real snapshot that arrived from another device.

Both kinds carry the same shape: `{ id, name, ts, data }`, where `data` is
a serialised snapshot of **every** synced table at that instant, never
just the one the user happened to be looking at. Restoring anything less
than the whole store is a correctness trap, see section 3.

## 2. Where it lives, and the storage-agnostic seam

Collectibles backs this with a Supabase `versions` table as the source of
truth, plus a `localStorage` cache for offline reads and instant list
paint. Neither fact belongs in the reusable pattern. Wrap access behind
one small interface and the save/restore/prune logic below never needs to
know or care what is underneath:

```js
// Any backing store implements this. Collectibles' concrete version wraps
// Supabase REST; a new app could implement the same three methods over
// IndexedDB, a different REST API, or localStorage alone, with zero
// change to the calling code in sections 3-4.
const VersionStore = {
  async list()          { /* -> [{id,name,ts,data}], newest first */ },
  async save(version)   { /* upsert one row, -> true ONLY on confirmed write */ },
  async remove(id)      { /* best-effort delete of one row */ }
};
```

`save()` reporting `true` only on a genuinely confirmed write matters:
pruning (4) must never delete a remote row on the strength of a write that
may have silently failed. Guard every write with the preview-guard rule
from `Sync Engine.md` (its rule 1): a `file://`/`localhost` preview must
never reach the real backend, and must never report a fake success.

## 3. Restore semantics: backup first, never a silent loss

Restoring is destructive by definition, it replaces the running state. The
house rule this pattern exists to enforce: **snapshot the CURRENT state as
its own new version BEFORE applying the older one, every time, no
exceptions.** That backup (Collectibles names it `'Auto-backup before
restore'`) is what turns "I restored the wrong version" into a two-click
recovery instead of a second, compounding data-loss event.

Sequence, in order, none skippable:

1. Confirm with the user first, this overwrites the live state.
2. Save a backup version of the current state. Only prune (4) after that
   save reports a confirmed success.
3. Parse the target snapshot's `data` inside a try/catch. A corrupt or
   truncated blob must bail out cleanly (message shown, function returns)
   before it ever reaches a mutation. Never let a bad parse partially apply.
4. Capture the set of row ids present **before** the restore, per table.
5. Apply the restored `data` onto every table it covers, not just the
   table currently in view.
6. Diff the before/after id sets per table, and explicitly delete (never
   just leave "dirty") any id that existed before but is absent after.
   Skipping this step is the single most non-obvious bug in the whole
   pattern: without it, a row the restore was meant to remove just sits in
   the backend, unsynced, and the next reconciliation merges it straight
   back in, so the restore silently fails to stick.
7. Re-render every affected view.

## 4. Prune policy

Both the manual save path and the auto-daily path push onto the same
newest-first list, then trim it to a fixed cap (Collectibles uses 50) by
dropping whatever falls past the cap, "oldest" meaning simply "off the
end" of that ordering. Cloud-side pruning mirrors the same rule, but runs
**only** after a save reports a confirmed write: query the backend's own
newest-N, delete anything older. A save that never reached the cloud must
never trigger a cloud delete, or a failed write plus an optimistic prune
compounds into actual data loss.

## 5. Local cache sizing, a hard-won lesson

An earlier version kept every version's full data blob (roughly 300 to
400KB each) in `localStorage`. At a 50-row cap that is up to ~18MB,
comfortably over a browser's ~5MB quota, and it surfaced as an opaque
"local storage full" failure with no obvious cause. The fix generalises
past this one field: **keep the full payload for only the newest N
entries locally (Collectibles uses N=2), degrade everything older to
metadata-only (name and date, no blob), and treat the remote store as the
sole full-fidelity source for anything outside that hot set.** Restoring
an entry whose local blob was trimmed transparently re-fetches it from the
backend first. If even the trimmed write still will not fit, fall back
further to metadata-only for every row, rather than let the quota error
propagate to the user.

## 6. Integration steps

1. Implement `VersionStore` (section 2) against your own backend, or start
   with a localStorage-only stub for a single-device app and add a real
   backend later. The calling code in sections 3-4 does not change.
2. Wire a save call at every point a user is about to do something risky,
   plus the daily auto-snapshot (a short `setTimeout` after load, and a
   6-hourly `setInterval` to catch long-lived tabs).
3. Wire restore per section 3, in order, no step skipped.
4. Run the local-cache trim (section 5) after every save, and once on
   boot, to compact any legacy full-blob rows left over from before the
   trim existed.
5. Render the list: name, formatted date, and a short meta line (row
   counts per table, parsed out of `data`) so a user can tell snapshots
   apart without opening each one.

## 7. Verification gotchas

1. **A prune that runs before the save it depends on is confirmed will
   delete data that was never actually backed up.** Always gate cloud
   pruning behind that save call's own success return, never fire-and-forget it.
2. **Restoring only the table currently on screen looks correct in
   testing and is wrong.** Test a restore with at least two different
   tables carrying different, independently verifiable values, and
   confirm both changed back, not only the one visible at the time.
3. **The stuck-row bug (3.6) does not show up in a single-device test.**
   It only surfaces once a second device (or a second tab acting as one)
   reconciles after the restore. Test with two independent clients against
   the same backend, not one.
4. **A corrupt snapshot must fail loudly to the restore action, not
   silently to the console.** Feed the restore path a deliberately
   truncated or invalid blob in a test and confirm the user sees a message
   and nothing mutates, rather than a half-applied table.
5. **A dismissed confirm proves nothing on its own.** Confirm that
   Cancelling truly leaves both the current state and the version list
   byte-for-byte untouched, not just that the dialog appeared.

## What was left out

- The concrete Supabase REST calls (`fetch` against the project URL, anon
  key headers, `Prefer: resolution=merge-duplicates`) are Collectibles-
  specific wiring, replaced above by the `VersionStore` interface.
- The specific seven tables Collectibles snapshots, and their field lists,
  are that app's own schema, not part of the reusable shape. The pattern
  only requires "every table the snapshot logically covers", whatever
  those are for a given app.
- The gallery card demos save, restore and prune entirely in memory, with
  a "simulate next day" button standing in for the real daily timer. It
  does not exercise the real Supabase calls, the cache-sizing trim (5), or
  the two-device reconciliation scenario in gotcha 3: all three need a
  real backend to verify properly, never assume the in-memory demo passing
  is evidence they work.
