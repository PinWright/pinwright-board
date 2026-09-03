---
id: B-niagara-module-input-enum-display-name-rejected
title: "niagara.set_module_input rejects the enum name for an enum-typed module input — only a bare integer index works, and the index is nowhere published"
status: OPEN
severity: Medium
category: bug
tags: [niagara, set-module-input, enum, coordinate-space, unsupported-input-value, discoverability]
encounters: 1
lastSeen: 2026-09-02T22:45:00+05:00
---

# An enum-typed module input takes only a raw integer, and `set_module_input` never says which integer

`niagara.set_static_switch` accepts an enum branch by **name** — the wiki's
`niagara.set_static_switch.md` says so explicitly ("Reach for the label"), the
response publishes `enumOptions[]` of `{index,name,displayName}`, and the docs
warn that guessing the index is unsafe because the authored names are a
permutation of the label order.

`niagara.set_module_input` gives an enum-typed **module input** none of that. A
name is refused outright:

```
niagara.set_module_input {
  assetPath:"/Game/FPS/VFX/Emitters/E_ImpConcrete_Sparks",
  entryId:"70FA7596…",                       // AddVelocityInCone
  inputName:"Cone Axis Coordinate Space",    // type ENiagaraCoordinateSpace
  value:"Local" }
-> [UNSUPPORTED_INPUT_VALUE] Module input values must be a bool, number,
   vector object/array, color object, or an existing override pin default string.
```

The same call with `value: 2` succeeds (`"value":"2.0"`). So the capability is
present; only the name spelling is missing.

## The index is not discoverable from the write verb

`niagara.inspect {includeStack:true}` *does* publish the option list for such an
input — the entry carries `typeInfo.isEnum:true`, `typeInfo.enum:
"/Script/Niagara.ENiagaraCoordinateSpace"` and `enumOptions: ["Simulation",
"World","Local"]` — but as a **bare string array with no indices**, unlike the
`{index,name,displayName}` objects `set_static_switch` publishes. A caller must
infer that array position == enum value. That inference happens to hold for a
C++ `UENUM` (verified against
`C:/UE_5.8/Engine/Plugins/FX/Niagara/Source/Niagara/Public/NiagaraTypes.h:631-644`,
`Simulation=0, World=1, Local=2`), but it is exactly the inference
`set_static_switch`'s own docs forbid for the user-defined enums Niagara's stock
modules use elsewhere — so a caller who learns the rule from one verb applies it
wrongly on the other.

The rejection payload publishes nothing at all: no `enumPath`, no `enumOptions`,
no hint that this input is an enum. `set_static_switch`'s `INVALID_VALUE`
rejection carries the whole branch table; this one carries a generic type list.

## Why it matters

`ENiagaraCoordinateSpace` inputs decide whether a velocity/orientation module
works in world or in the emitter's local space — i.e. whether an impact effect's
particles come out of the surface or fly off in a fixed world direction
regardless of how the actor is oriented. Getting it wrong is not a subtle
degradation, and a wrong integer is written without complaint (Niagara clamps an
out-of-range enum selector rather than erroring — the same failure mode
`set_static_switch` was hardened against, per its "An index outside the table is
refused, not written" note).

## Fix

1. Accept the enum entry name / display name on `set_module_input` for an
   enum-typed input, matched the way `set_static_switch` matches
   (case-insensitive, ignoring spaces and underscores), and echo
   `index`/`name`/`displayName` back.
2. Publish `enumPath` + `enumOptions[]` of `{index,name,displayName}` on the
   `UNSUPPORTED_INPUT_VALUE` rejection **and** in
   `niagara.inspect`'s `moduleInputs[]` entries, replacing the index-less bare
   string array.
3. Refuse an integer outside the table instead of writing it.

severity rationale: impact=a wrong-but-accepted value silently mis-orients an effect, and the correct value is only reachable by an inference the sibling verb's docs call unsafe × reach=every module with a coordinate-space / mode enum input (AddVelocityInCone, AddVelocity, MeshRotationRate, Collision, …) -> Medium

## History
- `#1-initial-repro` `OPEN` reporter — Hit while authoring `/Game/FPS/VFX/NS_Impact_Concrete` on UE 5.8 in the EAContentExamples58 checkout. `set_module_input` on `AddVelocityInCone`'s `Cone Axis Coordinate Space` with `value:"Local"` returned `UNSUPPORTED_INPUT_VALUE`; `value:2` succeeded. The option list exists in `niagara.inspect {includeStack:true}` but only as `enumOptions:["Simulation","World","Local"]` with no indices, so the reporter confirmed `Local == 2` out-of-band against `NiagaraTypes.h:631-644` rather than from any RPC. The rejection payload carries no enum information whatsoever. Contrast `niagara.set_static_switch`, which takes the label, publishes `{index,name,displayName}` on success *and* on `INVALID_VALUE`, and refuses an out-of-table index.
