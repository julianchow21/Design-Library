# Sync Engine

localStorage-first sync with a cloud backend: the UI always reads and writes
local state instantly, a debounced background flush pushes dirty rows to the
cloud, and a load-time reconciliation merges cloud state back in without
clobbering unsynced local edits. Extracted from Kujira Forex (`index.html`,
lines 361 to 463 at time of writing: `isLocalhostPreview`, `_dirty`,
`markDirty`, `sbBatchUpsert`, `sbFetchAll`, `sbDelete`, `_queueDelete`,
`flushPendingDeletes`, `_flushDirty`, `mergeTable`, `loadData`). Generalised
below: no Supabase URL, no trade schema, config object instead of
Forex's `DB`/`TABLES` globals.

Category: Flows. This is a systems pattern (data layer spanning several
functions and two localStorage keys plus a REST backend), too large for a
single gallery `<template>` card. The "Sync Engine" card in `index.html` is a
pure front-end visualiser of the dirty-queue-flush-retry mechanics only, no
network, no real backend. This file is the full pattern.

## 1. The five rules, non-negotiable

These are hard-learned loss classes. Breaking any one of them silently drops
a user's data at some point:

1. **Preview guard at the TOP of every flush and every write.** `file://` and
   `localhost` never talk to the cloud. Critically, a skipped write must
   **not** clear the dirty flag, or the next reload-and-merge treats the row
   as already synced and drops it for good.
2. **Snapshot dirty-row IDs before any `await`.** A user can keep editing
   while a batch is in flight. Take the ID set at the top of the flush, not
   partway through, or a fresh edit made mid-upload gets marked clean by the
   time the flush finishes even though it never actually reached the server.
3. **Sync by timestamp, never by row count.** `mergeTable` keeps the local
   row for any ID still in the dirty set, and otherwise takes the cloud row.
   Comparing counts (`cloud.length > local.length`) cannot tell "cloud has a
   genuinely newer state" apart from "cloud just has more rows for an
   unrelated reason", and picks wrong in exactly the cases that matter.
4. **Never sync a local preview.** A locally-computed preview object carries
   a fake or stale timestamp. If it reaches the reconciliation path it can
   look "newer" than real synced data and overwrite it.
5. **Echo the server's own timestamp back for concurrency**, never compare
   two timestamps taken from different clocks (client vs server clock drift
   makes "newer" undecidable otherwise).

## 2. Per-row dirty tracking, not a single boolean

A single `isDirty` flag for a whole table cannot express "these three rows
changed, the other 200 did not". Journal tracks dirt per row, per table:

```js
const _dirty = {};
TABLES.forEach(t => _dirty[t] = new Set());

function markDirty(table, id){
  if (_dirty[table]){ _dirty[table].add(id); _persistDirty(); }
}
```

Persisted to its **own** localStorage key (`DIRTY_KEY`, distinct from the
data key itself), so a dirty set survives a refresh even if the flush never
got to run:

```js
function _loadDirty(){
  try{
    const o = JSON.parse(localStorage.getItem(DIRTY_KEY) || '{}');
    TABLES.forEach(t => _dirty[t] = new Set(o[t] || []));
  }catch(e){}
}
function _persistDirty(){
  try{
    const o = {};
    TABLES.forEach(t => o[t] = [..._dirty[t]]);
    localStorage.setItem(DIRTY_KEY, JSON.stringify(o));
  }catch(e){}
}
```

Every mutation (create, edit, tag rename, bulk action) calls `markDirty`
immediately, then a debounced `saveData()` writes local state and arms a
single flush timer:

```js
let _saveTimer = null;
function saveData(){
  localStorage.setItem(APP.storageKey, JSON.stringify(DB));
  clearTimeout(_saveTimer);
  _saveTimer = setTimeout(_flushDirty, 1000);
}
```

## 3. Per-row retry on chunk failure, the poisoned-row problem

Upserts are batched (200 rows per request in Journal) for efficiency, but a
single malformed row in a 200-row batch must never wedge the other 199. The
doctrine: try the chunk, and only on failure fall back to retrying each row
in that chunk individually. Rows that succeed on the individual retry clear
their dirty flag normally. Only the row that fails even alone stays dirty
and gets logged, diagnosable rather than silently stuck:

