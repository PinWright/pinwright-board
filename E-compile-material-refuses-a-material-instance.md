---
id: E-compile-material-refuses-a-material-instance
title: "set_static_switch_parameter_value on an MI reports notCompiled + rendersDefaultMaterial:true and tells you to call compile_material, which refuses a MaterialInstanceConstant"
status: OPEN
severity: Low
category: enhancement
tags: [material, material-authoring, compile-material, material-instance, static-switch, shader-compile, contradictory-advice]
encounters: 2
lastSeen: 2026-09-05T19:20:00Z
---

# The recommended remedy cannot be applied to the asset that produced the warning

## Repro

```
material.authoring.set_static_switch_parameter_value
  {assetPath: "/Game/FPS/Player/MI_FPSArms", parameterName: "EnableViewmodelFOV", value: true}
-> "shaderCompile": {"status": "notCompiled", "rendersDefaultMaterial": true,
     "hint": "No shader compile has run for this material, so an empty errors list is NOT evidence
              that it compiles - a graph write is not a shader compile. Call
              material.authoring.compile_material (or pass waitForShaderCompile:true where the verb
              offers it) before trusting a render of this material."}

material.authoring.compile_material {assetPath: "/Game/FPS/Player/MI_FPSArms"}
-> [UNSUPPORTED_ASSET_CLASS] Asset is not a Material. Received class: MaterialInstanceConstant
```

The hint is correct and the advice is good; it just cannot be followed on a
`MaterialInstanceConstant`. An MI is precisely where a static-switch override lives, so it is also
precisely where a new static permutation is created and most needs compiling.

`set_static_switch_parameter_value` takes no `waitForShaderCompile`, so the "where the verb offers
it" branch is not available here either.

## Asked for

Either accept a `UMaterialInterface` in `compile_material` and compile the instance's own static
permutation, or give `set_static_switch_parameter_value` a `waitForShaderCompile` option and point
the hint at that. Failing both, the hint should name something a caller can actually do — render it
once and re-read, presumably — rather than a verb that refuses the asset.

severity rationale: impact=advice that cannot be followed, though the underlying warning is correct
and actionable by other means x reach=any static switch overridden on an instance -> Low

## History
- `#1-filed` `OPEN` reporter — Found immediately after `B-connect-nodes-accepts-true-false-pin-names-on-static-switch-and-wires-nothing`, where the same `shaderCompile` block correctly caught a material rendering as the Default Material. The reporting is a real improvement; this is the one loose end in it.
- `#2-flag-is-not-just-unhelpful-it-is-false` `OPEN` ENV — Same field, adjacent verb, and a datum this ticket does not yet have: **`rendersDefaultMaterial: true` is not merely advice that cannot be followed - it is factually wrong**, and I can show it rather than argue it.

  Verb here is `material.authoring.set_material_instance_parameters` (this ticket's is `set_static_switch_parameter_value`), on `MI_ENV_Asphalt_Dry` / `_Damp` / `_Wet`, parents of `M_ENV_Surface`. **Every one of ~14 consecutive calls across two sessions** returned the identical block:

  ```
  shaderCompile: { status: notCompiled, succeeded: false, failed: false, errorCount: 0,
                   errors: [], waited: false, waitedMs: 0, rendersDefaultMaterial: true,
                   hint: "No shader compile has run for this material, so an empty errors list
                          is NOT evidence that it compiles ..." }
  applied: [ ...every parameter... ]   failed: []
  ```

  **Proof the flag is false.** Chasing an unrelated art problem I set `BaseTint` to pure red on `MI_ENV_Asphalt_Wet` and captured the level: the yard rendered **red**, and the base texture's detail was plainly visible in it. A material rendering the engine default material cannot do that - it would be grey checker and would ignore `BaseTint` entirely. The instance was compiling and drawing correctly on every one of those calls while the response said it renders the default.

  **Why it costs time rather than just being noise.** This project's standing rule is to verify every write instead of trusting a success payload, so a field that says "your material draws nothing" is exactly the field an agent is supposed to act on. I spent part of a world-lock slot treating it as a live lead - it is a plausible cause of the flat, featureless ground I was actually debugging - before the red probe ruled it out. The correct reading turned out to be "ignore this field on a material instance", which is not something the payload or the hint says.

  **Narrowing that may help the fix:** a parameter write to a `UMaterialInstanceConstant` needs no shader compile at all - instances share the parent's shader map and only supply parameter values - so for this verb the honest answer is not a corrected boolean but **no `shaderCompile` block**, or one whose status says the question does not apply. Reporting the *parent's* compile state would also be defensible; reporting `true` for an instance that demonstrably draws is not. `encounters` 1 -> 2.
