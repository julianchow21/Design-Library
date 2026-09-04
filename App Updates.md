# App Updates

Offline-capable shell (service worker) plus an update-ready pill that tells an
already-open tab a new deploy exists. Extracted from Kujira Collectibles v3.31
(`sw.js`, whole file, plus `features.js`, the shell-version poll IIFE).
Generalised: no Pokemon/TCG content, ids or names below, placeholders only.

Category: Flows. This is a systems pattern (service worker plus a page-level
poll), too large for a single gallery `<template>` card. A small live demo of
the poll/pill flow sits in `index.html` as the "Update Pill" card; this file
is the full pattern.

## 1. Two independent layers, solving two different problems

- **"Is the code fresh."** Solved entirely by the service worker's
  network-first HTML strategy (below). Anyone who loads or reloads the page
  while online gets the newest shell, silently, no user action.
- **"The user already has a stale tab open and has no reason to reload it."**
  A reload only happens on next navigation. The update pill exists purely to
  close that gap: poll for a change, then let the user choose to reload now.

Neither layer depends on the other. A page with no service worker at all can
still run the poll/pill against its own plain HTTP headers.

## 2. Service worker strategy (`sw.js`)

- **install**: `skipWaiting()` plus precache a `CORE`/`SHELL` list of exact
  request URLs, **including query strings**. Precache matching is exact-URL,
  a missing or mismatched `?v=` silently misses (see 8).
- **activate**: delete every cache key except the current versioned `CACHE`
  name (and any deliberately separate runtime cache, e.g. a fetched-art
  cache, explicitly excluded from the wipe), then `clients.claim()`.
- **fetch**: never intercept non-GET, never intercept cross-origin (an API,
  a CDN, a backend proxy), those pass straight to the network.
  - HTML/navigation requests: **network-first**. Fetch, and on success
    re-cache the fresh copy and return it. On failure, fall back to the
    cached shell HTML, then a plain offline `Response`.
  - Other same-origin static assets: **cache-first**. Return a cache hit
    immediately. On a miss, fetch, cache it if the response is ok, and on
    total failure fall back to a cache match (optionally `ignoreSearch`) or
    a `504`.
- Bump the `CACHE` version string whenever every client should drop its old
  shell entirely. Cheap and safe, the next `install` simply refetches it.

## 3. Update pill (complements the service worker, does not replace it)

- **Poll**: `HEAD` the page's own URL with `cache:'no-store'`, read `ETag` or
  `Last-Modified`. The **first** probe only records that stamp as a baseline,
  it never shows the pill. Only a **later** probe whose stamp differs shows
  it, so the very first load of a brand-new tab is never mistaken for a
  pending update.
- **When it runs**: on `window load`, and on `visibilitychange` back to
  `visible` (a backgrounded tab that regains focus). Throttle repeats behind
  a minimum interval (Collectibles uses 30 minutes) so a tab left open all
  day is not hammering `HEAD` requests.
- **Pill UI**: the whole pill is one click target that reloads the page,
  with a dedicated dismiss (✕) that calls `stopPropagation` so dismissing
  never also triggers a reload. Dismissing only hides it, it does not cancel
  the pending update, the next successful check (or the user's own next
  reload) still picks up the fresh code.
- **Silent failure everywhere**: offline, blocked, or a host that exposes
  neither header, every branch is a no-op inside a `try`/`catch` (or a
  rejected-promise `.catch`). This is a background probe the user never
  asked for, it must never surface an error state.

## 4. Cache-bust `?v=` discipline and version badge alignment

- Every versioned asset tag (`tokens.css?v=X`, any `lib/*.js?v=X`) must bump
  its query string on ship. That query bump is what actually busts a stale
  **browser** cache entry, the service worker cache name alone does not.
- The service worker's own precache list needs the **identical** query
  string for that same entry. A mismatch does not error, the request just
  falls through to network every time (quietly slower) or, worse, precaches
  a URL the running app never actually requests.
- The visible version badge (e.g. `v3.31 (14 Jul)`) must always match the
  deploy. Comparing a local copy's badge against the live badge is the
  manual sanity check that pairs with, never replaces, the automatic pill.
