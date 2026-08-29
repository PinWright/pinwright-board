---
id: B-pcg-spawner-mixed-mesh-heights-silently-misscaled
title: "A PCGStaticMeshSpawner's weighted entries carry no per-entry scale, so one upstream TransformPoints range serves every mesh in a band — zone D shipped six 157 uu snags inside a range authored for 1300 uu trees, and no PinWright response reports the condition even though pcg.generate already walks the components that would prove it"
status: OPEN
severity: Medium
category: bug
tags: [pcg, generate, static-mesh-spawner, mesh-selector, weighted, transform-points, scale, silent-wrong-output, missing-warning, lint, vegetation]
encounters: 1
lastSeen: 2026-08-29T18:00:00+05:00
---

# One scale range, many mesh heights, no diagnostic

A `PCGStaticMeshSpawner`'s weighted entries choose *which* mesh a point gets. Scale comes from a
`TransformPoints` node upstream, which is one range for the whole band. So the moment a band's
entries differ meaningfully in mesh height, some of them are wrong by construction, and every
surface — the engine's, and PinWright's — is silent about it.

**Scope this correctly before reading further.** The absent per-entry scale is an engine structural
fact and PinWright cannot add a field to an engine `USTRUCT`. What is in scope, and what this ticket
asks for, is that PinWright's PCG surface lets a caller author this graph, execute it, and read back
a full success with no hint the condition exists — while already holding, in `pcg.generate`'s own
readback loop, the components that would prove it.

## Measured

Zone D of `PW_VegetationTest`, a 44-node graph with six `PCGStaticMeshSpawner` nodes
(`Docs/map/vegetation-style-split.md` § Zone D):

- **`TransformPoints_0`, the exposed-crest band, range 0.65–1.05.** Authored for ~1000–1400 uu
  stylised trees. One entry in that band was `SM_Dead_Tree`, a **157 uu** mesh. The six instances it
  drew came out at **102–165 uu** — 1-metre twigs standing in a crest band of 13-metre trees. This
  was a live defect in the shipped level *before* the re-speciation pass, not one that pass
  introduced; the pass found it by looking at the frame. Corrected to 8.0–14.5, which puts the same
  mesh at 1266–2168 uu.
- **`TransformPoints_4`, the shrub band, range 0.70–1.40.** Sized for `SM_Bush_Radiant`, a 344 uu
  mesh. Re-speciating the entries to Bird-of-Paradise variants (139–198 uu) halved the layer's
  height and the ridge read as bare (`SPL_A2_D1_eye_ridge.png`) until the range moved to 1.05–2.10.

Ratio in the first case is roughly **8x**. Nothing in any `pcg.generate` response distinguished the
before state from the after state — see `B-pcg-generate-instancecount-blind-to-species`, where the
same four generations returned an identical `instanceCount`.

## Mechanism — the entry has nowhere to put a scale, and this is deliberate in the engine

`FPCGMeshSelectorWeightedEntry`
(`C:/UE_5.8/Engine/Plugins/PCG/Source/PCG/Public/MeshSelectors/PCGMeshSelectorWeighted.h:23-24`)
has exactly two live members:

```cpp
UPROPERTY(EditAnywhere, Category = Settings)
FPCGSoftISMComponentDescriptor Descriptor;   // :35-36
int Weight = 1;                              // :38-39
```

Everything else on the struct (`:41-69`) is inside `#if WITH_EDITORONLY_DATA` and is either a
transient `DisplayName` or a `_DEPRECATED` upgrade field.

The descriptor has no transform either, and the engine's own type hierarchy says the omission is
intentional. `FPCGSoftISMComponentDescriptor`
(`C:/UE_5.8/Engine/Plugins/PCG/Source/PCG/Public/MeshSelectors/PCGISMDescriptor.h:12-13`) adds only
`ComponentTags` (`:24-25`) and `AdditionalCommaSeparatedTags` (`:27-28`) to
`FSoftISMComponentDescriptor`
(`C:/UE_5.8/Engine/Source/Runtime/Engine/Public/ISMPartition/ISMComponentDescriptor.h:308`), which
derives from `FISMComponentDescriptorBase` (`:18`) — no transform anywhere on that chain. The **one**
member of the family that carries a transform is the hard-reference sibling `FISMComponentDescriptor`
(`:259`), at `:300`:

```cpp
UPROPERTY(EditAnywhere, Category = "Component Settings", meta = (DisplayAfter = "ComponentClass"))
FTransform LocalTransform = FTransform::Identity;
```

PCG's soft descriptor does not inherit it. So per-entry scale is not an oversight PinWright can
patch around — a weighted mesh entry genuinely cannot express one.

