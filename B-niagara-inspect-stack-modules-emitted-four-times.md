---
id: B-niagara-inspect-stack-modules-emitted-four-times
title: "niagara.inspect / asset.dump emit every emitter stack module four times, byte-identical — the builder walks the same shared NodeGraph once per script, so 75% of the stack aspect is duplication and no entry says which scriptUsage it belongs to"
status: OPEN
severity: Medium
category: bug
tags: [niagara, niagara-inspect, asset-dump, stack, response-size, duplicate-entries, script-usage, missing-discriminator]
encounters: 1
lastSeen: 2026-08-28T09:40:00+05:00
---

# The stack aspect repeats every module 4x and drops the field that would have told the copies apart

`niagara.inspect {includeStack:true}` (and the `niagara_stack.json` / `niagara_model.json`
sidecars that share the builder) emit each emitter-owned module **four times** and each
system-owned module **twice**. The copies are **byte-identical** — same `entryId`, same
`index`, same `moduleInputs`, same `staticSwitchInputs`. They carry no information the
first copy does not.

Two costs, one cause:

1. **~75% of the aspect is waste.** On a stock three-emitter system the modules array is
   154 entries where 39 are distinct. The aspect already exceeds the inline display budget
   and spills to `HttpResponses/*.json` — measured 872–887 KB per call — so the multiplier
   lands squarely on a response agents already have to read off disk in slices.
2. **The discriminator that would have justified the copies is missing.** Module entries
   carry `ownerKind` / `ownerName` / `index` but **no `scriptUsage`**, so nothing says
   whether a module sits in EmitterSpawn, EmitterUpdate, ParticleSpawn or ParticleUpdate.
   A caller cannot tell from the stack aspect which stage a module belongs to — they have
   to infer it from module order, or pass `scriptUsage` blind to the edit verbs.

## Measured

`asset.duplicate` of `/Niagara/DefaultAssets/Templates/Systems/SimpleExplosion`,
`niagara.inspect {includeProperties:false, includeStack:true, includeGraphs:false, includeCompile:false}`:

```
total module entries:        154
distinct (ownerName,entryId): 39
repetition histogram:        {4x: 38 groups, 2x: 1 group}
the 2x group:                NS_Verify_A / SystemState  (system-owned)
groups whose copies differ:   0   (all 39 groups byte-identical after json.dumps(sort_keys))
deduped size:                ~25% of current
```

The 4x/2x split is exactly the number of scripts walked per owner — which is the mechanism.

## Root cause (guilty source lines)

`Plugins/PinWright/Source/PinWright/Private/Handlers/Niagara/NiagaraDumpBuilder.cpp`.

`BuildSystemStackArray` (`:~1010`) walks the system's two scripts, then **four scripts per
emitter handle**:

```cpp
        AddGraphStackModules(Modules, TEXT("system"), System->GetName(), GetGraphFromScript(System->GetSystemSpawnScript()));
        AddGraphStackModules(Modules, TEXT("system"), System->GetName(), GetGraphFromScript(System->GetSystemUpdateScript()));

        for (const FNiagaraEmitterHandle& Handle : System->GetEmitterHandles())
        {
            …
            AddGraphStackModules(Modules, TEXT("emitter"), EmitterName, … EmitterData->EmitterSpawnScriptProps.Script …);
            AddGraphStackModules(Modules, TEXT("emitter"), EmitterName, … EmitterData->EmitterUpdateScriptProps.Script …);
            AddGraphStackModules(Modules, TEXT("emitter"), EmitterName, … EmitterData->SpawnScriptProps.Script …);
            AddGraphStackModules(Modules, TEXT("emitter"), EmitterName, … EmitterData->UpdateScriptProps.Script …);
        }
```

`BuildEmitterStackArray` (`:1035`) does the same four calls for a standalone emitter asset.

**All four calls resolve to the same graph object.** `GetGraphFromScript`
(`NiagaraJsonHelpers.cpp:87-98`) returns `Source->NodeGraph`, and an emitter's four scripts
share a single `UNiagaraScriptSource` — so the four calls hand `AddGraphStackModules` the
identical `UNiagaraGraph` pointer. That function (`:~1050`) then enumerates the **whole**
graph each time and appends the lot:

