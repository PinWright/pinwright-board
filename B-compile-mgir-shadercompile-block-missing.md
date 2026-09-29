---
id: B-compile-mgir-shadercompile-block-missing
title: "material.compile_mgir returns no shaderCompile block at all (even with waitForShaderCompile:true), although the method page and material.mgir say the verdict lives there"
status: OPEN
severity: Medium
category: bug
tags: [material, mgir, compile_mgir, shadercompile, response-shape, docs-mismatch]
encounters: 1
lastSeen: 2026-09-29T18:40:00Z
---

# `material.compile_mgir` omits the documented `shaderCompile` block

## Symptom

`material.compile_mgir` is documented (method page and `material.mgir` § Compile / decompile) to
return a `shaderCompile` block — "branch on `shaderCompile.status`", "pass
`waitForShaderCompile: true` ... and report the real verdict", with per-asset
`shaderCompile.materials[]` and failures raised into `warnings[]`.

The live response carries no such key. Observed key set (PDS unreal-fpv checkout, UE 5.8 Linux,
editor not in PIE, `waitForShaderCompile: true`, `save: false`, `mode: "Append"`, target
`/App/MultiplayerLevelEditor/PostProcess/M_SelOverlay_O1_k005`, a translucent Modulate surface
material with two Custom HLSL nodes):

```
['assetPaths', 'blocksCompiled', 'consumerRefresh', 'expressionsCreated', 'mode',
 'saveDetail', 'saveRequested', 'saveState', 'saved', 'world']
```

Earlier in the same session, six other `compile_mgir` calls (post-process material with a Custom
node, four new surface materials; `save: true`, with and without `waitForShaderCompile: true`,
some while PIE was active → `saveState: blockedByPie`) also returned `shaderCompile: None` /
no `warnings`.

## Impact

The docs tell the caller that `blocksCompiled` is not a shader verdict and that `shaderCompile`
is the measurement. With the block missing, a caller has no shader verdict from this verb and
must add a separate `material.authoring.compile_material` per asset (which did return
`compileStatus: completed`, `compileErrors: []` for all of them). A caller that reads
`r.get('shaderCompile', {}).get('status')` gets `None` and may treat it as "not compiled" or
skip the check.

## Repro

1. Any `material.compile_mgir` call with `waitForShaderCompile: true` on a small material.
2. Inspect the response keys: no `shaderCompile`.

## Expected

Either the `shaderCompile` block as documented (status + per-material breakdown), or the docs
updated to point callers at `material.authoring.compile_material`.

## History

- `#1-filed-missing-block` `OPEN` reporter — Found while prototyping map-editor selection outline/overlay materials via MGIR on the PDS unreal-fpv checkout. Worked around with a follow-up `material.authoring.compile_material` per asset.
