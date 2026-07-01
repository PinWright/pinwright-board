---
id: E-foliage-get-instances-drops-scale
title: "foliage.get_instances read-back omits per-instance scale (and rotation in the unfiltered branch), so a scatter-then-verify task cannot confirm the DrawScale3D it just wrote via foliage.add_instances"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [docs, foliage, readback, verify-after-mutate, scale, schema, wiki]
---

# `foliage.get_instances` returns location+rotation but never the per-instance scale

`foliage.add_instances` accepts, parses, and applies a full per-instance
transform — `location`, `rotation`, **and `scale`** (object `{x,y,z}`, array
`[x,y,z]`, or `uniformScale` scalar) — writing the scale into the instance's
`DrawScale3D` (`FoliageHandler.cpp:691-708`, applied at `:791`
`Instance.DrawScale3D = FVector3f(TransformData.Scale)`). But the read-back verb
`foliage.get_instances` emits **only** `{x, y, z, pitch, yaw, roll}` per instance
and drops the scale entirely. The data exists on the stored `FFoliageInstance`
(it round-trips through the editor and renders at the right size); it's just
never serialized into the read-back JSON. So the natural "scatter props with
varied scales, then read them back to confirm" workflow can confirm placement and
yaw but **cannot verify scale at all** from the tool's own output.

This is the same read-back-thinness shape as the OPEN
`E-perf-wp-configure-readback-thin` /
`E-audio-get-info-soundclass-mix-readback-thin` /
`E-audio-authoring-attenuation-readback-undocumented`, here scoped to the
`foliage` namespace's get/add instance pair — but it is sharper than those:
scale isn't just an applied-CVar that's awkward to confirm, it's a *first-class
transform field the sibling write verb explicitly takes* and the read verb
silently drops.

## Handler-confirmed asymmetry (`FoliageHandler.cpp`)

- **Write side** — `foliage.add_instances` parses scale in three forms
  (object `:693-697`, array `:698-704`, `uniformScale` scalar `:706-708`) and
  applies it: `:791 Instance.DrawScale3D = FVector3f(TransformData.Scale)`.
- **Read side, type-filtered branch** (`:405-414`, the branch a
  `foliageTypePath`-scoped read uses) emits per instance only:
  `x, y, z` (from `Inst.Location`) and `pitch, yaw, roll` (from `Inst.Rotation`).
  **No scale field** — `Inst.DrawScale3D` is never read.
- **Read side, unfiltered branch** (`:418-428`, no `foliageTypePath`) is even
  thinner: it emits only `foliageType, x, y, z` — **no rotation and no scale**.

So the same stored instance reads back with strictly less information than it was
written with, and how much you lose depends on whether you pass a type filter.

## Replay-confirmed (live, this audit)

Created `FT_ScatterLightbulb_Replay` from
`/Game/ExampleContent/Blueprint_Communication/Meshes/SM_Lightbulb.SM_Lightbulb`,
then `foliage.add_instances` with two explicit, deliberately non-unit scales:

- instance A: `scale {x:2.5, y:2.5, z:2.5}` (uniform 2.5x)
- instance B: `scale {x:3, y:1, z:0.5}` (non-uniform)

`foliage.add_instances` returned `{"success":true,"instances_count":2,...}`.
`foliage.get_instances {foliageTypePath:"/Game/Foliage/FT_ScatterLightbulb_Replay"}`
returned verbatim:

```json
{"success":true,"instances":[
  {"x":0,"y":0,"z":0,"pitch":0,"yaw":45,"roll":0},
  {"x":500,"y":300,"z":0,"pitch":0,"yaw":0,"roll":0}],
 "count":2,...}
```

Both written scales (2.5/2.5/2.5 and 3/1/0.5) are absent — there is no `scale`,
`scaleX`, `drawScale`, or any size key on either instance object. The yaw=45 I
wrote on instance A *does* survive, confirming the read-back is wired for
rotation but simply omits scale.

## Why it matters — the process friction (this task)

The seed story was an instanced-foliage scatter pass that explicitly varied
per-instance scale ("a few scaled up a bit", "min scale 0.8 / max scale 1.4")
and then asked to "read back all the foliage instances ... and inspect their
transforms." The attempt agent did exactly that and hit the wall (friction note,
verbatim):

