---
id: B-foliage-paint-alignmaxangle-zero-sentinel
title: "foliage.paint's rotation block emits alignMaxAngleDeg: 0 beside alignedCount: 12, which reads as 'clamped to zero degrees' — 0 is the engine's 'no limit', it is the CDO default and no PinWright verb can set anything else, so the misleading form is the only form the field ever takes"
status: OPEN
severity: Low
category: bug
tags: [foliage, paint, response-shape, sentinel-value, align-to-normal, rotation, legibility, docs, engine-parity, vegetation]
encounters: 1
lastSeen: 2026-08-30T19:00:00+05:00
---

# A sentinel published as a bare number, next to a count that contradicts it

`foliage.paint`'s `rotation` block pairs what the resolved `UFoliageType` asked for with what was
measured. When the type aligns to normal it emits both halves —
`Source/PinWright/Private/Handlers/Environment/FoliageHandler.cpp:1045-1048`, re-derived at plugin
HEAD `1a9e5778`:

```cpp
if (bTypeAlignToNormal) {
  RotationObj->SetNumberField(TEXT("alignMaxAngleDeg"), TypeAlignMaxAngle);
  RotationObj->SetNumberField(TEXT("alignedCount"), AlignedCount);
}
```

Measured on the **13:32 build, plugin commit `d8f1bc32`** (`FoliageHandler.cpp` is byte-identical
at `d8f1bc32` and HEAD `1a9e5778`), 12 instances painted onto the zone-E scree at 10-34 degrees.
Source of record, read directly: `X:/src/unreal/EAContentExamples58/Docs/map/vegetation-agent-brief.md:770-782`,
project commit `7d629ad9`. The block came back

    {randomYaw:true, randomPitchAngleDeg:12, alignToNormal:true, alignedCount:12}

with `alignMaxAngleDeg: 0` beside it, every `placed[]` row carrying a non-zero pitch/yaw/roll and
the clumps **visibly leaning** in the capture. Read literally the block says the maximum
adjustment away from vertical was zero degrees and that all 12 instances were nevertheless
aligned. Those cannot both be true, and the reader has no way from the response to tell which
field to disbelieve.

## Zero is the engine's "no limit" — confirmed, so the number is right and the label is missing

`FFoliageInstance::AlignToNormal`
(`C:/UE_5.8/Engine/Source/Runtime/Foliage/Public/InstancedFoliage.h:109-135`) computes the full
alignment first and only then considers clamping it, under an explicit guard at `:119-120`:

```cpp
// limit the maximum pitch angle if it's > 0.
if (AlignMaxAngle > 0.f)
{
    int32 MaxPitch = static_cast<int32>(AlignMaxAngle);
    ...
}
```

With `AlignMaxAngle == 0` the clamp block is skipped entirely and the instance takes the whole
surface normal. **0 does not mean "capped at 0"; it means "not capped".** The declaration carries
no hint of that: `UFoliageType::AlignMaxAngle`
(`C:/UE_5.8/Engine/Source/Runtime/Foliage/Public/FoliageType.h:226-228`) is documented only as
*"The maximum angle in degrees that foliage instances will be adjusted away from the vertical"*,
with `UIMin = 0, ClampMin = 0, UIMax = 359, ClampMax = 359`. Nothing states the sentinel; it
exists solely in the `> 0.f` guard above.

PinWright's value is a faithful echo, not a bug in the number: `TypeAlignMaxAngle` is read once
off the resolved type at `FoliageHandler.cpp:846` and passed straight to the engine's own routine
at `:978` (`Info->Instances[InstanceIndex].AlignToNormal(GroundNormal, TypeAlignMaxAngle)`) — the
same argument the engine's own painter passes at
`C:/UE_5.8/Engine/Source/Runtime/Foliage/Private/InstancedFoliage.cpp:5548`. So the fix is
labelling, never changing the value.

## The misleading form is the only form the field can take

Not an edge case, and this is the part that decides the ticket is worth filing:

- `AlignMaxAngle = 0.0f` is the **CDO default**, set in `UFoliageType::UFoliageType`
  (`InstancedFoliage.cpp:580`) at `:594` — three lines after `AlignToNormal = true;` at `:585`.
  So the shipped default type both aligns and has no limit, which is exactly the combination that
  produces the contradictory-looking pair.
- **No PinWright verb writes it.** A whole-`Source/` grep for `AlignMaxAngle` returns three hits,
  all in `FoliageHandler.cpp`: the read at `:846`, the apply at `:978` and the emit at `:1046`.
  `foliage.add_type` has no parameter for it (its `ApplyFoliageScaleAndAlign` sets scale and
  `AlignToNormal` only). Every foliage type PinWright creates therefore carries 0, and every
  `foliage.paint` response with `alignToNormal: true` emits `alignMaxAngleDeg: 0`.

So there is no configuration reachable through the plugin in which this field reads sensibly.

## The natural remedy makes the output worse

