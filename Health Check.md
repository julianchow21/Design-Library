# Health Check

Data-integrity auditor: a registry of small, independent checks scans the
app's own in-memory tables and returns a flat list of findings, each tagged
with a severity, an optional one-line fix hint, and the specific rows it
came from. Extracted from Kujira Collectibles v3.31 (`app.js`, lines 3387
to 3684: `runHealthCheck`, the fingerprint-based ignore list,
`healthGotoRow`). Generalised below: no Collectibles tables, no card or
grading fields.

Category: Flows. The registry, fingerprint model and row-jump plumbing
together are a systems pattern (a data-layer sweep plus a modal-scale UI),
too large for a single gallery `<template>` card. The "Health Check" card
in `index.html` runs the same registry shape over two small invented
tables with seeded problems, no Collectibles data. This file is the full
pattern.

## 1. The registry shape

Each check is a small function, not a config object: `(ctx, out) => void`.
It reads whatever it needs from `ctx` and pushes zero or more findings onto
`out`. Nothing central needs to know what a check inspects, adding one is
"append a function to the array":

```js
function pushFinding(list, sev, area, message, fix, details){
  list.push({ sev, area, message, fix: fix || '', details: details || [] });
}

const CHECKS = [
  (ctx, out) => {
    const dups = findDuplicateIds(ctx.orders);
    if (dups.length) pushFinding(out, 'fail', 'Orders',
      `${dups.length} order id(s) appear more than once.`,
      'Check the changelog for the timestamp, delete or renumber one.', dups);
  },
  // ...one function per rule, independent, no shared state between them
];

function runHealthCheck(ctx){
  const out = [];
  CHECKS.forEach(check => check(ctx, out));
  return out;
}
```

`sev` is one of `'fail' | 'warn' | 'info'`. The source additionally hides
`warn` from its own modal by design (a business decision in its own
comment: existing warnings had already been triaged once, only the ignore
count in the footer surfaces them again), a project-specific choice, not
part of the reusable shape. The generalised version here, and the gallery
demo, show all three, decide per project whether `warn` should be visible
by default or start pre-filtered.

## 2. Cross-table consistency checks

The most valuable class of check reads more than one table and confirms
they agree. The two source examples: a sale whose linked inventory item is
not actually marked Sold (revenue double-counted), and a sale that points
at an inventory id which no longer exists (orphaned reference). Same
shape, generalised:

```js
(ctx, out) => {
  const mismatched = [];
  ctx.payments.forEach(p => {
    const order = ctx.orders.find(o => o.id === p.orderId);
    if (order && order.status !== 'Paid') mismatched.push({ label: describe(order), key: order.key });
  });
  if (mismatched.length) pushFinding(out, 'fail', 'Payments',
    `${mismatched.length} order(s) have a payment recorded but are not marked Paid.`,
    'Open the order and set its status to Paid.', mismatched);
}
```

A row that a check reports must carry a **stable, collision-proof key**,
never the business id the check itself might be proving is duplicated (see
section 1's example, two rows can legitimately share the same visible
"order id", the internal list key must not). Generate or carry a separate
synthetic key per row for this reason, never key a list by the field a
check exists to distrust.

## 3. Fingerprint dismissal model

Persisted to a dedicated localStorage key. Each finding hashes to `sev +
'|' + area + '|' + stem(message)`, where `stem()` makes the hash tolerant
of a changing count so "3 orders have X" and "1 order has X" still dismiss
as the same underlying issue:

```js
function fingerprint(f){
  const stem = f.message.toLowerCase()
    .replace(/^\d+\s*/, '')     // drop the leading count
    .replace(/\b\d+\b/g, '#')   // collapse any other numbers
    .split('.')[0]              // first sentence only
    .trim()
    .slice(0, 80);
  return `${f.sev}|${f.area}|${stem}`;
}
```

Dismissing adds the fingerprint to a `Set`, persisted as a JSON array.
Every future run filters findings against this set before display, so a
dismissed issue stays hidden until the user explicitly restores it (a
single "Restore ignored" action clears the whole set, there is no
per-finding undo, matching the source). Because the hash is content-derived
rather than index-derived, it survives the run order or the exact wording's
numbers changing, but not a genuine wording change to the check's own
message (intentional, a materially different message is a different
issue).

## 4. Linking a finding back to its row

Each finding's `details` array carries `{ label, key }` per offending row.
`key` is `null` when there is nothing to jump to (an orphaned reference,
the other side of the link is genuinely gone). The UI renders a "View"
action only when `key` is present:

```js
function gotoRow(key){
  const row = document.querySelector(`[data-row-key="${key}"]`);
  if (!row) return;
  row.scrollIntoView({ behavior: 'smooth', block: 'center' });
  row.classList.add('is-hit');
  setTimeout(() => row.classList.remove('is-hit'), 1800);
}
```

In the source, this also closes the health modal and switches the app to
the row's own tab first (`showPage`), since the row can live on a
different screen entirely. A single-page or single-panel context (like the
gallery demo) can skip the navigation step and jump straight to the
highlight.

## 5. When to run

- **On demand**, from a menu action, the common case, cheap enough (a pass
  over in-memory arrays, no network) to run any time.
- **After a sync merge**, a good second trigger, a cloud merge is exactly
  when timestamp reconciliation or a dirty-row edge case is most likely to
  have just created an inconsistency worth surfacing immediately.
- Not on every keystroke or every render. A check registry over any
  non-trivial dataset should stay an explicit action, not an implicit one,
  so the user is never surprised by a modal appearing mid-edit.

## 6. Integration steps

1. Define your row and table shapes, then write one small function per
   rule, append to `CHECKS`, nothing else changes.
2. Give every check a stable `area` label (used in both the grouping
   header and the fingerprint), keep it short and consistent across checks
   that touch the same feature.
3. Ensure every row your checks can reference carries a synthetic,
   collision-proof key (section 2), add one if the natural id cannot be
   trusted.
4. Wire a single "Run check" entry point that builds `ctx` from your real
   data and calls `runHealthCheck(ctx)`, render the result, offer Ignore
   and Restore ignored actions backed by the fingerprint set.
5. Decide per project whether `warn` should show by default (section 1).

## 7. Verification gotchas

1. **A dismiss test needs two full runs, not one.** Confirming a finding's
   Ignore button hides it is not enough, re-run the check and confirm it
   stays hidden, that is the entire point of persisting a fingerprint
   rather than an in-memory-only filter.
2. **Test the collision case directly.** If two rows can legitimately share
   a display id (the very thing one check might flag), verify the row-jump
   still lands on the correct one, this is only possible if the jump key is
   never the colliding business id (section 2).
3. **An orphaned reference must degrade to "no link", never a dead click or
   a thrown error.** Test the specific finding whose `key` is `null`.
4. **A stemmed fingerprint must not be so aggressive it merges two different
   real issues.** Slicing to a fixed length and collapsing numbers is tuned
   for "same issue, different count". If two genuinely different checks in
   the same area produce similarly-worded messages, a too-short slice can
   make them collide, test with the real message set, not just the demo's,
   before trusting the stem length as-is.

## What was left out

- The source's date-canonicalisation and eBay currency-drift checks are
  domain-specific one-offs, not reused here. The two kept as worked
  examples (duplicate id, cross-table mismatch) are the ones with the most
  reusable shape.
- Bulk auto-fix actions attached to a specific finding (the source's
  "Auto-fix all dates" and "Backfill holding period" buttons) are a
  reasonable extension, not part of the core registry, a fix action just
  needs its own button rendered next to that finding and re-running the
  check afterwards.
