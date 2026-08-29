---
id: B-pcg-generate-blind-to-non-point-data
title: "Every countable field pcg.generate returns is structurally zero for a graph whose output is neither points nor spawned resources, and the wiki names instanceCount as the number to judge a generation by — so a Procedural Vegetation graph that worked and one that is broken produce byte-identical, confidently-zero receipts"
status: OPEN
severity: High
category: bug
tags: [pcg, generate, readback, procedural-vegetation, silent-wrong-conclusion, response-shape, data-types, doc-defect, unobservable-output]
encounters: 1
lastSeen: 2026-08-29T17:15:00+05:00
---

# The response has five counters and none of them can see this graph's output

`pcg.generate` publishes six countable fields. For a Procedural Vegetation graph, **all of them are
structurally zero regardless of whether the graph did anything**, and `dataCount` — the one field
that is not zero — counts containers rather than content. Nothing in the payload distinguishes *"the
graph produced nothing"* from *"the graph produced something this readback cannot count"*, and the
shipped documentation resolves that ambiguity in the wrong direction.

## The fields, and why each is zero

`Source/PinWrightPCG/Private/Handlers/PCG/PCGGenerateReadback.h`, `BuildGenerationPayload`
(`:243-319`) writes: `pointCount` `:266`, `dataCount` `:267` (both gated on `graphOutputAvailable`),
`instanceCount` `:283`, `instancedComponentCount` `:284`, `spawnedActorCount` `:285` (gated on
`resourceCountsAvailable`), and `localComponentCount` `:300`.

**`pointCount` counts only `UPCGBasePointData`, by cast.** `CountGeneratedPoints` (`:74-99`) iterates
`Output.TaggedData` at `:79` and:

```cpp
if (const UPCGBasePointData* PointData = Cast<UPCGBasePointData>(Data))   // :87
{
    Total += PointData->GetNumPoints();                                    // :89
}
```

Anything that fails that cast contributes nothing and is skipped — the file says so in a comment at
`:73`. `UPVData` is not a `UPCGBasePointData`, so a PV graph's output is invisible here by
construction, not by accident.

**The one engine conversion that could have rescued it is a stub.** `UPVData::ToBasePointData`
(`C:/UE_5.8/Engine/Plugins/Experimental/ProceduralVegetationEditor/Source/ProceduralVegetation/Private/DataTypes/PVData.cpp:54-60`):

```cpp
const UPCGBasePointData* UPVData::ToBasePointData(FPCGContext* Context, const FBox& InBounds, TSubclassOf<UPCGBasePointData> PointDataClass) const
{
    TRACE_CPUPROFILER_EVENT_SCOPE(UPCGPrimitiveData::CreatePointData);

    UPCGBasePointData* Data = FPCGContext::NewObject_AnyThread<UPCGBasePointData>(Context, GetTransientPackage(), PointDataClass);
    return Data;
}
```

A brand-new, empty point data. The plant's `FManagedArrayCollection` (member initialised at `:14`,
`Initialize` at `:17-20`) is never touched. The rest of `UPVData`'s spatial surface is stubbed the
same way — `GetBounds()` (`:44-47`) returns a `ForceInit` box, `SamplePoint()` (`:49-52`) returns
`false` — while `CopyInternal` (`:62-76`) *does* copy the collection, so the stubbing is specific to
the PCG conversion surface rather than to the type.

**Stated precisely, because it is easy to mis-cite:** PinWright never calls `ToBasePointData` —
zero occurrences in `PCGGenerateReadback.h`. It is cited because it is the only route by which a
`UPVData` could ever have become countable, and it returns empty. The proximate reason `pointCount`
is zero is the failed cast at `:87`.

**The three resource counters need spawned managed resources, and PV spawns none.**
`instanceCount` / `instancedComponentCount` / `spawnedActorCount` are accumulated by walking
`ForEachManagedResource` for `UPCGManagedISMComponent` and `UPCGManagedActors` — the mechanism
`B-pcg-generated-graph-output-empty-after-generate` `#2` added and this ticket is not questioning.
A PV graph terminating in an Export node creates neither, so all three are structurally zero too.

**`dataCount` is a container count.** `CountGeneratedData` (`:102-105`) is one line:

```cpp
return Output.TaggedData.Num();                                            // :104
```

