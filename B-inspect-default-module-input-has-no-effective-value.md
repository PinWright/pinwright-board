---
id: B-inspect-default-module-input-has-no-effective-value
title: "niagara.inspect reports an unset module input as valueMode:'default' with no value, and the graph aspect leaves ParameterMapGet default pins unnamed and unpaired — so an unset input's effective value is unreadable, and pairing them by position is provably wrong"
status: OPEN
severity: Medium
category: bug
tags: [niagara, inspect, module-input, defaults, parameter-map-get, graph, unreadable, nir, decompile, add-velocity-in-cone]
encounters: 1
lastSeen: 2026-09-05
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
invisible.

The graph aspect holds the numbers but not the pairing. A module script's
`UNiagaraNodeParameterMapGet` carries its per-parameter fallbacks on **input pins whose `name` is
`None`**, alongside the named output pins they belong to, and nothing records which goes with
which.

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

Three values, three outputs, no linkage. Type narrows it to one — the enum must be
`Cone Axis Coordinate Space = Local` — and leaves the two `Vector3f` cases undecidable: is
`Module.Cone Axis` `(1,0,0)` or `(0,0,0)`? One is a legal cone axis and the other is degenerate,
so the answer changes what the module does.

**Pairing by position is not the answer, and there is proof in the same dump.** Node
`NiagaraNodeParameterMapGet_6` in the same script:

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
not merely unreliable here — on this node it is wrong.

## No other surface answers it either

- `niagara.inspect {includeStack:true}` on a placed `AddVelocityInCone` (scratch emitter
  `/Game/FPS/VFX/Scratch_TicketRetry/E_TR_Curve`, entryId `0D4FE3034FD238310C94839A6E262269`)
  reports all seven inputs — `Cone Angle`, `Cone Axis`, `Cone Axis Coordinate Space`,
  `Use Velocity Falloff On Cone Axis`, `Velocity Distribution Along Cone Axis`,
  `Velocity Falloff Away From Cone Axis`, `Velocity Strength` — as `valueMode: "default"` with no
  value. The enum input carries `enumOptions: ["Simulation","World","Local"]`, i.e. the legal set
  but not the chosen one.
- `niagara.decompile_nir` on the module script emits
  `%NiagaraNodeParameterMapGet_1.Module.Cone Axis = get $Module.Cone Axis : Vector3f @(-976,-224)` —
  name and type, no default — and emits **no line at all** for the unnamed default pins, so NIR is
  strictly lossier than `niagara.graph.get` here.
- The rapid-iteration stores hold only what something wrote, so `parametersOnly` is empty for an
  unset input by construction.

## Why it matters

This is what made `NS_Blood`'s cone axis undecidable: the effect needed to know which way the
spray points before changing it, and no verb would say. The general shape is worse than one
effect — an author cannot read the configuration they are about to modify, so every edit to a
stock module is a blind write, and `niagara.set_module_input`'s own wiki advice to *"screen a pin
before writing"* can only report `valueMode`, never the value being displaced.

Note that `niagara.get_curve_keys` has exactly this gap for data-interface inputs and reports it
honestly rather than silently: an un-overridden curve input returns
`[DATA_INTERFACE_NOT_FOUND] Module input '<x>' carries no override value and its module script
declares no default data interface.` The scalar/vector/enum case does not even say that much — it
returns a field-shaped object with the field missing.

## Expected

Either of these closes it; the first is what authoring needs.

1. `niagara.inspect {includeStack:true}` publishes the **effective** value on a
   `valueMode: "default"` input — resolved from the module script's `ParameterMapGet` fallback —
   alongside the existing `valueMode`, so `default` keeps meaning "not overridden" while the
   caller still learns what runs. Mark it (`valueSource: "scriptDefault"`) so it is never mistaken
   for an authored value, the same way `nir.txt`'s new ` @default` tag does for renderer
   properties (see `B-nir-renderer-omits-default-valued-properties`).
2. Failing that, `niagara.graph.get` names each `ParameterMapGet` default pin, or records the
   output pin it backs. The engine holds this pairing — `UNiagaraNodeParameterMapGet` resolves a
   default pin per output pin — so the serializer is dropping a relation it has, not inventing
   one. Anything less leaves the caller guessing, and the `MapGet_6` evidence above shows the
   obvious guess is wrong.

## Cross-ref

- `B-nir-renderer-omits-default-valued-properties` — the same class of defect one aspect over
  (a default-valued *renderer property* was invisible), and its `@default` tag is the model for
  fix 1. Fixed there; unfixed here.
- `B-niagara-set-curve-keys-unreachable-module-input-di` `#11` — `get_curve_keys` refuses on an
  un-overridden module input rather than returning the script default, which is the
  data-interface-shaped instance of this same gap.

severity rationale: impact=an author cannot read the configuration they are about to overwrite,
and the only inference available from the published data is provably wrong on a stock module x
reach=every unset input on every stock module, which is most of every stack -> Medium

## History
- `#1-initial-repro` `OPEN` VFX — Filed while retrying six workaround tickets against the rebuilt plugin (PLAN rule 2) on the FPS VFX stream, EAContentExamples58, UE 5.8, live editor port 27145, 2026-09-05. Every table above is a verbatim read of a live response: `niagara.graph.get` on the engine module script for the two `ParameterMapGet` nodes, `niagara.inspect {includeStack:true}` on a placed copy in the scratch emitter, and `niagara.decompile_nir` on the same module script. **This was retried specifically to see whether the rebuilt plugin now resolves defaults. It does not** — the three unnamed default pins and the missing effective value are unchanged from the pre-update behaviour, so this is a fresh report and not a `#N-verified-in-fps-build`. The `MapGet_6` reverse-ordering finding is new and is the reason the "pair them by position" workaround must not be adopted; it rests on reading 45 as the cone angle in degrees and 0.5 as the distribution, which is semantic inference, not a source read. Not source-confirmed: no read of `NiagaraDumpBuilder.cpp`'s pin serializer or of `UNiagaraNodeParameterMapGet::GetDefaultPin` was made; the claim that the engine holds the pairing is from the engine's documented per-output default-pin model, not from this codebase.
