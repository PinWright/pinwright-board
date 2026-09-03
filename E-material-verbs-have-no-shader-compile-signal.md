---
id: E-material-verbs-have-no-shader-compile-signal
title: "Only material.authoring.compile_material reports shader compile errors; every other material write verb returns success for a material that fails to compile and draws nothing, and none of them mention it"
status: OPEN
severity: Medium
category: enhancement
tags: [material, compile_mgir, add_custom_expression, connect_nodes, shader-compile, verification, silent-false-success, docs]
encounters: 1
lastSeen: 2026-09-03T02:00:00Z
---

# Material write verbs give no signal that the shader failed

## Symptom

`material.compile_mgir`, `material.graph.*` and `material.authoring.*` all return
success-shaped payloads (`blocksCompiled`, `expressionsCreated`, `nodeId`,
`"Nodes connected."`) that describe the **graph** write. None of them says anything about
whether the material's **shader** compiles. A material with a malformed Custom HLSL node
writes a valid `.uasset`, passes `asset.save`, appears with the right domain, blend mode
and connected `mainInputs` under `material.authoring.get_material_info`, and renders
nothing at all.

`material.authoring.compile_material` is the one verb that surfaces the truth:

```
call("material.authoring.compile_material", {materialPath:"/Game/FPS/UI/Materials/M_HUD_RadarBack"})
-> {"compileSucceeded":false, "compiledWithErrors":true, "compileStatus":"failed",
    "compileErrors":["/Engine/Generated/Material.ush:3746:12: error: use of undeclared identifier 'Input'", ...]}
```

It is excellent — it names the file, line, shader type and permutation. The gap is that
nothing routes an author to it. Its own wiki page is titled as a compile trigger, and
`material.compile_mgir.md` never mentions it, so the natural reading of "compile_mgir"
is that compiling is what it already did.

## Why it matters here

Verifying a material by looking at it costs a PIE session, and on a shared editor a PIE
session costs a world-lock slot behind a queue. `compile_material` answers the same
question in 3-10 seconds with no world, no lock and no PIE — but only if you know to ask.
Two UI materials shipped through a full authoring, saving, disk-verification and PIE
capture cycle before this verb was tried; it found both failures immediately.

## Suggested change

- `material.compile_mgir`, `material.graph.add_expression`/`create_nodes` and
  `material.authoring.connect_nodes` should either return the shader compile status or
  carry a `hint` naming `material.authoring.compile_material`, the way
  `blueprint.inspect` already hints at `asset.dump_folder`.
- Cross-link it from `material.compile_mgir.md` and `material.mgir.md`, and say plainly
  that a successful MGIR compile is a graph write, not a shader compile.
- `visual-review.md` should list "compile the material first" ahead of "capture it",
  since the cheap check strictly dominates the expensive one.
