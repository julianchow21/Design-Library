# Pattern Library

Reusable UI/UX pattern library. Single-file gallery in `index.html`, opens straight from `file://`.

## Purpose

Julian pastes Inspect-panel code from sites he admires. The session model re-implements it clean on house tokens (`tokens.css`), and the clean pattern gets stored here for reuse in future Kujira projects.

This library is the sole home for reusable UI ideas (the root CLAUDE.md routes them here). Boundary with the starter: `web-app-starter/workbench.html` demos only the starter's own shipped `lib/` engines, everything else reusable lives here. Patterns proven in real projects flow onward via the Promotion rule below.

## Intake rule

- Pasted external code is reference only. It is NEVER stored verbatim, not even in a comment, for copyright and reusability reasons.
- Every pattern in this gallery is a clean re-implementation on house tokens: same idea, house colours, spacing, radius, motion.
- If a pattern cannot be reasonably rebuilt on tokens (e.g. it depends on a library or a very specific asset), leave it out rather than approximate it badly.

## How to add a pattern

1. In `index.html`, append a new `<template class="pattern" data-title="..." data-category="Components|Flows" data-source="..." data-note="...">` block near the existing one. `data-note` is plain text, no angle brackets, they get parsed as real (invisible, selector-matching) elements when the note is injected via innerHTML.
2. Put the pattern's HTML inside the template, with a scoped `<style>` tag inside it if it needs CSS (prefix class names so they cannot collide with another pattern or the gallery shell, e.g. `.dp-*` for the empty-state seed, `.dg-*` for the dot globe).
3. Use `tokens.css` variables only, never a hardcoded colour, radius or duration.
4. Bump the version badge (`.ver` in the topbar markup, and the `<title>`/comment if referenced) in the same edit, e.g. `v0.2 (DD Mon)`.
5. Open the file from `file://` and check: card renders, filter shows it under the right category, copy button copies the template's raw HTML, both themes look right, long titles/notes do not break the card.

### Patterns with a script tag

A pattern's HTML may include a `<script>` (e.g. a canvas animation). `render()` extracts cloned `<script>` elements and re-creates them fresh before appending, because a cloned script keeps its "already started" flag and never runs otherwise (see `insertLiveContent()`). Rules for a scripted pattern:

- Wrap the whole script body in an IIFE, `(function(){ ... })()`, so multiple instances (or repeated re-renders from the filter cycle) never collide on globals.
- Find the pattern's own root/canvas relative to itself, e.g. `document.currentScript.parentElement.querySelector('.xx-canvas')`, never a global `id`. The same template can be live more than once during a render.
- Idempotent: every instance must own only its own closured state. Cycling filters re-clones and re-runs the script each time, an old instance must never be able to affect a new one.
- Self-terminating: any `requestAnimationFrame` loop or `MutationObserver` must check `element.isConnected` every tick and stop (and disconnect any observer) the moment its element leaves the DOM. Without this, cycling filters stacks a new loop on every re-render and old ones never die.
- Respect `prefers-reduced-motion`: render one static frame, no continuous animation.
- Read colours from `tokens.css` via `getComputedStyle`, never hardcode, and re-read on an `html` class change (`MutationObserver` on `document.documentElement`, `attributeFilter:['class']`) so the theme toggle recolours the pattern live.

## Promotion rule

A pattern proven useful across two or more real projects gets promoted into `templates/web-app-starter/` as a first-class piece of the starter (its own CSS block or a documented snippet), not left only here. Leave the copy here too, the library stays the full catalogue.

## Files

- `tokens.css`, unmodified copy of the starter's tokens. Never edit locally, if the starter's tokens change, re-copy.
- `index.html`, the whole gallery: shell CSS, filter/render/copy JS, and every pattern's `<template>`.
- A systems pattern (architecture spanning more than markup/CSS/one script, e.g. service worker integration, an asset pipeline, a kill-switch chain) does not fit a `data-note` attribute. It gets its own `<Pattern Name>.md` file alongside `index.html` (Title Case, two words max), plus a normal gallery card whose `data-note` points to that file and whose live demo covers only the piece that can be genuinely demoed (e.g. `Launch Intro.md`, demoed as its CSS-only fallback tier).
