---
id: E-compile-material-refuses-a-material-instance
title: "set_static_switch_parameter_value on an MI reports notCompiled + rendersDefaultMaterial:true and tells you to call compile_material, which refuses a MaterialInstanceConstant"
status: OPEN
severity: Low
category: enhancement
tags: [material, material-authoring, compile-material, material-instance, static-switch, shader-compile, contradictory-advice]
encounters: 1
lastSeen: 2026-09-05T19:17:00Z
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
