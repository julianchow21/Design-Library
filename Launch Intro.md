# Launch Intro

Full-screen launch splash that boots the app behind it, then progressively
enhances from a pure-CSS greeting into a WebGL scene, with a hard ceiling and
an instant skip at any moment. Extracted from Kujira Collectibles v3.23
(`features.js`, the `Launch intro` IIFE, plus `index.html` and `styles.css`
support and `sw.js` precache entries). Generalised: no Pokemon/TCG content,
ids or names below, placeholders only.

Category: Flows. This is a systems pattern (WebGL, service worker, asset
pipeline), too large for a single gallery `<template>` card. A small live
demo of the CSS-only fallback tier sits in `index.html` as the "Launch
Intro" card; this file is the full pattern.

## 1. Architecture

- `#intro` is a fixed, full-viewport div painted via a **critical inline
  style directly in the HTML**, not waiting for the external stylesheet, so
  the brand backdrop is on screen at 0ms.
- z-index above every other overlay in the app (Collectibles used
  `100000`, above its prior highest overlay at `9999`).
- The app boots normally underneath it. DB init, cloud sync, whatever the
  app needs to do on load, all happen concurrently. The intro never gates
  or blocks the app.
- The div's own markup plus its CSS in the stylesheet is a **complete,
  standalone greeting on its own**. Everything past that is progressive
  enhancement layered on top. Any failure anywhere just leaves the CSS
  layer already on screen, never a broken half-loaded scene.
- The WebGL layer loads via a **lazy dynamic `import()` of a vendored
  module** (not a CDN, so it survives offline/service-worker precache).
  The main bundle never pays for it, and an import failure degrades
  gracefully to the CSS layer.
- **Hard ceiling**: an independent force-remove timer guarantees the
  overlay cannot outlive a fixed budget (Collectibles used 9.5s) no matter
  what state the scene or asset loading is in.
- Tap, click and keydown at any moment trigger an immediate skip, which
  converges on the same teardown as the natural end.

## 2. Kill-switch chain and CSS fallback

Checked in order, first match wins, each falls back to the same CSS-only
greeting:

1. Settings toggle off (user preference, e.g. `localStorage`)
2. `prefers-reduced-motion: reduce`
3. No WebGL support
4. Dynamic import of the scene library failed

Every kill-switch path shows the CSS greeting for a short fixed duration
then dissolves. A mid-scene skip does **not** hard-cut either: it triggers
an accelerated version of the scene's own natural ending beat (compressed
timing), so a skip still lands the intended visual beat, just fast, never
a jarring cut.

Wrap any settings-toggle storage read/write in try/catch (private
browsing can throw or silently no-op) and default to enabled if unreadable.

## 3. Teardown checklist

Must be idempotent (single "already removed" flag) since it can be
triggered by more than one race at once (natural end, skip, hard ceiling,
caught error):

- cancel the `requestAnimationFrame` loop
- remove every listener added (skip on pointerdown/keydown, pointermove
  parallax, resize)
- traverse the scene graph disposing every geometry, every material (and
  any texture/map on it), and call `.dispose()` on any `InstancedMesh`
- `renderer.dispose()`, then `forceContextLoss()` if available, then
  remove the canvas element from the DOM
- clear every pending timer (kill-switch dissolve, hard ceiling, ending tail)
- revoke every object URL created from a loaded blob
- null every module-scope reference to scene/camera/renderer/mesh objects
- remove the overlay div itself from the DOM

Debug hook pattern: expose a small `window.__xDebug` object, safe to call
before start, mid-run, and after teardown:

