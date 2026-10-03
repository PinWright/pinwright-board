---
id: E-volume-set-extent-units-class-dependent-docs
title: "volume.set_volume_extent/set_volume_bounds document a WORLD half-extent but apply extent/100 as actor scale on non-brush trigger shapes (sphere, capsule, box), so the result is unscaledExtent x extent/100"
status: OPEN
severity: Medium
category: ergonomic
tags: [volume, set_volume_extent, extent, units, scale, docs, wiki, discoverability]
encounters: 2
costly: 1
lastSeen: 2026-07-01T19:06:56.5680937+03:00
rice: [2, 2, 1, 2]
priority: 17
---

# volume.set_volume_extent and set_volume_bounds promise a WORLD half-extent but apply extent/100 as actor scale on trigger sphere/capsule/box

`volume.set_volume_extent` documents `extent` as a "WORLD half-extent" in its summary and param
description (`Source/PinWright/Private/Handlers/Volume/VolumeHandler.cpp:1212-1224`).
`volume.set_volume_bounds` documents `bounds` as WORLD min/max corners (`:1434-1446`). Both
verbs keep that promise only for `ABrush` volumes (`Cast<ABrush>` at `:1246`, `:1489`). Any
other volume goes to the `else` branch, which sets
`SetActorScale3D(extent / 100)` (`:1278`, `:1506`). These include `ATriggerSphere` and
`ATriggerCapsule` (from `volume.create_trigger_sphere` / `create_trigger_capsule`, `:384`,
`:447`) and `ATriggerBox`, which are all `ATriggerBase`, not brushes. The resulting size is
therefore the shape's own unscaled extent times `extent/100`, not `extent`:

- On a `TriggerSphere` of radius 350, `extent:{500,500,500}` gives scale 5 and a world
  half-extent of `{1750,1750,1750}` (History `#1`).
- On a `TriggerBox` (default box extent 40), `extent:{400,120,250}` gives `{160,128,128}`.
  That is 0.4 x extent, with the readback floored by the ~128 editor-sprite bounds
  (History `#2`, ~4 extra diagnostic calls).

The call still reports success. `newExtent` is measured (`:1290-1293`), so it contradicts
`requestedExtent`, but nothing flags the mismatch. The documented contract is wrong for every
trigger shape that has a typed create verb, apart from the brush-based `create_trigger_volume`.

**Fix:** In the non-brush branch of both verbs, size the shape component itself instead of
writing `extent/100` as scale. For a `UBoxComponent`, set `SetBoxExtent(extent)` with the
actor at scale 1. For a `USphereComponent`, set `SetSphereRadius`; refuse a non-uniform
extent, or document that the largest axis is used. For a `UCapsuleComponent`, set
`SetCapsuleSize(radius from x/y, half-height from z)`. Refuse (`INVALID_ARGUMENT`) a non-brush
class with no known shape component. As a minimum alternative, narrow both summaries to say
what the non-brush branch does.

**Acceptance:** After `set_volume_extent {extent:{500,500,500}}` on a radius-350 TriggerSphere,
`newExtent` and `get_volumes_info` both report ~`{500,500,500}`. A TriggerBox with
`{400,120,250}` reads back `{400,120,250}` (bounds query excluding the sprite).
`set_volume_bounds` on a trigger shape spans the requested corners. A test covers one sphere
and one box.

