---
id: E-actor-duplicate-no-rotation-scale
title: "actor.duplicate accepts only offset+newName (no rotation/scale) — any placed/oriented/scaled copy always needs a second actor.set_transform, doubling the call count for 'place N oriented copies'"
status: OPEN
severity: Low
category: ergonomic
tags: [actor, duplicate, transform, rotation, scale, ergonomics, partial-transform-verb]
encounters: 1
lastSeen: 2026-07-02T03:26:06.4106887+03:00
---

# `actor.duplicate` exposes only `offset` + `newName` — an oriented/scaled copy always costs a paired `actor.set_transform`

`actor.duplicate` places a copy of an existing actor but its transform surface is
partial: it takes `offset` (a relative translation) and `newName`, and **no
rotation and no scale**. So every copy that must land at a distinct orientation
or size — which is the whole point of "array these along a curve / scatter these
with variety" — needs an immediate, separate `actor.set_transform` to apply the
location + yaw + scale. A single "make a placed, oriented, scaled copy" intent is
split into two calls.

## Evidence — this task (park bollards along a spline)

The audited task duplicated one bollard template 8 times, each copy needing a
distinct along-curve yaw and a slightly varied scale. Because `actor.duplicate`
carries neither, all 8 successful duplicates were immediately followed by 8
paired `actor.set_transform` calls to apply location+yaw+scale — 16 calls for
what is conceptually 8 placements. The agent's friction note (verbatim):

> "actor.duplicate takes only offset+newName (no rotation/scale) so each copy
> needed a follow-up actor.set_transform."

This is the same "transform-family verb takes a partial transform, forcing a
follow-up `set_transform`" shape already filed for the spawn verbs as
`E-spawn-no-scale-param` (`actor.spawn`/`actor.spawn_from_blueprint` took
location+rotation but no scale; resolved by adding an optional `scale` param).
`actor.duplicate` is the sibling gap — it has the position lever (`offset`) but
lacks the rotation and scale levers.

## What it should be

Add optional `rotation` ({pitch,yaw,roll}) and `scale` ({x,y,z}) params to
`actor.duplicate`, applied to the new actor in the same place the duplicate
already applies `offset` (build the copy's `FTransform` from
source-transform + offset + the supplied rotation/scale, or apply them post-copy
exactly as `actor.set_transform` does). Then a fully-placed copy is one call.
Keep both optional (default: inherit the source's rotation/scale, i.e. current
behavior) so nothing breaks.

**Workaround (works today, reliably):** `actor.duplicate` then
`actor.set_transform` on the returned name — this is the very pattern the
`geometry.md` `array_linear`/`array_radial` steer-notes prescribe. Clean, but one
extra call per oriented/scaled copy.

**Note:** in this specific task the whole duplicate+set_transform loop would have
been avoided by the purpose-built `geometry.duplicate_along_spline` (filed as
undiscoverable in `E-duplicate-along-spline-undiscoverable`). This ticket is the
standalone gap for the **general, non-spline** duplicate case, where no array
helper applies.

severity rationale: impact=pure ergonomic friction (one extra `set_transform`
call per oriented/scaled copy; a reliable documented workaround exists) × reach=
`actor.duplicate` is a common verb but the oriented/scaled-copy variant that needs
the follow-up is a subset of its uses → Low. (Sibling `E-spawn-no-scale-param` was
rated Medium on every-session spawn reach; duplicate's need is narrower.)

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of a park-bollard-along-a-spline
  task (focus `geometry.duplicate_along_spline`; 42 calls; outcome done). PROCESS
  finding (CallAnalyzer + friction note): `actor.duplicate` exposes only
  `offset`+`newName`, so each of the 8 copies that needed a distinct along-curve
  yaw + varied scale required a paired `actor.set_transform` — 8 duplicate + 8
  set_transform for 8 placements. Verbatim friction: "actor.duplicate takes only
  offset+newName (no rotation/scale) so each copy needed a follow-up
  actor.set_transform." Same "partial-transform verb → forced follow-up
  set_transform" family as `E-spawn-no-scale-param` (spawn missing scale, IN-REVIEW),
  but a different verb/handler (`LifecycleHandler.cpp` `actor.duplicate` vs
  `SpawnHandler.cpp`) so tracked separately with a shared `partial-transform-verb`
  family tag. Proposed: add optional `rotation` + `scale` params to
  `actor.duplicate` (symmetric with `offset`), defaulting to the source's values.
  Dedup: ripgrep over OPEN+closed found `E-spawn-no-scale-param` (spawn verbs, the
  family sibling — different method/handler, its fix adds scale only to spawn),
  `F-duplicate-editor-subobjects` (DONE — duplicating widget-tree children /
  actor components, not the actor.duplicate transform surface), and
  `E-actor-duplicate-locked-level-opaque-error` (locked-level error wording,
  unrelated). No existing ticket covers `actor.duplicate`'s missing rotation/scale
  params.