- a getter for the current state-machine value
- one or two "probe" functions that project a world-space object through
  the camera and back through the renderer, to measure its actual
  on-screen size/position (the "did the important asset land where and how
  large it should" QA check)

## 4. Asset pipeline patterns

- All remote imagery goes through a **dedicated Cache API bucket**, opened
  from window/page context (`caches.open('app-intro-art')`), a distinct
  cache name the service worker's install/precache never touches and its
  activate cleanup must explicitly exclude (see 5).
- Pattern: match first, fetch and `put` only on a genuine miss, then
  `blob()` then `URL.createObjectURL()` for use as a texture source.
  Track every created object URL for later revocation.
- **Memoisation rule**: only cache a genuine miss that resolved via a
  successful fetch. Never cache a transient failure (network error,
  non-ok response) as if it were a permanent absence, or one offline blip
  poisons every future session with a false "no art available".
- **Featured content array**: a small, hand-picked list of remote ids,
  each verified once against the live source (identity and image both
  checked), always attempted first, before any data-derived picks.
- **Data-driven selection**: pull candidates from the app's own data,
  excluding anything already in the featured list, weighted or sorted by
  a meaningful signal (e.g. quantity-desc, then a secondary tiebreaker),
  capped to a small pool, with a further per-record copy cap so one
  heavily-weighted record can't flood the scene.
- Every remote fetch (featured or data-derived) is wrapped so a failure
  resolves to null/empty and the visual simply has a gap at that slot,
  never a crash, never a stalled promise.

## 5. Service worker integration

- The dynamic-import URL for the vendored scene library needs an
  **exact-match entry in the service worker's precache list, including its
  query string** (e.g. a cache-busting `?v=X.Y`). Cache-first matching is
  an exact URL match: a mismatched or missing query on either side means
  the request falls through to network, or worse, hits a stale cache
  entry.
- A vendored library's own internal relative imports (e.g. a core module
  it loads itself with a fixed path) usually carry no query of their own,
  so precache that one without a query too. Bump the outer file's query
  and the service worker's own cache-version constant together as one
  ritual.
- **Cache version bump ritual**: bump the service worker's own version
  constant whenever core app files change, so every client drops its
  stale shell. On `activate`, delete every cache key except the current
  version **and** except the dedicated intro-art bucket, a long-lived
  runtime cache, not a versioned precache. Forgetting the exclusion wipes
  the whole art cache on every deploy.
- Cross-origin asset requests are left alone by the service worker's
  fetch handler entirely (early-return on origin mismatch). The intro's
  own cache-bucket logic in page context does the caching for those
  instead, fetched with an explicit `cors` mode so the response is
  readable (an opaque no-cors response can't be usefully cached or read
  back as a texture anyway).

## 6. Scene construction notes

