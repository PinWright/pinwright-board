---
id: B-connect-nodes-accepts-true-false-pin-names-on-static-switch-and-wires-nothing
title: "material.authoring.connect_nodes reports 'Nodes connected.' for inputName True/False on a StaticSwitchParameter - the names get_material_node_details itself returns - but the compiler then fails with 'Missing A input' and the material renders as Default Material"
status: IN-REVIEW
severity: High
category: bug
tags: [material, material-authoring, connect-nodes, get-material-node-details, static-switch, pin-names, silent-success, default-material]
encounters: 2
lastSeen: 2026-09-05T20:50:00Z
---

# Two verbs disagree about a StaticSwitchParameter's pin names, and the disagreement is silent

## Repro

UE 5.8, EAContentExamples58, on `/Game/FPS/Player/M_FPSArms`.

`material.authoring.add_static_switch_parameter {parameterName: "EnableViewmodelFOV", defaultValue: false}`,
then read it back:

```
material.authoring.get_material_node_details {nodeId: <the switch>}
-> "nodeType": "MaterialExpressionStaticSwitchParameter",
   "inputs": [ {"name": "True",  "connectedNodeId": ""},
               {"name": "False", "connectedNodeId": ""} ],
   "outputs": [ {"index": 0, "name": "Output"} ]
```

The details verb says the pins are `True` and `False`. Wiring them with those names is accepted:

```
connect_nodes {sourceNodeId: <MaterialFunctionCall>, targetNodeId: <switch>, inputName: "True"}
-> {"message": "Nodes connected."}
connect_nodes {sourceNodeId: <Constant3Vector>,      targetNodeId: <switch>, inputName: "False"}
-> {"message": "Nodes connected."}
```

Both succeed. `compile_material` on the material then also reports success —
`compileSucceeded: true, compiledWithErrors: false, compileErrors: []` — because it builds the
**default** static permutation, in which the unconnected input is not reached.

The truth surfaces only when something forces the other permutation:

```
material.authoring.set_static_switch_parameter_value
  {assetPath: <the MI>, parameterName: "EnableViewmodelFOV", value: false}
-> "shaderCompile": {"status": "failed", "errorCount": 1,
                     "errors": ["(Node StaticSwitchParameter) Missing A input"],
                     "rendersDefaultMaterial": true,
                     "hint": "The shader FAILED to compile, so this material renders as the engine
                              Default Material... A capture of this material is not evidence of anything."}
```

The compiler wants `A` and `B`. Re-wiring with `inputName: "A"` and `"B"` — same two calls, same
nodes, different names — and re-compiling gives
`shaderCompile: {succeeded: true, rendersDefaultMaterial: false}`.

## Why it is worth a High

A material that renders as the Default Material looks like a **badly authored** material, not a
broken one, and the Default Material on a skinned mesh is pale grey with fine speckle — entirely
plausible as "my albedo and roughness are wrong". This cost two builds of a first-person viewmodel:

- verified the instance resolved `SleeveTint` = (0.035, 0.038, 0.045) through `MaterialEditingLibrary`
  — correct, and irrelevant;
- retuned tiling 9 -> 2, `Specular` 0.32 -> 0.10, `Roughness` 0.62 -> 0.80 — no pixel moved;
- replaced an `AutoExposureBias` hack with a pinned `editor.screenshot {exposure:{mode:"fixed", ev100:0}}`
  — correctly exposed frame, same pale arms;
- hid the third-person body, then the arms themselves, to establish which component was drawing;
- chased a genuine but unrelated second defect in the mesh material slots
  (`B-convert-to-skeletal-mesh-leaves-sections-without-a-material-slot`).

Each was a reasonable next step on the evidence available, and none could have found it, because the
two verbs that could have said "this pin is not connected" both reported success.

## Asked for

1. **`connect_nodes` must refuse an `inputName` it cannot bind**, listing the valid names.
   `PIN_NOT_FOUND` is already the shape it uses elsewhere — it returned exactly that for `UVs` on a
   TextureSample in this same session, which is why "Nodes connected." read as trustworthy here.
2. **`get_material_node_details` must report the names the compiler uses.** If `True`/`False` are
   display labels over `A`/`B`, report both; returning only the label that does not work is the trap.
3. Consider having `compile_material` build the non-default static permutations, or warn that it did
   not: a material is only proven by compiling the permutation an instance actually selects, and
   nothing in the chain does that today.

## Related

`E-compile-material-refuses-a-material-instance` — the remedy the failure hint recommends cannot be
followed for a `MaterialInstanceConstant`, which is where a static switch override lives.

severity rationale: impact=ships a material that renders as the engine default while three separate
verbs report success x reach=any material with a static switch, the standard way to gate an optional
feature -> High

## Fix

The defect was true: `material.authoring.connect_nodes` resolved an input, assigned the wire, and unconditionally returned success after `PostEditChange` without reading the selected `FExpressionInput` back. UE 5.8 stores StaticSwitchParameter inputs in reflected fields `A`/`B` while `GetInputName()` exposes `True`/`False`; the resolver now pairs both aliases with the exact cached input pointer, returns the complete accepted-name set, and the handler returns `PIN_NOT_FOUND` with `result.candidates` before mutation or `CONNECTION_FAILED` when read-back does not retain the requested source.

Files changed: `Source/PinWright/Private/Material/MaterialExpressionFactory.h`, `Source/PinWright/Private/Material/MaterialExpressionFactory.cpp`, `Source/PinWright/Private/Handlers/Material/MaterialAuthoringHandler.cpp`, `Source/PinWright/Private/Tests/Material/TestMaterialConnectSourcePin.cpp`, and `Docs/wiki-src/material.authoring.md`. Test id: `PinWright.material.authoring.connect_nodes.StaticSwitchInputAliases`.