```js
for (const {t, ids, rows} of jobs){
  for (let i = 0; i < rows.length; i += 200){
    const chunk = rows.slice(i, i + 200);
    try {
      await sbBatchUpsert(t, chunk);
      chunk.forEach(r => _dirty[t].delete(r.id));
    } catch (e) {
      // A single bad row must not wedge the whole chunk. Retry per-row so
      // good rows still clear their dirty flag, only the offending row
      // stays dirty (diagnosable, not a silent full-chunk stall).
      for (const row of chunk){
        try { await sbBatchUpsert(t, [row]); _dirty[t].delete(row.id); }
        catch (e2){ poisoned.push({ table: t, id: row.id, error: e2.message }); }
      }
    }
  }
}
```

A poisoned row is never dropped. It stays in the dirty set, gets retried on
the very next flush (the debounced timer or the periodic belt-and-braces
`setInterval`), and surfaces in a toast plus a console error so the failure
is visible, not swallowed.

## 4. Queued deletes with a TTL

A delete that fails offline cannot just be forgotten, but it also cannot
retry forever if the record is legitimately gone from the server for other
reasons. Journal queues the delete to its own localStorage key with a
timestamp, retries opportunistically, and expires the queue entry after 7
days rather than retrying indefinitely:

```js
function _queueDelete(table, id){
  const l = JSON.parse(localStorage.getItem(PDEL_KEY) || '[]');
  if (!l.some(x => x.table === table && x.id === id)){
    l.push({ table, id, ts: Date.now() });
    localStorage.setItem(PDEL_KEY, JSON.stringify(l));
  }
}
async function flushPendingDeletes(){
  if (isLocalhostPreview() || !sbConfigured()) return;
  const l = JSON.parse(localStorage.getItem(PDEL_KEY) || '[]');
  const keep = [];
  for (const it of l){
    try {
      const r = await fetch(`${SB_URL}/rest/v1/${it.table}?id=eq.${it.id}`, { method:'DELETE', headers: SB_HDR });
      if (!r.ok) throw 0;
    } catch (e){
      if (Date.now() - it.ts < 7 * 864e5) keep.push(it); // 7-day TTL, then give up
    }
  }
  localStorage.setItem(PDEL_KEY, JSON.stringify(keep));
}
```

A periodic `setInterval` (Journal uses 60s) calls `flushPendingDeletes()` as
a belt-and-braces retry, covering the case where a delete fails and no
further dirty write ever fires the normal debounced flush again.

## 5. Timestamp reconciliation on load, and its known limitation

On load, local state renders immediately (localStorage-first, the UI never
waits on the network). Cloud state is then fetched and compared by
timestamp: if the newest cloud row is not newer than the local snapshot, the
cloud fetch is discarded outright. Otherwise, `mergeTable` combines them,
letting any row still in the dirty set win locally over the cloud copy:

```js
function mergeTable(cloudRows, localRows, dirtySet){
  if (!dirtySet || dirtySet.size === 0) return cloudRows;
  const byId = new Map(cloudRows.map(r => [r.id, r]));
  for (const id of dirtySet){
    const lr = localRows.find(r => r.id === id);
    if (lr) byId.set(id, lr);
  }
  return [...byId.values()];
}
```

**Documented limitation, do not silently "fix" this without a live backend
to test against:** the newest-cloud-row check is table-level, a single max
timestamp across the whole table, not per-row optimistic concurrency. A
single stale local row can currently win or lose against the whole cloud
table's state, rather than being resolved row-by-row. This is acceptable for
a single-user app where conflicting concurrent writers are rare, and is
flagged in the source as a `TODO` to revisit once there is a live backend to
validate a per-row fix against, not a gap to redesign blind.

## 6. Generalising away from Forex's globals

Journal hardcodes `DB` and `TABLES` as module-level globals. To reuse this
engine in another project without forking it, wrap the same logic behind a
small config object instead:

