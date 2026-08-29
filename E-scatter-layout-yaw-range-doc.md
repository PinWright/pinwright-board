---
id: E-scatter-layout-yaw-range-doc
title: "`spatial.scatter_layout` draws yaw on [0, 360) and returns it normalised to (-180, 180] by FTransform's quaternion round-trip, and no doc states either range — the distribution was measured and is uniform, so this is a documentation defect, not a distribution defect"
status: OPEN
severity: Low
category: ergonomic
tags: [spatial, scatter_layout, yaw, rotation, frotator, quaternion, normalization, docs, response-shape, range-check, test-gap]
encounters: 1
lastSeen: 2026-08-29T00:00:00+05:00
---

# The number is right; the range nobody wrote down is not the range that comes back

`spatial.scatter_layout` draws a full-circle yaw — `Handlers/Spatial/ScatterLayoutHandler.cpp:432-436`:

```cpp
// FRotator is (Pitch, Yaw, Roll): pitch and roll stay literal zeroes here.
const FTransform Point(
    FRotator(0.0, bRandomYaw ? (UnitYaw * 360.0) : 0.0, 0.0),    // :434
    FVector(X, Y, PlaneZ),
    FVector(Scale));
```

`UnitYaw` is `FRandomStream::GetFraction()`, so the drawn value is on `[0, 360)`. It is then stored
in an `FTransform`, i.e. as a quaternion, and the response is built from that —
`Utils/JsonBuilders.cpp:27-34`, the rotation line at `:31`:

```cpp
Obj->SetObjectField(TEXT("rotation"), BuildRotatorJson(Transform.GetRotation().Rotator()));
```

`FQuat::Rotator()` returns yaw in `(-180, 180]`. So a point drawn at 300 degrees comes back as
`-60`. The transform is identical and every consumer that feeds it straight to
`foliage.add_instances` / `actor.spawn_batch` / `actor.set_instance_transforms` is unaffected. Only
a caller that reads the number and range-checks it is bitten — and there is nothing wrong to see
until they check against a range they inferred rather than read.

## The distribution was measured; it is not the defect

`F-scatter-layout-verb`'s `#3` verified this verb live over **34 calls**, including two runs at the
same seed byte-identical across all nine numbers per transform and a full nearest-neighbour bearing
chi-square pass. Yaw is uniform. This ticket asserts nothing about the draw, and a fixer should not
go looking at `ScatterLayoutPointStream`: the values are correct, the range they are reported in is
simply undocumented and is not the one the source constant suggests.

## What the docs actually say — correcting the report as it reached me

I was told yaw is documented as 0-360. **It is not documented at all.** Every statement about yaw on
this verb is qualitative:

- `ScatterLayoutHandler.cpp:201-203`, the `randomYaw` param — *"Yaw each point fully at random
  (default true). Pitch and roll are ALWAYS zero and there is no knob for them"* — served verbatim
  as the `randomYaw` row of `Saved/PinWright/wiki/spatial.scatter_layout.md:17`.
- `Docs/wiki-src/spatial.md:395` — *"pitch and roll are **always** zero and have no knob"*.
- `Docs/wiki-src/level-building.instancing-and-scatter.md:78`, the doctrine the verb implements —
  *"Vary per-instance uniform scale by ~±15% and **yaw fully**"*.
