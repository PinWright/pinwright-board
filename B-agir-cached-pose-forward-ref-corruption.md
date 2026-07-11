---
id: B-agir-cached-pose-forward-ref-corruption
title: "anim.compile_agir saves an uncompilable AnimBlueprint: use_cached_pose forward reference leaves SaveCachedPoseNode null, the 'EarlyValidation will fix up' claim is false, and no save-time gate blocks it"
status: OPEN
severity: Critical
category: bug
tags: [agir, animgraph, cached-pose, forward-ref, compile-agir, integrity-gate, cold-load, corrupt-asset]
encounters: 1
lastSeen: 2026-07-11T04:52:35.3332310+03:00
---

# `anim.compile_agir` persists a structurally-broken AnimBlueprint on a cached-pose forward reference

Compiling AGIR text into an AnimBlueprint whose `output` consumes a
`use_cached_pose` **before** the matching `save_cached_pose` appears in the
text (a forward reference — the normal emission order for a locomotion
AnimBP, whose AnimGraph output line precedes the cache-body block) does the
following:

1. `anim.compile_agir` emits a non-fatal warning
   `AGIR_CACHED_POSE_FORWARD_REF: use_cached_pose source='Locomotion' did not
   resolve at compile time; engine EarlyValidation will fix up on AnimBP
   compile` (x2) and returns success.
2. The `use_cached_pose` node is created with its `SaveCachedPoseNode` weak
   pointer left **null** — the linkage is never established.
3. `asset.save` persists the AnimBlueprint. **No save-time integrity gate**
   walks the AnimGraph to reject a dangling `use_cached_pose` before write.
4. On a **cold editor restart**, opening the asset succeeds but
   `blueprint.compile` FAILS with structural corruption:
   `Use cached pose 'Locomotion' does not have an associated Save Cached Pose
   node` (x2), `compiled:false`, `status:"Error"`.

So the "benign / EarlyValidation will fix up" reassurance is **false**: the
engine's EarlyValidation does NOT re-resolve the linkage on cold load, and the
tool has already reported success and saved. This is save-time asset
corruption the integrity gate let through — the exact class the widget/BP
corruption work (`B-bp-saved-state-corruption-mcp-edits`) hardens against, but
for a completely different asset type (AnimBlueprint), code path (AGIR
cached-pose compile), and root cause (unresolved `SaveCachedPoseNode` weak
ptr), so it is not covered by that ticket's Blueprint-graph / widget-tree
gates.

## Root cause (source-confirmed)

- Compile side (the defect) —
  `Plugins/PinWright/Source/PinWright/Private/AGIR/AGIRCompiler_CachedPose.cpp`,
  `CompileUseCachedPoseInstruction`. When the cache name is not yet in
  `CacheNameMap` (forward ref), it takes the else branch and only warns,
  leaving `SaveCachedPoseNode` null:

  Lines 177-186 (verbatim):
  ```
      if (UAnimGraphNode_SaveCachedPose* const* SavePtr = CacheNameMap.Find(CacheName))
      {
          UseNode->SaveCachedPoseNode = *SavePtr;
      }
      else
      {
          OutWarnings.Add(FString::Printf(
              TEXT("AGIR_CACHED_POSE_FORWARD_REF: use_cached_pose source='%s' did not resolve at compile time; engine EarlyValidation will fix up on AnimBP compile (line %d)"),
              *CacheName, Inst.SourceLine));
      }
  ```

  The false premise is stated in the comment at lines 164-169: "The engine's
  EarlyValidation re-resolves SaveCachedPoseNode from this name on next AnimBP
  compile, so leaving the weak ptr null when the save node hasn't been seen yet
  is recoverable." The cold-load `compile_failed` refutes this for the standard
  Mannequin locomotion topology.

- Confirming symptom — the decompiler derives the `source=` label from the
  weak pointer, not from a stored name:
  `Plugins/PinWright/Source/PinWright/Private/AGIR/AGIRTextEmitter.cpp` lines
  677-681:
  ```
      FString SourceName;
      if (UseNode && UseNode->SaveCachedPoseNode.IsValid())
      {
          SourceName = UseNode->SaveCachedPoseNode->CacheName;
      }
  ```
  Because the forward-ref path leaves `SaveCachedPoseNode` null, the variant's
  post-transfer decompile emits `use_cached_pose source=` with an **empty**
  label (the round-trip drops `Locomotion` to empty) — the observable signal
  that the linkage was never resolved and was saved broken.

## What it should do

The compile must not leave a `use_cached_pose` unlinked at persist time. Any of:
1. **Two-pass / deferred resolution** in `anim.compile_agir` — after all
   instructions are compiled, resolve every deferred `use_cached_pose` against
   the completed `CacheNameMap` and set `SaveCachedPoseNode` (the fix sketch
   already recorded on `F-agir-cliff-completion` #6: Pass 1 creates Save nodes,
   Pass 2 resolves Use nodes). Downgrade `AGIR_CACHED_POSE_FORWARD_REF` from a
   "benign" note to a resolved link.
