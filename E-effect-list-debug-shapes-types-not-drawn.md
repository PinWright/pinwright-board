---
id: E-effect-list-debug-shapes-types-not-drawn
title: "effect.list_debug_shapes lists the static shape-TYPE catalog, not the drawn shapes — name parallels clear_debug_shapes/draw_debug_shape which DO operate on real shapes"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [effect, list_debug_shapes, debug-shape, naming, misleading, discoverability]
---

# `effect.list_debug_shapes` is named like an enumerate-what's-drawn verb but returns a constant type catalog

`effect.list_debug_shapes` (`Handlers/VFX/EffectHandler.cpp:251-272`) returns a
**hardcoded constant list of the 11 supported shape *types*** and never inspects
the world. Its two sibling verbs in the same namespace operate on the **actual
drawn shapes**: `effect.draw_debug_shape` (`:283`) draws a real shape into the
world, and `effect.clear_debug_shapes` (`:277`) tears down the drawn shapes via
`FlushPersistentDebugLines`. All three share the `debug_shape(s)` token, so the
plural `list_debug_shapes` reads as the "enumerate what is currently drawn"
companion to `clear_debug_shapes` ("clear what is currently drawn"). It is not —
it is a static capability catalog. The summary string "List available debug shape
types" is accurate, but the *method name alone* (which is what an agent picks
from the namespace index before reading the page) misleads.

## Why it concretely misleads

An agent doing a blockout layout drew 6 persistent debug shapes (a circle, a
sphere, 4 arrows) and then, per the task ("list the debug shapes again so I can
confirm the markers are registered"), called `effect.list_debug_shapes` a second
time *to verify the markers it had just drawn*. It got back the identical static
type list — no information about what was drawn — and concluded (verbatim
friction): *"effect.list_debug_shapes only returns the static set of SUPPORTED
shape types (same 11 before and after drawing), so it cannot confirm that specific
markers are 'registered/drawn'; the per-draw success:true is the only real
confirmation, and there is no method to enumerate currently-active debug shapes."*

This is not a one-off agent misread: the existing board ticket
`F-effect-draw-debug-shapes-batch` (OPEN) carries the *same* false premise in its
own body — line 35: *"`effect.list_debug_shapes` lists *all*"* and line 113:
*"`clear_debug_shapes`/`list_debug_shapes` are already whole-scene-scoped (one
call clears/lists all)"* — treating `list_debug_shapes` as if it lists the drawn
shapes. The name has now misled both an attempt agent and a prior reporter.

## Quotable demonstration (replayed via `mcp__editor-automation__call`)

Drew one real debug sphere, then listed — the list is unchanged (does not reflect
the drawn sphere):

```
effect.draw_debug_shape {preset:"oracle_repro", shapeType:"sphere",
                         location:[0,0,50], color:[255,0,0,255], size:40, duration:600}
  -> {"success":true,"shapeType":"sphere","location":"0.00,0.00,50.00","duration":600}

effect.list_debug_shapes {}
  -> {"shapes":["sphere","box","circle","line","point","coordinate",
                "cylinder","cone","capsule","arrow","plane"],"count":11}
```

Identical output before any draw and after a draw — the constant catalog, never
the world contents. (Cleaned up afterward with `effect.clear_debug_shapes`.)

## Fix (self-describing result — the least-churny in-code option)

The cheapest in-code fix is to make the **result self-describing** so the
method-name implication is corrected at the call site, without growing the
~1,200-method catalog or depending on a wiki edit:

- Add a `shapeTypes` key carrying the same catalog array (so the name's "types"
  meaning is reachable in the result shape), keeping the legacy `shapes`/`count`
  keys for back-compat.
- Add a one-line `note` string: *"These are the supported shape TYPES, not the
  shapes currently drawn. To draw use effect.draw_debug_shape; to tear down use
  effect.clear_debug_shapes. There is no verb that enumerates currently-drawn
  shapes (per-draw success is the confirmation)."*

This was chosen over the two alternatives considered: a **rename + back-compat
alias** (`effect.list_debug_shape_types`) would add a second
`REGISTER_RPC_HANDLER` block to the catalog — duplicate-entry churn for a
no-error, clean-outcome Low-friction nit, and there is no alias mechanism in
`Handlers/HandlerRegistration.h` — and a **wiki disclosure line** on
`docs/wiki-src/effect.md` belongs to the downstream wiki process, not this
ticket. The self-describing result corrects the reading directly at the call
site where the agent saw the friction.

The underlying "no verb enumerates currently-drawn shapes" remains a separate
capability point (the UE `DrawDebug*` primitives have no queryable per-shape
registry), but the task here *succeeded* (per-draw success confirmed each marker;
cleanup left zero orphans), so this is filed as the ergonomic naming/result
mismatch — not a blocking gap. If a future task is genuinely blocked by the
absence of an enumerate-drawn verb, that is a separate F- ticket.

## Not a duplicate of

- `F-effect-draw-debug-shapes-batch` (OPEN) — that is the missing *batch draw*
  capability on `effect.draw_debug_shape` (N round-trips for N markers). This is
  the *naming/result-semantics* mismatch on the sibling `list_debug_shapes`; in
  fact that ticket *embeds* the misconception this ticket documents. Different
  verb, different friction class.

## History
- `#1-initial-repro` `OPEN` reporter — REALISM-mode campfire-clearing blockout
  audit (namespace `effect`; attempt outcome `done`/clean — task fully succeeded,
  both lights verified, zero orphans). Per-finding judge filed THIS ergonomic
  finding (no OUTCOME overlap with the clean attempt). `effect.list_debug_shapes`
  (`EffectHandler.cpp:251-272`) returns a hardcoded 11-element shape-*type* catalog
  and never reads the world; its siblings `draw_debug_shape` (`:283`) and
  `clear_debug_shapes` (`:277`) operate on real drawn shapes, so the shared
  `debug_shape(s)` token makes the plural `list_debug_shapes` read as
  "enumerate-what's-drawn." Replayed: drew a sphere, then `list_debug_shapes`
  returned the identical static catalog (quoted above). The agent called it twice
  trying to confirm its drawn markers and got nothing about them; the existing
  `F-effect-draw-debug-shapes-batch` body repeats the same false "lists all"
  premise. Proposed fixes: `_types` rename+alias, or a self-describing
  `shapeTypes`+`note` result, or a wiki disclosure line.
- `#2-reword-and-fix` `IN-REVIEW` developer — Reworded the ticket from a
  three-option pick-one to the single least-churny in-code fix (self-describing
  result), retiring the rename+alias and wiki-line options per the adversarial
  review (catalog-growing duplicate `REGISTER_RPC_HANDLER` for a no-error,
  clean-outcome Low nit; no alias mechanism exists in `HandlerRegistration.h`;
  the wiki line is the downstream wiki process). Implemented in
  `Source/PinWright/Private/Handlers/VFX/EffectHandler.cpp`
  (`effect.list_debug_shapes`, formerly :251-272): the result now carries a
  `shapeTypes` key (same catalog), keeps the legacy `shapes`/`count` keys for
  back-compat, and adds a `note` string disambiguating supported TYPES from
  currently-drawn shapes and pointing at `effect.draw_debug_shape` /
  `effect.clear_debug_shapes`. Regression test
  `PinWright.effect.list_debug_shapes.ResultIsSelfDescribing` in
  `Source/PinWright/Private/Tests/Assets/TestVFXHandlers.cpp` invokes the real
  handler via `InvokeHandlerWithCapture` and asserts the `shapeTypes` array
  (sphere/arrow present), the `note` string, and that the note names the draw
  verb — reverting the fix flips it.
