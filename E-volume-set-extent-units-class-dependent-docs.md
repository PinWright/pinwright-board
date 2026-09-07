---
id: E-volume-set-extent-units-class-dependent-docs
title: "volume.set_volume_extent 'extent' is documented as a bare 'New extent {x,y,z}' but means absolute world half-extent for brush volumes and scale×100 for non-brush trigger spheres/capsules — the class-dependent meaning is undiscoverable from the wiki"
status: OPEN
severity: Low
category: ergonomic
tags: [volume, set_volume_extent, extent, units, scale, docs, wiki, discoverability]
encounters: 2
costly: 1
lastSeen: 2026-07-01T19:06:56.5680937+03:00
---

# `volume.set_volume_extent`'s `extent` param has two incompatible meanings by volume class, and the wiki documents neither

`volume.set_volume_extent` takes an `extent {x,y,z}` whose registered description
is, in full, `"New extent {x,y,z}"` (`VolumeHandler.cpp:1292`). Nothing states the
**units** or that the **meaning flips by volume class**:

- **Brush volumes** (`ABlockingVolume`, `ATriggerBox`, `ATriggerCapsule` — all `ABrush`)
  take the `Cast<ABrush>` branch and pass `extent` to `CreateBoxBrushForVolume`
  (`VolumeHandler.cpp:1314`), i.e. `extent` is intended as an **absolute world-unit
  half-extent**.
- **Non-brush volumes** (`ATriggerSphere`, an `ATriggerBase` — NOT an `ABrush`) take
  the else branch and apply `SetActorScale3D(extent / 100)` (`VolumeHandler.cpp:1318`),
  i.e. `extent` is a **scale multiplier ×100**, not a world extent. For a 350-radius
  trigger sphere, `extent={500,500,500}` sets actor scale `{5,5,5}` and the bounding
  half-extent becomes 350×5 = `{1750,1750,1750}` — five times the "500" the caller typed.

An agent reading the wiki before calling cannot know which interpretation applies to
the actor it just created, so a literal target like "grow the central trigger to a
500-unit extent" silently produces a 1750 result, and the only way to predict the
output is to know the actor's class is non-brush and recompute `baseRadius × (extent/100)`.
This is the request-side discoverability twin of `E-geometry-warp-extent-semantics`
(units/semantics of a deformer `extent` undocumented) — there the convention is
symmetric-half-extent, here it is class-dependent-units.

## Distinct from the judge's filing

