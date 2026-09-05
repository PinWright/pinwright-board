---
id: E-niagara-module-input-default-value-unreported
title: "niagara.inspect reports valueMode:\"default\" on an unset module input but never what that default IS — and the graphs aspect cannot be used to recover it, because the MapGet default pins are anonymous and their output-pin linkage is unserialized"
status: OPEN
severity: Medium
category: ergonomic
tags: [niagara, inspect, module-inputs, defaults, ParameterMapGet, introspection, parity-ue58]
encounters: 1
lastSeen: 2026-09-05
---

# An unset module input reads back as "unset" and nothing else

`niagara.inspect {includeStack:true}` emits, per placed module, a `moduleInputs[]` entry with
`name` / `type` / `typeInfo` / `valueMode` (the `moduleInputs` array added by
`E-niagara-input-schema-readback` `#2`). For an input with no override pin the entry is:

```json
{"name":"Cone Axis","type":"Vector3f",
 "typeInfo":{"name":"Vector3f","sizeBytes":12},
 "valueMode":"default"}
```

There is **no `value` field at all** — not the declared default, not a null, nothing. The readback
says the input is unset and stops. So a caller can see *that* an input is defaulted but never *what
it is defaulted to*, which is exactly the question that decides whether an unset input is harmless
or a bug. The Niagara stack UI shows that value greyed-out in the same row, so this is a readback
gap, not an engine limitation.

## The graphs aspect does not close it — it cannot be joined

The obvious fallback is to inspect the module SCRIPT's graph and read the default off its
`UNiagaraNodeParameterMapGet`. That does not work either. On
`/Niagara/Modules/Spawn/Velocity/AddVelocityInCone`, `niagara.inspect {includeGraphs:true}`
emits `NiagaraNodeParameterMapGet_1` as three named output pins and three **anonymous** input pins:

```
output  Local.Module.ConeVector                  Vector3f
output  Module.Cone Axis                         Vector3f
output  Module.Cone Axis Coordinate Space        ENiagaraCoordinateSpace
input   name:"None" displayName:""  Vector3f                 defaultValue:"1.000000,0.000,0.000"
input   name:"None" displayName:""  Vector3f                 defaultValue:"0.000,0.000,0.000"
input   name:"None" displayName:""  ENiagaraCoordinateSpace  defaultValue:"Local"
```

Nothing says which default belongs to which output. Two independent reasons the caller cannot
work it out:

1. **Array order is not the pairing.** On the sibling `NiagaraNodeParameterMapGet_6` in the same
   graph the outputs are `[Module.Cone Angle, Module.Velocity Distribution Along Cone Axis]` and the
   defaults are `["0.500000", "45.000000"]` — reversed, since Cone Angle's default is 45. On
   `NiagaraNodeParameterMapGet_0` the order is neither forward nor reversed (one output pin is even
   emitted *after* the default pins), and the type sequences do not line up either way.
2. **The real linkage is unserialized, and could not be joined even if it were emitted.** It lives
   in `UNiagaraNodeParameterMapGet::PinOutputToPinDefaultPersistentId` (`TMap<FGuid,FGuid>`), which
   the graph serializer does not emit. Fetching it out-of-band via `property.get` does not help,
   because that map is keyed on each pin's **`PersistentGuid`** while the `id` inspect publishes per
   pin is its **`PinId`** — measured on this node, the map's three keys
   (`E39C284D...`, `F734A3BD...`, `2AC467DB...`) and its three values
   (`453BF498...`, `177539E1...`, `856B693B...`) match **none** of the six pin `id`s inspect
   reported. The two id spaces never meet, so no amount of graph reading resolves a default.

## The route that does work — two RPCs and a manual float decode

```
property.get  {objectPath:"<script>.<script>:NiagaraScriptSource_0.NiagaraGraph_0",
               propertyName:"VariableToScriptVariable"}
  -> "(Name=\"Module.Cone Axis\",...)" : "...NiagaraGraph_0.NiagaraScriptVariable_1"
property.list {objectPath:"...NiagaraGraph_0.NiagaraScriptVariable_1"}
  -> DefaultMode "Value"
     Variable {"VarData":[0,0,128,63, 0,0,0,0, 0,0,0,0], "Name":"Module.Cone Axis"}
```

