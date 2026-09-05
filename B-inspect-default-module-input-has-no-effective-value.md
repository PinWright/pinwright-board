---
id: B-inspect-default-module-input-has-no-effective-value
title: "niagara.inspect reports an unset module input as valueMode:'default' with no value, and the graph aspect's ParameterMapGet default pins are anonymous and unjoinable — so an unset input's effective value is unreadable by any verb"
status: OPEN
severity: Medium
category: bug
tags: [niagara, inspect, module-input, defaults, parameter-map-get, graph, unreadable, nir, decompile, add-velocity-in-cone, persistent-guid, merged-duplicate]
encounters: 2
lastSeen: 2026-09-05
supersedes: [E-niagara-module-input-default-value-unreported]
---

# An unset module input has no readable value on any surface

`niagara.inspect {includeStack: true}` publishes every module input a caller has not written as

```json
{"name": "Cone Axis", "type": "Vector3f",
 "typeInfo": {...}, "valueMode": "default"}
```

with **no `value` field at all**. `valueMode: "default"` says only *"nobody overrode this"*; it
does not say what the module will therefore use. Since the overwhelming majority of a stock
module's inputs are unset, the overwhelming majority of a stack's effective configuration is
invisible. The Niagara stack UI shows that value greyed-out in the same row, so this is a readback
gap, not an engine limitation.

The graph aspect holds the numbers but not the pairing. A module script's
`UNiagaraNodeParameterMapGet` carries its per-parameter fallbacks on **input pins whose `name` is
`None`**, alongside the named output pins they belong to, and nothing published records which goes
with which.

## Measured, live editor port 27145, UE 5.8, EAContentExamples58, 2026-09-05

`niagara.graph.get {assetPath: "/Niagara/Modules/Spawn/Velocity/AddVelocityInCone.AddVelocityInCone"}`,
node `NiagaraNodeParameterMapGet_1`:

| direction | pin name | type | defaultValue | links |
|---|---|---|---|---|
| output | `Local.Module.ConeVector` | `Vector3f` | `''` | 1 |
| output | `Module.Cone Axis` | `Vector3f` | `''` | 1 |
| output | `Module.Cone Axis Coordinate Space` | `ENiagaraCoordinateSpace` | `''` | 1 |
| input | `None` | `Vector3f` | `'1.000000,0.000,0.000'` | 0 |
| input | `None` | `Vector3f` | `'0.000,0.000,0.000'` | 0 |
| input | `None` | `ENiagaraCoordinateSpace` | `'Local'` | 0 |

Three values, three outputs, no published linkage. Type narrows it to one — the enum must be
`Cone Axis Coordinate Space = Local` — and leaves the two `Vector3f` cases undecidable: is
`Module.Cone Axis` `(1,0,0)` or `(0,0,0)`? One is a legal cone axis and the other is degenerate,
so the answer changes what the module does.

### Pairing by position is not the answer, and there is proof in the same graph

Node `NiagaraNodeParameterMapGet_6` in the same script:

| direction | pin name | type | defaultValue |
|---|---|---|---|
| output | `Module.Cone Angle` | `NiagaraFloat` | `''` |
| output | `Module.Velocity Distribution Along Cone Axis` | `NiagaraFloat` | `''` |
| input | `None` | `NiagaraFloat` | `'0.500000'` |
| input | `None` | `NiagaraFloat` | `'45.000000'` |

Both outputs are `NiagaraFloat`, so type cannot disambiguate, and the semantics settle it the
other way: a **cone angle of 45 degrees** with a **distribution of 0.5** is the only reading that
makes sense, so the default pins are in the **reverse** of the output order. A caller pairing by
index would conclude `Cone Angle = 0.5` and `Velocity Distribution = 45`. Positional inference is
not merely unreliable here — on this node it is wrong. Reported independently by both reporters
(see History `#1` and `#2`), which is as close to confirmation as this board gets. `#2` adds that
on `NiagaraNodeParameterMapGet_0` the order is neither forward nor reversed — one output pin is
emitted *after* the default pins — so there is no consistent convention to lean on at all.

