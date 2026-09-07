---
id: B-static-switch-input-connection-lost-across-save-and-reload
title: "New material-function outputs lack persistent IDs, so saved function-call consumers can disconnect on reload"
status: OPEN
severity: High
category: bug
tags: [material, material-function, function-output, persistent-id, static-switch, serialisation, save, reload, default-material, silent-revert]
encounters: 3
costly: 2
lastSeen: 2026-09-07T06:50:00Z
---

# Function-output IDs can invalidate a saved consumer on reload

## Current diagnosis and accepted scope

Classification: reformulate. Worker brief: PARTLY TRUE. PinWright's typed and generic FunctionOutput creation paths omitted ConditionallyGenerateId. A caller can serialize an invalid output ID while its transient output pointer remains usable; loading the function creates an ID and caller remapping drops the old reference. This is a source-confirmed defect matching the reported null/-1 state. The original failed graph and IDs are unavailable, so attribution of the historical M_FPSArms incident remains an inference. A/B serialization and post-hoc connection timing are not established causes.

Production sites are `Source/PinWright/Private/Material/MaterialExpressionFactory.cpp` (both owner overloads) and `Source/PinWright/Private/Handlers/Material/MaterialAuthoringHandler.cpp` (`add_function_output`). Engine `Runtime/Engine/Private/Materials/MaterialExpressions.cpp` initializes output IDs in `UMaterialExpressionFunctionOutput::PostLoad`, caches them in function calls, and remaps callers by ID in `UpdateFromFunctionResource`; an unmatched ID clears the consumer input to null/INDEX_NONE. `Runtime/Engine/Public/Materials/MaterialExpressionMaterialFunctionCall.h` serializes `ExpressionOutputId` but marks `ExpressionOutput` transient. The native creation precedent is `Editor/MaterialEditor/Private/MaterialEditingLibrary.cpp`.

## What happened, in order, all measured

On `/Game/FPS/Player/M_FPSArms`, a `MaterialExpressionStaticSwitchParameter` named
`EnableViewmodelFOV` gating a `MaterialFunctionCall` into World Position Offset.

1. Bound input 0 through the engine API — `MaterialEditingLibrary.connect_material_expressions(call,
   '', switch, 'True')` returned **True**.
2. Read it back: `get_material_node_details` reported
   `A: {Expression: MaterialExpressionMaterialFunctionCall_0, OutputIndex: 0}`. **Connected.**
3. `compile_material`: `shaderCompile.succeeded: true`, **`rendersDefaultMaterial: false`**.
4. `asset.generate_thumbnail` on the instance: `usingDefaultMaterial: false`,
   `meanLuminance 0.132` — dark, correct.
5. **Rendered correctly in a PIE frame** (`Docs/fps/evidence/player/b09-01_idle.png`): the arms read
   as dark fabric where every previous build had shown pale grey.
6. `asset.save {force: true}` -> `saveState: "written"`, `sizeBytes: 18041`.

Then the editor restarted (twice, for unrelated crashes). On the next session, with the `.uasset`
unchanged on disk at 18 041 bytes:

```
get_material_info M_FPSArms
-> shaderCompile: {status: "failed", errors: ["(Node StaticSwitchParameter) Missing A input"],
                   rendersDefaultMaterial: true}
get_material_node_details <switch>
-> A: {Expression: null, OutputIndex: -1}
   B: {Expression: MaterialExpressionConstant3Vector_0}     <- input 1 survived
```

Input 0 is null again. Input 1, connected in the same session by the same means, survived. The next
PIE frame had pale arms again.

## Why this is worse than the sibling ticket

`B-connect-nodes-accepts-true-false-pin-names-on-static-switch-and-wires-nothing` covers
`connect_nodes` failing to bind input 0 while reporting success. This is a different failure and it
defeats the workaround for that one: the connection **was** made (by the engine API, not the verb),
**was** verified by read-back, **was** proven by a shader compile and by a rendered frame, and
**was** saved with the strongest success signal the save verb has. Every check available to a caller
passed. It still did not survive serialisation.

The original report treated restart as the only verification route. The current tree has asset.reload for eviction and disk readback. The regression below saves and unloads both the function dependency and its consumer before reloading. Ordinary load_asset/LoadObject on a resident object is not disk verification.

Cost here: three builds of a first-person viewmodel rendering as the engine Default Material. Twice
I reported the material fixed on evidence that was, at the time, complete and correct.

## Original requested investigation

1. Find why input 0 of `MaterialExpressionStaticSwitchParameter` is not serialised, or is dropped on
   load, when input 1 on the same node is. `A` and `B` are both `FExpressionInput` on the same
   class, so an asymmetry between them is the place to look.
2. If the write path is at fault rather than the load path, `asset.save` should not report
   `saveState: "written"` over a graph it did not fully serialise.
3. A post-save verification hook would make this class of defect visible: re-read the graph from the
   saved package rather than from memory and diff. `asset.save`'s own docs already warn that reading
   back from `load_asset` returns the in-memory object — this is exactly the case that warning
   exists for, and nothing currently acts on it.

