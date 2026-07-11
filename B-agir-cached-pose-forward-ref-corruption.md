---
id: B-agir-cached-pose-forward-ref-corruption
title: "anim.compile_agir leaves use_cached_pose SaveCachedPoseNode unresolved on a forward reference (use-before-save): a lossy AGIR round-trip + violated compile-time linkage invariant — NOT cold-load asset corruption (engine EarlyValidation self-heals the persisted asset)"
status: IN-REVIEW
severity: Medium
category: bug
tags: [agir, animgraph, cached-pose, forward-ref, compile-agir, roundtrip]
encounters: 1
lastSeen: 2026-07-11T04:52:35.3332310+03:00
claimedBy: fuzz2
claimedAt: 2026-07-11T05:22:26.5096351+03:00
---

# `anim.compile_agir` leaves a `use_cached_pose` unresolved on a cached-pose forward reference

Compiling AGIR text in which the AnimGraph `output` consumes a `use_cached_pose`
**before** the matching `save_cached_pose` appears in the text (a forward
reference — the normal emission order for a locomotion AnimBP, whose AnimGraph
output line precedes the cache-body block) leaves the `use_cached_pose` node's
`SaveCachedPoseNode` weak pointer **null at AGIR compile time**, emitting only a
non-fatal `AGIR_CACHED_POSE_FORWARD_REF` warning.

Two observable consequences, both proven by a red test on current source:

1. **Violated compile-time linkage invariant.** The plugin's own contract — the
   one the sibling `PinWright.AGIR.CachedPose.RoundTrip` test asserts for the
   backward-reference (save-before-use) ordering — is that `SaveCachedPoseNode`
   resolves *at AGIR compile time*, before any AnimBP compile. On the
   forward-reference ordering that resolution silently does not happen, so the
   compiler's behavior is **order-dependent**.
2. **Lossy warm AGIR round-trip.** `AGIRTextEmitter` derives the decompiled
   `use_cached_pose source=` label from `SaveCachedPoseNode->CacheName`
   (`AGIRTextEmitter.cpp:677-681`), so a re-decompile immediately after a
   forward-ref compile drops the cache name (`Locomotion`) to an **empty**
   label. The AGIR round-trip (`decompile_agir` -> `compile_agir` ->
   `decompile_agir`) — a core PinWright capability — is not faithful for the
   canonical cached-pose-fronted (locomotion) shape.

## What this is NOT (corrected from the original report)

The original report (`#1-initial-repro`) framed this as **Critical save-time
asset corruption**: that `anim.compile_agir` reports success + `asset.save`
persists an AnimBlueprint that then FAILS `blueprint.compile` on a cold editor
restart, and that the code comment "engine EarlyValidation will fix up" is a
**false premise**. Engine source refutes that framing:

- `Engine/Source/Editor/AnimGraph/Private/AnimGraphNode_UseCachedPose.cpp::EarlyValidation`
  (lines 28-76) re-resolves `SaveCachedPoseNode` from the node's serialized
  `NameOfCache` on **every** compile: when the weak ptr is null/unlinked and
  `!NameOfCache.IsEmpty()`, it walks `GraphBlueprint->GetAllGraphs()` and sets
  `SaveCachedPoseNode` to the `UAnimGraphNode_SaveCachedPose` whose
  `CacheName == NameOfCache`. `OnProcessDuringCompilation` then links via that
  re-resolved node.
- Both inputs to that recovery are serialized and PinWright writes both:
  `NameOfCache` is a serialized `UPROPERTY()` (`AnimGraphNode_UseCachedPose.h:47-48`)
  written reflectively at `AGIRCompiler_CachedPose.cpp:170` on every use node;
  the save side's `CacheName` is a serialized `UPROPERTY(EditAnywhere)` set at
  `AGIRCompiler_CachedPose.cpp:86`.
- So the persisted asset **self-heals** on the next AnimBP compile (including the
  cold-load compile). The comment at `AGIRCompiler_CachedPose.cpp:164-169` is
  substantially **correct**, not a false premise.