`B-blocking-volume-no-brush-geometry` `#7-additional-trigger-sphere-extent-echo-mismatch`
(IN-REVIEW, the judge's filing for this same task) is the **response-side bug**: the
`newExtent` echo (`VolumeHandler.cpp:1325-1329`) blindly re-emits the raw request instead
of the post-apply `GetActorBounds()` half-extent, so the echo and the `get_volumes_info`
readback can never agree. Its fix is "make `newExtent` reflect the applied bounds."

This ticket is the **request-side docs gap**: even *after* #7 makes the echo accurate, an
agent reading the wiki still has no way to know — *before* calling — that `extent={500,500,500}`
on a trigger sphere means "scale 5×" (→ 1750) rather than "set the half-extent to 500 world
units." The param's class-dependent meaning is the friction, independent of whether the echo
is fixed. Same overlay/file-process relationship as `E-geometry-deformer-echo-mesh-counts`
(response shape) vs `E-geometry-warp-extent-semantics` (request param meaning) in the geometry
namespace.

## What it should do (docs only)

Add a `### volume.set_volume_extent` section to the **`docs/wiki-src/volume.md`** overlay
(currently a 3-line namespace blurb with no per-method sections — the same overlay
`E-volume-type-filter-discovery`, `E-volume-create-name-vs-volumename`, and
`E-volume-get-info-no-limit-spills` already want extended). It should state that `extent`'s
meaning depends on the volume's class:
- for **brush** volumes (blocking / trigger box / trigger capsule) `extent` is an absolute
  **world-unit half-extent**; and
- for **non-brush** trigger spheres (and other `ATriggerBase`) `extent` is applied as
  **actor scale = extent/100**, so the resulting bounding half-extent is
  `baseRadius × (extent/100)`, NOT the number passed.

One short note that tells the caller to expect `radius×(extent/100)` for trigger spheres
removes the surprise. (Enriching the `RPC_PARAM_OPT("extent", ..., "New extent {x,y,z}")`
description at `VolumeHandler.cpp:1292` would help too, but the overlay is the named target
per the docs process.)

## Friction evidence (this task — focus `volume.create_trigger_sphere`, namespace `volume`, 9 calls, outcome `ergo`)

The task was otherwise frictionless — per the self-report "wiki had clean per-method param
pages, every call succeeded first try with no retries." The single friction was units:
`volume.set_volume_extent {volumeName:"ArenaCenterTrigger", extent:{500,500,500}}` (a 350-radius
`TriggerSphere`) returned `newExtent {500,500,500}` but `get_volumes_info` read back
`extent {1750,1750,1750}`. Friction note, verbatim: *"set_volume_extent({500,500,500}) returned
newExtent {500,500,500} but the get_volumes_info readback reports ArenaCenterTrigger's extent as
{1750,1750,1750} (350 base radius × 5 scale = 1750), so the create radius is in world units while
set_volume_extent appears to be applied as a scale multiplier and the readback reports base×scale
— the units between create sphereRadius, set_volume_extent, and the get_volumes_info extent field
do not round-trip to the same number, which is confusing for verifying a literal target extent."*
Source-confirmed: `set_volume_extent` brush vs non-brush branch at `VolumeHandler.cpp:1311-1319`
(non-brush = `SetActorScale3D(extent/100)` :1318); param doc `"New extent {x,y,z}"` :1292 with no
units/class note. No call errored — pure discoverability overhead for predicting/verifying a
literal target extent.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the arena gameplay-triggers task (focus `volume.create_trigger_sphere`, namespace `volume`, 9 calls, outcome `ergo`; judge filed `B-blocking-volume-no-brush-geometry` #7 for the response-side echo bug). Distinct request-side docs angle: `volume.set_volume_extent`'s `extent` is documented as a bare `"New extent {x,y,z}"` (`VolumeHandler.cpp:1292`) but means an absolute world-unit half-extent for brush volumes (`Cast<ABrush>`→`CreateBoxBrushForVolume` :1314) and a scale×100 for non-brush trigger spheres (`SetActorScale3D(extent/100)` :1318), so `extent={500,500,500}` on a 350-radius `TriggerSphere` produces a `{1750,1750,1750}` bounding half-extent (350×5) — five times the typed value. An agent reading the wiki cannot predict which interpretation applies before calling; the friction note (verbatim above) shows the resulting confusion verifying a literal target extent. Fix (downstream docs): add a `### volume.set_volume_extent` section to `docs/wiki-src/volume.md` stating the class-dependent units (brush = absolute world half-extent; non-brush sphere = scale = extent/100, bounding half-extent = baseRadius×(extent/100)). Same overlay `E-volume-type-filter-discovery`/`E-volume-create-name-vs-volumename`/`E-volume-get-info-no-limit-spills` already target. Distinct from #7 (response-side echo fix) — this gap remains even after the echo is corrected. Sibling pattern of `E-geometry-warp-extent-semantics` (request param meaning) vs `E-geometry-deformer-echo-mesh-counts` (response shape).
- `#2-additional-triggerbox-extent-semantics` `OPEN` reporter — Additional evidence (arena gameplay/atmosphere-volumes task, no seed; `volume` namespace, 13 executing RPCs, outcome tool_bug). Extends this ticket beyond the trigger-SPHERE case to `ATriggerBox`, and **corrects the class model in the body**: per the judge's source replay in `B-blocking-volume-no-brush-geometry` `#8`, `ATriggerBox` is an `ATriggerBase` (NOT an `ABrush`), so `set_volume_extent` takes the **non-brush** `SetActorScale3D(extent/100)` branch (`VolumeHandler.cpp:1318`) too — the same class-dependent-units surprise, but on a class this ticket's body still lists under "brush = absolute world half-extent." Concretely: `set_volume_extent {Arena_AmbushTrigger, extent:{400,120,250}}` echoed `newExtent {400,120,250}` but `get_volumes_info` read back `{160,128,128}`; then `{380,110,240}`→`{152,128,128}`; then `{120,400,220}`→`{128,160,128}`. Pattern = `max(0.4×extent, 128)` per axis, i.e. `SetActorScale3D(extent/100)` applied to the engine-default ~40-unit half-extent box (40×(extent/100)=0.4×extent), then floored at the ~128 editor-Sprite billboard bound (400→scale4→160; 120→scale1.2→48 but floored 128; 250→scale2.5→100 but floored 128). An agent reading the wiki still cannot predict, before calling, that a TriggerBox `extent` becomes `scale×40` with a ~128 floor, so the round-trip fails: the agent burned ~4 extra diagnostic set/get RPCs reverse-engineering the `max(0.4x,128)` rule and had to reorient the corridor onto the one axis whose scaled value clears the 128 floor to make the tighten visible. Friction note verbatim: *"volume.get_volumes_info reports its extent through a lossy transform (~0.4x the set half-extent, floored at 128 per axis) so set extents don't round-trip like the other four volume classes do — it took two diagnostic re-tightens to reverse-engineer the max(0.4x,128) rule and reorient the corridor."* (The CallAnalyzer's proposed ticket blamed `get_volumes_info` as lossy; the judge's `#8` replay shows `get_volumes_info`'s `GetActorBounds()` is the truthful side — the caller's "lossy get" mischaracterization is itself a symptom of this discoverability gap.) Docs fix widened: the `### volume.set_volume_extent` overlay note must cover **all** the `ATriggerBase` trigger classes (box/capsule/sphere) as non-brush `scale=extent/100`, not just the sphere, and state the ~128 editor-Sprite floor for `ATriggerBox`. Severity unchanged (Low, docs discoverability — encounters is a tiebreak, never a severity input); bumped encounters→2.