- The response spec, `spatial.md:413`, says only *"Each `transforms[]` entry is `{location,
  rotation, scale}`"*. No component ranges.

So there is no false sentence to correct — there is an absent one, and the absence is what makes the
`(-180, 180]` return surprising. A reader who resolves "fully at random" against `UnitYaw * 360.0`
lands on `[0, 360)`, which is the value the verb draws and not the value it reports. This is a
weaker claim than "the docs say 0-360" and it is the one the tree supports; the defect and the fix
are unchanged.

## Why it got through the suite

`PinWright.spatial.scatter_layout.SameSeedIsByteIdentical`
(`Source/PinWright/Private/Tests/Spatial/TestScatterLayout.cpp`) asserts the sibling components and
not this one. It parses yaw at `:113` and uses it only for cross-run equality (`:182`, and again at
`:478` for the overlapping-regions property). Its range block, `:204-228`, checks pitch and roll are
literal zero (`:209`) and that scale is uniform inside `0.85..1.15` (`:215-216`) — **there is no yaw
assertion of any kind.** Verified before asserting it. That is a reasonable place for the gap: the
test was written against the Rotator trap the doctrine page warns about, which is a pitch/roll
concern. A one-line `Yaw > -180.0 && Yaw <= 180.0` added there pins whichever range the fix chooses.

## What it should do

Document the returned range, in the response spec rather than the `randomYaw` param, since it is
true whether or not `randomYaw` is on: *`rotation.yaw` comes back in `(-180, 180]` — the transform
round-trips through a quaternion, so a drawn 300 is reported as -60; pitch and roll are exactly 0.*
Add the matching assertion to `SameSeedIsByteIdentical`.

Normalising the response to `[0, 360)` instead is the wrong fix: it would make this verb's rotation
field disagree with every other transform response in the plugin, all of which go through
`BuildTransformJson`.

## Distinct from

- `E-control-time-of-day-pitch-misreports-after-normalize` (IN-REVIEW, Low) — **the family
  precedent, and the opposite direction.** There, `environment.control.set_time_of_day` echoes the
  pre-normalization Euler value it *tried* to set (pitch 165) while the actor stores the normalized
  one (15), so the response reports a value the engine did not keep. Here the response reports
  exactly what is stored; it is the documentation that never named the range. Same `FTransform`
  quaternion round-trip, mirrored failure — that one echoes past the normalization, this one
  documents past it. Worth one fixer seeing both, but no shared code: that is
  `EnvironmentHandler.cpp` echoing a local, this is `JsonBuilders::BuildTransformJson` behaving
  correctly and undocumented.
- `B-sequencer-get-binding-transform-rotation-fields-rotated` (DONE, High) — rotation values
  returned in the **wrong fields**. Silent wrong data, correctly High. Here the value is in the
  right field and is the right value; only its range is unstated.
- `E-scatter-layout-maxpoints-clamped-jitter-refused` — the other residual from the same
  verification pass, on the parameter side rather than the response side.

## Dedup

Board-wide search across all statuses for `scatter_layout` and for yaw/rotation-range tickets: the
rotation-adjacent files are `E-control-time-of-day-pitch-misreports-after-normalize`,
`B-sequencer-get-binding-transform-rotation-fields-rotated`, `E-actor-duplicate-no-rotation-scale`
and `E-pwmodel-transform-rotate-pivots-origin`, none of which is about a documented-versus-returned
range on this verb.

## History
- `#1-normalised-range-undocumented` `OPEN` reporter — Re-derived at HEAD. `spatial.scatter_layout` draws yaw as `UnitYaw * 360.0` on `[0, 360)` (`Handlers/Spatial/ScatterLayoutHandler.cpp:434`), builds an `FTransform` from it, and the response goes out through `JsonBuilders::BuildTransformJson` (`Utils/JsonBuilders.cpp:27-34`), whose `:31` reads `Transform.GetRotation().Rotator()` — a quaternion round-trip that returns yaw in `(-180, 180]`. A point drawn at 300 is reported as -60. **Documentation defect, not a distribution defect, and the ticket says so:** `F-scatter-layout-verb`'s `#3` measured this verb over 34 live calls including byte-identical same-seed runs and a nearest-neighbour bearing chi-square; the yaw distribution is uniform and the values are correct. Correcting the report as it reached me: yaw is not documented as 0-360, it is not documented at all. Every statement is qualitative — `ScatterLayoutHandler.cpp:201-203` and the generated `spatial.scatter_layout.md:17` ("Yaw each point fully at random ... Pitch and roll are ALWAYS zero"), `Docs/wiki-src/spatial.md:395`, and the doctrine page `level-building.instancing-and-scatter.md:78` ("yaw fully") — and the response spec at `spatial.md:413` gives no component ranges. So there is no false sentence, only an absent one; a reader who resolves "fully at random" against the `* 360.0` constant infers `[0, 360)` and is wrong about the wire. Weaker claim than reported, same defect and same fix. Why it got through, verified before asserting: `PinWright.spatial.scatter_layout.SameSeedIsByteIdentical` (`Tests/Spatial/TestScatterLayout.cpp`) parses yaw at `:113` and uses it only for cross-run equality (`:182`, `:478`); its range block at `:204-228` asserts pitch/roll are literal zero (`:209`) and scale is uniform in `0.85..1.15` (`:215-216`), and makes **no yaw-range assertion**. Ask: state the returned range in the response spec (not on `randomYaw`, since it holds either way) and add the assertion. Normalising to `[0, 360)` is the wrong fix — it would make this the only transform response in the plugin disagreeing with `BuildTransformJson`. Cross-linked `E-control-time-of-day-pitch-misreports-after-normalize` (IN-REVIEW, Low) as the family precedent in the opposite direction — that verb echoes the pre-normalization value the engine did not keep; this one returns the normalized value the docs did not promise — and `B-sequencer-get-binding-transform-rotation-fields-rotated` (DONE, High), where rotation values land in the wrong fields and are genuinely wrong data. Severity Low, argued rather than defaulted: impact class is Low by the rubric's own wording — a docs gap, cosmetic on the wire, with no blocked task, no wrong value and no workaround needed, since every downstream consumer of `transforms[]` feeds the number straight back into a transform and is unaffected. It bites exactly one caller: the one that range-checks the field. Reach bump declined — `spatial.scatter_layout` is not an almost-every-session verb, and Low is the floor so the rare-path bump-down has nowhere to go.
