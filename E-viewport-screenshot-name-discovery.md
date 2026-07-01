---
id: E-viewport-screenshot-name-discovery
title: "Screenshot lives under editor.* not viewport.* — viewport.screenshot is the natural guess and nothing cross-references it"
status: OPEN
severity: Low
category: ergonomic
tags: [docs, editor, screenshot, viewport, discovery, naming]
encounters: 1
lastSeen: 2026-06-23T10:08:23Z
---

# "viewport.screenshot" is the natural guess; the real method is editor.screenshot, with no cross-reference

A caller whose intent is "screenshot the viewport" naturally reaches for
`viewport.screenshot` — viewport is the noun being captured, and the broader
ecosystem talks about viewport screenshots. But there is no `viewport.*`
namespace screenshot method (and no `docs/wiki-src/viewport.md` overlay at all);
the real method is `editor.screenshot`, parked under the `editor` namespace's
"Stats / capture" line (`docs/wiki-src/editor.md`). Nothing on the viewport-shaped
path points there:

- The `editor.md` overlay lists `editor.screenshot` but never says "this is the
  viewport screenshot you were looking for" or cross-references a `viewport.*`
  guess.
- There is no `viewport.md` overlay to redirect from.
- The viewport-adjacent reader `system.inspect.get_viewport_info`
  (`E-viewport-info-camera-transform`, DONE) — the other place a "viewport"
  intent lands — does not cross-link to the capture verb either.

So a caller has to *know* that screenshots live under `editor.*` rather than
`viewport.*`. This is the same name-vs-namespace discovery friction
`E-actor-list-no-class-filter` (OPEN) flagged for `actor.list` (`filter` vs class
intent) and `E-widget-screenshot-docs-eye-vs-visibility` (DONE) flagged for the
widget screenshot prep verbs — the right method exists, but the obvious guess has
no on-page cue pointing to it.

This is the *discovery* angle, distinct from `F-editor-viewport-screenshot`
(DONE), which was the *capability* gap (`editor.screenshot` failing `NO_VIEWPORT`
outside PIE; now falls back to the level viewport). The capability is there; what
is missing is the breadcrumb from the `viewport.screenshot` guess to
`editor.screenshot`.

## What it should do

Docs-only (cheapest, no behavior change). In `docs/wiki-src/editor.md`:

1. In the `### editor.screenshot` area (or the "Stats / capture" bullet), add a
   one-line note: screenshots live under `editor.*`, not `viewport.*` — there is
   no `viewport.screenshot`; use `editor.screenshot` (async tracked job,
   poll `system.job_status`, returns `{path,width,height}`; falls back to the
   active level viewport outside PIE per `F-editor-viewport-screenshot`).
2. Optionally seed a tiny `docs/wiki-src/viewport.md` overlay (or extend the
   `system.inspect.get_viewport_info` guidance) that redirects "screenshot the
   viewport" intents to `editor.screenshot`.

Optional ergonomic follow-up (out of this docs scope): register a `viewport.screenshot`
alias, or have the dispatcher's unknown-method error for a `viewport.screenshot`-ish
call suggest `editor.screenshot`, so the natural guess self-corrects.

## Evidence

From the `effect.activate_niagara` struggle audit (namespace `effect`, outcome
`clean`). The story's own step 7 phrasing hedged the method name: *"Take an
offscreen viewport screenshot ... (viewport.screenshot or the equivalent
screenshot RPC)."* Friction note, verbatim: *"no viewport.screenshot exists, so I
used the documented equivalent editor.screenshot (async -> polled
system.job_status)."* The agent resolved it without a wasted RPC (it went
straight to `editor.screenshot` rather than firing a failing
`viewport.screenshot`), so the cost was one discovery/mapping step rather than a
round-trip — but the mis-reach is real and predictable enough that the task
author wrote the wrong name into the story. A single cross-reference on the
`editor.md` screenshot entry removes the guess-and-map step.

## History
- `#1-initial-audit` `OPEN` reporter — Filed from the effect.activate_niagara struggle audit (outcome clean). "Screenshot the viewport" naturally reads as `viewport.screenshot`, but the real method is `editor.screenshot` under the `editor` namespace, with no `viewport.*` overlay and no cross-reference from any viewport-shaped path (`editor.md` screenshot entry, `system.inspect.get_viewport_info`). The agent and the story author both reached for `viewport.screenshot`; the agent mapped it to the documented `editor.screenshot` without a wasted RPC, so the cost was a discovery step, not a round-trip. Distinct from `F-editor-viewport-screenshot` (DONE, the capability) — this is the breadcrumb. Proposed: docs-only — add a `viewport.*`→`editor.screenshot` note on the `editor.md` screenshot entry (and optionally a tiny `viewport.md` redirect). Tagged `docs`; overlay to edit is `docs/wiki-src/editor.md`.
