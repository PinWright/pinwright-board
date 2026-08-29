---
id: B-pcg-generate-instancecount-blind-to-species
title: "pcg.generate's instanceCount cannot see a species change — four materially different graph edits returned byte-identical 26664 because a weighted mesh selector partitions a fixed point set, while the wiki tells callers instanceCount is the number to judge a generation by and the response names no mesh anywhere"
status: OPEN
severity: High
category: bug
tags: [pcg, generate, readback, silent-wrong-verification, mesh-selector, weighted, instance-count, deciding-number-unreported, vegetation, wiki-overclaim]
encounters: 1
lastSeen: 2026-08-29T18:00:00+05:00
---

# The field that proves a generation ran is the field a caller uses to prove a generation is right, and it cannot tell those apart

`pcg.generate` reports `instanceCount`. The generated wiki page tells the caller, in bold, that
this is **the** number to judge a generation by. It is a correct answer to "did the graph put
geometry in the world" and it is structurally incapable of answering "did the graph put the
geometry I asked for in the world" — and there is no other field in the response that can, because
the payload never names a mesh.

## Measured

Zone D of `PW_VegetationTest` was re-speciated (`Docs/map/vegetation-style-split.md` § Zone D and
§ Findings 1): a 44-node `UPCGGraph` with six `PCGStaticMeshSpawner` nodes, whose weighted mesh
entries were rewritten from a stylised palette to a photoreal one. Four separate, materially
different edits, each followed by `pcg.generate`:

| edit | `instanceCount` | `instancedComponentCount` |
|---|---:|---:|
| baseline | 26664 | 19 |
| three spawners re-meshed | **26664** | 18 |
| a whole species dropped from a band | **26664** | 16 |
| weights changed | **26664** | 15 |

`instanceCount` is **byte-identical across all four**. The independent census that showed the edits
had landed walked the actor's ISM components in `python.execute`: canopy trees went from 280 (of
which 214 stylised) to 291 (of which **0** stylised), `HillTree_P2` 60 → 256, `SM_Dead_Tree` 6 →
35, groundcover from 6857 untextured grass quads to 23 093 Nordic instances. Every one of those is
invisible to the verb's own numbers.

The only field that moved at all is `instancedComponentCount` (19 → 18 → 16 → 15), which counts
*components*, not species — it moves when the mesh set's cardinality changes and stays put when one
mesh is swapped for another, so it is a coincidence detector, not a verification surface.

## Mechanism — why the number cannot move, and why that is correct engine behaviour

The graph's spawners use `UPCGMeshSelectorWeighted`
(`C:/UE_5.8/Engine/Plugins/PCG/Source/PCG/Public/MeshSelectors/PCGMeshSelectorWeighted.h:73-74`,
entries at `:100`). Its selection loop is bounded by the **input point count**, not by anything
weight-derived:

```cpp
while (CurrentPointIndex < InPointData->GetNumPoints())
```
`C:/UE_5.8/Engine/Plugins/PCG/Source/PCG/Private/MeshSelectors/PCGMeshSelectorWeighted.cpp:215`

Each point draws exactly one bucket from the cumulative weight table built at `:151-152`
(`RandomWeightedPick` at `:231-234`) and emits exactly one instance at `:243`
(`InstanceList.Instances.Emplace(InTransform);`). **One point in, one instance out.** Weights
*partition* the point set; they cannot resize it. Neither can swapping which mesh a bucket names.
So for any edit confined to `MeshEntries`, the summed instance count is an invariant of the
upstream sampler, and `26664` is the sampler's number being reported four times.

Nothing here is a bug in PCG. The bug is that the plugin publishes this invariant as the caller's
verification signal.

## The readback discards the identity it is already holding

`FGenerationReadback` walks the component's managed resources and, for each
`UPCGManagedISMComponent`
(`Plugins/PinWright/Source/PinWrightPCG/Private/Handlers/PCG/PCGGenerateReadback.h:156`), does
exactly two things:

```cpp
++Out.InstancedComponentCount;                                    // :161
Out.InstanceCount += static_cast<int64>(ISMC->GetInstanceCount()); // :162
```