- The reporter's own cold-load error string — `Use cached pose 'Locomotion'
  does not have an associated Save Cached Pose node` — actually **proves**
  `NameOfCache` persisted non-empty (the `@@`/title renders from `NameOfCache`
  via `GetNodeTitle`), which means EarlyValidation's refresh loop **did** run. A
  genuine cold-load compile failure could then only occur if **no**
  `save_cached_pose` with a matching `CacheName` was persisted at all — a
  save-side / decompile-transfer defect (a missing or mis-named save node, cf.
  `B-agir-state-machine-output-pose-unbound`), which is a **different** defect
  from the use-node null weak ptr this ticket is about, is **not** reproduced by
  the red test, and would have to be re-filed with that real root cause if ever
  reproduced on a truly fresh editor.

Net: no persistent asset corruption is substantiated. The real, reproducible
defect is the compile-time-invariant + warm-round-trip-fidelity gap above.
Severity downgraded Critical -> Medium accordingly (real, broad-reach
round-trip fidelity defect in a core capability, but self-healing and non-
corrupting).

## Root cause (source-confirmed)

`Plugins/PinWright/Source/PinWright/Private/AGIR/AGIRCompiler_CachedPose.cpp`,
`CompileUseCachedPoseInstruction`. The block-scoped cache-name table is consumed
in the **same single pass** that fills it (`AGIRCompiler.cpp:981-1004`
`CompileBlockIntoGraph`: "Pass-1 save handlers populate it, Pass-1 use handlers
consume it"), so a `use_cached_pose` emitted before its `save_cached_pose`
resolves against an incomplete map and takes the else branch that only warns
and leaves `SaveCachedPoseNode` null (the link is set only in the found branch,
line 179). Pose-pin wires already defer to a post-loop Pass-2
(`PendingWires`/`ResolvePendingPoseWires`, `AGIRCompiler.cpp:1006`), but the
cache-name->SaveCachedPoseNode linkage is not deferred.

## What it should do — Fix (adopted)

**Deferred (two-pass) cache-linkage resolution**, mirroring the in-file
`PendingWires`/`ResolvePendingPoseWires` pattern and the fix sketch recorded on
`F-agir-cliff-completion` #6 (Pass 1 creates Save nodes; Pass 2 resolves Use
nodes): queue each `use_cached_pose`'s cache-name resolution during the
instruction loop and resolve it against the **completed** `CacheNameMap` after
every instruction in the block has compiled. This makes `SaveCachedPoseNode`
resolution order-independent and restores a faithful warm round-trip. A genuinely
dangling use (no `save_cached_pose` with that name anywhere in the block) stays a
non-fatal warning (the compile still succeeds; the engine self-heals cross-graph
or the AnimBP compile legitimately reports the missing save).

Invariant to restore: `use_cached_pose` resolves `SaveCachedPoseNode` at AGIR
compile time regardless of whether its `save_cached_pose` was emitted before or
after it — the same invariant the backward-ref `RoundTrip` test already asserts.

### Rejected fix — the save-time integrity gate (original option #2)

The original "AnimBlueprint save-time integrity gate that rejects any
`use_cached_pose` with a null `SaveCachedPoseNode` before persist" is **not**
adopted and should not ship: it would **false-positive on every AnimBP the
engine's EarlyValidation self-heals** (the null weak ptr is engine-recoverable
whenever `NameOfCache` + a matching save node persist, which is the normal
case), repeating the over-broad pre-save heuristic removed in
`B-integrity-gate-false-positive-asyncaction-proxies` (#2-removed-orphan-heuristic)
and violating agent-conventions.md:149 ("Prefer source-level fixes + narrow
invariant checks over post-hoc heuristic validation"). The source-level Pass-2
resolution above is the correct fix.

## History
- `#1-initial-repro` `OPEN` reporter — Cold-load-confirmed corruption from a REALISM AGIR-transfer task (capture `ABP_Manny` AGIR, rebuild into a throwaway `ABP_Manny_Variant`). `anim.compile_agir` (mode=Replace, save=true) reported success with `AGIR_CACHED_POSE_FORWARD_REF` warnings (x2) claiming benign EarlyValidation fixup; `asset.save force=true` persisted it. On a real editor cold restart, `editor.open_asset` succeeded (`open_ok:true`) but `blueprint.compile` returned `compiled:false`, `status:"Error"` with `Use cached pose 'Locomotion'  does not have an associated Save Cached Pose node` (x2). Root-caused in source: `AGIRCompiler_CachedPose.cpp:177-186` (`CompileUseCachedPoseInstruction`) leaves `UseNode->SaveCachedPoseNode` null on a forward reference and only warns; the comment at 164-169 asserts EarlyValidation recovers it — the cold load refutes that for the Mannequin locomotion topology. Confirming symptom: `AGIRTextEmitter.cpp:677-681` derives the decompiled `use_cached_pose source=` label from `SaveCachedPoseNode->CacheName`, so the null link makes the variant's decompile emit an empty source (the `Locomotion` label dropped to empty). No AnimBlueprint save-time integrity gate blocks the dangling node. Distinct from `B-bp-saved-state-corruption-mcp-edits` (widget-tree / Blueprint-graph CreateDelegate corruption — different asset type, code path, and root cause) and from `B-agir-state-machine-output-pose-unbound` (a compile-time `AGIR_SYMBOL_NOT_FOUND` reject, not a save-time-passing / cold-load-failing corruption). Fix options: deferred two-pass `SaveCachedPoseNode` resolution in `anim.compile_agir` (per `F-agir-cliff-completion` #6), and/or an AnimBlueprint save-time integrity gate that rejects an unlinked `use_cached_pose` before persist.
- `#2-reword` `IN-REVIEW` developer — REWORD (severity Critical -> Medium, root cause corrected, fix scope narrowed to option #1). Independently verified against plugin + engine source: the forward-ref null weak ptr and the empty warm re-decompile label are real (red test `PinWright.AGIR.CachedPose.ForwardRefResolvesLinkage` reproduces both on current source), but the original Critical "save-time asset corruption / uncompilable AnimBlueprint / EarlyValidation false premise" framing is **contradicted by engine source**: `AnimGraphNode_UseCachedPose::EarlyValidation` (UE 5.7 `AnimGraphNode_UseCachedPose.cpp:28-76`) re-resolves `SaveCachedPoseNode` from the serialized `NameOfCache` (PinWright writes it at `AGIRCompiler_CachedPose.cpp:170`) against a matching serialized save `CacheName` (`:86`) on every AnimBP compile, so the persisted asset self-heals on cold load. Real defect reframed to: forward-ref leaves the cache linkage unresolved at AGIR compile time -> violates the plugin's own compile-time invariant (asserted for backward refs by `PinWright.AGIR.CachedPose.RoundTrip`) + a lossy warm AGIR round-trip. Adopted fix: deferred/two-pass `SaveCachedPoseNode` resolution mirroring the in-file `PendingWires`/`ResolvePendingPoseWires` pattern (per `F-agir-cliff-completion` #6). Dropped the original save-time integrity-gate option as WONTFIX — it would false-positive on every engine-self-healed AnimBP, repeating the removed anti-pattern from `B-integrity-gate-false-positive-asyncaction-proxies` (agent-conventions.md:149). Implementing per adopted fix; will flip green the reproduced red test.
</content>
</invoke>