**Workaround:** For a trigger shape, pass `extent = target / unscaledShapeExtent * 100` per axis.
Alternatively, set the shape component's `SphereRadius` / `BoxExtent` / `CapsuleHalfHeight`
with `property.set`, and check `newExtent` in the response.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the arena gameplay-triggers task (focus `volume.create_trigger_sphere`, namespace `volume`, 9 calls, outcome `ergo`; judge filed `B-blocking-volume-no-brush-geometry` #7 for the response-side echo bug). Distinct request-side docs angle: `volume.set_volume_extent`'s `extent` is documented as a bare `"New extent {x,y,z}"` (`VolumeHandler.cpp:1292`) but means an absolute world-unit half-extent for brush volumes (`Cast<ABrush>`→`CreateBoxBrushForVolume` :1314) and a scale×100 for non-brush trigger spheres (`SetActorScale3D(extent/100)` :1318), so `extent={500,500,500}` on a 350-radius `TriggerSphere` produces a `{1750,1750,1750}` bounding half-extent (350×5) — five times the typed value. An agent reading the wiki cannot predict which interpretation applies before calling; the friction note (verbatim above) shows the resulting confusion verifying a literal target extent. Fix (downstream docs): add a `### volume.set_volume_extent` section to `docs/wiki-src/volume.md` stating the class-dependent units (brush = absolute world half-extent; non-brush sphere = scale = extent/100, bounding half-extent = baseRadius×(extent/100)). Same overlay `E-volume-type-filter-discovery`/`E-volume-create-name-vs-volumename`/`E-volume-get-info-no-limit-spills` already target. Distinct from #7 (response-side echo fix) — this gap remains even after the echo is corrected. Sibling pattern of `E-geometry-warp-extent-semantics` (request param meaning) vs `E-geometry-deformer-echo-mesh-counts` (response shape).
- `#2-additional-triggerbox-extent-semantics` `OPEN` reporter — Additional evidence (arena gameplay/atmosphere-volumes task, no seed; `volume` namespace, 13 executing RPCs, outcome tool_bug). Extends this ticket beyond the trigger-SPHERE case to `ATriggerBox`, and **corrects the class model in the body**: per the judge's source replay in `B-blocking-volume-no-brush-geometry` `#8`, `ATriggerBox` is an `ATriggerBase` (NOT an `ABrush`), so `set_volume_extent` takes the **non-brush** `SetActorScale3D(extent/100)` branch (`VolumeHandler.cpp:1318`) too — the same class-dependent-units surprise, but on a class this ticket's body still lists under "brush = absolute world half-extent." Concretely: `set_volume_extent {Arena_AmbushTrigger, extent:{400,120,250}}` echoed `newExtent {400,120,250}` but `get_volumes_info` read back `{160,128,128}`; then `{380,110,240}`→`{152,128,128}`; then `{120,400,220}`→`{128,160,128}`. Pattern = `max(0.4×extent, 128)` per axis, i.e. `SetActorScale3D(extent/100)` applied to the engine-default ~40-unit half-extent box (40×(extent/100)=0.4×extent), then floored at the ~128 editor-Sprite billboard bound (400→scale4→160; 120→scale1.2→48 but floored 128; 250→scale2.5→100 but floored 128). An agent reading the wiki still cannot predict, before calling, that a TriggerBox `extent` becomes `scale×40` with a ~128 floor, so the round-trip fails: the agent burned ~4 extra diagnostic set/get RPCs reverse-engineering the `max(0.4x,128)` rule and had to reorient the corridor onto the one axis whose scaled value clears the 128 floor to make the tighten visible. Friction note verbatim: *"volume.get_volumes_info reports its extent through a lossy transform (~0.4x the set half-extent, floored at 128 per axis) so set extents don't round-trip like the other four volume classes do — it took two diagnostic re-tightens to reverse-engineer the max(0.4x,128) rule and reorient the corridor."* (The CallAnalyzer's proposed ticket blamed `get_volumes_info` as lossy; the judge's `#8` replay shows `get_volumes_info`'s `GetActorBounds()` is the truthful side — the caller's "lossy get" mischaracterization is itself a symptom of this discoverability gap.) Docs fix widened: the `### volume.set_volume_extent` overlay note must cover **all** the `ATriggerBase` trigger classes (box/capsule/sphere) as non-brush `scale=extent/100`, not just the sphere, and state the ~128 editor-Sprite floor for `ATriggerBox`. Severity unchanged (Low, docs discoverability — encounters is a tiebreak, never a severity input); bumped encounters→2.
- `#3-rephrased` `OPEN` developer — Rephrased as a contract defect. The old premise, that `extent` was a bare "New extent {x,y,z}", is stale: `set_volume_extent` now documents a WORLD half-extent and measures `newExtent` (`VolumeHandler.cpp:1212-1224`, `:1290-1293`). The non-brush branch still applies `SetActorScale3D(extent/100)` (`:1278`), and `set_volume_bounds` does the same (`:1506`), so the WORLD contract is false for trigger shapes. Corrected the class list: TriggerBox and TriggerCapsule are `ATriggerBase` (non-brush), as `#2` found. Updated the stale `:1292`/`:1314`/`:1318` citations. Severity Low -> Medium: the verb applies something other than its documented contract on a normal level-building path, a caller must notice the measured mismatch and pre-divide by the shape extent, and `#2` cost ~4 diagnostic calls. Id/category kept (E-); the manager may re-file it as B-. RICE R 1->2 (2 encounters), C 0.8->1 (verified), E 1->2 (shape-component sizing in two verbs plus tests).