Accepted implementation scope is output-ID initialization and regression coverage. General automatic post-save graph comparison and changes to asset.save are out of scope: writing an invalid ID is still a real package write, followed here by semantic invalidation during load.

## Workaround, and why I took it

Removed the static switch entirely. `M_FPSArms` is used by one component — the first-person
viewmodel — so the switch had no purpose there; it was mirroring a pattern from a weapon master
shared with world and AI weapons. The `MaterialFunctionCall` now drives World Position Offset
directly, so there is no input-0 to lose.

After: `M_FPSArms.uasset` 16 898 bytes, `EnableViewmodelFOV` **0 occurrences** in the bytes,
`MF_ViewmodelFOV` still 2, `compile_material` -> `succeeded: true, rendersDefaultMaterial: false`,
`generate_thumbnail` -> `usingDefaultMaterial: false, meanLuminance 0.131`. The stale
`EnableViewmodelFOV` override on `MI_FPSArms` was cleared with
`clear_parameter_override` so the instance does not carry an override for a parameter that no longer
exists.

Anyone who needs the switch (a material shared between viewmodel and world) cannot take this
workaround, which is why the ticket stands.

severity rationale: impact=valid-looking function-call connections can disappear on reload and cause fallback rendering x reach=consumers of newly created function outputs missing persistent IDs -> High. Reach is not every static-switch material.

## Fix

The root cause is missing persistent identity on newly created function outputs. Both owner overloads in `Source/PinWright/Private/Material/MaterialExpressionFactory.cpp` now call `ConditionallyGenerateId(false)` after successful property application and before collection insertion. `Source/PinWright/Private/Handlers/Material/MaterialAuthoringHandler.cpp` applies the same initialization in `material.authoring.add_function_output`. A valid explicitly supplied ID is preserved.

Added `Source/PinWright/Private/Tests/Material/TestStaticSwitchFunctionOutputPersistence.cpp`, test `PinWright.material.authoring.connect_nodes.StaticSwitchFunctionOutputPersistence`, and documented the creation contract in `docs/wiki-src/material.authoring.md`. Deliberate non-changes: no FunctionInput, engine, MGIR, asset.save, generic post-save comparison, connection-time ID regeneration, or repair of existing malformed assets/lost connections. The original incident attribution remains inferred.

## Regression and verification

`PinWright.material.authoring.connect_nodes.StaticSwitchFunctionOutputPersistence` creates function outputs through both production handler routes in independent `/Game/PinWrightTests/<GUID>` fixtures. It asserts an immediate nonzero output ID and the matching caller ID, connects StaticSwitch A through authoring and B through graph handlers, and also covers the reporter's direct engine True-pin connection. It saves the function dependency first and the consumer second, releases strong owners, unloads both through native package unloading (including safe ResetLoaders), proves eviction, then reloads the consumer and its dependency. Readback checks stable output/caller IDs, the reconstructed output pointer, and surviving A/B connections to nodes in the reloaded material. Local assertions also cover the material factory overload and preservation of an explicitly supplied ID.

Counterfactual: reverting initialization fails the immediate output-ID validity assertion; the old caller can save a zero ID, then fail to match the function's postload-generated ID and lose A while B survives. Resident-only readback fails the eviction assertions.

NOT RUN: build, automation, editor, and runtime verification. Source-only implementation; IN-REVIEW.

## History
- `#1-filed` `OPEN` reporter — Found on the FPS PLAYER stream after the arms viewmodel rendered pale in a frame that followed two editor restarts, having rendered dark before them. The measurement chain above is the whole story; the earlier ticket's `connect_nodes` defect is real but separate, and this one survives its workaround.
- `#2-narrowed-by-a-control-it-is-the-POST-HOC-binding-path` `OPEN` reporter — **The WEAPONS stream ran the restart test on an independently authored switch and input 0 SURVIVED, which narrows this from a serialisation bug to a connection-path bug.** Their control, on `/Game/FPS/Weapons/Materials/M_WPN_OpticLens`, same node class (`MaterialExpressionStaticSwitchParameter`), after a full save -> process death -> reload:

```
properties.A = {Expression: ".../M_WPN_OpticLens:MaterialExpressionTransform_1", OutputIndex: 0}
properties.B = {Expression: ".../MaterialExpressionConstant3Vector_0"}
compile_material -> compileSucceeded true, shaderCompile.rendersDefaultMaterial FALSE
```

  **The difference is how the input was set, not what node it is.** Theirs was authored in a single `material.compile_mgir` `Extend` call, which emits `True:` / `False:` as part of the expression's construction — the input is populated while the expression is being created. Mine was bound afterwards with `MaterialEditingLibrary.connect_material_expressions(call, '', switch, 'True')` on an expression that already existed. That post-hoc binding is the half that did not survive.

  So the search is much smaller than `#1` implied: not "`FExpressionInput` A does not serialise" but "an `FExpressionInput` written after the expression exists is not marked dirty, or is not written through to the serialised input, while the same field set at construction is". Retitled accordingly.

  It also makes this and `B-connect-nodes-accepts-true-false-pin-names-on-static-switch-and-wires-nothing` two depths of one problem rather than two problems: `connect_nodes` reports success and binds nothing at all; `connect_material_expressions` binds something that passes every in-session check — read-back, shader compile, rendered frame, `saveState: "written"` — and then does not serialise. Both are the post-hoc connection path being unreliable on this node. **Practical guidance until it is fixed: author expression inputs at construction (`material.compile_mgir` `Extend`) rather than connecting them afterwards**, which is a working route past both tickets and is what the other stream did without hitting either.

  Credit where due: the control was theirs, run at my request after I warned them their hook might have the same defect. It did not, and their negative result is worth more to the fix than my positive one.