Material shader permutation compilation remains deliberately unchanged. The inspection response now retains its existing `name` field and adds optional `internalName` when a reflected property differs, so the StaticSwitch `True`/`False` display labels are paired with `A`/`B` without changing other node types.

### Verifier follow-up

Restored the legacy reflected-property fallback as the final resolver step for hidden inputs such as `TextureSampleParameter2D.TextureObject`, while preserving the display/internal/function-call/derived alias scan and candidate reporting. Added root/main-material read-back with typed `CONNECTION_FAILED` on a lost source. Added the optional `internalName` field in the shared MGIR detail payload and documented the backward-compatible shape. Source-only follow-up; no editor, build, or automation run.

Files changed in this follow-up: `Source/PinWright/Private/Material/MaterialExpressionFactory.cpp`, `Source/PinWright/Private/Handlers/Material/MaterialAuthoringHandler.cpp`, `Source/PinWright/Private/MGIR/MGIRExpressionUtils.h`, `Source/PinWright/Private/Tests/Material/TestMaterialConnectSourcePin.cpp`, `Docs/wiki-src/material.authoring.md`, and `Docs/wiki-src/material.graph.md`. Coverage: `PinWright.material.authoring.connect_nodes.StaticSwitchInputAliases` (extended with `get_material_node_details` A/B assertions), `PinWright.material.authoring.connect_nodes.TextureObjectReflectedFallback`, and `PinWright.material.authoring.connect_nodes.MainInputReadBack`.

## History
- `#1-filed` `OPEN` reporter — Found on the FPS PLAYER stream. The new `shaderCompile` block on `set_static_switch_parameter_value` is what finally surfaced it; its `rendersDefaultMaterial` flag and the "a capture of this material is not evidence of anything" hint are exactly the right reporting. The gap is that nothing says it at authoring time, when the wrong pin name is used. Fixed on my side by re-wiring to `A`/`B`; `M_FPSArms` now compiles with `rendersDefaultMaterial: false`.
- `#2-it-is-not-the-NAME-it-is-input-index-0-and-the-first-fix-did-not-stick` `OPEN` reporter — **Sharpening `#1`, which blamed the pin name. The name is a red herring: `connect_nodes` cannot bind input index 0 of a `StaticSwitchParameter` under EITHER name, and reports success both times.** Evidence, all from one asset (`/Game/FPS/Player/M_FPSArms`):

  1. `connect_nodes {inputName: "True"}` -> `"Nodes connected."`; `connect_nodes {inputName: "A"}` -> `"Nodes connected."`. Both for input 0.
  2. `connect_nodes {inputName: "False"}` and `{inputName: "B"}` -> also success, and those two **did** take.
  3. `get_material_node_details` afterwards: `A: {Expression: null, OutputIndex: -1}`, `B: {Expression: MaterialExpressionConstant3Vector_0}`. Input 0 null after two successful-looking writes; input 1 connected.
  4. `MaterialEditingLibrary.connect_material_expressions(call, '', switch, 'A')` -> **False** (the engine refuses that name); the same call with `'True'` -> **True** and the connection persists. So the engine's name for input 0 is `True`, while the material COMPILER's error calls it `A` (`"(Node StaticSwitchParameter) Missing A input"`). Three layers, three names.

  **And the first fix silently did not stick, which is the part that cost the most.** After `#1` I re-wired with `A`/`B`, `compile_material` answered `compileSucceeded: true, rendersDefaultMaterial: false`, and I recorded the material as fixed. A later `get_material_info` on the same asset came back `shaderCompile: {status: "failed", errors: ["(Node StaticSwitchParameter) Missing A input"], rendersDefaultMaterial: true}` — the `B` half had persisted and the `A` half never existed, so the successful compile had been of a graph that was still broken. Two builds of a first-person viewmodel rendered as the engine Default Material on the back of that.

  Fixed for real via `connect_material_expressions(..., 'True')`, then `compile_material` -> `shaderCompile.succeeded: true, rendersDefaultMaterial: false`, saved (`M_FPSArms.uasset` 18 041 B, `saveState: "written"`), and confirmed by `asset.generate_thumbnail` -> `usingDefaultMaterial: false`, `meanLuminance 0.132`.

  Adds to the asks in `#1`: `connect_nodes` must verify the connection it just made and refuse when the engine call returns false, rather than reporting `"Nodes connected."` unconditionally — the engine already answers with a bool, so the information is there and is being discarded.
- `#3-resolve-aliases-and-read-back` `IN-REVIEW` developer — Changed material input resolution to bind display and reflected internal names to the same input pointer, made `connect_nodes` return `PIN_NOT_FOUND` with all candidate names before mutation, and gated success on `Input.Expression == Source`. Added `PinWright.material.authoring.connect_nodes.StaticSwitchInputAliases` on a transient material; source-only verification only, no editor/build/suite run.
- `#4-verifier-follow-up-corrections` `IN-REVIEW` developer — Restored the final reflected-property fallback for hidden `TextureObject`, added optional detail `internalName` aliases, and gated main-material success on source read-back with `CONNECTION_FAILED`. Added/extended `PinWright.material.authoring.connect_nodes.TextureObjectReflectedFallback`, `PinWright.material.authoring.connect_nodes.MainInputReadBack`, and `PinWright.material.authoring.connect_nodes.StaticSwitchInputAliases`; source-only verification, no editor/build/suite run.
