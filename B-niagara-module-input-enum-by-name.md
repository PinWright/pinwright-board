---
id: B-niagara-module-input-enum-by-name
title: "niagara.set_module_input rejects an enum module input's name — only a bare integer is accepted, and it is written to the pin as a float (\"2.0\")"
status: OPEN
severity: Medium
category: bug
tags: [niagara, set-module-input, enum, coordinate-space, display-name, unsupported-input-value, type-inference]
encounters: 1
lastSeen: 2026-09-02T22:45:00+05:00
---

# `niagara.set_module_input` has no enum path: the label is rejected, the index is accepted untyped and unchecked

Many stock Niagara module inputs are **enum-typed**, not switch-typed — most
visibly `ENiagaraCoordinateSpace` on `AddVelocityInCone.Cone Axis Coordinate Space`,
`MeshRotationRate.Coordinate Space`, `SpriteFacingAndAlignment.Facing Coordinate Space`
and `Collision.Analytical Collision Plane Space`. These are ordinary module inputs
(they appear under `stack.modules[].moduleInputs[]` with
`type: "ENiagaraCoordinateSpace"`), not static switches, so
`niagara.set_static_switch` does not address them and its enum resolution — fixed
in `B-niagara-static-switch-enum-display-name` — does not apply.

`niagara.set_module_input`'s value inference handles bool, number, vector and
colour only, so the enum's name is refused:

```
niagara.set_module_input { ..., inputName: "Cone Axis Coordinate Space", value: "Local" }
-> [UNSUPPORTED_INPUT_VALUE] Module input values must be a bool, number,
   vector object/array, color object, or an existing override pin default string.
```

A bare integer is accepted — and that is the second half of the defect:

```
niagara.set_module_input { ..., inputName: "Cone Axis Coordinate Space", value: 2 }
-> success:true, value:"2.0"
```

Two problems with that success:

1. **The echo is a float spelling of an enum.** `"2.0"` is what the number path
   produces; a Niagara enum pin default is not a float literal. The caller has no
   way to tell from the response whether the pin now carries a resolvable branch
   or an unreadable string, and the same asymmetry
   `B-niagara-static-switch-enum-display-name` `#1` recorded on the decode side
   ("silently coerces a miss to index 0") applies here.
2. **No range check.** The static-switch path now refuses an index outside the
   table and publishes `enumOptions[]` with every rejection. The module-input path
   publishes nothing and refuses nothing, so a wrong index writes a wrong branch
   that compiles clean and validates clean — exactly the failure mode
   `B-niagara-static-switch-enum-display-name` was fixed to stop.

## Why it matters

Coordinate space is the difference between an effect that fires along the
emitter's own axis and one that fires along world X. Every directional impact,
muzzle and spray effect needs `Local`/`Simulation` chosen deliberately. Guessing
the integer works only because `ENiagaraCoordinateSpace` happens to be a plain
C++ `UENUM` in declaration order — the user-defined enums Niagara uses elsewhere
permute their order, which is the trap `B-niagara-static-switch-enum-display-name`
documents, and nothing here protects against it.

## Repro

1. `asset.duplicate` `/Niagara/DefaultAssets/Templates/Emitters/SimpleSpriteBurst`
   to a scratch path.
2. `niagara.add_module` `/Niagara/Modules/Spawn/Velocity/AddVelocityInCone`,
   `scriptUsage: "ParticleSpawnScript"` — keep the returned `nodeId`.
3. `niagara.set_module_input` on it, `inputName: "Cone Axis Coordinate Space"`,
   `value: "Local"` -> **FAIL** `UNSUPPORTED_INPUT_VALUE`.
4. Same call with `value: 2` -> `success:true, value:"2.0"`.
5. Same call with `value: 97` -> also accepted (no table, no range check).

## Expected

The enum branch `niagara.set_static_switch` already has, reused here rather than
reimplemented: resolve a string through `BlueprintEnumHelpers::TryResolveEnumLiteralToValue`
plus the space/underscore-collapsed label match, range-check a number against the
same table, encode the pin default in the form Niagara reads back (not `"%f"`),
and attach `enumPath` + `enumOptions[]` to both the success echo and the
rejection — so a caller can discover the branch table from the response instead of
guessing.

Blocked-on note: resolving the enum requires the input's **declared type**, the
same discovery gap named in `F-niagara-dynamic-input-nested-inputs` and
`B-niagara-set-module-input-vec2` `#2` (`GetStackFunctionInputs` +
`FCompileConstantResolver`). An `inputType` hint on the request would sidestep it.

## Distinct from related tickets

- `B-niagara-static-switch-enum-display-name` (DONE) is the same class on
  `set_static_switch`. Static switches and enum module inputs are different code
  paths and different pin kinds; the fix did not carry across.
- `B-niagara-set-module-input-vec2` is the neighbouring hole in the same
  inference table (no FVector2D branch) and shares the root cause: the table
  guesses a type from JSON shape instead of reading the declared one.

severity rationale: impact=soft blocker with a silent-wrong-value hazard — the
task is doable by guessing an index, and a wrong guess is written without
complaint x reach=every enum-typed module input, which includes the coordinate
space on every velocity, orientation and collision module -> Medium

## History
- `#1-initial-repro` `OPEN` reporter — Found building the FPS impact VFX systems under `/Game/FPS/VFX/` (map as forcing function; host `CLAUDE.md` § "What this project is for"), 2026-09-02, UE 5.8, EAContentExamples58 checkout, live editor on port 27145. Both calls above were executed: `value:"Local"` returned the `UNSUPPORTED_INPUT_VALUE` text quoted verbatim, `value:2` returned `success:true ... "value":"2.0"` on `AddVelocityInCone` in a fresh `SimpleSpriteBurst` duplicate. The out-of-range case (step 5) is **stated, not run** — inferred from the response carrying no `enumOptions[]` and from the number path having no table to check against; a verifier should run it. Not source-confirmed: no read of `InferNiagaraInputType` / `JsonValueToPinDefaultString` was made in this session; the "float spelling" reading is from the echoed `"2.0"`, and the cross-reference to the static-switch fix is from that ticket's own `#2`/`#3` history, not from the code.