### Root cause: the linkage exists, under an id the API does not expose

*(measured by the `#2` reporter — this is the finding that turns the ticket from "unserialized" to
"serialized under the wrong key")*

The pairing lives in `UNiagaraNodeParameterMapGet::PinOutputToPinDefaultPersistentId`
(`TMap<FGuid,FGuid>`), which the graph serializer does not emit. Fetching it out of band with
`property.get` does not rescue it either: **that map is keyed on each pin's `PersistentGuid`,
while the `id` `inspect` publishes per pin is its `PinId`.** Measured on this node, the map's three
keys (`E39C284D…`, `F734A3BD…`, `2AC467DB…`) and its three values (`453BF498…`, `177539E1…`,
`856B693B…`) match **none** of the six pin `id`s inspect reported. The two id spaces never meet,
so no amount of graph reading resolves a default with what is published today.

## No other surface answers it either

- `niagara.inspect {includeStack:true}` on a placed `AddVelocityInCone` (scratch emitter
  `/Game/FPS/VFX/Scratch_TicketRetry/E_TR_Curve`, entryId `0D4FE3034FD238310C94839A6E262269`)
  reports all seven inputs — `Cone Angle`, `Cone Axis`, `Cone Axis Coordinate Space`,
  `Use Velocity Falloff On Cone Axis`, `Velocity Distribution Along Cone Axis`,
  `Velocity Falloff Away From Cone Axis`, `Velocity Strength` — as `valueMode: "default"` with no
  value. The enum input carries `enumOptions: ["Simulation","World","Local"]`, i.e. the legal set
  but not the chosen one. Reproduced by `#2` on `/Game/FPS/VFX/NS_Blood`, a different asset.
- `niagara.decompile_nir` on the module script emits
  `%NiagaraNodeParameterMapGet_1.Module.Cone Axis = get $Module.Cone Axis : Vector3f @(-976,-224)` —
  name and type, no default — and emits **no line at all** for the unnamed default pins, so NIR is
  strictly lossier than `niagara.graph.get` here.
- The rapid-iteration stores hold only what something wrote, so `parametersOnly` is empty for an
  unset input by construction.

## The route that does work — two RPCs and a manual float decode

*(found by the `#2` reporter)*

```
property.get  {objectPath:"<script>.<script>:NiagaraScriptSource_0.NiagaraGraph_0",
               propertyName:"VariableToScriptVariable"}
  -> "(Name=\"Module.Cone Axis\",...)" : "...NiagaraGraph_0.NiagaraScriptVariable_1"
property.list {objectPath:"...NiagaraGraph_0.NiagaraScriptVariable_1"}
  -> DefaultMode "Value"
     Variable {"VarData":[0,0,128,63, 0,0,0,0, 0,0,0,0], "Name":"Module.Cone Axis"}
```

`VarData` is the raw little-endian struct — `0x3F800000, 0, 0` = **`(1, 0, 0)`** — which the caller
must decode by hand against `typeInfo.sizeBytes`. The same route gives
`NiagaraScriptVariable_6` = `Module.Cone Axis Coordinate Space`, `VarData [2,0,0,0]` = `2` =
**`Local`**, and `NiagaraScriptVariable_7` = `Local.Module.ConeVector` = **`(0,0,0)`** — which
independently settles the ambiguity the pin dump leaves: the `(1,0,0)` belongs to `Cone Axis`, and
the `(0,0,0)` to the local temp. This is the authoritative source and it is nowhere near the verb a
caller is already holding.

## Why it matters