- Fog and palette pulled from **brand tokens**, but as a deliberately
  **fixed, theme-invariant** gradient/colour (literal values, not the
  app's live light/dark theme tokens). A splash is commonly designed to
  read the same regardless of the user's theme. Document this as an
  intentional exception to "tokens only", not an oversight, if the target
  project's pattern-library conventions otherwise mandate tokens
  everywhere.
- Repeated filler geometry (many identical small quads) built as a single
  `InstancedMesh`, one draw call for the whole field regardless of
  instance count, never N separate meshes.
- Any transparent mesh should use `FrontSide`, not `DoubleSide`. On
  three.js r185+, a transparent `DoubleSide` mesh costs 2 draw calls (back
  face then front face) versus 1 for `FrontSide`. Ordinary backface
  culling is enough for geometry to disappear cleanly with no visible pop
  once the camera passes it.
- Cap device pixel ratio (e.g. to 1.75) rather than leaving it uncapped,
  to bound render cost on high-DPI devices.
- Keep an explicit running draw-call budget (fixed elements plus capped
  variable elements) and size instance/copy caps so worst-case load still
  lands comfortably under it.
- **Compose aspect-scaled, never with fixed world positions tuned to one
  aspect ratio.** Derive any on-screen target size or position (e.g.
  solving a plane's world width from its fixed depth plus the camera's
  current vertical FOV and aspect) at the moment it's needed, so a target
  framing (e.g. "fills X% of frame width, centred") holds across desktop
  and narrow mobile portrait automatically, no separate hard-coded
  per-aspect correction.

## 7. Verification gotchas

Hard-won, stated as rules:

1. **A cache-first service worker serves stale code to test browsers.**
   Files changed on disk will not show up in an already-registered test
   session. Always test in a fresh/incognito context, or explicitly
   unregister/bypass the service worker for that run, or you are QAing
   yesterday's build.
2. **The canvas's own CSS fade-in poisons early visual captures.** The
   canvas fades from opacity 0 to 1 once the first frame renders, so a
   screenshot taken right at load captures a transparent or half-faded
   canvas, not the real composition. Let a natural run play out before
   judging visuals, never capture on the first frame.
3. **A 0x0-viewport backgrounded pane poisons WebGL composition
   judgements.** A browser pane/tab that boots pages at a 0x0 viewport
   while backgrounded breaks every aspect-driven size/position
   calculation in the scene. Never judge a WebGL scene's composition from
   a backgrounded preview pane. Use a real-viewport, foregrounded browser
   session for any visual claim about the 3D layer.
4. **Top-layer popovers paint above any z-index.** A native `popover`
   attribute element (or anything similarly promoted to the top layer)
   renders above literally everything else on screen, including an
   overlay meant to be the highest thing on screen. An app's own
   notification/toast system built on `popover` cannot be out-z-indexed
   by the intro. The overlay or the app must actively defer or queue those
   notifications while the overlay is up, never try to fight it with
   z-index.
   - *Status: a toast-deferral queue for this exact problem is planned for
     Collectibles v3.24. This section will be completed with the concrete
     mechanism once that change ships. Not yet captured here.*

## 8. Code skeleton

Generic placeholders throughout: rename the `intro` namespace prefix to
avoid global collisions, replace `FEATURED_IDS` and `getDataCandidates()`
with your own verified content and data source, replace asset paths with
your own vendored library and brand mark.

### Overlay markup (`index.html`)

```html
<!-- Launch intro: critical inline style so the brand backdrop paints
     before the stylesheet loads. This div plus its CSS is the complete
     fallback greeting if the scene never runs (kill-switch, reduced
     motion, no WebGL, import failure). Controller lives in your JS.
     Removed from the DOM on teardown. -->
<div id="intro" style="position:fixed;inset:0;z-index:100000;background:linear-gradient(160deg,#12101F 0%,#2E2752 100%);display:flex;align-items:center;justify-content:center;overflow:hidden">
  <div id="intro-stage"></div>
  <div id="intro-word"><img id="intro-word-icon" src="./assets/brand-mark.png" alt="Your brand" /></div>
  <div id="intro-skip">tap to skip</div>
</div>
```

### CSS fragment (`styles.css`)

```css
/* Fixed brand gradient regardless of theme (deliberately theme-invariant
   splash), literal hex, never the app's --bg/--text tokens. z-index above
   every existing overlay. */
#intro{position:fixed;inset:0;z-index:100000;background:linear-gradient(160deg,#12101F 0%,#2E2752 100%);display:flex;align-items:center;justify-content:center;overflow:hidden;opacity:1;transition:opacity 0.25s ease,transform 0.25s ease}
#intro.intro-out{opacity:0;transform:scale(1.03);pointer-events:none}
#intro-stage{position:absolute;inset:0}
#intro-stage canvas{display:block;width:100%!important;height:100%!important;opacity:0;transition:opacity 0.9s ease}
#intro-stage canvas.intro-in{opacity:1}
/* Hidden by default, the scene carries its own brand beat. Shown only on
   a kill-switch path. */
#intro-word{position:relative;display:flex;align-items:center;justify-content:center;user-select:none;opacity:0;transition:opacity 0.4s ease}
#intro-word.intro-word-show{opacity:1}
#intro-word-icon{width:64px;height:64px;animation:introIconPulse 2.8s ease-in-out infinite}
#intro-skip{position:absolute;left:0;right:0;bottom:calc(28px + env(safe-area-inset-bottom));text-align:center;font-size:11px;letter-spacing:0.12em;text-transform:uppercase;color:#B0AAC8;opacity:0.35;animation:introHint 3.2s ease-in-out 0.6s infinite;user-select:none}
@keyframes introIconPulse{0%,100%{opacity:0.85}50%{opacity:1}}
@keyframes introHint{0%,100%{opacity:0.35}50%{opacity:0.75}}
@media(prefers-reduced-motion:reduce){
  #intro{transition:opacity 0.2s linear}
  #intro.intro-out{transform:none}
  #intro-word-icon{animation:none}
  #intro-skip{animation:none;opacity:0.55}
}
```

### JS skeleton (fenced IIFE, near end of your app's script)

```js
/* ===== Launch intro =====
   Self-contained IIFE, does not touch the rest of the app. Plays a scene
   on launch while normal app init runs behind it (never blocks the app).
   The #intro div and its CSS are the fallback greeting on their own,
   everything below is progressive enhancement, any failure here just
   falls back to the CSS layer already on screen.
   Kill-switch order: settings toggle off, prefers-reduced-motion, no
   WebGL, scene-library import failure. Hard ceiling: force-removed
   regardless, at HARD_CEILING_MS. Tap/key ends it early via the same
   teardown. Debug: window.__introDebug.info(). */
(function () {
  var INTRO_KEY = 'app_intro_enabled';
  var KILLSWITCH_FADE_MS = 1200;
  var DISSOLVE_MS = 260;
  var HARD_CEILING_MS = 9500;

  var introState = 'idle';
  var introKillSwitch = null;
  var _renderer = null, _scene = null, _camera = null, _raf = null;
  var _introTimers = [];
  var _introObjectUrls = [];
  var _onPointerMove = null, _onResize = null;
  var _ending = false, _removed = false;

  // ── Settings toggle ──
  function introEnabled() {
    try { return localStorage.getItem(INTRO_KEY) !== 'false'; } catch (e) { return true; }
  }

  // ── Debug hook, always safe to call ──
  window.__introDebug = {
    get state() { return introState; },
    info: function () {
      return {
        state: introState, killSwitch: introKillSwitch,
        cameraZ: _camera ? _camera.position.z : null,
        drawCalls: _renderer ? _renderer.info.render.calls : 0,
        removed: _removed
      };
    }
  };

  // ── Asset pipeline: cache bucket, memoise real misses only ──
  async function introCachedImage(url) {
    try {
      var cache = await caches.open('app-intro-art');
      var res = await cache.match(url);
      if (!res) {
        var fresh = await fetch(url, { mode: 'cors' });
        if (!fresh || !fresh.ok) return null; // transient failure - never cached
        await cache.put(url, fresh.clone());
        res = fresh;
      }
      var blob = await res.blob();
      var objUrl = URL.createObjectURL(blob);
      _introObjectUrls.push(objUrl);
      return objUrl;
    } catch (e) { return null; }
  }
  function introLoadTexture(THREE, objectUrl) {
    return new Promise(function (resolve) {
      if (!objectUrl) { resolve(null); return; }
      var img = new Image();
      img.onload = function () {
        try {
          var tex = new THREE.Texture(img);
          tex.needsUpdate = true;
          resolve(tex);
        } catch (e) { resolve(null); }
      };
      img.onerror = function () { resolve(null); };
      img.src = objectUrl;
    });
  }

  // ── Replace with your own verified remote ids, always attempted first ──
  var FEATURED_IDS = ['featured-id-1', 'featured-id-2'];
  async function introBuildFeatured(THREE) {
    var results = await Promise.all(FEATURED_IDS.map(async function (id) {
      try {
        var url = await introResolveContentUrl(id); // your own lookup, generic
        var objUrl = await introCachedImage(url);
        var tex = await introLoadTexture(THREE, objUrl);
        return tex ? { texture: tex, copies: 1, featured: true } : null;
      } catch (e) { return null; }
    }));
    return results.filter(Boolean);
  }
  // ── Replace with your own data source, excluding anything in FEATURED_IDS,
  // sorted by a meaningful weight (e.g. quantity-desc), capped, per-record
  // copy cap so one record can't flood the scene ──
  async function introBuildDataCandidates(THREE) {
    var picked = getDataCandidates().slice(0, 14); // your own cap
    var results = await Promise.all(picked.map(async function (rec) {
      try {
        var objUrl = await introCachedImage(rec.imageUrl);
        var tex = await introLoadTexture(THREE, objUrl);
        return tex ? { texture: tex, copies: Math.min(3, rec.weight || 1), featured: false } : null;
      } catch (e) { return null; }
    }));
    return results.filter(Boolean);
  }
  function getDataCandidates() { return []; }       // stub: your app's own records
  function introResolveContentUrl(id) { return id; } // stub: your own id-to-url lookup

  function introHasWebGL() {
    try {
      var c = document.createElement('canvas');
      return !!(window.WebGLRenderingContext && (c.getContext('webgl2') || c.getContext('webgl')));
    } catch (e) { return false; }
  }

  // ── Scene build. Throws bubble to main's catch. ──
  async function introBuildScene(THREE, stageEl) {
    var pair = await Promise.all([
      introBuildFeatured(THREE).catch(function () { return []; }),
      introBuildDataCandidates(THREE).catch(function () { return []; })
    ]);
    var entries = pair[0].concat(pair[1]);
    if (_ending) return; // skipped while assets were loading

    var scene = new THREE.Scene();
    scene.fog = new THREE.Fog(0x12101F, 8, 48); // brand-toned, theme-invariant on purpose
    var camera = new THREE.PerspectiveCamera(55, stageEl.clientWidth / stageEl.clientHeight, 0.1, 100);
    var renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true, powerPreference: 'low-power' });
    renderer.setPixelRatio(Math.min(window.devicePixelRatio || 1, 1.75)); // DPR cap
    renderer.setSize(stageEl.clientWidth, stageEl.clientHeight);
    stageEl.appendChild(renderer.domElement);
    _renderer = renderer; _scene = scene; _camera = camera;

    // Repeated filler geometry: one InstancedMesh, one draw call regardless
    // of instance count.
    var fillerGeo = new THREE.PlaneGeometry(1, 1.4);
    var fillerMat = new THREE.MeshBasicMaterial({ transparent: true, side: THREE.DoubleSide, depthWrite: false, fog: true });
    var fillerMesh = new THREE.InstancedMesh(fillerGeo, fillerMat, 25);
    // ... position instances via dummy.updateMatrix()/setMatrixAt per your own layout ...
    scene.add(fillerMesh);

    // Real content entries: FrontSide only (not DoubleSide) - 1 draw call
    // each on r185+, ordinary backface culling still hides them cleanly
    // once the camera passes.
    entries.forEach(function (entry) {
      for (var i = 0; i < entry.copies; i++) {
        var mat = new THREE.MeshBasicMaterial({ map: entry.texture, transparent: true, side: THREE.FrontSide, depthWrite: false, fog: true });
        var mesh = new THREE.Mesh(fillerGeo, mat);
        // ... position aspect-scaled to your own composition, never a fixed
        // world position tuned at one aspect ...
        scene.add(mesh);
      }
    });

    function onResize() {
      if (!_renderer || !_camera) return;
      var w = stageEl.clientWidth, h = stageEl.clientHeight;
      if (!w || !h) return;
      _camera.aspect = w / h;
      _camera.updateProjectionMatrix();
      _renderer.setSize(w, h);
    }
    window.addEventListener('resize', onResize);
    _onResize = onResize;

    var canvasFadedIn = false;
    function tick() {
      _raf = requestAnimationFrame(tick);
      try {
        // ... your own choreography: dolly/camera move, ending beat, etc ...
        renderer.render(scene, camera);
        if (!canvasFadedIn) { canvasFadedIn = true; renderer.domElement.classList.add('intro-in'); }
      } catch (e) {
        introForceRemove(e);
      }
    }
    introState = 'scene';
    tick();
  }

  function introSkip() {
    if (_ending) return;
    introState = 'skipped';
    introDissolve(); // in a fuller build: trigger an accelerated ending beat if a scene is live
  }
  function introDissolve(ms) {
    if (_ending) return;
    _ending = true;
    var el = document.getElementById('intro');
    if (el) el.classList.add('intro-out');
    _introTimers.push(setTimeout(introTeardown, ms || DISSOLVE_MS));
  }
  function introShowWord() {
    var el = document.getElementById('intro-word');
    if (el) el.classList.add('intro-word-show');
  }
  function introForceRemove(err) {
    if (err) { try { console.warn('[intro] stopped:', err); } catch (e2) {} }
    _ending = true;
    introTeardown();
  }

  // ── Teardown: idempotent, safe from any race (natural end, skip, hard
  // ceiling, caught error) ──
  function introTeardown() {
    if (_removed) return;
    _removed = true;
    introState = 'done';
    _introTimers.forEach(function (id) { clearTimeout(id); });
    _introTimers.length = 0;
    if (_raf) { cancelAnimationFrame(_raf); _raf = null; }
    if (_onPointerMove) { window.removeEventListener('pointermove', _onPointerMove); _onPointerMove = null; }
    if (_onResize) { window.removeEventListener('resize', _onResize); _onResize = null; }
    window.removeEventListener('pointerdown', introSkip);
    window.removeEventListener('keydown', introSkip);
    try {
      if (_scene) {
        _scene.traverse(function (obj) {
          if (obj.isInstancedMesh && typeof obj.dispose === 'function') obj.dispose();
          if (obj.geometry) obj.geometry.dispose();
          if (obj.material) {
            var mats = Array.isArray(obj.material) ? obj.material : [obj.material];
            mats.forEach(function (m) { if (m.map) m.map.dispose(); m.dispose(); });
          }
        });
      }
      if (_renderer) {
        _renderer.dispose();
        if (_renderer.forceContextLoss) _renderer.forceContextLoss();
        if (_renderer.domElement && _renderer.domElement.parentNode) _renderer.domElement.parentNode.removeChild(_renderer.domElement);
      }
    } catch (e3) { /* best-effort disposal, never block removal */ }
    _introObjectUrls.forEach(function (u) { try { URL.revokeObjectURL(u); } catch (e4) {} });
    _introObjectUrls.length = 0;
    var el = document.getElementById('intro');
    if (el && el.parentNode) el.parentNode.removeChild(el);
    _scene = null; _camera = null; _renderer = null;
  }

  async function introMain() {
    var introEl = document.getElementById('intro');
    if (!introEl) return; // markup missing, nothing to control
    var stageEl = document.getElementById('intro-stage');

    window.addEventListener('pointerdown', introSkip, { passive: true });
    window.addEventListener('keydown', introSkip);
    _introTimers.push(setTimeout(function () { introForceRemove(); }, HARD_CEILING_MS));

    if (!introEnabled()) { introKillSwitch = 'settings-off'; introShowWord(); _introTimers.push(setTimeout(introDissolve, KILLSWITCH_FADE_MS)); return; }
    if (window.matchMedia && window.matchMedia('(prefers-reduced-motion: reduce)').matches) { introKillSwitch = 'reduced-motion'; introShowWord(); _introTimers.push(setTimeout(introDissolve, KILLSWITCH_FADE_MS)); return; }
    if (!introHasWebGL()) { introKillSwitch = 'no-webgl'; introShowWord(); _introTimers.push(setTimeout(introDissolve, KILLSWITCH_FADE_MS)); return; }

    var THREE;
    try {
      THREE = await import('./vendor/scene-lib.module.js?v=1.0'); // exact-match precache this URL, query and all
    } catch (e) {
      introKillSwitch = 'import-failed';
      introShowWord();
      _introTimers.push(setTimeout(introDissolve, KILLSWITCH_FADE_MS));
      return;
    }
    if (_ending) return; // skipped during import

    await introBuildScene(THREE, stageEl);
  }

  introMain().catch(function (e) { introKillSwitch = introKillSwitch || 'error'; introForceRemove(e); });
})();
```

### Service worker precache entries (`sw.js`)

```js
// Exact-match, query string and all. Bump the outer file's query and
// CACHE together whenever the vendored library or app shell changes.
const CORE = [/* ...your existing entries..., */ './vendor/scene-lib.module.js?v=1.0', './vendor/scene-lib-core.min.js'];

self.addEventListener('activate', (e) => {
  e.waitUntil((async () => {
    const keys = await caches.keys();
    // app-intro-art is a separate runtime cache for remote intro art,
    // never wipe it on a version bump
    await Promise.all(keys.filter((k) => k !== CACHE && k !== 'app-intro-art').map((k) => caches.delete(k)));
    await self.clients.claim();
  })());
});
```

## What was left out

- The Collectibles-specific card field, cast tableau, and choreography
  timings (walk duration, ending fade curves) are content decisions for
  that app, not part of the reusable pattern. Kept generic stubs
  (`getDataCandidates`, `introResolveContentUrl`) instead.
- Full pointer-parallax and multi-phase ending choreography from the
  source were trimmed from the skeleton to keep it under budget. The
  architecture note (section 1) and scene construction notes (section 6)
  describe the rules to reapply them per project.
- The toast-deferral mechanism for the top-layer popover problem (section
  7, point 4) is not yet captured, it ships in Collectibles v3.24, after
  this extraction.
