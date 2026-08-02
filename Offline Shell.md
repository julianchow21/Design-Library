# Offline Shell

A minimal service worker app shell: precache the static shell on install,
serve network-first with a cache fallback so a deploy is picked up
immediately when online, and never touch cross-origin or non-GET requests
so a live backend call is never served stale. Extracted from Kujira
Forex's `sw.js` in full (26 lines, the whole file). Generalised: swap
`kjr-journal-v3` and the `SHELL` list for your own app's assets.

Category: Flows. This is a systems pattern (a separate worker script,
registration, and cache lifecycle), too large for a single gallery
`<template>` card, and it genuinely cannot run from a live demo card at all,
service workers require a real HTTP(S) origin and never register from
`file://`. The "Offline Shell" card in `index.html` is diagrammatic only,
labelled HTML/CSS showing the request flow, no real network interception.
This file is the full pattern.

## 1. Install: precache the shell

```js
const CACHE = 'kjr-forex-v2'; // bump this string on every ship
const SHELL = [
  './', './index.html', './icon.svg', './manifest.webmanifest',
  './lib/theme-init.js?v=1.0', './lib/kjr-format.js?v=1.1', './lib/kjr-calendar.js?v=1.0'
];

self.addEventListener('install', e => {
  e.waitUntil(
    caches.open(CACHE).then(c => c.addAll(SHELL)).then(() => self.skipWaiting())
  );
});
```

`SHELL` lists every static file the app needs to boot offline, including
each vendored `lib/` script with its exact `?v=` query string, cache
matching is an exact URL match, a mismatched query falls through to network
or hits a stale entry. `self.skipWaiting()` makes the new worker take over
immediately rather than waiting for every open tab to close first.

## 2. Activate: drop every stale cache version

```js
self.addEventListener('activate', e => {
  e.waitUntil(
    caches.keys()
      .then(keys => Promise.all(keys.filter(k => k !== CACHE).map(k => caches.delete(k))))
      .then(() => self.clients.claim())
  );
});
```

Any cache key that is not the current `CACHE` string gets deleted, this is
what makes the version bump ritual (section 4) actually take effect:
without it, an old versioned cache lingers forever and a client can still
be served from it. `self.clients.claim()` lets the new worker start
controlling already-open tabs immediately, instead of only new navigations.

## 3. Fetch: network-first, cache fallback, and the exclusions that matter most

```js
self.addEventListener('fetch', e => {
  const req = e.request;
  // Only handle same-origin GETs. Never touch Supabase / API calls, they
  // must always hit the network so the user never sees stale cloud data.
  if (req.method !== 'GET' || new URL(req.url).origin !== location.origin) return;

  e.respondWith(
    fetch(req)
      .then(res => {
        const copy = res.clone();
        caches.open(CACHE).then(c => c.put(req, copy)).catch(() => {});
        return res;
      })
      .catch(() => caches.match(req).then(r => r || caches.match('./index.html')))
  );
});
```

Two things here carry the whole safety story:

- **The exclusion guard runs first, before any caching logic.** `req.method
  !== 'GET'` and an origin mismatch both bail out with a bare `return`,
  leaving the request completely untouched by the service worker. A
  `POST`/`PATCH`/`DELETE` to a backend, or any cross-origin call (a REST
  API, a CDN), is never intercepted, never cached, and never risks serving
  a stale response for data that must always be live.
- **Network-first, not cache-first.** Every same-origin GET tries the
  network first and updates the cache with whatever comes back. Only a
  genuine `fetch()` rejection (offline, DNS failure) falls back to the
  cache, and if even that misses, to `index.html` itself (an SPA shell
  fallback, so a deep offline reload still boots the app shell rather than
  showing a browser error page). This means an online user always gets the
  latest deploy immediately, cache-first designs trade that away for a
  faster repeat load and need an explicit "new version available" prompt
  to avoid serving yesterday's code indefinitely, network-first avoids
  needing that prompt at the cost of one network round-trip per load.

## 4. Registration and the version bump ritual

```js
if ('serviceWorker' in navigator && location.protocol !== 'file:'){
  navigator.serviceWorker.register('sw.js').catch(() => {});
}
```

Guarded by `location.protocol !== 'file:'` because registration always
fails from `file://`, this keeps a local file-opened preview silent instead
of throwing. The `.catch(() => {})` means a registration failure (unsupported
browser, disabled in settings) degrades to "just works online, no offline
support", never a hard error.

**Bump `CACHE` on every ship.** This is the entire mechanism that gets a new
build in front of returning users: the next `activate` event on a bumped
worker deletes every old-versioned cache key (section 2) and the browser's
own install/waiting/activate lifecycle swaps the worker over. Forgetting
the bump means the new deploy's files are fetched network-first anyway (so
online users still see it), but an offline user, or the fallback path on a
flaky connection, can still be served the previous version's cached shell
until the version string changes.

## What was left out

- No offline write queue or background sync registration, this shell is
  read-side caching only. Forex's actual offline write durability comes
  from the separate localStorage-first data layer (see `Sync Engine.md`),
  not from the service worker.
- No dedicated long-lived runtime cache bucket (contrast `Launch Intro.md`'s
  `app-intro-art` cache), because this app shell has no equivalent
  fetched-art asset class to protect from the version-bump wipe in section 2.
  If a future feature adds one, exclude its cache name from the `activate`
  cleanup filter explicitly, the same way `Launch Intro.md` documents.
