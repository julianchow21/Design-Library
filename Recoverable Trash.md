# Recoverable Trash

Soft-delete bin with a 30-day restore window, backed by a cloud table plus a
localStorage retry queue for the write path. Extracted from Kujira
Collectibles (`app.js`, lines at time of writing: `_queuePendingTrash` and
`flushPendingTrash` :296-336, `sendToTrash`/`sendBatchToTrash` :4761-4819,
`restoreFromTrash` :4854, `hardDeleteTrashEntry` :4839, `purgeExpiredTrash`
:4940, `renderTrash` :4949).

Category: Flows. This is a systems pattern (the write-path retry queue spans
a boot-time flush, a dedicated localStorage key, and a guard used by every
soft-delete call site), too large for a single gallery `<template>` card. The
"Recoverable Trash" card in `index.html` is a pure front-end visualiser of the
bin itself (30-day countdown, restore, expiry purge), no real network, no
queue. This file is the write-path doctrine behind it.

## 1. Preview guard, bail before queuing

Every write in this pattern starts with the same guard used across the app's
sync layer: `isLocalhostPreview()` returns true and the function returns
immediately, before touching the network or the retry queue. A local preview
never reaches production data, and critically it never enters the pending-write
queue either, since that queue exists specifically to protect a write that
was genuinely attempted and failed, not one that was intentionally skipped.

```js
async function sendToTrash(table, item, reason) {
  const trashEntry = { id: 'trash_' + Date.now() + '_' + Math.random().toString(36).slice(2,6),
    data: { originalTable: table, originalId: item.id, item, reason: reason || 'manual', deletedAt: new Date().toISOString() },
    updated_at: new Date().toISOString() };
  if (isLocalhostPreview()) {
    // Keep the snapshot entirely local (own DB.trash array + localStorage)
    // so the Trash tab and restore still work while developing offline.
    DB.trash.push(trashEntry);
    _saveLocalTrash();
    return;
  }
  try {
    const r = await fetch(SB_URL + '/rest/v1/trash', { method: 'POST',
      headers: { ...SB_HDR, 'Prefer': 'resolution=merge-duplicates' }, body: JSON.stringify(trashEntry) });
    if (!r.ok) throw new Error(await r.text());
  } catch (e) {
    _queuePendingTrash(trashEntry); // keep the snapshot until the cloud write lands
  }
}
```

## 2. The pending-write queue, why a failed trash write cannot just be dropped

A delete already happened locally (the item left its source table) by the
time `sendToTrash` runs. If the trash write itself then fails (offline, a
5xx, a timeout) and that failure is swallowed, the item is gone from
everywhere: not in its source table, not in the trash. The 30-day restore
promise would be broken silently, with no error surfaced to the user at the
moment it actually mattered.

The fix: a failed trash write is queued to its own localStorage key,
deduplicated by id, and retried the next time the app loads:

```js
const PENDING_TRASH_KEY = '_kjrPendingTrashWrites';

function _queuePendingTrash(entry) {
  try {
    const list = JSON.parse(localStorage.getItem(PENDING_TRASH_KEY) || '[]');
    if (!list.some(x => x.id === entry.id)) {
      list.push(entry);
      localStorage.setItem(PENDING_TRASH_KEY, JSON.stringify(list));
    }
  } catch (e) { console.warn('queuePendingTrash failed:', e); }
}

async function flushPendingTrash() {
  if (isLocalhostPreview()) return;
  let list;
  try { list = JSON.parse(localStorage.getItem(PENDING_TRASH_KEY) || '[]'); }
  catch { return; }
  if (!Array.isArray(list) || !list.length) return;
  const stillPending = [];
  for (const entry of list) {
    try {
      const r = await fetch(SB_URL + '/rest/v1/trash', { method: 'POST',
        headers: { ...SB_HDR, 'Prefer': 'resolution=merge-duplicates' }, body: JSON.stringify(entry) });
      if (!r.ok) throw new Error(await r.text());
    } catch (e) {
      // Keep retrying for the full 30-day retention window, matching the
      // restore promise itself. Dropping sooner would silently discard the
      // only remaining copy of the deleted item.
      const ts = new Date(entry.data?.deletedAt || entry.updated_at || 0).getTime();
      if (Date.now() - ts < 30 * 86400 * 1000) stillPending.push(entry);
      else console.warn('[Trash] dropping expired pending trash write:', entry.id, e.message);
    }
  }
  localStorage.setItem(PENDING_TRASH_KEY, JSON.stringify(stillPending));
}
```

`flushPendingTrash()` is called on boot and again once the app's session is
ready, covering both a cold load and a load where the first call ran before
auth/session state existed to make the request meaningful. The TTL is
deliberately the same 30 days as the restore window itself: a queued write
that is still failing after 30 days would have expired out of the bin anyway
even if it had landed, so there is no reason to retry it forever.

## 3. Restore is local-first and idempotent

`restoreFromTrash` writes the restored row back into local `DB` and calls
`saveData()` **before** attempting the cloud push, so a restore is never lost
locally even if the network call that follows fails:

```js
DB[originalTable].push(item);
markDirty(originalTable, item.id);
saveData();
let _cloudFailed = false;
try {
  _ts = await sbUpsert(sbTable, item.id, withoutId(item));
} catch (e) {
  _cloudFailed = true; // markDirty + saveData already queued this row for the
                        // next dirty flush, so fall through rather than abort
}
```

On a cloud failure the function does not roll back or re-throw. The row is
already dirty locally, which means the normal sync-engine dirty flush will
pick it up and retry on its own schedule, so the restore still completes from
the user's point of view, with a toast noting that cloud sync will retry.

A second guard makes the whole operation safe to invoke twice (a double-click,
or a retry after a prior cloud push actually landed despite an error being
reported locally): before inserting, `restoreFromTrash` checks whether the
row already exists in the target table by id, and if so treats the call as a
no-op that just clears the trash entry, rather than inserting a duplicate.

## 4. Expiry purge

`purgeExpiredTrash()` iterates the trash table and hard-deletes any entry
whose `deletedAt` is more than 30 days in the past. It runs automatically 5
seconds after boot and again whenever the Trash tab is opened
(`showPage('trash')` calls both `renderTrash()` and `purgeExpiredTrash()`),
so a user does not need to do anything for expired entries to actually leave
the table. The card's demo exposes this as a manual button instead, purely so
the effect is visible in one click without needing to fast-forward real time.

## What was left out

- The actual Supabase REST calls (`sbUpsert`, `sbDelete`, the `trash` table
  shape) are shown above only as the shape to follow, swap in your own
  backend's REST or RPC surface.
- Bulk trash (`sendBatchToTrash`) is the same doctrine applied to an array of
  items in one request, not shown separately here as it adds nothing new.
- The card's own demo restore is a visual no-op (there is no second "active
  items" list to restore into in a self-contained gallery card), it only
  removes the entry from the bin and shows a toast. The queue and restore
  logic above is the real doctrine.