```js
function createSyncEngine(config){
  // config = {
  //   tables: ['trades'],
  //   storageKey: 'app_v1',
  //   restUrl, restHeaders,        // your backend's REST base + auth headers
  //   getRow(table, id),           // read one row from your own store
  //   allRows(table),              // read all rows from your own store
  //   applyRow(table, row),        // write a merged/cloud row into your store
  // }
  const DIRTY_KEY = config.storageKey + '_dirty';
  const PDEL_KEY  = config.storageKey + '_pendingdel';
  const _dirty = {};
  config.tables.forEach(t => _dirty[t] = new Set());
  // ...same functions as above, but reading config.tables / config.restUrl /
  // config.getRow / config.applyRow instead of the module-level DB/TABLES.
  return { markDirty, saveData, flushDirty: _flushDirty, loadData, flushPendingDeletes };
}
```

Every app that adopts this gets its own instance, its own storage keys, and
its own backend, with zero shared mutable state between them. This is the
only change needed to lift the pattern out of Journal, the rules in sections
1 to 5 above do not change.

## What was left out

- Forex's actual Supabase REST calls (`sbBatchUpsert`, `sbFetchAll`,
  `sbDelete`) are shown above only as the shape to follow, replace the
  fetch URL and payload mapping with your own backend's REST or RPC surface.
- Row-level optimistic concurrency (a real per-row `updated_at` compare
  instead of the table-level max) is called out in section 5 as a known
  gap, not shipped here, because it needs a live backend to validate against
  real conflict scenarios rather than being designed blind.

## 7. Boot-time snapshot-before-pull (Manpower Portal)

Manpower Portal's boot sequence adds a second loss-prevention layer on top of the five rules above: a local snapshot taken before the remote pull, not after, plus a lock screen that blocks editing while that pull is in flight. Extracted from `index.html`, the boot-lock overlay markup (lines 1108 to 1120) and the boot controller (lines 5433 to 5475 at time of writing: `showBootLock`, `hideBootLock`, `bootLockFailed`, and the `boot()` sequence that calls `takeDailySnapshot()` then `pullFromRemote()`).

### The two failure modes this prevents

**FM1, edit-race.** A user can start editing the moment the page paints, before the first pull has even finished. If a live edit and an in-flight pull are both writing to the same local state, whichever one lands last wins, silently discarding the other. The boot lock overlay covers the whole UI for the duration of the pull, so no edit can begin until the pull has resolved one way or the other.

**FM2, lossy-snapshot.** The daily safety snapshot used to run after the pull. If that pull was itself lossy (a partial or corrupted response overwriting good local state), the snapshot's only job, capturing a safe rollback point, would end up snapshotting the already-damaged state instead. Moving the snapshot ahead of the pull means there is always one point-in-time capture of local state as it stood before any remote data touched it, taken on every boot regardless of what the pull then does.

### Boot order

```js
(async function boot(){
  // 1) honour-system identity for the audit log, deferred slightly so the
  //    lock screen has time to paint first
  // 2) seed the sync URL on first visit
  // 3) snapshot LOCAL state, before any pull (fixes FM2)
  takeDailySnapshot();
  // 4) pull remote data behind the boot lock, so no edit can race it (fixes FM1)
  if (getSyncUrl()){
    showBootLock();
    try {
      const ok = await pullFromRemote({ allowSeed: true, reason: 'boot' });
      if (ok) hideBootLock();
      else bootLockFailed('Pull returned an error.');
    } catch (e) {
      bootLockFailed(e && e.message ? e.message : '');
    }
  }
})();
```

Order matters here on its own. Step 3 before step 4 is the entire fix for FM2, and wrapping step 4 in the lock screen is the entire fix for FM1. Reordering either one reopens the failure mode it closes.

### Lock screen behaviour

While the pull runs, `showBootLock()` opens a dialog (`role="dialog" aria-modal="true"`) over the whole app: a spinner, "Loading latest data", and a short explanation. It carries no buttons while the pull is in flight, nothing to interact with until the pull settles. On success, `hideBootLock()` removes it. On failure, `bootLockFailed(msg)` swaps the copy to explain what went wrong and reveals two actions:

- **Retry**, calls `pullFromRemote()` again behind the same lock
- **Continue offline**, calls `hideBootLock()` and lets the user proceed on whatever local state (plus the pre-pull snapshot) is already on the device

Neither option loses data. Retry re-runs the same guarded pull, Continue offline simply stops waiting on the network and falls back to local state, which the FM2 fix guarantees was snapshotted safely moments earlier.
