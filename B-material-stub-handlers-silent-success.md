---
id: B-material-stub-handlers-silent-success
title: "material.authoring.set_cast_shadows / set_material_parameter are no-op stubs that return success"
status: DONE
severity: High
category: bug
tags: [material, material-authoring, stub, silent-failure]
---

# Stub material authoring handlers return success without doing the work

Two registered handlers in `MaterialAuthoringHandler.cpp` are implemented
as success-returning no-ops:

**`material.authoring.set_cast_shadows`** (lines ~2506–2523):
```cpp
bool CastShadows = Ctx.GetBool(TEXT("castShadows"), true);
Result->SetStringField(TEXT("assetPath"), AssetPath);
Result->SetBoolField(TEXT("castShadows"), CastShadows);
Ctx.SendSuccess(Result);
```
No write to the material — `Material` isn't even loaded. The wiki blurb
admits "currently a no-op that records the request", but the handler still
**returns success**, so callers wiring this into automation have no way to
detect that nothing happened. Echoing back the requested value
strengthens the false signal.

**`material.authoring.set_material_parameter`** (lines ~2414–2433):
```cpp
Result->SetBoolField(TEXT("parameterSet"), true);
Ctx.SendSuccess(Result);
```
Same shape — loads no asset, mutates nothing, replies `parameterSet: true`.
The wiki labels it a "stub generic dispatcher" but the contract is
indistinguishable from a working method.

**Why it matters:**

- A caller running `set_cast_shadows({assetPath, castShadows: false})` followed by
  `compile_material` will see `compiled: true` and conclude the shader was
  rebuilt with shadows disabled. Visual / runtime testing is required to
  catch the failure — the JSON-RPC layer cannot.
- `set_material_parameter` is the obvious entry point for anyone who hasn't
  read the wiki: the name, the param list (`assetPath`, `parameterName`,
  `parameterType`), and the `parameterSet: true` response all suggest it
  does what it says.

**Fix options (any of):**

1. **Implement them.** `set_cast_shadows` should toggle `Material->BlendableLocation`
   adjacent shadow flags via the proper editor surface (the wiki notes
   "per-shading-model property surface") and call `PostEditChange()`. The
   blocking factor is per-shading-model UPROPERTY routing; for many shading
   models the flag is `bCastDynamicShadowAsMasked` (Masked) or
   `BlendableOutputAlpha` etc. Investigate `FMaterialEditorUtilities`.
2. **Make them fail loud.** Replace `SendSuccess` with
   `SendError("NOT_IMPLEMENTED", "set_cast_shadows is a stub …")` until
   real implementation lands. Removes the silent-success footgun in one
   line each. For `set_material_parameter`, route through to the typed
   `set_scalar_parameter_value` / `set_vector_parameter_value` /
   `set_texture_parameter_value` based on `parameterType` instead of
   returning a meaningless `parameterSet: true`.

Option 2 is the minimum safe change; option 1 closes the capability gap.

## Repro

1. Create any material `M`.
2. Call `material.authoring.set_cast_shadows({assetPath: M, castShadows: false})`.
3. Open the material; cast-shadows is unchanged. RPC returned `castShadows: false`.
4. Symmetric repro for `set_material_parameter`.

## History
- `#1-stub-handlers-return-success` `OPEN` reporter — Material API audit confirmed `material.authoring.set_cast_shadows` and `material.authoring.set_material_parameter` are no-op stubs in `MaterialAuthoringHandler.cpp` that send `SendSuccess` without mutating any asset. The wiki flags both as stubs, but the handlers still return success-shaped JSON, leaving callers unable to distinguish "applied" from "ignored". Either implement them or convert to `SendError("NOT_IMPLEMENTED", …)` so the contract is honest.
- `#2-reviewed-and-confirmed` `OPEN` reporter — Re-read `MaterialAuthoringHandler.cpp` lines 2414–2433 (`set_material_parameter`) and 2506–2523 (`set_cast_shadows`). Both confirmed: `set_material_parameter` never calls `LoadObject` for the material and replies `parameterSet: true`; `set_cast_shadows` never loads the material and echoes the requested flag back as if applied. No duplicate tickets in the board folder. Severity High upheld — discovery-schema-driven callers and `compile_material` follow-ups have no way to detect the no-op, and the minimal Option-2 fix (swap `SendSuccess` → `SendError("NOT_IMPLEMENTED", …)`) is one line per handler.
- `#3-stubs-return-not-implemented` `IN-REVIEW` developer — Replaced SendSuccess in MaterialAuthoringHandler.cpp set_cast_shadows and set_material_parameter with SendError("NOT_IMPLEMENTED", ...). set_material_parameter's error message points callers at the typed set_scalar/vector/texture_parameter_value alternatives. Regression test TestMaterialAuthoringStubsNotImplemented.cpp invokes both and asserts ErrorCode == "NOT_IMPLEMENTED".
- `#4-verify-not-implemented` `DONE` tester — Verified: `material.authoring.set_cast_shadows` with `assetPath=/Game/App/UI/Test/W_McpVerifyTemp_B_material_stub_handlers_silent_success` and `castShadows=false` returned `Error: NOT_IMPLEMENTED`; `material.authoring.set_material_parameter` with `assetPath=/Game/App/UI/Test/W_McpVerifyTemp_B_material_stub_handlers_silent_success`, `parameterName=VerifyParam`, and `parameterType=scalar` returned `Error: NOT_IMPLEMENTED` with typed setter guidance.
