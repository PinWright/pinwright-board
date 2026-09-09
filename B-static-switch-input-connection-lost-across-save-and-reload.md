---
id: B-static-switch-input-connection-lost-across-save-and-reload
title: "New material-function outputs lack persistent IDs, so saved function-call consumers can disconnect on reload"
status: IN-REVIEW
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

- `#5-function-input-identity-and-rebuild-stability` `IN-REVIEW` developer — The #4 return is explained by two live production paths the #3 fix left open, both read off UE 5.8 source. **(a) The input half was never fixed.** `material.authoring.add_function_input` and both `FMaterialExpressionFactory::Create` overloads never initialised `UMaterialExpressionFunctionInput::Id`, so a call node caches an all-zero `ExpressionInputId`; `UMaterialExpressionFunctionInput::PostLoad` mints a valid one, `FindInputById` matches nothing and `UpdateFromFunctionResource` drops the input connection — which is exactly the lost `FOVScale` pin, alongside the output pin #3 had already covered. **(b) Rebuilding a function renumbers its pins.** `material.compile_mgir` `Append` empties the function graph (`ClearMaterialFunctionGraph`) and re-emits every pin as a new object with a new GUID, and outside a cook `ConditionallyGenerateId` is plain `FGuid::NewGuid()` (`MaterialExpressions.cpp` `CookDeterminism::NewGuid`, 5.8:403 — deterministic ONLY under `IsRunningCookCommandlet()`), so one recompile silently disconnects every already-saved consumer, with the consumers untouched and no error anywhere. Fixed: `Source/PinWright/Private/Material/MaterialExpressionFactory.cpp` (both overloads) and `Source/PinWright/Private/Handlers/Material/MaterialAuthoringHandler.cpp` `add_function_input` now initialise the input Id; new shared header `Source/PinWright/Private/Handlers/Material/MaterialFunctionIdentity.h` carries `EnsurePersistentIds` / `CaptureIds` / `RestoreIds` / `CollectUnstableFunctionCalls`; `SaveMaterialFunctionAsset` refuses to write an anonymous pin and `use_material_function` repairs + persists the function before caching its GUIDs; `Source/PinWright/Private/MGIR/MGIRCompiler.cpp` `CompileFunctionBlock` brackets the Append rebuild with capture/restore by pin NAME. **Audit of the four force-regenerate sites named in the brief:** `MGIRExpressionEmitter.cpp` `RegisterExpression` (FunctionInput) and `WireRootOutput` (FunctionOutput) both changed `ConditionallyGenerateId(true)` → `(false)` — forcing threw away the identity the Append snapshot restores and gained nothing (the factory already initialises a genuinely new pin). The three `MaterialAuthoringHandler.cpp` sites are inside `SeedMaterialLayerFunctionTemplate`, reachable only from `create_material_layer` / `create_material_blend` on an asset created moments earlier in the same handler, so no consumer can exist; left alone. **Detection:** `material.authoring.get_material_info` gains a `functionCallIdentity` block — `{unstable: [{nodeId, functionPath, pinKind, pinName, reason}], warning}` — emitted ONLY when a call node caches a pin GUID that will not re-link, with `reason` one of `missing-persistent-id` / `stale-persistent-id` / `duplicate-persistent-id`. **Stated limitation, because it bounds acceptance (a):** an asset poisoned BEFORE this guard shipped reads clean, and no in-memory read can see otherwise — the engine's `PostLoad` replaces the invalid on-disk GUID and `UpdateFromFunctionResource` discards the caller's stale GUID, both during load and before any verb can look. Re-saving such a function persists the GUID its `PostLoad` minted and ends the every-load loop; the wires already dropped must be re-authored. That is documented in `Docs/wiki-src/material.authoring.md` (which also supersedes the #3 output-only paragraph) and in `Docs/wiki-src/material.mgir.md` (Append pin identity), covering acceptance (b). Regression test `PinWright.material.authoring.use_material_function.FunctionPinIdentitySurvivesReload` in `Source/PinWright/Private/Tests/Material/TestMaterialFunctionPinIdentity.cpp`: authors a function input + output through the production handlers, binds a caller with `use_material_function`, wires BOTH pins through `connect_nodes`, saves function then consumer, unloads both packages and proves eviction, reloads from disk, then recompiles the function in MGIR `Append` and force-regenerates one pin to drive the detector. Counterfactuals: revert the `add_function_input` initialisation and “Production-created function input immediately has a persistent ID” fails (that assertion runs before `use_material_function`, so the repair there cannot mask it); revert the `CompileFunctionBlock` capture/restore and “Rebuilt input keeps its previous persistent ID” fails because the re-emitted pin carries a fresh `FGuid::NewGuid()`; revert `AddUnstableFunctionCallReport` and “A caller holding a GUID no pin carries is reported” fails because no `functionCallIdentity` block is emitted. NOT RUN: build, automation, editor and runtime verification — source-only, per sprint protocol.
- `#6-verifier-follow-up-unconditional-persist-and-orphan-warning` `IN-REVIEW` developer — Three verifier findings fixed. **(1) The #5 repair could never fire, and that was the whole point of the ticket.** `use_material_function` gated its function re-save on `EnsurePersistentIds(Func) > 0`, an IN-MEMORY validity read — but `UMaterialExpressionFunctionInput::PostLoad` / `FunctionOutput::PostLoad` call `ConditionallyGenerateId(false)` on every load, so a function whose `.uasset` holds an all-zero pin GUID still presents a valid one in memory. The count was always 0, the function was never re-written, the call node cached the GUID PostLoad had just minted, and the next load minted a different one: the legacy asset kept losing its wires forever, which is exactly the state #4 was returned over. `Source/PinWright/Private/Handlers/Material/MaterialAuthoringHandler.cpp` now dirties and writes the function UNCONDITIONALLY before `SetMaterialFunction`, and reports it: the response gains a `functionIdentity` block (`assetPath` + the standard `AddAssetSaveReport` shape, so a PIE-blocked write reports `saveState: blockedByPie`), plus a `warnings[]` entry when the write did not land, because the node's wires are not durable then. `FINALIZE_EXPR_AND_RESPOND` was inlined for this verb so the response can carry those fields; the verb description declares the write. **(2) Wording corrected** — `SaveMaterialFunctionAsset` repairs an anonymous pin and writes, it does not refuse; `Docs/wiki-src/material.authoring.md` said "refuses to write" and now says "repairs", and the Limit paragraph is rewritten now that binding actually repairs a legacy function. #5's text stands as written and is superseded here. **(3) A rebuild that renames or drops a pin is now reported.** `RestoreIds` (`Handlers/Material/MaterialFunctionIdentity.h`) takes an optional out-array and returns the snapshot names no rebuilt pin claimed; `MGIR/MGIRCompiler.cpp` `CompileFunctionBlock` turns those into a warning naming the function and each orphaned pin, carried on new `FMGIRCompileResult::Warnings` / `FMGIRCompiledBlock::Warnings` into `material.compile_mgir`'s existing `warnings[]` array (shape unchanged, entries added). Silently discarding them made a rename indistinguishable from a clean rebuild, at the one moment the disconnection is visible. Regression test extended with `RunLegacyPoisonedFunctionFixture` in the same test id: builds a function whose pins are saved with NO GUID by construction (asserting `!Id.IsValid()` before the save, since every production route now initialises it), evicts the function package so the bind is a genuine disk load, binds with `use_material_function`, asserts `functionIdentity.saved`, wires both pins, saves the consumer, evicts both packages and reloads. Counterfactual for (1): restore the `> 0` gate and the bind writes nothing, so the second reload mints GUIDs that do not match what the consumer cached — “Legacy call input is still wired after reload” and “Legacy BaseColor still reads the call after reload” both fail, and “Bind persisted the function's pin identity” fails outright. NOT RUN: build, automation, editor and runtime verification — source-only, per sprint protocol.