`VarData` is the raw little-endian struct — `0x3F800000, 0, 0` = `(1, 0, 0)` — which the caller
must decode by hand against `typeInfo.sizeBytes`. The same route on `NiagaraScriptVariable_6` gives
`Module.Cone Axis Coordinate Space` = `VarData [2,0,0,0]` = `2` = `Local`.

That is the authoritative source and it is nowhere near the verb a caller is already holding.

## What it cost

`/Game/FPS/VFX/NS_Blood`'s three directional emitters (Drips / Spray / Mist) left both
`Cone Axis` and `Cone Axis Coordinate Space` unset while the other five impact systems stated them
explicitly. WEAPONS spawns impact systems with `MakeRotFromX(ImpactNormal)`, so the effect is
either correct (`+X` / `Local`) or degenerate (a zero axis), and the two look identical in every
readback the plugin offers. One agent audited it, could not tell which, and had to hand the question
on; it was settled only by the `property.get` route above (it was `(1,0,0)` / `Local` — correct).
A single field on the entry that agent already had would have ended it.

## Fix

Extend `NiagaraDumpBuilder::BuildModuleInputsJson` (the builder `E-niagara-input-schema-readback`
`#2` added; feeds `niagara.inspect` includeStack and the `niagara_stack.json` sidecar) so a
`valueMode:"default"` entry also carries the declared default:

- `defaultMode` — `UNiagaraScriptVariable::DefaultMode` (`Value` / `Binding` / ...).
- `defaultValue` — decoded to the **same canonical pin-default string the `local` path already
  emits**, so `"(X=1.0,Y=0.0,Z=0.0)"` and `"2.0"` read identically whether stated or defaulted and
  a caller can diff the two without a type-aware branch.
- `defaultBinding` — `UNiagaraScriptVariable::DefaultBinding` when `DefaultMode` is `Binding`.

Source it from the module script graph's `VariableToScriptVariable` ->
`UNiagaraScriptVariable` (`UNiagaraGraph::GetScriptVariable(FNiagaraVariable)`), **not** from the
MapGet default pins — per the two reasons above those cannot be paired without also serializing
`PinOutputToPinDefaultPersistentId` and switching the graph aspect's pin `id` to `PersistentGuid`,
which is a larger and separately-breaking change. Bump the `niagara_stack.json` aspect version.

Secondary, lower priority, same node: the graph aspect emitting default pins as `name:"None"` with
no owner is misleading on its own terms. If the graph aspect is touched anyway, give each MapGet
default pin a `defaultForOutputPin` field naming the output pin it backs (resolvable inside the
builder, which has both the node and the map).

Acceptance: `niagara.inspect {includeStack:true}` on a system whose `AddVelocityInCone` has no
override on `Cone Axis` reports that input as
`{"valueMode":"default","defaultMode":"Value","defaultValue":"(X=1.0,Y=0.0,Z=0.0)"}`, and its
`Cone Axis Coordinate Space` as `{"valueMode":"default","defaultMode":"Value","defaultValue":"2.0",
"enumOptions":["Simulation","World","Local"]}` — matching the greyed-out values the Niagara stack UI
shows for the same two rows.

## History
- `#1-default-unreadable` `OPEN` reporter — Hit while stating `Cone Axis` / `Cone Axis Coordinate Space` explicitly on `/Game/FPS/VFX/NS_Blood`'s Drips/Spray/Mist emitters, 2026-09-05, UE 5.8, EAContentExamples58, live editor on port 27145, plugin freshly pulled to `origin/master` and rebuilt. Every call quoted above was executed against that editor: the `moduleInputs` shape is from `niagara.inspect {assetPath:"/Game/FPS/VFX/NS_Blood", includeStack:true}`; the MapGet pin dump and the `NiagaraNodeParameterMapGet_6` counter-example are from `niagara.inspect {assetPath:"/Niagara/Modules/Spawn/Velocity/AddVelocityInCone", includeGraphs:true}`; the `PinOutputToPinDefaultPersistentId` keys/values and the `VariableToScriptVariable` / `NiagaraScriptVariable_1` / `NiagaraScriptVariable_6` readbacks are from the `property.get` / `property.list` calls shown. **Not source-confirmed**: no read of `NiagaraDumpBuilder.cpp` or `NiagaraEdit*` was made this session — the builder and helper names in Fix come from `E-niagara-input-schema-readback` `#2`'s own history, and a developer should re-derive the call site before editing. The `UNiagaraGraph::GetScriptVariable` suggestion is likewise proposed, not verified to be the accessor the builder can reach.