Type-agnostic and content-blind. `dataCount: 3` cannot be distinguished from three point datas,
three param datas, or three `UPVData`s carrying a finished plant.

**No field anywhere reports the class of an output data object.** Grep for `dataTypes|dataType|GetClass()`
across `PCGGenerateReadback.h` and `PCGGenerateHandler.cpp`: **zero hits**. `CountGeneratedPoints`
performs a `Cast` at `:87` to decide whether to count — it has the class in hand at the cheapest
possible moment — and discards it.

## The documentation converts the zero into a wrong conclusion

This is what lifts the ticket out of the "a readback omits a field" band. `Docs/wiki-src/pcg.md`
does not leave the caller to interpret the zeros; it tells them which one to trust, twice.

`pcg.md:41`:

> poll `call("system.job_status", {ticket_id})` for the terminal result and **judge it by
> `instanceCount`**, because `pointCount` covers only data reaching the graph's Output node and may
> be omitted.

`pcg.md:65`:

> **`instanceCount` is the number to judge a generation by.** It counts the ISM/HISM instances PCG
> actually spawned and still manages (read from the component's managed-resource list), so it is
> **non-zero exactly when the graph put geometry in the world**.

And `pcg.md:66` closes the funnel: *"**Never treat a missing `pointCount` as 'produced nothing'** —
check `instanceCount`."*

For a PV graph the caller is therefore routed, explicitly and in two steps, from an unavailable
`pointCount` to a field that is **structurally zero for this entire class of graph**. The clause
*"non-zero exactly when the graph put geometry in the world"* is false for any graph whose product
is not ISM/HISM instances or spawned actors. Every one of these sentences is correct for the
spawner-graph case they were written for — `B-pcg-generated-graph-output-empty-after-generate`
earned them — and each becomes a wrong instruction the moment the output is a different shape.

## The empirical consequence: the question is unanswerable

Measured this session on a generated PV actor in the level, and this is the part worth reading
before rating the severity.

A generate was driven to completion through a wired Export node. The framing was set up correctly
and the framing metrics were green: camera set first and allowed to settle, one read-back, capture
at 1280×720 with exposure pinned, `framing.boundsInFrame: true`, `offAxis 1.9°` against a 49.5°
limit — that is, the generated actor's own bounds were provably centred in frame.

`Saved/Screenshots/OpenLevel/PV_generate_world_output.png`, viewed: sky above a horizon line in the
upper eighth of the frame, a flat pale blue-grey ground plane filling the rest, and the world-axis
gizmo in the bottom-left corner. No plant. No mesh. No geometry of any kind.

So: every readback counter zero, every framing check green, a capture that proves nothing is there,
and **no verb in the plugin can say whether the grower grew a plant.** The agent driving it recorded
that it could not determine the outcome, which is the correct thing to record and is exactly the
failure this ticket is about. Note what the capture does *not* establish — it cannot distinguish a
graph that produced nothing from a graph that produced something with no world representation, which
is precisely why the readback has to answer it.

## The ask

**`dataTypes[]` — the class name and count of each output data object**, emitted next to
`dataCount`:

```json
"dataCount": 3,
"dataTypes": [ { "class": "/Script/ProceduralVegetation.PVData", "count": 3 } ]
```

- Cheap: the class is already in hand at `PCGGenerateReadback.h:87`, where a `Cast` decides whether
  to count. Recording `Data->GetClass()->GetPathName()` alongside costs one map insert.
- Naturally bounded by **distinct classes**, not by content, so it needs no new cap — the same
  argument `B-pcg-generate-instancecount-blind-to-species` makes for `instancesByMesh`.
- It answers the actual question. `dataCount: 3` plus `dataTypes: [PVData ×3]` says *the graph ran,
  produced three PV data objects, and this readback cannot count inside them* — which is a true,
  actionable statement. Three zeros are not.

**And correct `pcg.md:41`, `:65` and `:66` in the same change.** `instanceCount` is the number to
judge a *spawner* generation by; it is not "non-zero exactly when the graph put geometry in the
world", and a graph whose output is neither points nor managed resources needs the doc to say so
rather than route the caller into a third structurally-zero field.

**Generalises well past PV.** The counting path has exactly one cast (`:87`); *anything* reaching
the graph's Output node that is not a `UPCGBasePointData` counts zero, whatever plugin defined it.
PV is the case measured here, not the boundary of the defect.

## Same shape as

The session's recurring class, stated on `B-foliage-paint-does-no-ground-projection` § *Same shape
as*: *the call succeeds, every number it reports is correct, and the output is wrong because the
deciding number was never reported.* This is the class in its purest form — not one missing number
but **every** number missing at once, with no field left that could contradict the others.

- `B-pcg-generate-instancecount-blind-to-species` (OPEN, High) — same verb, same response, same
  documentation-turns-a-true-number-into-a-false-conclusion aggravator, about species instead of
  data type. The two are one field apart and a fixer opening `PCGGenerateReadback.h` should have
  both; kept separate per the no-umbrella policy because `instancesByMesh` and `dataTypes` come from
  different accumulators and either can land without the other.
- `B-pcg-connect-pins-silently-replaces-edge` (OPEN, High, filed this session) — the other end of
  the same PV session. Having silently unwired the graph, nothing here could have told you.
- `B-pcg-spawner-mixed-mesh-heights-silently-misscaled` (OPEN, Medium) — same verb, same silence,
  about scale.

## Distinct from

- **`B-pcg-generated-graph-output-empty-after-generate` (DONE, Medium)** — the closest relative and
  deliberately **not** reopened. That ticket found `pointCount: 0` for a pass that spawned 97 and 64
  ISM instances, correctly diagnosed `GeneratedGraphOutput` as holding only what reached the Output
  node, and fixed it by counting managed resources instead. That fix works and is not in question:
  for a spawner graph, `instanceCount` now answers. This ticket is the case its fix does not reach —
  output that is neither points **nor** managed resources — which is a different mechanism, a
  different field, and a defect that only became visible once a PV graph could be built at all
  (`F-pcg-create-graph-class-parameter`, DONE this session). Reopening a DONE ticket to carry it
  would hide a new finding inside a closed one.
- `F-pcg-generate-readback` (IN-REVIEW) — the feature ticket that built this readback. Its
  acceptance criterion is a non-zero point count from a scatter graph, which is the spawner case and
  is met; this is a shape it did not scope.
- `B-pcg-generate-deadlocks-game-thread` — the ticketing/async path, unrelated to what the receipt
  contains.

## Dedup

Board-wide search for `pointCount`, `dataCount`, `dataTypes`, `instanceCount` and `pcg.generate`
readback tickets. Four files touch this response — the three above plus
`B-pcg-spawner-mixed-mesh-heights-silently-misscaled` — and each is distinguished. **Nothing on the
board asks for the class or type of an output data object, in any namespace.**

## Severity

**High.** Impact class is the rubric's High band — *silent wrong data on a normal path, where the
caller trusts a result and builds on it.* The subtlety a reviewer will reach for first, so it is
stated up front: **no field in this response is false.** `pointCount: 0` is arithmetically correct,
`instanceCount: 0` is arithmetically correct, `dataCount` is correct. What is false is the shipped
instruction for reading them (`pcg.md:41`, `:65`, `:66`), which nominates a structurally-zero field
as the one to judge the generation by and explicitly steers the caller to it when the other is
unavailable. A document that converts a correct number into a wrong conclusion is the High band with
an extra step — the same reasoning `B-pcg-generate-instancecount-blind-to-species` used on this same
verb, and `B-wiki-hism-recipe-resets-actor-transform` used to file a wiki defect as `B-` rather than
`E-`.

**Not Medium.** The tempting reading is *"a readback omits a field and forces a fallback"*, which is
the Medium band verbatim. It is refused because **there is no fallback.** The whole point of the
measurement above is that after the readback, the framing checks and the capture, the outcome of the
generation was still undetermined — and the capture cannot answer it either, since a graph with no
world representation and a graph that produced nothing look identical through a camera. A Medium
requires a route to the answer that costs extra calls; here the answer is unreachable through the
RPC surface.

**Not Critical.** Nothing crashes, nothing is corrupted, and no asset is lost. A generation whose
result cannot be read is re-runnable and the graph is intact.

**Reach modifier declined in both directions, and named.** No bump down to Medium: `pcg.generate` is
the `pcg` namespace's only execution verb, so every PCG session that runs anything hits it, and the
zero-counter case covers every non-point output type rather than PV alone — that is not a rare edge
path. No bump up: PCG authoring is not an every-session activity across the plugin's whole surface,
and a bump up from High lands on Critical, which the paragraph above declines on its merits. High
stands unmodified.

## History
- `#1-every-counter-is-structurally-zero` `OPEN` reporter — Filed from this session's Procedural Vegetation pass, the first on this project able to build a PV graph at all (`F-pcg-create-graph-class-parameter`, DONE this session). **Structural claim, derived from source and marked as such; the empirical half is a capture.** All six countable fields `pcg.generate` publishes are zero for a PV graph whether it worked or not: `pointCount` (`PCGGenerateReadback.h:266`) comes from `CountGeneratedPoints` (`:74-99`), which counts only what survives `Cast<UPCGBasePointData>` at `:87` and skips everything else per the comment at `:73`, and a `UPVData` is not a `UPCGBasePointData`; `instanceCount` / `instancedComponentCount` / `spawnedActorCount` (`:283-285`) come from walking managed `UPCGManagedISMComponent` / `UPCGManagedActors` resources, and a PV graph terminating in an Export node spawns neither; `dataCount` (`:267`) is `Output.TaggedData.Num()` (`CountGeneratedData`, `:102-105`, the return at `:104`), a container count that is type-agnostic and content-blind. Definitive negative, established by grep over `PCGGenerateReadback.h` and `PCGGenerateHandler.cpp` for `dataTypes|dataType|GetClass()` — **zero hits**: no field anywhere reports the class of an output data object, though `CountGeneratedPoints` holds the class at `:87` in order to decide whether to count it. Recorded precisely because it is easy to mis-cite: PinWright never calls `UPVData::ToBasePointData`; that engine function is cited because it is the only conversion that could ever have made a `UPVData` countable, and it is a stub — `PVData.cpp:54-60` allocates a brand-new empty `UPCGBasePointData` and returns it without touching the plant's `FManagedArrayCollection` (member `:14`, `Initialize` `:17-20`), with `GetBounds()` (`:44-47`) and `SamplePoint()` (`:49-52`) stubbed the same way while `CopyInternal` (`:62-76`) does copy the collection. **The documentation aggravator, which is what makes this a `B-` and not an `E-`:** `Docs/wiki-src/pcg.md:41` says to *"judge it by `instanceCount`"*, `:65` says *"`instanceCount` is the number to judge a generation by … non-zero exactly when the graph put geometry in the world"*, and `:66` says *"Never treat a missing `pointCount` as 'produced nothing' — check `instanceCount`"* — a two-step funnel that routes the caller from an unavailable field into a structurally-zero one, with a universal claim that is false for any output that is not ISM/HISM instances or spawned actors. **Empirical consequence, measured:** a completed generate through a wired Export node, camera set first and settled, one read-back, capture at 1280x720 with exposure pinned, `framing.boundsInFrame: true` and `offAxis 1.9°` against a 49.5° limit — the generated actor's bounds provably centred — produced `Saved/Screenshots/OpenLevel/PV_generate_world_output.png`, which on inspection shows sky, a horizon in the upper eighth, a flat pale ground plane, and the world-axis gizmo, and nothing else. The driving agent recorded that it could not determine whether the grower grew a plant; no verb can answer it, and the capture cannot either, because a graph with no world representation and a graph that produced nothing are identical through a camera. Ask: a `dataTypes[]` breakdown (class path + count per output data object) next to `dataCount`, built at `:87` where the class is already in hand, bounded by distinct classes rather than content so it needs no new cap; plus corrections to `pcg.md:41`/`:65`/`:66`. Generalises past PV — the counting path has exactly one cast, so anything reaching the Output node that is not a `UPCGBasePointData` counts zero regardless of which plugin defined it. Dedup: searched the board for `pointCount`, `dataCount`, `dataTypes`, `instanceCount` and generate-readback tickets; four files touch this response and all four are distinguished in the body, and **nothing on the board asks for the class or type of an output data object in any namespace**. `B-pcg-generated-graph-output-empty-after-generate` (DONE) deliberately **not** reopened — its fix is correct for the spawner case it was filed for, and this is the case that fix does not reach. Cross-linked into the session's recurring class rather than restating it. Severity High, with Medium argued and refused because the rubric's Medium requires a fallback route to the answer and here there is none, and Critical refused because nothing crashes or is lost; reach declined in both directions.