It has the `UInstancedStaticMeshComponent*` in hand and never calls `GetStaticMesh()`. Emission is
`PCGGenerateReadback.h:283-284`. `BuildGenerationPayload` (`:243-319`) emits `actor`, `graph`,
`graphPath`, `graphOutputAvailable`, `pointCount`, `dataCount`, `resourceCountsAvailable`,
`instanceCount`, `instancedComponentCount`, `spawnedActorCount`, `partitioned`,
`localComponentsWalked`, `localComponentCount`, `completionSignal`, `elapsedSeconds`, `warnings[]`
— **sixteen fields, none of which names a mesh**. The per-mesh identity is discarded one line
before it would have been free.

## The documentation claim, and where exactly it over-reaches

`Saved/PinWright/wiki/pcg.generate.md:26` (source overlay:
`Plugins/PinWright/Docs/wiki-src/pcg.md:65`):

> **`instanceCount` is the number to judge a generation by.** It counts the ISM/HISM instances PCG
> actually spawned and still manages (read from the component's managed-resource list), so it is
> non-zero exactly when the graph put geometry in the world.

Read the two halves separately, because only one of them is wrong. The **justification** — "non-zero
exactly when the graph put geometry in the world" — is true, and is precisely the property
`B-pcg-generated-graph-output-empty-after-generate` was fixed to deliver. The **headline** —
"the number to judge a generation by" — generalises past its own evidence, from a non-zero test to
a correctness test, in the same sentence. The namespace page repeats the short form
(`Saved/PinWright/wiki/pcg.md:45`, source `Docs/wiki-src/pcg.md:41`): *"judge it by
`instanceCount`"*.

**The C++ is honest and the overlay is not**, which localises the doc half of the fix.
`PCGGenerateReadback.h:275` reads *"Use instanceCount / spawnedActorCount to judge whether the
generation did work"* — "whether it did work" is exactly the claim the mechanism supports. The
registered handler description (`PCGGenerateHandler.cpp:111`) is equally careful: *"instanceCount
(ISM/HISM instances actually spawned)"*. Only the hand-authored overlay sentence promotes it.

## Follow-on, not a regression

`B-pcg-generated-graph-output-empty-after-generate` (DONE, Medium) is the ticket that introduced
`instanceCount`, replacing a `pointCount` that reported `0` for a pass that spawned 97 instances.
Its stated goal — *"the response should not be able to report `0` for a pass that spawned
instances"* — is met and is not in question here. This is the next question asked of the same
field: it can distinguish *something* from *nothing* and cannot distinguish *what* from *what
else*. The fix for that ticket walked past the answer, since `AccumulateManagedResourceCounts`
already visits every component.

## Fix

Add a per-mesh breakdown to the payload, keyed by mesh object path:

```json
"instancesByMesh": { "/Game/…/HillTree_P2.HillTree_P2": 256, "/Game/…/SM_Dead_Tree.SM_Dead_Tree": 35 }
```

The data is one call away inside the existing loop (`PCGGenerateReadback.h:156-163`:
`ISMC->GetStaticMesh()`), so this is an accumulate-into-a-map change, not a new traversal. Two
details worth deciding rather than discovering:

- **Cap and spill.** A graph with many meshes should obey the same oversized-echo policy the rest
  of the surface uses; `instancesByMesh` is bounded by distinct meshes, not by instances, so the
  natural cap is small.
- **Null mesh.** A managed ISM with no static mesh should get an explicit key rather than being
  dropped, since "a component spawned with no mesh" is itself a finding.

Then correct `Docs/wiki-src/pcg.md:65` and `:41` to say what the C++ already says — `instanceCount`
answers *whether*, `instancesByMesh` answers *what* — and, until the field exists, say plainly that
a species or weight change must be verified by walking the actor's ISM components.

## Same shape as

The session's recurring class, stated on `B-foliage-paint-does-no-ground-projection`: *the call
succeeds, every number it reports is correct, and the output is wrong because the deciding number
was never reported.* This is the variant where the unreported number is not merely absent from the
response but is **contradicted by the documentation**, which nominates a different, invariant number
as the one to trust.

Nearest siblings, all in this class and all different mechanisms:

- `B-foliage-remove-empties-ledger-not-component` (OPEN, Critical) — the deciding number is
  unreadable through any verb in the namespace.
- `B-wiki-hism-recipe-resets-actor-transform` (OPEN, Medium) — the deciding number is the actor's
  location, and a green `placed: 41` from an unrelated verb confirms the damage.
- `B-pcg-spawner-mixed-mesh-heights-silently-misscaled` (OPEN) — same verb, same response, same
  silence, about scale instead of species.

## Severity

**High.** Impact class is the rubric's High band — *silent wrong data on a normal path, where the
caller trusts a result and builds on it*. The subtlety worth stating, because a reviewer will
reach for it: no field in the response is false. `instanceCount` is arithmetically correct on every
one of the four calls. What is false is the **shipped instruction about how to read it**
(`pcg.generate.md:26`), and a caller who follows the plugin's own documentation gets a confident
confirmation of four graph edits that field could not have detected. A doc that converts a correct
number into a wrong conclusion is the High band's failure with an extra step, not a Low-band
docs item — the same reasoning `B-wiki-hism-recipe-resets-actor-transform` used to file a wiki
defect as `B-` rather than `E-`.

**Not Critical**: nothing crashes and nothing is corrupted or lost. A wrongly-verified generation is
re-runnable and the graph asset is intact.

**Reach modifier declined in both directions, and named.** No bump up: `pcg.generate` is the PCG
namespace's only execution verb, so every PCG session hits it, but PCG authoring is not itself an
every-session activity across the plugin's surface — and an up-bump from High lands on Critical,
which the rubric reserves for crashes and data loss. No bump down: this is not an edge path. It is
the default path for the default mesh selector (`UPCGMeshSelectorWeighted` is what
`PCGStaticMeshSpawner` ships with), and it fires on *every* edit confined to mesh entries, which is
the most common kind of PCG edit there is. High stands unmodified.

## History
- `#1-instancecount-invariant-under-species-change` `OPEN` reporter — Measured during the zone C/D
  re-speciation of `PW_VegetationTest` (`Docs/map/vegetation-style-split.md` § Zone D, § Findings
  1): four materially different edits to one 44-node graph's `PCGStaticMeshSpawner` mesh entries —
  three spawners re-meshed, a whole species dropped, weights changed — each followed by
  `pcg.generate`, all four returning `instanceCount: 26664` byte-identical. Only
  `instancedComponentCount` moved (19 → 18 → 16 → 15), and it counts components rather than
  species. The edits had in fact landed: an independent `python.execute` census of the actor's ISM
  components read canopy trees 280 (214 stylised) → 291 (0 stylised), `HillTree_P2` 60 → 256,
  `SM_Dead_Tree` 6 → 35, groundcover 6857 grass quads → 23 093 Nordic. MECHANISM (engine source,
  UE 5.8): `UPCGMeshSelectorWeighted`'s loop is bounded by `InPointData->GetNumPoints()`
  (`PCGMeshSelectorWeighted.cpp:215`), each point draws one bucket from the cumulative weight table
  (`:151-152`, `:231-234`) and emits exactly one instance (`:243`) — so weights partition a fixed
  point set and no mesh-entry edit can move the sum. That is correct PCG behaviour; the defect is
  publishing the invariant as the verification signal. READBACK: `PCGGenerateReadback.h:156-163`
  holds the `UInstancedStaticMeshComponent*` and never calls `GetStaticMesh()`; the full payload
  (`:243-319`) has sixteen fields and none names a mesh. DOC CLAIM, and its half-correction: the
  overlay sentence at `Docs/wiki-src/pcg.md:65` (shipped as `Saved/PinWright/wiki/pcg.generate.md:26`)
  promotes "non-zero exactly when the graph put geometry in the world" — true — into "the number to
  judge a generation by" — not true — in one sentence, and `pcg.md:41` repeats the short form; but
  the **C++ is honest** (`PCGGenerateReadback.h:275` "to judge whether the generation did work",
  `PCGGenerateHandler.cpp:111` "instances actually spawned"), so the doc half of the fix is
  localised to the hand-authored overlay. NOT a regression and NOT a re-file of
  `B-pcg-generated-graph-output-empty-after-generate` (DONE): that ticket introduced this field to
  stop it reporting `0` for a real generation, which it does correctly; this is the next question
  the same field cannot answer, and its fix walked past the answer because
  `AccumulateManagedResourceCounts` already visits every component. Asked for: `instancesByMesh`
  keyed by mesh object path, accumulated in the existing loop, with a decided cap/spill policy and
  an explicit key for a null mesh. Rated **High** on the rubric's silent-wrong-data band with the
  subtlety stated out loud — no field in the response is false, the shipped instruction for reading
  it is; declined Critical (nothing crashes, nothing is lost) and declined the reach modifier in
  both directions (every PCG session hits this verb but PCG is not an every-session namespace, and
  an up-bump lands on Critical; not an edge path either, since `UPCGMeshSelectorWeighted` is the
  shipped default selector and this fires on every mesh-entry edit).