Scale therefore comes from `UPCGTransformPointsSettings`
(`C:/UE_5.8/Engine/Plugins/PCG/Source/PCG/Public/Elements/PCGTransformPoints.h:12-13`), whose
`ScaleMin` / `ScaleMax` (`:67-71`) are a single range applied to the point set *before* the spawner
sees it — i.e. before the mesh for each point has been chosen. The ordering is what makes it
unfixable at the caller's level: the range is committed one node upstream of the decision it would
need to depend on.

## The engine does not warn, and it holds the data that would let it

`UPCGStaticMeshSpawnerSettings`
(`C:/UE_5.8/Engine/Plugins/PCG/Source/PCG/Public/Elements/PCGStaticMeshSpawner.h:26-27`) has no
scale field and no height diagnostic. Its entire warning surface, enumerated in
`C:/UE_5.8/Engine/Plugins/PCG/Source/PCG/Private/Elements/PCGStaticMeshSpawner.cpp`, is: invalid
selector `:287`, identical spawn `:354` / `:472`, first-input-only `:384`, invalid input/actor
`:400` / `:408` / `:416`, attribute overwrite `:505`, load failures `:757` / `:774`. **Nothing
inspects mesh bounds.** The only bounds-aware member is `bApplyMeshBoundsToPoints`
(`PCGStaticMeshSpawner.h:104-105`), and its own comment says it *writes* bounds onto points; it does
not compare them:

```cpp
/** Sets the BoundsMin and BoundsMax attributes of each point to reflect the StaticMesh spawned at its location */
bool bApplyMeshBoundsToPoints = true;
```

That flag defaults **true**, which means the element already resolves and touches every entry's mesh
bounds on the normal path. The comparison is a subtraction away from data it is already computing.

## What PinWright can do, and where it goes

The fix is a **diagnostic, not a scale field**. Two candidate homes, both cheap because the traversal
already exists:

1. **`pcg.generate` `warnings[]`** — the readback already walks every managed ISM component
   (`Plugins/PinWright/Source/PinWrightPCG/Private/Handlers/PCG/PCGGenerateReadback.h:154-174`,
   with the component in hand at `:156`). Reading `GetStaticMesh()->GetBounds()` per component and
   emitting a warning when the tallest and shortest mesh a single spawner produced differ by more
   than a threshold costs one extra read in a loop that already runs. This composes with
   `B-pcg-generate-instancecount-blind-to-species`'s `instancesByMesh`: once the response names
   meshes, the height spread is derivable by the caller even without the warning, and the warning
   becomes the ergonomic layer on top rather than the only signal.
2. **A static check at author time** — the same comparison over a graph's `MeshEntries` without
   executing anything, on `pcg.inspect` or a `pcg.lint`-shaped verb. Strictly better, because it
   fires before the level acquires the wrong geometry rather than after, and because it can see
   entries whose weight happened to draw zero points this run.

Threshold: state it rather than tune it silently. A ratio of ~1.5x between the tallest and shortest
mesh in one spawner is where a shared range starts to be visibly wrong for one of them; zone D's
case was ~8x. Whatever number is chosen belongs in the warning text, so a caller who disagrees knows
what was measured.

Whichever lands, the wiki should say plainly on the PCG page what the engine does not: **entries in
one weighted spawner share a scale range, so mixing mesh heights in one band is a defect the graph
cannot express its way out of** — split the band, or normalise the meshes.

## Same shape as

`B-pcg-generate-instancecount-blind-to-species` (OPEN, High) — same verb, same response, same
silence, about species instead of scale, and the same loop is the natural place to fix both. Read
together they say: `pcg.generate` reports how *many* things exist and nothing about *what* they are
or *how big* they are.

The session's recurring class (`B-foliage-paint-does-no-ground-projection` § *Same shape as*): the
call succeeds, every number it reports is correct, and the output is wrong because the deciding
number was never reported. Here the deciding number is the ratio between the tallest and shortest
mesh in one band, and the shipped consequence is six 1-metre trees that survived into a level review.

## Severity

**Medium.** Impact class is the rubric's Medium band — *"a readback omits a field and forces a
fallback"*. The fallback is real and was used: walking the actor's ISM components in
`python.execute` and reading mesh bounds per component, which is how the defect was found at all.

**Deliberately not High**, and the distinction is worth stating because the sibling ticket *is*
High. High requires the caller to trust a result that is a lie. Here nothing in any PinWright
response is false and no PinWright document nominates a field that would catch this — there is no
over-claim to point at, only an absence. The wrong scale is also authored by the caller's own
`TransformPoints` range rather than written by a PinWright verb. What PinWright owns is the missing
diagnostic, and a missing diagnostic on a condition the engine also declines to diagnose is the
Medium band, not the High one. A reviewer who reads the shipped six twigs as "the caller trusted the
tool and built on it" would rate this High; the counter-argument is that the tool never said
anything about scale in the first place.