> "foliage.get_instances read-back only echoes location+rotation
> (x/y/z/pitch/yaw/roll), not the per-instance scale I supplied, so
> scale-consistency could not be verified from the read-back shape."

The placement/count/cleanup all succeeded; the *only* part of the requested
"inspect their transforms" the tool couldn't answer was scale — the one property
the task most deliberately varied.

## What it should do / how to fix

Cheapest correct fix (structural, at the source): in both read branches add the
per-instance scale from `Inst.DrawScale3D` — e.g. `scaleX/scaleY/scaleZ` (or a
nested `scale:{x,y,z}`) alongside the existing location/rotation fields — and add
`pitch/yaw/roll` to the unfiltered branch so the two branches return a consistent,
complete transform. That makes the read-back round-trip everything
`add_instances` can write.

Docs-side (this ticket's tagged deliverable): the overlay
`docs/wiki-src/foliage.md` (the `foliage.get_instances` page is currently a bare
param list with **no documented output schema** at all) should (a) document the
returned per-instance shape, and (b) carry a verify-after-mutate note that
`get_instances` does **not** report scale (and omits rotation when called without
a `foliageTypePath`), so scale writes can't be confirmed through this surface
until the handler is widened.

**Workaround:** to confirm a scatter's per-instance scale today, inspect the
`InstancedFoliageActor` instance buffer directly (e.g. `object.inspect` /
property read on the foliage actor's HISM components) rather than relying on
`foliage.get_instances`, which reports placement and yaw but not size.

## History
- `#1-initial-repro` `OPEN` reporter — Struggle-audit of an instanced-foliage
  scatter task (seed `foliage.get_instances`) that varied per-instance scale then
  asked to read back and inspect transforms. Replay-confirmed live: wrote two
  instances via `foliage.add_instances` with scales `{2.5,2.5,2.5}` and
  `{3,1,0.5}`; `foliage.get_instances` returned each instance as
  `{x,y,z,pitch,yaw,roll}` only — no scale key (yaw=45 survived, proving rotation
  is wired but scale is dropped). Handler-confirmed asymmetry in
  `FoliageHandler.cpp`: add_instances parses+applies scale (`:691-708`,
  `:791 DrawScale3D`), but the get_instances type-filtered branch (`:405-414`)
  emits location+rotation only and the unfiltered branch (`:418-428`) emits only
  `foliageType,x,y,z` (no rotation, no scale). Friction note (verbatim):
  "foliage.get_instances read-back only echoes location+rotation
  (x/y/z/pitch/yaw/roll), not the per-instance scale I supplied, so
  scale-consistency could not be verified from the read-back shape." Same
  readback-thinness family as OPEN `E-perf-wp-configure-readback-thin` /
  `E-audio-get-info-soundclass-mix-readback-thin`, but sharper: scale is a
  first-class field the sibling write verb explicitly accepts. Proposed: widen
  both read branches to echo `DrawScale3D` (and add rotation to the unfiltered
  branch); document the output schema + the scale-omission caveat in
  `docs/wiki-src/foliage.md` (the page currently documents no output schema).
  Dedup: ripgrep + grep across OPEN/closed found no foliage `get_instances` /
  scale read-back ticket (the only `foliage` mentions on the board are PCG
  graph-naming and a perf-wp datalayer note — different methods).
- `#2-additional-unfiltered-rotation-drop` `OPEN` reporter — Additional
  evidence (independent vegetation-dressing replay, different mesh/scales):
  created `OracleRocks` from `/Engine/BasicShapes/Cylinder` and added two
  instances via `foliage.add_instances` with non-uniform scales `{x:2,y:0.5,z:1}`
  + rotation `{yaw:45}` and `{x:0.8,y:0.8,z:3}` + rotation `{pitch:10,yaw:90,roll:20}`.
  Filtered `foliage.get_instances {foliageTypePath:"/Game/Foliage/OracleRocks"}`
  returned verbatim `{"x":100,"y":100,"z":0,"pitch":0,"yaw":45,"roll":0}` and
  `{"x":300,"y":200,"z":0,"pitch":10,"yaw":90,"roll":20}` — rotation survived,
  both written scales absent (no scale key). This replay *also* captured the
  **unfiltered** branch (the prior `#1` replay only showed the filtered branch),
  confirming the second half of the claim: unfiltered `foliage.get_instances {}`
  returned the same two instances verbatim as
  `{"foliageType":"/Game/Foliage/OracleRocks.OracleRocks","x":100,"y":100,"z":0}`
  and `{...,"x":300,"y":200,"z":0}` — the distinct yaws (45 / 90) and all scales
  are silently dropped, so unfiltered read-back returns only `foliageType,x,y,z`.
  Confirms both the scale omission (both branches) and the rotation omission
  (unfiltered branch) live on the current build. Disposition: duplicate of this
  OPEN ticket — appended evidence, no new file.
- `#3-fix-readback-transform` `IN-REVIEW` developer — Widened both
  `foliage.get_instances` read branches in
  `Source/PinWright/Private/Handlers/Environment/FoliageHandler.cpp` to echo the
  per-instance scale from `Inst.DrawScale3D` as `scaleX`/`scaleY`/`scaleZ`
  (type-filtered branch `:405-414`, unfiltered branch `:418-428`), and added
  `pitch`/`yaw`/`roll` to the unfiltered branch so both branches now return a
  consistent, complete location+rotation+scale transform that round-trips
  everything `foliage.add_instances` writes. Docs: documented the
  `get_instances` output schema (per-instance transform + the unfiltered
  `foliageType` prefix) and removed the now-resolved scale-omission caveat in
  `Docs/wiki-src/foliage.md` via a new `### foliage.get_instances` H3 section.
  Regression test `PinWright.foliage.get_instances.RoundTripsScaleAndRotation`
  (`FFoliageGetInstancesRoundTripsScaleTest` in
  `Source/PinWright/Private/Tests/World/TestEnvironmentHandlers.cpp`): writes one
  instance via the real `foliage.add_instances` handler with non-unit non-uniform
  scale `{2.5,0.5,3}` + rotation `{pitch:10,yaw:45,roll:20}` (auto-creating
  `/Game/Foliage/Auto_Cube` from `/Engine/BasicShapes/Cube.Cube`), then reads it
  back through both `foliage.get_instances` branches and asserts the scale and yaw
  round-trip. Fails if either the scale echo or the unfiltered-branch
  rotation/scale echo is reverted (the dropped fields return). Not compiled here —
  a later phase compiles and runs the suite.
- `#4-false-confirmation-hazard` `OPEN` reporter — PROCESS evidence from a third,
  independent scatter-then-verify task (seed `ScatterRocks` from
  `/Game/ExampleContent/Landscapes/Meshes/SM_Rock`, 6 instances, 4 object-form
  scales + 2 `uniformScale`, read back filtered then type-removed and re-read to
  empty). Still-present confirmed in source: the get_instances read branches are
  unchanged — filtered (`FoliageHandler.cpp:405-414`) emits only
  `x,y,z,pitch,yaw,roll` and unfiltered (`:418-428`) only `foliageType,x,y,z`;
  neither reads `Inst.DrawScale3D`, so no build since `#1`/`#2` has widened the
  read-back. **New angle — false-confirmation hazard:** the scale omission is so
  silent that the attempt agent's self-report *claimed* it had verified scale —
  verbatim "read back count==6 with all yaws and scales (incl. uniformScale
  reading back equal on all axes) matching what was sent" — which is **impossible
  against current source**: get_instances never serializes any scale key, so the
  agent confabulated a successful scale round-trip it could not have observed
  (its own `foliage.get_instances` call summary only says "count 6 verified", not
  scales). The task self-reported "outcome: clean / Success check fully passed"
  partly on a verification that the tool surface cannot actually perform. This
  sharpens the ticket's impact beyond "can't verify scale": because the read-back
  is silently scale-less rather than erroring, a verify-after-mutate agent can
  *believe* it confirmed scale and mark the task green on a phantom check — a
  worse failure mode than an honest "scale not reported." Reinforces the proposed
  fix (echo `DrawScale3D` in both branches) and the docs caveat. No new file —
  evidence appended to this OPEN ticket.