2. **Save-time integrity gate for AnimBlueprints** — before `asset.save`
   persists, walk the AnimGraph and reject (or auto-repair) any
   `UAnimGraphNode_UseCachedPose` whose `SaveCachedPoseNode` is null / whose
   `NameOfCache` has no matching `UAnimGraphNode_SaveCachedPose`, mirroring what
   `ValidateBlueprintGraphIntegrity` does for Blueprint/Widget assets. A save
   should never write an AnimBlueprint that fails its own `blueprint.compile` on
   cold load.

Invariant to restore: an AnimBlueprint that `anim.compile_agir` reports as
compiled-and-saved must still `blueprint.compile` cleanly after an editor
restart.

## Verbatim repro (cold-load-confirmed this iteration)

1. `anim.decompile_agir { assetPath: "/Game/Characters/Mannequins/Animations/ABP_Manny" }` — capture the source AGIR (locomotion state machine fronted by a cached pose named `Locomotion`).
2. `animation.authoring.create_anim_blueprint` — create `/Game/Characters/Mannequins/Animations/ABP_Manny_Variant` on `SK_Mannequin`.
3. `anim.compile_agir { context: "/Game/Characters/Mannequins/Animations/ABP_Manny_Variant", mode: "Replace", text: "<the captured AGIR verbatim>", save: true }` — returns success with warnings `AGIR_CACHED_POSE_FORWARD_REF: use_cached_pose source='Locomotion' did not resolve at compile time; engine EarlyValidation will fix up on AnimBP compile` (x2).
4. `asset.save { assetPath: "/Game/Characters/Mannequins/Animations/ABP_Manny_Variant", force: true }` — ok.
5. Cold-restart the editor (editor.quit discard=true, confirmed process exit, headless relaunch).
6. `editor.open_asset { assetPath: "/Game/Characters/Mannequins/Animations/ABP_Manny_Variant" }` — `open_ok:true`.
7. `blueprint.compile { assetPath: "/Game/Characters/Mannequins/Animations/ABP_Manny_Variant" }` — `compiled:false`, `status:"Error"`, errors: `Use cached pose 'Locomotion'  does not have an associated Save Cached Pose node` (x2). Outcome: `compile_failed`.

Editor stayed up throughout (no crash). The corruption surfaces only on the
cold load — the warm post-compile decompile looked "non-empty with the same
state machine," masking it.

severity rationale: impact=corruption × reach=every-session -> Critical (AGIR
round-trip transfer of any cached-pose-fronted AnimBlueprint — the canonical
locomotion shape — emits the use before the save, so the forward-ref path is
the normal case, not a corner case).

## History
- `#1-initial-repro` `OPEN` reporter — Cold-load-confirmed corruption from a REALISM AGIR-transfer task (capture `ABP_Manny` AGIR, rebuild into a throwaway `ABP_Manny_Variant`). `anim.compile_agir` (mode=Replace, save=true) reported success with `AGIR_CACHED_POSE_FORWARD_REF` warnings (x2) claiming benign EarlyValidation fixup; `asset.save force=true` persisted it. On a real editor cold restart, `editor.open_asset` succeeded (`open_ok:true`) but `blueprint.compile` returned `compiled:false`, `status:"Error"` with `Use cached pose 'Locomotion'  does not have an associated Save Cached Pose node` (x2). Root-caused in source: `AGIRCompiler_CachedPose.cpp:177-186` (`CompileUseCachedPoseInstruction`) leaves `UseNode->SaveCachedPoseNode` null on a forward reference and only warns; the comment at 164-169 asserts EarlyValidation recovers it — the cold load refutes that for the Mannequin locomotion topology. Confirming symptom: `AGIRTextEmitter.cpp:677-681` derives the decompiled `use_cached_pose source=` label from `SaveCachedPoseNode->CacheName`, so the null link makes the variant's decompile emit an empty source (the `Locomotion` label dropped to empty). No AnimBlueprint save-time integrity gate blocks the dangling node. Distinct from `B-bp-saved-state-corruption-mcp-edits` (widget-tree / Blueprint-graph CreateDelegate corruption — different asset type, code path, and root cause) and from `B-agir-state-machine-output-pose-unbound` (a compile-time `AGIR_SYMBOL_NOT_FOUND` reject, not a save-time-passing / cold-load-failing corruption). Fix options: deferred two-pass `SaveCachedPoseNode` resolution in `anim.compile_agir` (per `F-agir-cliff-completion` #6), and/or an AnimBlueprint save-time integrity gate that rejects an unlinked `use_cached_pose` before persist.