This is what made `NS_Blood`'s cone axis undecidable. Its three directional emitters (Drips /
Spray / Mist) left both `Cone Axis` and `Cone Axis Coordinate Space` unset while the other five
impact systems stated them explicitly. WEAPONS spawns impact systems with
`MakeRotFromX(ImpactNormal)`, so the effect is either correct (`+X` / `Local`) or degenerate (a
zero axis), and the two look identical in every readback the plugin offers. One agent audited it,
could not tell which, and had to hand the question on; it was settled only by the
`property.get` route above — the emitters were **accidentally correct**, `(1,0,0)` / `Local`, and
are now stated explicitly so the fleet is uniform. Two builds of uncertainty for a value the
engine had the whole time.

The general shape is worse than one effect: an author cannot read the configuration they are about
to modify, so every edit to a stock module is a blind write, and `niagara.set_module_input`'s own
wiki advice to *"screen a pin before writing"* can only report `valueMode`, never the value being
displaced.

Note that `niagara.get_curve_keys` has exactly this gap for data-interface inputs and reports it
honestly rather than silently: an un-overridden curve input returns
`[DATA_INTERFACE_NOT_FOUND] Module input '<x>' carries no override value and its module script
declares no default data interface.` The scalar/vector/enum case does not even say that much — it
returns a field-shaped object with the field missing.

## Fix

Extend `NiagaraDumpBuilder::BuildModuleInputsJson` (the builder `E-niagara-input-schema-readback`
`#2` added; feeds `niagara.inspect` includeStack and the `niagara_stack.json` sidecar) so a
`valueMode:"default"` entry also carries the declared default:

- `defaultMode` — `UNiagaraScriptVariable::DefaultMode` (`Value` / `Binding` / …).
- `defaultValue` — decoded to the **same canonical pin-default string the `local` path already
  emits**, so `"(X=1.0,Y=0.0,Z=0.0)"` and `"2.0"` read identically whether stated or defaulted and
  a caller can diff the two without a type-aware branch. Mark the provenance
  (`valueSource: "scriptDefault"`) so it is never mistaken for an authored value — the same job
  `nir.txt`'s new ` @default` tag does for renderer properties, see
  `B-nir-renderer-omits-default-valued-properties`.
- `defaultBinding` — `UNiagaraScriptVariable::DefaultBinding` when `DefaultMode` is `Binding`.

Source it from the module script graph's `VariableToScriptVariable` -> `UNiagaraScriptVariable`
(`UNiagaraGraph::GetScriptVariable(FNiagaraVariable)`), **not** from the MapGet default pins: per
the root-cause section those cannot be paired without also serializing
`PinOutputToPinDefaultPersistentId` *and* switching the graph aspect's pin `id` to
`PersistentGuid`, which is a larger and separately-breaking change. Bump the
`niagara_stack.json` aspect version.

Secondary, lower priority, same node: the graph aspect emitting default pins as `name:"None"` with
no owner is misleading on its own terms. If the graph aspect is touched anyway, give each MapGet
default pin a `defaultForOutputPin` field naming the output pin it backs — resolvable inside the
builder, which has both the node and the map.

Acceptance: `niagara.inspect {includeStack:true}` on a system whose `AddVelocityInCone` has no
override on `Cone Axis` reports that input as
`{"valueMode":"default","defaultMode":"Value","defaultValue":"(X=1.0,Y=0.0,Z=0.0)"}`, and its
`Cone Axis Coordinate Space` as `{"valueMode":"default","defaultMode":"Value","defaultValue":"2.0",
"enumOptions":["Simulation","World","Local"]}` — matching the greyed-out values the Niagara stack UI
shows for the same two rows.

## Cross-ref

- `E-niagara-module-input-default-value-unreported` — **merged into this ticket** and reduced to a
  stub. Filed independently and minutes apart by the `NS_Blood` agent; its root-cause,
  working-route and `NS_Blood` outcome are folded in above and credited to History `#2`. This
  ticket is canonical, and `bug` is the classification: an unreadable value is not a convenience
  gap, it made a live contract question undecidable.