```cpp
    void AddGraphStackModules(TArray<…>& Modules, const FString& OwnerKind, const FString& OwnerName, const UNiagaraGraph* Graph)
    {
        …
        const TArray<const UNiagaraNodeFunctionCall*> FunctionNodes = CollectStackModuleNodesInExecutionOrder(Graph);
        for (int32 Index = 0; Index < FunctionNodes.Num(); ++Index)
        {
            Modules.Add(MakeObjectValue(BuildStackModuleJson(FunctionNodes[Index], OwnerKind, OwnerName, Index)));
        }
    }
```

`Index` restarts at 0 on every call, which is why the copies match even in that field.

Note the signature: `AddGraphStackModules` takes no usage argument, so it *cannot* stamp
`scriptUsage` — that is the same root cause as consequence 2 above. The sibling
`AddScriptCompileEntry`, two functions below in the same file, does take a `Usage` and does
set `scriptUsage` on its entries, so the field is established vocabulary in this builder;
the stack path just never got it.

## What it should do

One fix addresses both consequences: **thread the usage through and stop re-walking a
shared graph.** Either

- pass the usage into `AddGraphStackModules`, stamp `scriptUsage` on each entry, and filter
  the shared graph's nodes to the ones belonging to that usage (the output nodes of the
  graph already partition it by usage) — the copies stop being identical because they stop
  being copies; or
- if partitioning is more than this is worth, call it **once** per owner and stamp no usage,
  which removes the duplication but leaves consequence 2 open and should then say so in the
  wiki.

The first is strictly better and is what makes the aspect answer "which stage is this module
in?", a question `set_module_input` callers currently have to guess at.

Aspect versions in `Handlers/Asset/AssetDumpCache.cpp` need a bump either way, since the
sidecars change shape.

## Impact

Not silent-wrong-data — the entries are correct, there are just three redundant copies of
each. Filed Medium rather than Low because the multiplier applies to an aspect that already
spills to file, and because reading the module list at all currently requires deduping it
first (`sort -u`, or a `(ownerName, entryId)` set), which every consumer has to reinvent and
which a naive consumer will skip — a caller that iterates `modules[]` to count or to drive a
loop gets 4x the work and 4x the writes.

## Distinct from related tickets

- `B-asset-dump-niagara-emitter-duplicate-files` (DONE) — two *filenames* with identical
  content for standalone emitter assets. This is duplication *within* one array.
- `E-niagara-inspect-no-param-readback-projection` (IN-REVIEW) — no projection to keep a
  readback inline; its fix shipped `parametersOnly` / `parameterName` on the **parameters**
  aspect. This is the **stack** aspect, and a projection would not help: the bytes are
  redundant, not merely unwanted.
- `B-niagara-entry-id-not-unique-across-emitters` — filed alongside this from the same dump.
  That one is distinct entries colliding on their id; this one is one entry repeated. The
  two look alike in a dump and are unrelated in cause.
- `B-niagara-module-input-stack-infer` (IN-REVIEW) — consumes the missing `scriptUsage`
  discriminator from the other side: it is about the edit verbs inferring a stage. Fixing
  consequence 2 here would give that inference something to read.

## History
- `#1-initial-repro` `OPEN` reporter — Found while behaviourally verifying nine Niagara IN-REVIEW tickets against the built binary (PinWright HEAD `b79ba53e`), 2026-08-28, UE 5.8, on a scratch `asset.duplicate` of `SimpleExplosion` (deleted afterwards, never saved). Noticed because a module-list diff needed `sort -u` to be readable at all; quantified off the saved response payload rather than by eye — 154 entries, 39 distinct `(ownerName, entryId)` pairs, 38 groups at exactly 4x and one system-owned group at 2x, and **all 39 groups byte-identical** under `json.dumps(sort_keys=True)`, so no copy carries a field the others lack. Mechanism confirmed in source rather than inferred: the 4x/2x split matches the script counts in `BuildSystemStackArray` / `BuildEmitterStackArray` exactly, and `GetGraphFromScript` (`NiagaraJsonHelpers.cpp:87-98`) returns `Source->NodeGraph`, which the four emitter scripts share — so the same graph is enumerated four times by `AddGraphStackModules`, whose signature takes no usage and therefore cannot stamp `scriptUsage`. Board searched before filing (`spill` / `response size` / `duplicate entries` / niagara inspect+dump ticket names, all statuses): the two nearest, `B-asset-dump-niagara-emitter-duplicate-files` and `E-niagara-inspect-no-param-readback-projection`, are different surfaces and are cross-referenced above. Not verified: whether the `graphs` aspect and the `niagara_stack.json` / `niagara_model.json` sidecars repeat identically — they share `BuildStackModuleJson` but were not separately measured, and the sidecars were not regenerated.