- `#3-function-output-id-initialization` `IN-REVIEW` developer — Reformulated from general post-hoc StaticSwitchParameter persistence to missing persistent FunctionOutput IDs. Initialized new output IDs in both FMaterialExpressionFactory::Create overloads and material.authoring.add_function_output while preserving valid IDs. Added PinWright.material.authoring.connect_nodes.StaticSwitchFunctionOutputPersistence with production-handler creation, saved dependency/consumer packages, genuine eviction, and reloaded ID/A/B assertions. Updated material.authoring documentation. Static inspection only; build, automation, editor, and runtime verification were not run. The original incident attribution remains inferred; asset.save graph verification is outside this fix.
- `#4-returned` `OPEN` PLAYER — Live confirmation of the reformulated diagnosis, plus the half the fix does not reach. Fresh editor process (log opened 2026-09-07T06:38:37Z) loading `/Game/FPS/Player/M_FPSArms`, last saved 2026-09-06T06:30:41Z with `MaterialFunctionCall(/Game/FPS/Player/MF_ViewmodelFOV) -> WorldPositionOffset`. After reload, `material.decompile_mgir` emits the call node as `%nD2112FA44C52 = function_call ` + backtick + `/Game/FPS/Player/MF_ViewmodelFOV.MF_ViewmodelFOV` + backtick + `()` — **no arguments** — and the material block has **no `output WorldPositionOffset:` line at all**. `get_material_node_details {nodeId:"Main"}` lists only BaseColor/Metallic/Specular/Roughness/Normal. So BOTH the call's `FOVScale` input and its consumer link into WPO were dropped by the load, and the orphaned `CollectionParameter(MPC_WPN_Viewmodel, ViewmodelFOVScale)` that fed the input is left dangling in the graph.

  Two things this adds. **First, it kills the node class named in the original title.** The graph that reverted this time contains no `StaticSwitchParameter` at all — build 07 deleted it precisely so there would be no input 0 to lose, wired the function call straight into WPO, and the wire still did not survive. Whatever the mechanism is, it is not specific to a static switch, and it takes out two different pins on the same consumer in one load, which is what an invalidated function-side ID predicts and which a "one post-hoc input write fails to dirty" theory does not.

  **Second, the control in the same process.** `/Game/FPS/Weapons/Materials/M_WPN_OpticLens`, another stream's material, implements the identical viewmodel-FOV math with plain expression nodes (`WorldPosition(WPT_CameraRelative)` -> `Transform` World->View -> `Multiply` by an appended `(scale-1, scale-1, 0)` -> `Transform` View->World) and **has no MaterialFunctionCall in it**. In the same fresh process it decompiles with `output WorldPositionOffset:` intact and both `True:`/`False:` inputs of its own `StaticSwitchParameter` bound. Surviving graph: no function call. Reverting graph: function call. That is the difference the two assets actually have.

  **The gap the `#3` fix leaves open, and why this is returned rather than confirmed.** `#3` initializes output IDs in the two `FMaterialExpressionFactory::Create` overloads and in `add_function_output` — creation-time only. `MF_ViewmodelFOV` was authored 2026-09-03, before that shipped, so whatever it serialized is what it still has, and every consumer of it will keep silently losing its wires on every single load, forever. Nothing in the plugin detects this or repairs it: there is no verb that reports "this function's output ID is invalid", `material.decompile_mgir` renders the damaged graph as perfectly legal text (a zero-argument `function_call` and a missing `output` line both being things a caller might legitimately have authored), and the only route back is to notice the missing wire by eye and rebuild. Suggested acceptance for this ticket, beyond the creation-time fix: (a) a detection path that flags a function whose outputs carry invalid IDs, or a repair on load/save, and (b) a note in the `material.mgir` / `material.authoring` docs that a function authored before the fix stays poisoned, since a caller reading only the status would conclude their existing assets are safe.

  **Workaround used, and it is the durable one:** drop the `MaterialFunctionCall` and inline the math as plain expression nodes in one `material.compile_mgir` `Append` document, node-for-node on the structure the surviving control uses. `M_FPSArms` now carries `WorldPosition(WPT_CameraRelative)` -> `Transform` -> `Multiply` -> `Transform` -> `output WorldPositionOffset` with no function dependency, so there is no function-output ID left to invalidate. Cost: the HLSL-equivalent node chain is now duplicated in two materials owned by two streams, which is the real price of the defect.