- `B-nir-renderer-omits-default-valued-properties` — the same class of defect one aspect over
  (a default-valued *renderer property* was invisible), and its `@default` tag is the model for
  the provenance marker. Fixed there; unfixed here.
- `B-niagara-set-curve-keys-unreachable-module-input-di` `#11` — `get_curve_keys` refuses on an
  un-overridden module input rather than returning the script default, which is the
  data-interface-shaped instance of this same gap.
- `E-niagara-input-schema-readback` `#2` — added the `moduleInputs[]` array this ticket asks to
  extend.

severity rationale: impact=an author cannot read the configuration they are about to overwrite,
the only inference available from the published data is provably wrong on a stock module, and it
cost two builds of uncertainty on a shipped system x reach=every unset input on every stock
module, which is most of every stack -> Medium

## History
- `#1-initial-repro` `OPEN` VFX — Filed while retrying six workaround tickets against the rebuilt plugin (PLAN rule 2) on the FPS VFX stream, EAContentExamples58, UE 5.8, live editor port 27145, 2026-09-05. The two pin tables and the "no other surface" bullets are verbatim reads of live responses: `niagara.graph.get` on the engine module script for the two `ParameterMapGet` nodes, `niagara.inspect {includeStack:true}` on a placed copy in scratch emitter `/Game/FPS/VFX/Scratch_TicketRetry/E_TR_Curve` (deleted after the run), and `niagara.decompile_nir` on the same module script. **This was retried specifically to see whether the rebuilt plugin now resolves defaults. It does not** — the three unnamed default pins and the missing effective value are unchanged from the pre-update behaviour, so this is a fresh report and not a `#N-verified-in-fps-build`. The `MapGet_6` reverse-ordering finding rests on reading 45 as the cone angle in degrees and 0.5 as the distribution, i.e. semantic inference, not a source read. Not source-confirmed: no read of `NiagaraDumpBuilder.cpp` or of `UNiagaraNodeParameterMapGet` was made.
- `#2-merged-from-E-niagara-module-input-default-value-unreported` `OPEN` reporter — **Filed independently as `E-niagara-module-input-default-value-unreported` (board commit `5192d5b`) by the `NS_Blood` agent, minutes apart from `#1`, post-rebuild, on a different asset; merged here on the coordinator's instruction and the source ticket reduced to a stub.** Two reporters reproducing the same defect on different assets after the rebuild is the strongest statement available that the rebuild did not touch it. Everything this encounter contributed is marked in the body above: the **`PersistentGuid` vs `PinId` root cause** (the map's three keys `E39C284D…`/`F734A3BD…`/`2AC467DB…` and three values `453BF498…`/`177539E1…`/`856B693B…` matching none of the six published pin ids — which reclassifies the linkage from "unserialized" to "serialized under an identifier the API does not expose"), the **`NiagaraNodeParameterMapGet_0` observation** that its order is neither forward nor reversed, the **working `VariableToScriptVariable` -> `UNiagaraScriptVariable` read route** with the three decoded `VarData` values (`NiagaraScriptVariable_1` = `Module.Cone Axis` = `(1,0,0)`, `_6` = `Cone Axis Coordinate Space` = `2` = `Local`, `_7` = `Local.Module.ConeVector` = `(0,0,0)`), the **whole `## Fix` section**, and the **`NS_Blood` outcome** — its Drips/Spray/Mist emitters were accidentally correct and are now stated explicitly. Their calls were executed against the same editor on port 27145, 2026-09-05, plugin freshly pulled to `origin/master` and rebuilt. Their non-confirmations carry over unchanged: no read of `NiagaraDumpBuilder.cpp` or `NiagaraEdit*` was made, the builder and helper names in `## Fix` come from `E-niagara-input-schema-readback` `#2`'s own history and a developer should re-derive the call site before editing, and `UNiagaraGraph::GetScriptVariable` is proposed rather than verified to be reachable from the builder. Severity kept at Medium (both reporters agreed); category taken from this ticket (`bug`) rather than theirs (`ergonomic`).
