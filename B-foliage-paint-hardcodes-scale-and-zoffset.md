---
id: B-foliage-paint-hardcodes-scale-and-zoffset
title: "foliage.paint writes DrawScale3D=1 and ZOffset=0 on every instance, never reading the ScaleX/Y/Z or ZOffset ranges of the UFoliageType it resolved — the scale/offset half of the rotation defect just fixed on the same four lines"
status: OPEN
severity: Medium
category: bug
tags: [foliage, paint, scale, zoffset, foliage-type, drawscale3d, hardcoded, engine-parity, vegetation]
encounters: 1
lastSeen: 2026-08-29T22:00:00+03:00
---

# The type carries a scale range; the verb writes 1.0 anyway

`foliage.paint` builds each `FFoliageInstance` and then hardcodes two of its fields
(`Source/PinWright/Private/Handlers/Environment/FoliageHandler.cpp:916-920`):

```cpp
// Still hardcoded, and named as such: the type's ScaleX/Y/Z range and ZOffset range are two
// more fields this verb does not read. They are the scale/offset half of
// B-foliage-paint-ignores-align-to-normal-and-random-yaw and are NOT fixed here.
Instance.DrawScale3D = FVector3f(1.0f);
Instance.ZOffset = 0.0f;
```

The resolved `UFoliageType` carries both. `ScaleX/ScaleY/ScaleZ` are `FFloatInterval`s the
plugin itself writes — `foliage.add_type` sets them from its `minScale` / `maxScale` params
(`FoliageHandler.cpp:201-202`, via `ApplyFoliageScaleAndAlign`) — and `ZOffset` is a second
interval on the same object. Engine painting reads both through
`UFoliageType::GetRandomScale()` in `FPotentialInstance::PlaceInstance`. `foliage.paint`
reads neither.

Same class as the rotation defect: **the verb auto-creates the type, writes a contract onto
it, and then declines to honour it.** A caller who does `foliage.add_type {minScale: 0.5,
maxScale: 1.5}` and then paints with that type gets every instance at exactly 1.0 — the two
parameters they set are silently inert on this path. On the auto-create path the type is built
with the default `ScaleX` interval, so the symptom is the absence of the natural size variation
engine-painted foliage has, and a bed of identically-sized plants is not obviously wrong until
compared against one painted in the editor.

## Filed on the sibling ticket's own instruction

`B-foliage-paint-ignores-align-to-normal-and-random-yaw` (IN-REVIEW, High) fixed the rotation
half of exactly these lines and explicitly refused to absorb this one:

> `DrawScale3D` (hardcoded to `1.0f` at `:464`, while the type carries a `ScaleX/Y/Z` range)
> and `ZOffset` (hardcoded `0.0f` at `:465`) ... say so and the scale/offset half needs its
> own ticket.

Its `#2` fix left both lines in place and replaced the surrounding comment with the one quoted
above, which names this ticket as the owner. Line numbers have moved (`:464-465` → `:919-920`)
because the rotation fix inserted ahead of them.

Related, distinct:
- `B-create-procedural-ignores-scale-and-normal-fields` (the same two fields on
  `foliage.create_procedural`, which *accepts* the keys and writes them to a property that
  path never reads — written-but-unread, not hardcoded).
- `E-foliage-get-instances-drops-scale` — the readback side; it is why measuring this needs a
  direct `FFoliageInfo::Instances` read rather than the verb's own response. **Note it is now
  partly stale**: `foliage.get_instances` publishes `scaleX/scaleY/scaleZ` off
  `Inst.DrawScale3D` at `FoliageHandler.cpp:1398-1400`, so the readback exists and will report
  a flat `1,1,1` here — which makes this ticket cheap to verify.

## Severity

**Medium**, not High. Impact class would be "silent wrong / hardcoded data on a normal path"
(High) except that it is no longer silent: the `foliage.paint` registry description now states
it in the verb's own documentation (`FoliageHandler.cpp:548`) —

> Instance SCALE and ZOffset are still hardcoded to 1 and 0 and do NOT read the type's
> ScaleX/Y/Z or ZOffset ranges. `foliage.add_instances` is the richer literal-placement verb
> (per-instance rotation and scale).

— and it names the workaround in the same breath. That is the rubric's Medium: "doable, but
only via a documented workaround". Reach modifier not applied: `foliage.paint` is a common
verb but not an almost-every-session one, and `add_instances` already serves the callers who
need scale.

**Escalation condition, stated so a re-triage does not have to re-derive it:** if that
disclosure sentence is ever removed from the registration string — or if `foliage.paint` gains
its own `minScale`/`maxScale` params and continues to ignore them — this returns to **High**
on the silent-wrong-data class, matching the rotation half it is the sibling of.

**Workaround:** use `foliage.add_instances`, which takes a per-instance `scale` and writes it
(`FoliageHandler.cpp:1881`, `Instance.DrawScale3D = FVector3f(TransformData.Scale)`), and
sample the type's range yourself. There is no workaround for `ZOffset`.

**Fix:** read the resolved type the way the engine does — `FoliageType->GetRandomScale()` for
`DrawScale3D` and the type's `ZOffset` interval for the offset — at the same site the rotation
fix now reads `RandomYaw` / `RandomPitchAngle` (`:905-915`), so the two halves share one read
of the type. Then drop the disclosure sentence from the registration string and echo the
applied range next to the measured values the way the `rotation` block already does, so the
response proves the type was honoured rather than asserting it.

## History
- `#1-scale-and-zoffset-still-literal` `OPEN` reporter — Source-only, verified against the current working tree (which carries the uncommitted rotation fix from `B-foliage-paint-ignores-align-to-normal-and-random-yaw` `#2`). `FoliageHandler.cpp:919-920` still reads `Instance.DrawScale3D = FVector3f(1.0f); Instance.ZOffset = 0.0f;` under a comment (`:916-918`) that names this exact residue and routes it to a ticket that did not then exist; that comment is the sibling fix's own hand-off. Confirmed the type carries what the verb ignores: `ApplyFoliageScaleAndAlign` writes `FoliageType->ScaleX.Min/.Max` from `foliage.add_type`'s `minScale`/`maxScale` (`:201-202`), so a caller's explicit scale request is inert on the paint path. Confirmed `foliage.add_instances` is a real workaround — it writes `Instance.DrawScale3D` from the caller's per-instance transform (`:1881`) — and that the verb's own registration string discloses the gap (`:548`), which is the whole basis for rating this Medium rather than High; escalation condition recorded above. **No editor call was made and no foliage was painted this pass**: the mechanism is established from source, and the measurement that would confirm it end-to-end is cheap now that `foliage.get_instances` publishes `scaleX/Y/Z` off `DrawScale3D` (`:1398-1400`) — paint through a type with `minScale 0.5 / maxScale 1.5` and assert the readback is not uniformly `1,1,1`. Dedup: greps over the board for `DrawScale3D` and `ZOffset` return only `B-create-procedural-ignores-scale-and-normal-fields` (different verb, written-but-unread rather than hardcoded), `E-foliage-get-instances-drops-scale` (readback side, now partly stale — the fields are published today), `B-foliage-paint-does-no-ground-projection` and the sibling rotation ticket; none owns this pair on `foliage.paint`.
