---
id: B-niagara-entry-id-not-unique-across-emitters
title: "niagara entryId is a raw NodeGuid, so emitters duplicated from the same template share ids — a right entryId paired with the wrong emitter silently edits the wrong emitter instead of erroring, and the response echoes the id back unchanged"
status: OPEN
severity: Medium
category: bug
tags: [niagara, entry-id, identity, multi-emitter, silent-wrong-target, set_module_input, docs]
encounters: 1
lastSeen: 2026-08-28T09:40:00+05:00
---

# `entryId` is not a key: it is unique only *within* one emitter, and nothing says so

Every `niagara.*` module verb addresses a module by `entryId`. The name, the shape
(a bare GUID) and the wiki all read as a primary key. It is not one. `entryId` is
the raw `UNiagaraNodeFunctionCall::NodeGuid`, and duplicating an emitter copies its
graph **including the NodeGuids**, so two emitters descended from the same template
carry byte-identical `entryId`s for their corresponding stock modules.

The consequence is narrow but it is the failure shape this plugin cares about most:
**a correct `entryId` paired with the wrong `emitter` succeeds against the wrong
emitter's module rather than erroring**, and the success response echoes the
`entryId` that was asked for, so nothing in the result reveals the mistake.

For every *non*-colliding module the same caller error is caught — `MODULE_NOT_FOUND`.
The guard exists and works. It evaporates on exactly the modules that template-derived
emitters share, which on a multi-emitter system (a fish school, a layered effect) are
most of them.

## Measured collision

`asset.duplicate` of `/Niagara/DefaultAssets/Templates/Systems/SimpleExplosion`,
read with `niagara.inspect {includeStack:true}`. Of 36 distinct `entryId`s in the
system, **2 appear under more than one owner**:

| entryId | module | nodeName | owners |
|---|---|---|---|
| `1EA681674152F5B73C37F3AE70009629` | `SolveForcesAndVelocity` | `NiagaraNodeFunctionCall_8` | `OmnidirectionalBurst`, `SimpleSpriteBurst`, `UpwardMeshBurst` (all three) |
| `5DDC9AC54A37F08F656EB0AED8F19794` | `SpawnBurst_Instantaneous` | `NiagaraNodeFunctionCall_5` | `OmnidirectionalBurst`, `UpwardMeshBurst` |

`SimpleSpriteBurst`'s own `SpawnBurst_Instantaneous` is `68A8CD574D62C54866BE778FB68D9342`
— a different id for the same stock module, so the collision tracks duplication
lineage, not module identity. An agent cannot predict which ids collide.

## Verbatim repro (behavioural, against the built binary)

PinWright HEAD `b79ba53e`, UE 5.8, on the duplicate above. All four calls use the
**same** colliding `entryId` `5DDC9AC5…` and the same `inputName`.

```
# 1. wrong-but-plausible emitter: SUCCEEDS, writes the wrong emitter
niagara.set_module_input {emitter:"OmnidirectionalBurst", entryId:"5DDC9AC5…",
                          inputName:"Spawn Count", value:111}
  -> success:true, pinId:"A752DEAF4082B94DFAAA0396DA7E452F", value:"111.0"

# 2. intended emitter: also succeeds, different node
niagara.set_module_input {emitter:"UpwardMeshBurst", entryId:"5DDC9AC5…",
                          inputName:"Spawn Count", value:222}
  -> success:true, pinId:"437A1222485C914C289FED98848CA13F", value:"222.0"

# readback — two different modules were written, both reporting the same entryId
OmnidirectionalBurst | entryId 5DDC9AC5… | Spawn Count valueMode=local value=111.0
UpwardMeshBurst      | entryId 5DDC9AC5… | Spawn Count valueMode=local value=222.0
SimpleSpriteBurst    | entryId 68A8CD57… | Spawn Count valueMode=default value=None
```

The two responses differ only in `pinId`, which the caller has no baseline for. If
the caller meant `UpwardMeshBurst` and passed `OmnidirectionalBurst`, call 1 is a
silent wrong-target write that looks exactly like a correct one.

**Two controls prove the guard exists everywhere else and only fails here:**

```
# non-colliding id + wrong emitter -> correctly refused
niagara.set_module_input {emitter:"UpwardMeshBurst", entryId:"68A8CD57…", …}
  -> [MODULE_NOT_FOUND] Module entry '68A8CD57…' was not found.

# colliding id + NO emitter -> correctly refused, no first-match fallback
niagara.set_module_input {entryId:"5DDC9AC5…", …}   (emitter omitted)
  -> [MODULE_NOT_FOUND] Module entry '5DDC9AC5…' was not found.
```

## Root cause (guilty source lines)

**Id side.** `Plugins/PinWright/Source/PinWright/Private/Handlers/Niagara/NiagaraDumpBuilder.cpp:922`:

```cpp
        Obj->SetStringField(TEXT("entryId"), Node->NodeGuid.ToString());
```

A raw engine NodeGuid, emitted with no owner qualification. PinWright does not own
that GUID and cannot make it unique — `UNiagaraEmitter` duplication copies the graph
verbatim, which is *why* the ids collide.

**Resolve side.** `Handlers/Niagara/NiagaraEditTypes.cpp:1072-1075`:

```cpp
            OutTarget.Node = FindNode(OutTarget.Graph, TargetSpec.EntryId);
            …
            return OutTarget.ModuleNode ? FNiagaraEditError() : FNiagaraEditError::Make(
                TEXT("MODULE_NOT_FOUND"), …);
```

`FindNode` searches the graph of the **already-resolved emitter**. That scoping is
correct and is what makes the two controls above refuse — but it also means the
resolver never sees, and so can never report, that the id it was handed is ambiguous
across the system.

## Impact

Every verb taking `entryId` inherits this: `set_module_input`, `reset_module_input`,
`set_static_switch`, `clear_module_overrides`, `remove_module`, `move_module`,
`set_module_script`. It is latent — it needs a caller to pair a right id with a wrong
emitter — so it is filed Medium, not High. It would be High if any verb ever resolved
`entryId` without an emitter, because then a colliding id would pick a first match;
today none do, and the two controls above are the evidence.

The realistic trigger is an agent that harvests `entryId`s from one `niagara.inspect`
and replays them later without carrying `ownerName` alongside — the ids look globally
unique in the dump, and 34 of 36 of them are.

## What it should do — documentation and an echo, NOT a new id

Recommending **against** minting a unique id. `entryId` is the engine's NodeGuid;
changing it to a composite would break every stored id and every existing caller, and
would buy nothing behaviourally, since the resolver is already correctly emitter-scoped.

Two cheap changes close the gap instead:

1. **Say it.** `Docs/wiki-src/niagara.md` and the `entryId` param docs should state that
   `entryId` is unique only *within* one emitter, that `emitter` is load-bearing rather
   than a convenience, and that ids must be carried together with their `ownerName`.
   Today the param doc is just "Module entry id or node id".
2. **Echo what was resolved.** The mutation response should name the emitter and module
   it actually acted on (`ownerName` / `emitter`, and the module `name`), the same way
   `set_module_input` already echoes `value` and `replacedOverride`. A wrong-emitter
   write then shows up in the result instead of being invisible. This is the half that
   turns a silent wrong target into a visible one.

Optional and strictly better if cheap: when the resolved system contains the same
`entryId` under more than one emitter, add an `ambiguousEntryId: true` note (or the
list of owners) to the success response, so the caller learns the id is not a key at
the moment it uses it as one.

## Distinct from related tickets

- `B-niagara-module-input-stack-infer` — that is about which *stage* (`scriptUsage`) a
  module is inferred into within one emitter; this is about which *emitter*, one level up.
- `B-asset-dump-folder-accounting-uses-duplicate-package-paths` — duplicate package
  paths in dump accounting, a different surface with no addressing consequence.
- `B-niagara-inspect-stack-modules-emitted-four-times` — filed alongside this from the
  same dump; that one is repetition of whole entries, this one is collision of ids
  across distinct entries. Different mechanisms, different fixes.

## History
- `#1-initial-repro` `OPEN` reporter — Found while behaviourally verifying nine Niagara IN-REVIEW tickets against the built binary (PinWright HEAD `b79ba53e`, UBT "Target is up to date"), 2026-08-28, UE 5.8, on a scratch `asset.duplicate` of `SimpleExplosion` (deleted afterwards, never saved). Noticed as an oddity in the stack dump, then promoted to a filing after a behavioural check: two writes with the **same** `entryId` and different `emitter` values both succeeded and landed on different modules (`111.0` on `OmnidirectionalBurst`, `222.0` on `UpwardMeshBurst`, confirmed by readback), while the same id with the `emitter` omitted, and a non-colliding id with a wrong `emitter`, were both correctly refused `MODULE_NOT_FOUND`. So the resolver is emitter-scoped and safe by construction; what is missing is any signal that `entryId` is not a global key, which is the whole basis for filing this Medium rather than High. Root cause read in source, both halves: `NiagaraDumpBuilder.cpp:922` emits the raw `Node->NodeGuid`, and `NiagaraEditTypes.cpp:1072-1075` resolves it inside the already-selected emitter's graph. Board searched before filing (`entryid`, and `not unique` / `collision` / `nodeguid` across all statuses): nineteen tickets mention `entryId`, none is about its uniqueness or scope. Not verified: whether the collision also reaches `event handler` / `simulation stage` entryIds, which `NiagaraEditTypes.cpp:1085-1134` parses as usage GUIDs down a different path — flagged, not tested.
