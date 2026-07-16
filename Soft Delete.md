# Soft Delete

A trash-bin delete: removing a row never destroys it, it moves the row into
a separate Trash store with a 30-day restore window, and only one explicit,
confirmed action ever discards it for good. Extracted from Kujira
Collectibles v3.31 (`app.js`, the trash-bin block plus the pending-write
retry queue and the undo/redo snapshot doctrine it shares with every other
mutation). Generalised: table names, field names and confirm-dialog helpers
below are placeholders, swap in your own.

Category: Flows. This is a systems pattern (a separate store, an offline
retry queue, a scheduled prune, and a sync interaction), too large for a
single gallery `<template>` card. The "Trash Flow" card in `index.html`
demos the piece that is genuinely demoable on an in-memory store: delete,
restore, and empty-with-confirm. This file is the full pattern, including
the parts a static card cannot show (an offline write, a 30-day prune, a
background sync push).

## 1. Architecture: a separate store, not a deleted flag

Two ways to build "soft delete": add a `deleted`/`deletedAt` column to the
row's own table (every read filters `WHERE deleted_at IS NULL`), or move the
row wholesale into a separate `trash` table. Collectibles uses the second,
because several different source tables (items, sales, saved views, ...)
all need the same restore/prune behaviour, and a single trash store lets
one restore/prune/empty implementation serve all of them:

```js
// One shape for every deleted row, regardless of source table.
{ id: 'trash_' + Date.now() + '_' + rand(4),
  data: { originalTable, originalId: item.id, item, reason, deletedAt: isoNow() },
  updated_at: isoNow() }
```

A single-table app can use the lighter deleted-flag model instead, its
integration steps (section 7) still apply, just as an update rather than a
move. The trade-off: a flag means every existing query needs the filter
added, forgetting one leaks a "deleted" row back into a live view. A
separate store means restore/prune logic lives in one place, at the cost of
duplicating the row's shape into a second table.

## 2. Delete: snapshot before the destructive removal

The row is captured into the trash store first, the removal from its home
table happens only once that copy safely exists, never the other order:

```js
function softDelete(table, item){
  sendToTrash(table, item);        // 1. the full row is now safe elsewhere
  removeFromSource(table, item.id); // 2. only now does it leave its table
}
```

This is the same "snapshot before mutation" doctrine as undo/redo (see the
"Undo Stack" pattern): capture the pre-mutation state, then mutate, never
try to reconstruct what was lost after the fact.

## 3. The 30-day prune

A scheduled or on-load sweep hard-deletes anything past its window, using
`deletedAt`, the moment the row actually left its table, not the row's own
`updated_at` (a delayed write must not reset a countdown that already
started):

```js
async function purgeExpiredTrash(){
  var cutoff = Date.now() - 30 * 86400000;
  for (const entry of await fetchTrash())
    if (new Date(entry.data.deletedAt).getTime() < cutoff) await hardDelete(entry.id);
}
```

## 4. The offline pending-write recovery queue

If the network write that lands a row in the trash store fails (offline,
5xx), the row has often already been removed from its source table for a
responsive UI. Losing the trash write at that point would lose the row
outright, so the **full trash entry**, not an id reference, gets queued to
local storage and retried:

```js
try { await writeTrashEntry(entry); }
catch(e){ queuePendingWrite(entry); } // full entry: the source row is already gone
```

This queue is deliberately separate from a general "dirty row" sync queue.
A dirty-row queue typically re-reads the row from local state to retry, but
after an optimistic delete there is nothing left locally to re-read, the
entry itself must carry the complete snapshot. On flush, an entry still
younger than 30 days keeps retrying; one older than that gets dropped, its
restore promise already expired regardless of whether the write ever lands,
retrying forever past that point buys nothing.

## 5. Restore, and how it interacts with sync

Restore commits locally first, network second, so the user-visible action
never blocks on connectivity:

```js
async function restoreFromTrash(trashId){
  const entry = findTrashEntry(trashId);
  if (alreadyPresent(entry.originalTable, entry.item.id)) { await hardDeleteTrashEntry(trashId); return; } // guards a double-click/retry
  insertIntoSource(entry.originalTable, entry.item);       // 1. local commit, always
  saveLocally();
  try {
    const ts = await pushUpsert(entry.originalTable, entry.item);
    stampServerTimestamp(entry.item.id, ts);               // stops a false conflict on the next merge
  } catch(e) { /* local restore stands, normal retry queue covers the push */ }
  await hardDeleteTrashEntry(trashId);                      // 2. only now does the trash copy retire
}
```

Cloud failure degrades to "restored locally, sync will retry", never to
"restore silently did nothing". The trash entry is only removed once the
restore is durably committed locally, so a crash mid-restore leaves the
safety copy intact rather than losing both copies at once.

## 6. Empty trash: the one true hard delete

Every other operation in this pattern is reversible. Empty trash is not,
so it is the only path gated behind an explicit confirm naming the count:

```js
async function emptyTrash(){
  const entries = await fetchTrash();
  if (!entries.length) return; // nothing to confirm, button stays disabled
  if (!await confirmDialog(`Permanently delete all ${entries.length} items? This cannot be undone.`)) return;
  await hardDeleteAll(entries);
}
```

There is deliberately no further backup behind this confirm. It is where
the user's intent to permanently discard is expressed, keeping a shadow
copy after an explicit forever-delete would undermine that decision, not
protect it.

## 7. Integration steps for a new app

1. Add a `trash` store (table or array): `id`, `data` (`originalTable`,
   `originalId`, full `item`, `reason`, `deletedAt`), `updated_at`.
2. Route every delete through one `sendToTrash(table, item, reason)` helper,
   never a raw delete from any call site, so nothing can bypass the safety
   net.
3. Never let a local/dev/preview environment write real trash rows to a
   shared backend, keep a fully local fallback so development still
   exercises restore and empty-trash without touching production data.
4. Add the pending-write retry queue (section 4) in the same catch path as
   the trash write, expiring entries at the same 30-day mark as the
   retention window itself.
5. Build restore local-first (section 5): insert, mark dirty, save, then
   attempt the network push, never the reverse.
6. Add a prune sweep (section 3) on load or on a schedule.
7. Empty trash is the one hard-delete surface (section 6), always behind an
   explicit, count-naming confirm.
8. Build a Trash view: list, restore, empty, a designed empty state, and a
   days-left countdown per row, so the safety window is visible, not just
   an unlabelled timestamp.

## 8. Verification gotchas

1. **Test the offline path literally.** Go offline, delete a row, confirm
   the full entry lands in the pending-write queue (not just the source row
   vanishing), then go back online and confirm it flushes and disappears
   from the queue.
2. **A double-click or retried restore must never duplicate a row.** Check
   presence by id before inserting, not just before starting the restore.
3. **The countdown compares against `deletedAt`, never `updated_at`.** A
   write that lands late must not reset the 30-day clock to "now".
4. **Prove the dev/local guard actually blocks writes to the shared store**,
   and that its local fallback still lets restore and empty-trash be tested
   without a live backend.
5. **A prune sweep and a list fetch can race.** Guard render against an id
   that a concurrent purge already removed, never crash or show a blank row
   for it.

## What was left out

- Fan-in from several source tables into one trash store is a real
  Collectibles detail (items, sales and saved views all route through the
  same helper); keep your own list of which tables participate, it is a
  project decision, not part of the reusable shape.
- The general dirty-row conflict/merge rule for concurrent edits (a stale
  local write losing to a newer server copy) is separate sync doctrine, not
  repeated here.
- No UI mockup beyond the code shapes above, the demoable slice (delete,
  restore, empty-with-confirm on an in-memory store) is the "Trash Flow"
  card in `index.html`. The offline queue, the prune sweep and the sync
  push in sections 3 to 5 cannot run from a static gallery card.
