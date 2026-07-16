# AI Router

One `callAI(prompt)` facade in front of several AI providers, each with its
own auth style, request shape and response parsing, so the rest of the app
only ever calls one function and never branches on which provider is
active. Extracted from Kujira Collectibles v3.31 (`app.js`, lines 9191 to
9371: `_resolveAIProvider`, `callAI`). Generalised below: no Collectibles
prompts, and the single resolved-provider pick in the source is written up
here as a genuine ordered fallback chain (section 2 explains the
difference).

Category: Components. The provider config, classification and fallback
chain make this a systems pattern, too large for a single gallery
`<template>` card. The "AI Router" card in `index.html` is a pure visual
simulation of the chain walk (fixed setTimeout outcomes), no network call,
no key of any kind is read, stored, logged or displayed anywhere in the
demo. This file is the full pattern.

## 1. Provider config shape

```js
const PROVIDERS = [
  { id: 'gemini',     label: 'Gemini',     keyLS: 'app_gemini_key' },
  { id: 'groq',       label: 'Groq',       keyLS: 'app_groq_key' },
  { id: 'openrouter', label: 'OpenRouter', keyLS: 'app_openrouter_key' },
  { id: 'anthropic',  label: 'Anthropic',  keyLS: 'app_anthropic_key' }
];
```

Each provider is data, not a branch. Adding a fifth provider is one array
entry plus one `call()` implementation (section 3), nothing else in the
router changes.

## 2. Fallback order and resolution

The source picks the **cheapest configured option first** (its own
comment: free tiers first, Anthropic last), and lets an explicit user
choice in settings override the auto order if that provider actually has a
key:

```js
function resolveOrder(explicit){
  const configured = PROVIDERS.filter(p => !!localStorage.getItem(p.keyLS));
  if (explicit && configured.some(p => p.id === explicit)){
    return [explicit, ...configured.filter(p => p.id !== explicit).map(p => p.id)];
  }
  return configured.map(p => p.id); // free-tier-first array order above
}
```

**Difference from the shipped source, stated plainly:** `_resolveAIProvider`
in Collectibles picks exactly one provider by this order and `callAI` calls
only that one, a failed call returns an error string with no runtime
fallback to another provider. The chain built here is a deliberate
generalisation: attempt provider 1, and on a `transport` or `quota`
classification (section 3) fall through to the next **configured**
provider, stopping at the first success. A provider with no key is skipped
without ever being called, it was never attempted so it cannot be counted
as failed. Carrying this generalisation back into Collectibles itself would
be a genuine improvement, that is a separate, real-project decision, not
made by this library entry.

## 3. Error classification

Three categories, decided from the shape of the failure, not from which
provider produced it:

- **disabled**, no key configured for that provider. Never attempted, never
  counted as a failure, just skipped in the chain.
- **transport**, the request itself did not complete, a network error, or a
  timeout (`AbortSignal.timeout(90000)` in the source). Worth falling
  through to the next provider, the provider itself may be fine.
- **quota**, the request completed and the provider said no, an
  auth or rate-limit shaped HTTP status (401, 403, 429). Falling through to
  a different provider is still correct here (a different provider has its
  own separate quota), never assume a quota error on provider A says
  anything about provider B.

```js
function classify(err){
  if (err.name === 'TimeoutError' || err.name === 'TypeError') return 'transport';
  if (err.status === 401 || err.status === 403 || err.status === 429) return 'quota';
  return 'transport'; // default to retryable rather than silently stopping the chain
}
```

The source's own Gemini branch has a narrower, related case worth keeping:
a chain of current model-name aliases (Google rotates names), a `404` or
"not found" response tries the next alias, any other error stops
immediately, because retrying a different model name cannot fix an auth or
quota problem. Model-alias fallback and provider fallback are the same
idea one level down.

## 4. Key handling, done safely

- Every provider's key lives under its **own** localStorage entry, trimmed
  on write, read fresh on every call, never cached in a module-level
  variable that could linger or get logged by accident.
- Never hardcode a key anywhere in source, never commit one, never log a
  key or a request or response object that might contain one, log the
  provider id and the classification instead, not the payload.
- The source's own comment states the residual risk plainly: a key typed
  into a client-side settings panel is still readable by anyone who can
  read the page. For a real deployment, proxy the call through a small
  backend that holds the key server-side instead of trusting the browser.
- Placeholders only, anywhere near this pattern, in this file, in the demo,
  or in a real project's own README, never a real-looking key. Example
  shape only: `sk-REPLACE_WITH_YOUR_OWN_KEY`.

## 5. Retry and cooldown

Not present in the shipped source, a failed call simply returns an error
string, the next attempt is whatever the user does next (ask again). Worth
adding when generalising:

- A short **cooldown** per provider after a `quota` classification, that
  provider is known-limited for a while, do not immediately re-attempt it
  on the very next unrelated call. A timestamp per provider id kept in
  memory is enough, no persistence needed.
- **No automatic retry loop** within a single provider's own transport
  failure, beyond what section 3's chain already provides by moving to the
  next provider. A tight retry loop against a provider that is genuinely
  down just burns time the user is waiting on.

## 6. Integration steps

1. List your providers as data (section 1), one `call(prompt)`
   implementation per provider, each returning text or throwing.
2. Wrap every provider call in `try/catch`, classify the thrown error
   (section 3), never let a raw provider error string reach the UI
   unclassified.
3. Drive the chain off `resolveOrder()` (section 2), stopping at first
   success, skipping any provider with no key.
4. Surface the final state honestly: which provider answered, or, if the
   chain is exhausted, a designed message telling the user to add a key
   rather than a blank or broken panel.
5. Keep every key in its own localStorage slot (section 4), never in a
   shared object that gets logged or serialised whole.

## 7. Verification gotchas

1. **A provider with no key must never appear as failed.** Skipped and
   failed are different states with different colours and different text,
   confusing them makes a perfectly normal "not configured" look like an
   outage.
2. **Test the classification branches with the actual shapes your fetch
   layer throws.** A browser `fetch()` rejection is a `TypeError`, not an
   HTTP status. An aborted `AbortSignal.timeout` is a `TimeoutError`. An
   auth or quota failure is a resolved, non-throwing response with a bad
   status. Three different code paths, confirm each lands in the category
   you expect.
3. **Never screenshot or log a real key while verifying this pattern.** Use
   an obviously fake string for any manual test of the key-storage path.
4. **A demo or test harness must make zero real network calls.** Confirm in
   the browser's network panel that nothing fires to any provider's domain
   during a simulated run, a "demo" that quietly does call out is a data
   and cost risk, not a demo.

## What was left out

- The source's per-model-name retry loop for Gemini specifically (its own
  free-tier model names get renamed periodically) is domain detail for that
  one provider, section 3 keeps only the general lesson, a "not found"
  response retries a sibling, any other error stops.
- Streaming responses, token-by-token UI updates, are not covered, the
  source's `callAI` is a single non-streaming call per provider.