- `#7-connect-nodes-verified-through-a-freed-pin-pointer` `IN-REVIEW` developer — The `#6` regression test's legacy fixture failed at `material.authoring.connect_nodes` with `CONNECTION_FAILED Input pin 'Scale' did not retain the requested source node.` while the wire had in fact been made. Root cause is in `Source/PinWright/Private/Handlers/Material/MaterialAuthoringHandler.cpp` and predates this ticket: the verb wrote through the `FExpressionInput*` returned by `FMaterialExpressionFactory::FindExpressionInputByName`, then called `Material->PostEditChange()` and read the SAME pointer back. On a `MaterialFunctionCall` target that pointer aims into `UMaterialExpressionMaterialFunctionCall::FunctionInputs`, and `UMaterial::PostEditChangePropertyInternal` -> `UpdateCachedExpressionData` -> `FMaterialCachedExpressionData::AnalyzeMaterial` -> `UpdateForExpressions` calls `FunctionCall->UpdateFromFunctionResource()` (`MaterialCachedData.cpp` 5.8:725-734), which does `TArray OriginalInputs = MoveTemp(FunctionInputs)` and lets that buffer free at scope exit (`MaterialExpressions.cpp` 5.8:16102). The read-back therefore inspected released memory: it happened to still spell the new expression in the first fixture and did not in the legacy one, so a correct connection was refused. The verb now re-resolves the pin by name after `PostEditChange` and verifies the live pointer (`!InputPtr || InputPtr->Expression != SourceExpr`). No behaviour change for a genuinely dropped wire — the engine re-links the rebuilt pin by GUID, so a pin whose identity is broken still reports CONNECTION_FAILED. Covered by the existing `PinWright.material.authoring.use_material_function.FunctionPinIdentitySurvivesReload`; counterfactual: revert the re-resolve and the legacy fixture's `material.authoring.connect_nodes` verifies a freed buffer again, which is the observed failure. `material.graph.connect_nodes` does not read back after its edit and needs no change. NOT RUN: build, automation, editor and runtime verification — source-only.