**Reach modifier declined in both directions.** Not an every-session path: it needs a weighted
spawner with entries whose meshes differ substantially in height, which is a real and common
authoring pattern but not a universal one. Not a rare edge path either: `UPCGMeshSelectorWeighted`
is what `PCGStaticMeshSpawner` ships with, mixed-height bands are the normal way to author a
vegetation tier, and this graph hit it in two bands out of six. Medium stands unmodified.

## History
- `#1-one-range-serves-every-mesh` `OPEN` reporter — Found during the zone D re-speciation of
  `PW_VegetationTest` (`Docs/map/vegetation-style-split.md` § Zone D, § Findings 3). Zone D's
  exposed-crest band ran `TransformPoints_0` at 0.65–1.05, authored for ~1000–1400 uu stylised
  trees, with `SM_Dead_Tree` (a 157 uu mesh) among its weighted entries: the six instances it drew
  measured **102–165 uu**, and were shipped in the level before this pass, found by looking at the
  frame rather than by any response. Corrected to 8.0–14.5 (1266–2168 uu). Second instance in the
  same graph: `TransformPoints_4` at 0.70–1.40 was sized for `SM_Bush_Radiant` (344 uu) and the
  Bird-of-Paradise replacements are 139–198 uu, so the shrub layer lost half its height until the
  range moved to 1.05–2.10. MECHANISM (engine source, UE 5.8): `FPCGMeshSelectorWeightedEntry`
  (`PCGMeshSelectorWeighted.h:23-24`) has exactly two live members, `Descriptor` (`:35-36`) and
  `Weight` (`:38-39`) — everything else on the struct is editor-only transient or `_DEPRECATED`; and
  the descriptor chain carries no transform (`FPCGSoftISMComponentDescriptor`
  `PCGISMDescriptor.h:12-13` → `FSoftISMComponentDescriptor` `ISMComponentDescriptor.h:308` →
  `FISMComponentDescriptorBase` `:18`), while the **hard**-reference sibling `FISMComponentDescriptor`
  (`:259`) does carry `FTransform LocalTransform` at `:300` and PCG's soft descriptor does not
  inherit it. So the omission is deliberate in the engine and per-entry scale cannot be expressed;
  scale comes from `UPCGTransformPointsSettings::ScaleMin`/`ScaleMax`
  (`PCGTransformPoints.h:67-71`), one range applied a node **upstream** of the mesh choice it would
  need to depend on. ENGINE ALSO SILENT: `UPCGStaticMeshSpawnerSettings`
  (`PCGStaticMeshSpawner.h:26-27`) has no scale field and its whole warning surface
  (`PCGStaticMeshSpawner.cpp:287`, `:354`, `:384`, `:400`, `:408`, `:416`, `:472`, `:505`, `:757`,
  `:774`) never inspects bounds — though `bApplyMeshBoundsToPoints` (`:104-105`) defaults true and
  already resolves every entry's mesh bounds on the normal path, so the comparison is a subtraction
  away from data it computes anyway. SCOPED DELIBERATELY: PinWright cannot add a field to an engine
  `USTRUCT`, so the ask is a **diagnostic**, in either of two homes — a `pcg.generate` `warnings[]`
  entry built in the readback loop that already holds every managed ISM component
  (`PCGGenerateReadback.h:154-174`, component at `:156`), or, better, a static author-time check
  over `MeshEntries` on `pcg.inspect`/a lint verb, which fires before the level acquires the
  geometry and can see entries that drew zero points. Threshold proposed at ~1.5x tallest:shortest
  and asked to be printed in the warning text (zone D's case was ~8x). Composes with
  `B-pcg-generate-instancecount-blind-to-species` (OPEN, High): once the response names meshes the
  spread is caller-derivable, and the warning becomes ergonomics rather than the only signal.
  Rated **Medium** on the rubric's readback-omission band, with the fallback named (a
  `python.execute` walk of the ISM components reading mesh bounds, which is how this was found);
  **explicitly not High** even though the sibling ticket is, because no PinWright response is false
  here and no PinWright document nominates a field that would catch it — there is no over-claim,
  only an absence, and the wrong scale is authored by the caller's own `TransformPoints` range. The
  counter-reading (the shipped twigs mean the caller trusted the tool) is recorded in the body so a
  reviewer can take it. Reach declined both ways: mixed-height bands are a normal authoring pattern
  but not universal, and `UPCGMeshSelectorWeighted` is the shipped default selector, so this is
  neither every-session nor an edge path.
