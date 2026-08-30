---
id: B-niagara-set-module-input-vec2
title: "niagara.set_module_input cannot set a Vector2D input — a 2-component value is wrongly rejected (and a 3-component value silently mistypes the pin)"
status: IN-REVIEW
severity: Medium
category: bug
tags: [niagara, set-module-input, vector2d, type-inference, sprite-size, unsupported-input-value]
---

# `niagara.set_module_input` has no FVector2D path: the natural 2-component value is rejected, a wrong 3-component value silently succeeds

`niagara.set_module_input` infers the Niagara input type from the JSON
shape of `value` when no override pin exists yet (the common
fresh-module-input case). The inference table in
`NiagaraEditHandler.cpp` (`InferNiagaraInputType` + `JsonValueToPinDefaultString`,
the two anonymous-namespace helpers near lines 73-182) handles:

- bool, number (float)
- arrays of length **>= 3** -> Vec3, **>= 4** -> Vec4
- objects with `{r,g,b[,a]}` -> Color, `{x,y,z[,w]}` -> Vec3/Vec4

There is **no case for a 2-component vector** (FVector2D / `FVector2f`).
A 2-element array (`[8,8]`) and a 2-field object (`{x:8,y:8}`) both fall
through `JsonValueToPinDefaultString` to an empty string, so
`InferNiagaraInputType` returns false and the call is rejected:

```
[UNSUPPORTED_INPUT_VALUE] Module input values must be a bool, number,
vector object/array, color object, or an existing override pin default string.
```

This blocks the standard way to set a Vector2D module input. The most
common one is **`Sprite Size`** on the core `Initialize Particle` module
(`/Niagara/Modules/Spawn/Initialization/InitializeParticle`), a
`Vector 2D` pin — setting sprite dimensions is a routine VFX-authoring
step.

## Two defects, same missing case

1. **A valid documented Vector2D value is wrongly rejected.** The error
   text claims a "vector object/array" is accepted, but only 3- and
   4-component vectors are. The natural Vec2 spelling for a Vec2 input
   fails, and the message misdirects the caller into thinking the *value*
   is the problem (the agent who hit this assumed a size-mode static
   switch was gating the pin; the real cause is the absent Vec2 branch).

2. **A wrong 3-component value silently succeeds with a mistyped pin.**
   Passing `[8,8,0]` to the same `Sprite Size` (Vec2) input returns
   `success:true` and creates an override pin **typed Vec3** on a Vec2
   input — a silent type mismatch the inference produced. So the only
   value that "works" for a fresh Vec2 input is the *wrong-shaped* one,
   and it corrupts the pin type.

## Why it matters

`set_module_input` is the primary module-authoring path. FVector2D is a
first-class Niagara input type (Sprite Size, UV scale/offset, 2D ranges).
A caller doing exactly what the input wants — a 2-component value — is
blocked, while the wrong 3-component value is accepted and mistypes the
override pin. The error string actively masks the real cause.

## Repro (verbatim, replay-confirmed via mcp__editor-automation__call)

1. `niagara.create_system` `{name:"NS_OracleVec2", savePath:"/Game/FuzzVFX"}` -> ok
2. `niagara.create_emitter` `{name:"E_OracleVec2", savePath:"/Game/FuzzVFX"}` -> ok
3. `niagara.add_emitter` `{systemPath:".../NS_OracleVec2.NS_OracleVec2", emitterPath:".../E_OracleVec2.E_OracleVec2", name:"OracleEmber"}` -> ok
4. `niagara.add_module` `{assetPath:".../NS_OracleVec2.NS_OracleVec2", emitter:"OracleEmber", modulePath:"/Niagara/Modules/Spawn/Initialization/InitializeParticle.InitializeParticle", scriptUsage:"ParticleSpawnScript"}` -> `nodeId:"A215F1F84255F2EDE8BBB7955395F9F1"`
5. **FAIL** `niagara.set_module_input` `{assetPath:".../NS_OracleVec2.NS_OracleVec2", emitter:"OracleEmber", entryId:"A215F1F84255F2EDE8BBB7955395F9F1", inputName:"Sprite Size", value:{x:8,y:8}, scriptUsage:"ParticleSpawnScript"}`
   -> `[UNSUPPORTED_INPUT_VALUE] Module input values must be a bool, number, vector object/array, color object, or an existing override pin default string.`
6. **FAIL** same call with `value:[8,8]` -> identical `[UNSUPPORTED_INPUT_VALUE]`.
7. **WRONGLY PASSES** same call with `value:[8,8,0]` (a 3-component value on a Vec2 input)
   -> `{success:true, operation:"set_module_input", inputName:"Sprite Size", pinId:"38A8207141C184572DF4AE9E5188BB62", index:0}` — creates a Vec3-typed override pin on a Vector2D input.

**Workaround:** none clean for a fresh pin. Once any override pin exists
(e.g. created by the wrong `[8,8,0]` call above, which leaves it Vec3),
the `ExistingPin` branch accepts the Niagara string form
`"(X=8.0,Y=8.0)"` — but the pin is then mistyped, so this is not a real
workaround.