Stated because it is what lifts this above cosmetic. A caller who reads `alignMaxAngleDeg: 0` as a
clamp and "fixes" it — opening the type and setting a non-zero maximum, or asking for the value to
be raised — **introduces a pitch clamp that was not there**, and the foliage stops following the
ground it was following correctly. The misreading's obvious repair is a regression, and nothing in
the response or the docs would catch it.

## The docs name the field without explaining it

`Docs/wiki-src/foliage.md:172-173` (generated `Saved/PinWright/wiki/foliage.paint.md:62`):

> The `rotation` block echoes what the type asked for (`randomYaw`, `randomPitchAngleDeg`,
> `alignToNormal`, `alignMaxAngleDeg`) next to the **measured** `alignedCount` [...]

Correct, and silent on the sentinel. A caller who goes to the docs to resolve the contradiction
finds the field listed and nothing more, so the only way through is the engine header.

## Fix

Keep the number — it is the type's value, and the block's whole design is "echo what the type
asked for next to what was measured" (`FoliageHandler.cpp:1038-1039`), so replacing 0 with `null`
would break that contract in order to fix a label. Add the meaning beside it, in the same
disclosure style the block already uses:

```cpp
RotationObj->SetNumberField(TEXT("alignMaxAngleDeg"), TypeAlignMaxAngle);
RotationObj->SetBoolField(TEXT("alignMaxAngleClamps"), TypeAlignMaxAngle > 0.f);
```

`alignMaxAngleClamps: false` next to `alignMaxAngleDeg: 0` is unambiguous and matches the guard
the engine actually evaluates, so the two can never drift. Then add one clause to
`Docs/wiki-src/foliage.md:173` saying 0 is the engine's "no limit" rather than a zero cap.

## Filed rather than folded into the ticket whose fix added the field

`B-foliage-paint-ignores-align-to-normal-and-random-yaw` (IN-REVIEW, High) introduced the
`rotation` block at its `#2` — *"a new `rotation` block echoes what the type asked for next to the
MEASURED `alignedCount`"* — so this is a legibility defect in that fix's own new reporting.
It is filed separately on that ticket family's own precedent:
`B-capture-open-level-pose-params-photograph-stale-grass` `#4` found a defective field in the
block its `#2` had just added and carved it out to `B-capture-grass-instances-always-zero` rather
than reopening, on the reasoning that *"a wrong density number in the reporting `#2` added is a
different defect, and reopening here would put two tickets on one fact"*. The same applies here,
with the extra reason that the align ticket's tester scope is explicitly *"the rotation half
only"* (its `#3`) and its `DONE` should not be held on a label.

## Same shape as — an inversion of the session's recurring class

The class stated on `B-foliage-paint-does-no-ground-projection` § *Same shape as* is *the call
succeeds, every number it reports is correct, and the output is wrong because the deciding number
was never reported.* This is that class turned inside out: the deciding number **is** reported and
**is** correct, and the reader draws the opposite conclusion from it, because the number is a
sentinel published without its meaning. Recorded as an inversion rather than a member so a fixer
does not go looking for a missing measurement — there isn't one.

Nearest sibling on the board: `B-capture-grass-instances-always-zero` (OPEN, High) — also a field
in a rotation-block-style disclosure that always reads 0. It is rated far higher because there the
0 is **false** (grass exists and is not counted); here the 0 is **true** and only reads as its own
opposite. That difference is the whole gap between High and Low, and is why they are separate
tickets rather than one.

## Dedup

Board-wide greps for `alignMaxAngleDeg` and `AlignMaxAngle` return exactly two files —
`B-foliage-paint-ignores-align-to-normal-and-random-yaw` (which added the field) and
`B-foliage-paint-does-no-ground-projection` (which cites the engine's `Settings->AlignMaxAngle` at
`InstancedFoliage.cpp:5546-5549` while establishing that `foliage.paint` never reached
`PlaceInstance` at all). Neither says anything about the value 0 or how it reads. The other
`foliage.paint` tickets — `B-foliage-paint-hardcodes-scale-and-zoffset`,
`B-foliage-paint-does-no-ground-projection` (DONE) — are about fields the verb does not read, not
about a field it reports ambiguously. **Nothing on the board covers a sentinel value published
without its meaning, in any namespace.**

## Severity

**Low.** The rubric's Low band is *"pure friction. Docs, discoverability, naming, a response spill
that only forces a `Read`, or cosmetic."* This is naming: the value written is correct, the
alignment performed is correct, and the 12 instances are on the ground at the angles they should
be. Nothing needs re-doing; a reader needs one label.

**Medium argued and declined.** The Medium band includes *"doable, but only via [...] a source
dive"*, and resolving this contradiction does require reading
`InstancedFoliage.h:120` — the docs do not carry it and the response cannot. Declined because the
*task* needs no dive: the paint already did the right thing, and a caller who never reads the
field is never harmed. Only interpreting the receipt costs the dive, which is the Low band's
subject.

