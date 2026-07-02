---
id: B-duplicate-along-spline-scalevariation-clobbers-base-scale
title: "geometry.duplicate_along_spline with scaleVariation>0 sets an ABSOLUTE uniform scale near 1.0, silently discarding the source actor's own (possibly non-uniform) scale instead of multiplying it"
status: OPEN
severity: Medium
category: bug
tags: [geometry, duplicate_along_spline, spline, scale, scalevariation, silent-wrong-data, scalevariation-overwrites-base-scale]
encounters: 1
lastSeen: 2026-07-02T03:24:42.1858782+03:00
---

# `geometry.duplicate_along_spline` `scaleVariation` overwrites (not multiplies) the source's base scale — a thin/custom-scaled template comes out as fat uniform ~1.0 copies

`geometry.duplicate_along_spline` is meant to place `count` duplicates of a
source `DynamicMeshActor` along a spline, and `scaleVariation` is documented as
"Random scale variation range" — i.e. it should make each copy *slightly*
larger/smaller than the template so the row reads as natural rather than
mechanically identical. Instead, when `scaleVariation > 0` the handler sets each
copy's scale to an **absolute** uniform `FVector(1 ± scaleVariation)` and never
reads the source actor's scale. So the template's own scale — which is the whole
point of a "make one post to use as the template, then array it" workflow — is
**silently thrown away** the moment you ask for any variation.

The perverse consequence: `scaleVariation = 0` PRESERVES the template scale (the
underlying `DuplicateActor` copies it), but turning the variation feature ON
*destroys* it. A thin bollard prototype (scale `0.35, 0.35, 1`) arrayed with a
little variation comes out as a row of near-1.0 **uniform fat cylinders** — the
opposite of "matching stone bollards." This is exactly the park-bollard task
that seeded this finding.

## Verbatim repro (replayed live via `mcp__pinwright__call` this session)

Source with a non-uniform, non-1.0 scale:
```
geometry.create_cylinder {name:"ThinBollard", radius:15, height:100,
                          location:{x:0,y:0,z:50}, scale:{x:0.35,y:0.35,z:1}}
actor.get_transform {actorName:"ThinBollard"}
  -> {"location":[0,0,50],"rotation":[0,0,0],"scale":[0.35,0.35,1]}
```
Array it along the pre-existing S-curve spline with a little variation:
```
geometry.duplicate_along_spline {actorName:"ThinBollard",
    splineActorName:"GardenWalkSpline", count:4, alignToSpline:true, scaleVariation:0.15}
  -> {"sourceActor":"ThinBollard","splineActor":"GardenWalkSpline",
      "count":4,"splineLength":2143.5068359375,"alignToSpline":true}
```
Read the duplicates back:
```
actor.get_transform {actorName:"ThinBollard_Dup0"}
  -> {..."scale":[0.9431302845478058,0.9431302845478058,0.9431302845478058]}
actor.get_transform {actorName:"ThinBollard_Dup2"}
  -> {..."scale":[1.119301438331604,1.119301438331604,1.119301438331604]}
```
The `(0.35, 0.35, 1)` thin-post proportions are gone; every copy is a **uniform**
scale near 1.0. Control: the same call with `scaleVariation:0` (or a default-scale
source) leaves the copies at the template's scale — confirming it is the
variation branch specifically that clobbers it.

## Guilty source line

`Plugins/PinWright/Source/PinWright/Private/Handlers/Geometry/AdvancedMeshOpsHandler.cpp`, in the placement loop (lines 841-845):
```cpp
if (ScaleVariation > 0.0)
{
    double ScaleFactor = 1.0 + FMath::RandRange(-ScaleVariation, ScaleVariation);
    NewActor->SetActorScale3D(FVector(ScaleFactor));   // line 844: absolute, ignores SourceActor scale
}
```
`FVector(ScaleFactor)` is an absolute uniform scale built from 1.0; it never
consults `SourceActor->GetActorScale3D()`.

## What it should do

Multiply the variation onto the source's base scale, preserving axis proportions:
```cpp
NewActor->SetActorScale3D(SourceActor->GetActorScale3D() * ScaleFactor);
```
(`DuplicateActor` already copies the source scale, so with `scaleVariation == 0`
the current code is fine; only the `>0` branch needs to become multiplicative.)

severity rationale: impact=silent-wrong-data (caller trusts scaleVariation to add
variety to their template; it instead silently resets the template's base/non-uniform
scale — the readback "lies" only via the copies' transforms) × reach=rare (a
specialized geometry array verb, not an every-session method; workaround: pass
scaleVariation:0 and vary scale afterward via actor.set_transform) -> Medium

## History
- `#1-initial-repro` `OPEN` reporter — Seed method `geometry.duplicate_along_spline` (SEED mode). The attempt agent never called the seed (it hand-rolled the array via object.call_function GetTransformAtDistanceAlongSpline + actor.duplicate + actor.set_transform, and its friction wrongly claimed "there is no array-actors-along-spline helper" — the helper exists and works). Interrogating the seed directly surfaced this bug. Replay-confirmed live: source `ThinBollard` scale `(0.35,0.35,1)` arrayed with `scaleVariation:0.15` yields copies at uniform scale `~0.943` and `~1.119` (verbatim above) — the non-uniform source scale is discarded. Root cause: `AdvancedMeshOpsHandler.cpp:844` `SetActorScale3D(FVector(ScaleFactor))` sets an absolute uniform scale built from 1.0 rather than `SourceActor->GetActorScale3D() * ScaleFactor`. The base method otherwise works: 8 evenly-spaced, spline-aligned duplicates placed correctly along the S-curve.