## Fix

Add an FVector2D case to both `InferNiagaraInputType` and
`JsonValueToPinDefaultString` (`NiagaraEditHandler.cpp`):
- a 2-element array and a `{x,y}`-only object should serialize to the
  Niagara Vec2 pin default string `(X=..,Y=..)` and infer
  `FNiagaraTypeDefinition::GetVec2Def()`.
Independently, the inference should not accept an array/object whose
component count does not match the target pin's declared type
(reject `[8,8,0]` on a Vec2 input rather than creating a mistyped pin),
or at minimum coerce to the pin's true type. Note the typed Vec2 path for
the *parameter*-store sibling `niagara.set_parameter` already shipped
(`B-niagara-bool-vec2-struct-rawbytes`), but that fix did not touch this
module-input inference table.

## Cross-ref

- `B-niagara-bool-vec2-struct-rawbytes` (DONE) — added typed Vec2 support
  to `niagara.set_parameter` / `BuildParameterValueJson` (a different RPC
  and code path). The `set_module_input` inference helpers were not
  covered, so Vec2 module inputs remain unsettable.
- `B-niagara-module-input-stack-infer` (IN-REVIEW) — same RPC,
  `ApplyModuleMutation`, different defect (stack inference, not value
  inference).

## History
- `#2-vec2-inference-branch` `IN-REVIEW` developer — Fixed defect #1 (the primary, root-cause half): added an FVector2D branch to both module-input value-inference helpers in `Source/EditorAutomationRpcGateway/Private/Handlers/Niagara/NiagaraEditHandler.cpp`. `JsonValueToPinDefaultString` now formats a 2-element array and an `{x,y}`-only object (checked after the `{x,y,z}` case) to the Niagara Vec2 pin default string `(X=..,Y=..)`; `InferNiagaraInputType` now infers `FNiagaraTypeDefinition::GetVec2Def()` for a length-2 array (the `Num()` ladder: >=4 Vec4 / >=3 Vec3 / else Vec2) and for an `{x,y}`-only object, mirroring the explicit-type Vec2 path already at :1154. A 1-element array still yields an empty default string (the array branch guards `Num()>=2`) and stays rejected via the existing `OutDefaultValue.IsEmpty()` gate, so no behavior change there. The error string at :719 ("vector object/array") is now accurate for all of Vec2/Vec3/Vec4 and was left as-is. Defect #2 (the silent mistype of `[8,8,0]` on a Vec2 input) is NOT fixed here: detecting a component-count mismatch needs the module's DECLARED input type at the fresh-pin path, and the only exported discovery API (`FNiagaraStackGraphUtilities::GetStackFunctionInputs`) requires an `FCompileConstantResolver`/SVM context the non-API overload can't supply cross-module — a materially larger, riskier change (both the correctness and adversarial lenses flagged it as the over-scoped half). Recommend a follow-up ticket for declared-type coercion rather than bundling it here. Regression test: `Source/EditorAutomationRpcGateway/Private/Tests/Niagara/TestNiagaraSetModuleInput.cpp::FNiagaraSetModuleInputAcceptsVector2DTest` (`EditorAutomationRpcGateway.niagara.set_module_input.AcceptsVector2D`) — adds an `InitializeParticle` module to ParticleSpawn on a live transient system, dispatches the real `niagara.set_module_input` RPC with `{x:8,y:8}` (Sprite Size) and `[16,32]` (a distinct input, forcing the fresh-pin inference branch), asserts neither is rejected with UNSUPPORTED_INPUT_VALUE, and locates the created override pin by its returned `pinId` to assert it is typed `GetVec2Def()` (not Vec3/empty). Reverting the helper branch fails the test (rejection → no success, no pinId). Did not compile/run (later phase).
- `#1-initial-repro` `OPEN` reporter — Replay-confirmed via mcp__editor-automation__call: on a fresh `Initialize Particle` module in `ParticleSpawnScript`, `niagara.set_module_input` for the `Sprite Size` (Vector 2D) input rejects both `{x:8,y:8}` and `[8,8]` with `[UNSUPPORTED_INPUT_VALUE] Module input values must be a bool, number, vector object/array, color object, or an existing override pin default string.`, while `[8,8,0]` (a 3-component value) returns `success:true` and creates a Vec3-typed override pin on the Vec2 input. Source-confirmed: `NiagaraEditHandler.cpp` `JsonValueToPinDefaultString` only formats arrays of length >= 3 / objects with `{x,y,z}` (lines ~91-140) and `InferNiagaraInputType` only infers Vec3/Vec4 from >= 3-length arrays / `{x,y,z}` objects (lines ~162-180) — there is no 2-component (FVector2D) branch, so a 2-element value yields an empty default string and is rejected as UNSUPPORTED_INPUT_VALUE; the error text claims "vector object/array" is supported, masking that only 3-/4-component vectors are. The parameter-store Vec2 fix in `B-niagara-bool-vec2-struct-rawbytes` does not cover this module-input inference path.
- `#3-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 2 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