**Escalation condition, so a re-triage does not re-derive it:** if `foliage.add_type` (or
`foliage.paint`) ever gains an `alignMaxAngleDeg` parameter while the response keeps publishing
the bare number, this becomes wrong-data-shaped and moves to **Medium or High** — a caller passing
0 to mean "no limit" and a caller reading 0 back as "clamped" would then be looking at the same
number with opposite meanings on the same wire.

**Reach modifier declined in both directions, and named.** No bump up: `foliage.paint` is a common
vegetation verb but not an almost-every-session one across the plugin's surface. No bump down —
and this is the argument against dropping the ticket: it is not a rare edge path, because
`AlignMaxAngle` is 0 in the CDO (`InstancedFoliage.cpp:594`) and no PinWright verb can set it to
anything else, so **every** aligned paint the plugin can produce emits the misleading pair. The
frequency is 100% of the cases the field appears in; only the severity of each case is small.

## History
- `#1-zero-sentinel-published-without-its-meaning` `OPEN` reporter — `foliage.paint` emits `alignMaxAngleDeg` from the resolved type at `FoliageHandler.cpp:1046`, inside `if (bTypeAlignToNormal)` at `:1045`, immediately beside `alignedCount` at `:1047`. Measured on the **13:32 build (`d8f1bc32`; `FoliageHandler.cpp` is byte-identical at `d8f1bc32` and HEAD `1a9e5778`)**, source of record `X:/src/unreal/EAContentExamples58/Docs/map/vegetation-agent-brief.md:770-782` at project commit `7d629ad9`, read directly: 12 instances on the zone-E scree (10-34 deg) returned `{randomYaw:true, randomPitchAngleDeg:12, alignToNormal:true, alignedCount:12}` with `alignMaxAngleDeg: 0`, every `placed[]` row carrying non-zero pitch/yaw/roll and the clumps visibly leaning in the capture. **Confirmed in the engine that 0 means "no limit", which is the check this ticket turned on and would have been dropped without:** `FFoliageInstance::AlignToNormal` (`InstancedFoliage.h:109-135`) computes the full alignment and only then clamps, under `// limit the maximum pitch angle if it's > 0.` / `if (AlignMaxAngle > 0.f)` at `:119-120`, so at 0 the clamp block is skipped and the instance takes the whole normal; `UFoliageType::AlignMaxAngle` (`FoliageType.h:226-228`) is documented only as "The maximum angle in degrees that foliage instances will be adjusted away from the vertical" with `ClampMin = 0`, stating the sentinel nowhere. PinWright's number is a faithful echo and must not change: read once at `FoliageHandler.cpp:846`, passed to the engine's routine at `:978`, the same argument the engine's own painter passes at `InstancedFoliage.cpp:5548`. **The misleading form is the only form:** `AlignMaxAngle = 0.0f` is the CDO default (`UFoliageType::UFoliageType`, `InstancedFoliage.cpp:580`, the assignment at `:594`, three lines after `AlignToNormal = true;` at `:585`), and a whole-`Source/` grep for `AlignMaxAngle` returns three hits, all in `FoliageHandler.cpp` (`:846` read, `:978` apply, `:1046` emit) — no verb writes it, `foliage.add_type` has no parameter for it — so every aligned paint the plugin can produce emits `alignMaxAngleDeg: 0`. The docs name the field without explaining it (`Docs/wiki-src/foliage.md:172-173`, generated `foliage.paint.md:62`), so the only route to the meaning is the engine header. Recorded as what lifts this above cosmetic: the misreading's natural repair — setting a non-zero maximum on the type — **introduces** a pitch clamp that was not there, so the obvious fix is a regression. Fix is a label, not a value: add `alignMaxAngleClamps: (TypeAlignMaxAngle > 0.f)` beside the number, mirroring the guard the engine evaluates so the two cannot drift, and one clause in `Docs/wiki-src/foliage.md:173`. Filed separately rather than folded into `B-foliage-paint-ignores-align-to-normal-and-random-yaw` (IN-REVIEW, High), whose `#2` added this block, on that family's own precedent — `B-capture-open-level-pose-params-photograph-stale-grass` `#4` carved a defective field out of the block its `#2` had just added rather than reopening, "a wrong density number in the reporting `#2` added is a different defect" — and because that ticket's tester scope is explicitly the rotation half only (its `#3`). Rated **Low** on the naming band with Medium argued and declined (the task needs no source dive, only the receipt's interpretation does) and the escalation condition named: if `alignMaxAngleDeg` ever becomes an input while the bare number is still published, 0 carries opposite meanings on the same wire and this moves up. Reach declined both ways, with the no-bump-down argued explicitly since it is the reason not to drop the item: 100% of the cases the field appears in are the misleading case. Dedup: board greps for `alignMaxAngleDeg` / `AlignMaxAngle` return only the align ticket that added the field and `B-foliage-paint-does-no-ground-projection`, neither of which mentions the value 0 or how it reads; nothing on the board covers a sentinel published without its meaning in any namespace. Cross-linked to `B-capture-grass-instances-always-zero` as the near sibling and the contrast that explains the rating gap — there the 0 is false, here it is true and only reads as its own opposite.
