---
id: B-attach-render-target-phantom-param
title: "render.attach_render_target_to_volume reports attached:true for a nonexistent texture parameter (silent false-success)"
status: OPEN
severity: Medium
category: bug
tags: [render, render-target, post-process, unvalidated-texture-param, silent-false-success]
encounters: 1
lastSeen: 2026-07-10T22:06:52.1197512+03:00
---

## What's wrong

`render.attach_render_target_to_volume` never checks that `parameterName`
actually names a texture parameter on `materialPath`. It creates a MID of the
base material, calls `SetTextureParameterValue(FName(parameterName), RT)`
blindly, adds the MID as a WeightedBlendable, and returns `attached: true`
unconditionally.

If `parameterName` does not exist on the material, the MID stores a phantom
override that the material's shader never reads, so the render target is not
sampled by the volume at all — yet the caller is told `attached: true` and
trusts the RT is now driving the post-process feed. This is a silent
false-success: the volume gets a blendable material that renders its *default*
texture (or a hardcoded SceneTexture), never the live render target.

This is easy to hit in practice. Post-process texture parameters frequently
come from `TextureSampleParameter2D` subclasses (e.g. `AntialiasedTextureMask`)
whose material *parameter name* differs from the auto-generated expression
object name, so a caller reading the expression name off `asset.dump` and
passing it here gets `attached: true` with an inert binding and no signal that
anything is wrong.

Related (same pattern, different handler, not replayed here):
`material.authoring.set_texture_parameter_value`
(`MaterialAuthoringHandler.cpp:1973`) likewise calls
`SetTextureParameterValueEditorOnly` without validating the parameter exists.
Filed at method level; sharing the `unvalidated-texture-param` family tag so
future occurrences can merge.

## What it should do

Before creating/adding the MID, verify the material actually exposes a texture
parameter named `parameterName` (e.g. `BaseMat->GetTextureParameterValue(FName,
OutTex)` / enumerate `GetAllTextureParameterInfo`). If it does not exist, return
a clear error (`PARAM_NOT_FOUND`) that ideally lists the material's valid
texture parameter names, rather than reporting `attached: true`. Only report
`attached: true` when the render target was bound to a real, sampled parameter.

## Verbatim repro (replayed at HEAD via mcp__pinwright__call)

Setup:
- `render.create_render_target` args `{name: RT_OracleReplay, width: 256, height: 256, packagePath: /Game/RenderTargets}` -> `assetPath: /Game/RenderTargets/RT_OracleReplay.RT_OracleReplay`
- Volume: `/Game/Maps/ExampleProjectWelcome.ExampleProjectWelcome:PersistentLevel.PostProcessVolume_2` (fresh, unmodified)
- Material: `/Game/ExampleContent/PostProcessing/Materials/PPMAT_BlendablePostProcess2`

Call with a deliberately nonexistent parameter name:

`render.attach_render_target_to_volume` args:
`{volumePath: ".../PostProcessVolume_2", targetPath: "/Game/RenderTargets/RT_OracleReplay", materialPath: "/Game/ExampleContent/PostProcessing/Materials/PPMAT_BlendablePostProcess2", parameterName: "ThisParameterDoesNotExist_XYZ123"}`

Response (silent false-success):

```json
{"renderTarget":"/Game/RenderTargets/RT_OracleReplay","materialPath":"/Game/ExampleContent/PostProcessing/Materials/PPMAT_BlendablePostProcess2","parameterName":"ThisParameterDoesNotExist_XYZ123","attached":true,"actorPath":".../PostProcessVolume_2","existsAfter":true,"actorClass":"PostProcessVolume"}
```

The made-up parameter name is echoed back and `attached: true` is returned with
no error, even though `ThisParameterDoesNotExist_XYZ123` is not a parameter on
the material.

## Guilty source

`Plugins/PinWright/Source/PinWright/Private/Handlers/Render/RenderHandler.cpp`
lines 444-455:

```cpp
UMaterialInstanceDynamic* MID = UMaterialInstanceDynamic::Create(BaseMat, Volume);
if (MID)
{
    MID->SetTextureParameterValue(FName(*ParamName), RT);   // no existence check
    Volume->Settings.AddBlendable(MID, 1.0f);
    ...
    Result->SetBoolField(TEXT("attached"), true);           // always true
    AddActorVerification(Result, Volume);
    Ctx.SendSuccess(Result);
}
```

severity rationale: impact=silent-false-success (High) x reach=rare (one
specialized render-target->PPV method) -> Medium

## History

- `#1-initial-repro` `OPEN` reporter — Replay-confirmed at HEAD: passing a nonexistent `parameterName` (`ThisParameterDoesNotExist_XYZ123`) to `render.attach_render_target_to_volume` returns `attached: true` with no error; the handler (`RenderHandler.cpp:447`) calls `SetTextureParameterValue` without validating the parameter exists, so the render target is never actually sampled by the volume. Seeded family tag `unvalidated-texture-param`; noted sibling `material.authoring.set_texture_parameter_value` (`MaterialAuthoringHandler.cpp:1973`) has the same non-validation pattern via a separate code path.