- Ship ritual: bump every asset `?v=`, the service worker's `CACHE`
  constant/precache entries, and the visible badge **together in the same
  edit**. A mismatch between any two of these means either a stale cache or
  a badge that is lying about what is actually live.

## 5. Integration steps for a new app

1. Add `sw.js` with the network-first HTML / cache-first static split above
   (or start from the App Starter variant, see 7, and add the split
   later if the app needs it).
2. Register it once, after load: `if ('serviceWorker' in navigator)
   window.addEventListener('load', () => navigator.serviceWorker
   .register('./sw.js'))`.
3. Add the update-pill IIFE (see the Update Pill card's script) so it HEADs
   the app's own root URL.
4. Style the pill on house tokens only. In a real app, position it `fixed`
   to a screen corner, the gallery card scopes it to the card body instead,
   purely so the demo cannot visually escape its own card.
5. Fold the `?v=` bump and the service worker's `CACHE` bump into the same
   ship-checklist step as the version badge bump, never three separate steps.

## 6. Teardown and kill-switch notes

- A service worker itself needs no per-page-load teardown, it has no
  in-memory state tied to any one tab by design.
- The only kill switch at this layer is coarse: bump `CACHE`. Every previous
  client drops its old shell on its next `activate`. There is no per-feature
  kill switch here (contrast the Launch Intro pattern's settings-toggle kill
  chain, that one is client-side JS state, this one is the network layer).
- To force-retire a service worker in an emergency, ship a page that runs
  `navigator.serviceWorker.getRegistrations().then(rs => rs.forEach(r =>
  r.unregister()))` then reloads. Treat this as a one-time flush: remove it
  again once every known client has run it, do not leave it shipping forever.
- The pill has no timers or animation frames to tear down, its only
  lifecycle risk is its own `visibilitychange` listener living past the
  page it was meant for. In a real single-page app there is exactly one
  instance for the app's whole life, so this is a non-issue there, it only
  matters in a multi-instance context like this gallery (handled in the
  pattern's own script via an `isConnected` check plus a `MutationObserver`).

## 7. The App Starter variant

`templates/App Starter/sw.js` is deliberately smaller: a single
`SHELL` array precached on `install`, one network-first `fetch` handler for
every same-origin GET (no HTML-vs-asset split), falling back to
`caches.match(req)` then `caches.match('./index.html')` on failure. No
`ETag`/`Last-Modified` poll and no update-pill UI ship with it at all.

The Update Pill pattern integrates with **either** variant unchanged: the
pill only ever `HEAD`s the page's own URL and compares one header, it has no
dependency on which caching strategy sits behind it. Wire it into a
starter-based app exactly as in step 5, no starter-specific change needed.
If this pattern is later promoted into the starter itself (see the library's
Promotion rule), the natural next step is adding Collectibles' HTML/asset
split plus the pill to the starter's `sw.js`/`index.html`, neither ships
there today.

## 8. Verification gotchas

- A precache entry with a missing or mismatched `?v=` query misses
  `caches.match` silently, no error, just a permanent fall-through to
  network. Diff the exact request URL (query included) against the precache
  list entry by entry after any asset rename or version bump.
- Local `file://` preview cannot register a service worker at all (it needs
  `https` or `localhost`), and `fetch()` cannot `HEAD` a `file://` URL
  either, so the pill's poll silently no-ops there by design. That is not a
  bug, do all real verification against the deployed app or a `localhost`
  server, the same rule already used for Supabase writes.
- After bumping `CACHE`, confirm the old cache key is actually gone: check
  DevTools Application > Cache Storage, or run `caches.keys()` in the
  console. A leftover stale key alongside the new one means the `activate`
  handler's filter condition is wrong.
- `res.headers.get('ETag')` can come back null even same-origin, if the
  host does not set it (a CDN or proxy in front of the real host can strip
  either header). Test the pill against the real deployed host, never an
  assumption.
- A `visibilitychange` check firing while an earlier `load`-time check is
  still in flight can race, both only ever compare against the same
  baseline variable so it is harmless today, worth remembering before
  adding anything stateful to the check later.

Related: Offline Shell.md in this folder covers the same service-worker strategy as shipped in Journal, without the update-ready UI. Candidate for unification, see Backlog.md.
