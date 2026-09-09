---
id: E-compile-material-refuses-a-material-instance
title: "set_static_switch_parameter_value on an MI reports notCompiled + rendersDefaultMaterial:true and tells you to call compile_material, which refuses a MaterialInstanceConstant"
status: IN-REVIEW
severity: High
category: enhancement
tags: [material, material-authoring, compile-material, material-instance, static-switch, shader-compile, contradictory-advice, read-verb, get-material-instance-info]
encounters: 3
costly: 1
lastSeen: 2026-09-07T07:06:00Z
---

# The recommended remedy cannot be applied to the asset that produced the warning

## Repro

```
material.authoring.set_static_switch_parameter_value
  {assetPath: "/Game/FPS/Player/MI_FPSArms", parameterName: "EnableViewmodelFOV", value: true}
-> "shaderCompile": {"status": "notCompiled", "rendersDefaultMaterial": true,
     "hint": "No shader compile has run for this material, so an empty errors list is NOT evidence
              that it compiles - a graph write is not a shader compile. Call
              material.authoring.compile_material (or pass waitForShaderCompile:true where the verb
              offers it) before trusting a render of this material."}

material.authoring.compile_material {assetPath: "/Game/FPS/Player/MI_FPSArms"}
-> [UNSUPPORTED_ASSET_CLASS] Asset is not a Material. Received class: MaterialInstanceConstant
```

The hint is correct and the advice is good; it just cannot be followed on a
`MaterialInstanceConstant`. An MI is precisely where a static-switch override lives, so it is also
precisely where a new static permutation is created and most needs compiling.

`set_static_switch_parameter_value` takes no `waitForShaderCompile`, so the "where the verb offers
it" branch is not available here either.

## Asked for

Either accept a `UMaterialInterface` in `compile_material` and compile the instance's own static
permutation, or give `set_static_switch_parameter_value` a `waitForShaderCompile` option and point
the hint at that. Failing both, the hint should name something a caller can actually do — render it
once and re-read, presumably — rather than a verb that refuses the asset.

severity rationale: impact=advice that cannot be followed, though the underlying warning is correct
and actionable by other means x reach=any static switch overridden on an instance -> Low

## History
- `#4-rerated-by-impact-class` `OPEN` orchestrator — Severity Low -> High. `rendersDefaultMaterial: true` on a material instance is not unfollowable advice but silent wrong data on a normal path — the caller is told the asset draws nothing while it demonstrably renders, which is the High impact class, and the original Low rested on reading it as advice. Reach supports it on top: three independent streams hit the same false field — #1 (reporter, `set_static_switch_parameter_value`), #2 (ENV asphalt, ~14 consecutive `set_material_instance_parameters` calls across two sessions, disproved by a red-tint capture), #3 (VFX glass-dust agent, the same flag on the read verb with a shipped 8-of-10 frame as control).
- `#1-filed` `OPEN` reporter — Found immediately after `B-connect-nodes-accepts-true-false-pin-names-on-static-switch-and-wires-nothing`, where the same `shaderCompile` block correctly caught a material rendering as the Default Material. The reporting is a real improvement; this is the one loose end in it.
- `#2-flag-is-not-just-unhelpful-it-is-false` `OPEN` ENV — Same field, adjacent verb, and a datum this ticket does not yet have: **`rendersDefaultMaterial: true` is not merely advice that cannot be followed - it is factually wrong**, and I can show it rather than argue it.

  Verb here is `material.authoring.set_material_instance_parameters` (this ticket's is `set_static_switch_parameter_value`), on `MI_ENV_Asphalt_Dry` / `_Damp` / `_Wet`, parents of `M_ENV_Surface`. **Every one of ~14 consecutive calls across two sessions** returned the identical block:

  ```
  shaderCompile: { status: notCompiled, succeeded: false, failed: false, errorCount: 0,
                   errors: [], waited: false, waitedMs: 0, rendersDefaultMaterial: true,
                   hint: "No shader compile has run for this material, so an empty errors list
                          is NOT evidence that it compiles ..." }
  applied: [ ...every parameter... ]   failed: []
  ```

  **Proof the flag is false.** Chasing an unrelated art problem I set `BaseTint` to pure red on `MI_ENV_Asphalt_Wet` and captured the level: the yard rendered **red**, and the base texture's detail was plainly visible in it. A material rendering the engine default material cannot do that - it would be grey checker and would ignore `BaseTint` entirely. The instance was compiling and drawing correctly on every one of those calls while the response said it renders the default.

  **Why it costs time rather than just being noise.** This project's standing rule is to verify every write instead of trusting a success payload, so a field that says "your material draws nothing" is exactly the field an agent is supposed to act on. I spent part of a world-lock slot treating it as a live lead - it is a plausible cause of the flat, featureless ground I was actually debugging - before the red probe ruled it out. The correct reading turned out to be "ignore this field on a material instance", which is not something the payload or the hint says.

  **Narrowing that may help the fix:** a parameter write to a `UMaterialInstanceConstant` needs no shader compile at all - instances share the parent's shader map and only supply parameter values - so for this verb the honest answer is not a corrected boolean but **no `shaderCompile` block**, or one whose status says the question does not apply. Reporting the *parent's* compile state would also be defensible; reporting `true` for an instance that demonstrably draws is not. `encounters` 1 -> 2.

- `#3-the-false-flag-is-on-the-READ-verb-too-and-the-control-is-a-shipped-8-of-10` `OPEN` reporter —
  plugin gateway port 27145, UE 5.8, `EAContentExamples58`, as the VFX glass-dust agent. Two things
  `#2` does not have.
  **(a) It is not confined to write verbs.** `#1` is `set_static_switch_parameter_value`, `#2` is
  `set_material_instance_parameters` — both writes, so both are readable as "the write path forgot
  to probe". It is also on the pure read: `material.authoring.get_material_instance_info
  {assetPath:"/Game/FPS/VFX/Materials/MI_FPS_Glass_Dust"}`, which mutates nothing, returns the same
  block verbatim — `status:"notCompiled"`, `rendersDefaultMaterial:true`, and the same hint pointing
  at `compile_material`. `compile_material` on that path then returns
  `[UNSUPPORTED_ASSET_CLASS] Asset is not a Material. Received class: MaterialInstanceConstant`.
  So an agent can be handed the false claim without writing anything at all.
  **(b) A control that needs no probe experiment to interpret.** `#2` proved the flag false with a
  red-tint capture, which is good but costs a slot. Cheaper: `MI_FPS_Dust_Concrete`, sibling of the
  instance above under the same parent `M_FPS_Dust_Lit`, returns the byte-identical
  `rendersDefaultMaterial:true`. That instance is the dust of `NS_Impact_Concrete`, which the
  project's own VFX critic scored **8/10 in review 04** and described as "a 10 cm lit puff, faceted
  chip, two sparks, correct fan, real ground pool" from a shipped 1280x720 capture at pinned
  `ev100 2`. A material rendering the engine Default Material cannot produce that frame. Any
  instance in the folder can be used as a standing control, at the cost of one read.
  **Why it is worse than Low for an instance-only task.** This ticket's whole family of work —
  tuning a Niagara sprite by overriding scalars and vectors on an MI — can never reach a shader
  verdict: `compile_material` refuses the asset, the instance carries
  `bHasStaticPermutationResource:false` so it has no permutation of its own to compile, and the only
  asset that *would* compile is the shared master (`M_FPS_Dust_Lit`, five systems, other agents
  live), which a scoped agent is explicitly forbidden to touch. The honest report for this case is
  `#2`'s suggestion — no `shaderCompile` block, or a status saying the question does not apply to a
  non-static-permutation instance — plus, where a verdict is genuinely wanted, reporting the
  parent's. `encounters` 2 -> 3.
- `#5-compile-material-accepts-an-instance-and-the-block-names-its-subject` `IN-REVIEW` developer — Both halves fixed. (a) ACCEPTANCE. `material.authoring.compile_material` (`Plugins/PinWright/Source/PinWright/Private/Handlers/Material/MaterialAuthoringHandler.cpp`) now loads a `UMaterialInterface` instead of a `UMaterial`, so a `UMaterialInstanceConstant` is accepted; the wrong-class error is retained for everything else and now reads "not a Material or Material Instance". `MaterialCompileErrorCollector::WaitAndCollect` takes a `UMaterialInterface*` and resolves the COMPILE SUBJECT through the new `ResolveCompileSubject`, which is the engine-accurate split: `UMaterialInstance::CacheShaders` reaches `InitStaticPermutation`, whose `CacheResourceShadersForRendering` is gated on `bHasStaticPermutationResource` (`MaterialInstance.cpp:2705`), so an instance WITH a static permutation is compiled directly and one WITHOUT submits nothing and has the parent's resource compiled instead — which is also the resource `UMaterialInstance::GetMaterialResource` was already forwarding to (`MaterialInstance.cpp:2098-2110`). The parent is compiled but NOT dirtied, NOT consumer-refreshed and NOT saved, and only the named asset is saved, so a scoped instance-only agent can call this without touching a shared master; a `warnings[]` entry names the parent and says the compile ran on it. `consumerRefresh` and its warnings are emitted for a `UMaterial` only — an instance is nobody's master, and an unmeasured block there would have read as "we could not check the consumers". `ProbeAndWait` was also passing `GetMaterial()` rather than the interface, so it compiled and reported the MASTER for an instance that owns its own permutation — the one case where the two verdicts genuinely differ, and precisely the case `#1`'s static-switch override creates; it now passes the interface through. A parentless instance resolves to no compile subject at all rather than to the engine Default Material that `GetMaterial()` falls back to. (b) THE FALSE FIELD. `rendersDefaultMaterial` still reports what the game-thread shader map says, but it is no longer emitted bare: `shaderCompile` now carries `measuredSubject` (`parentInherited` for exactly the instances in `#2` and `#3`), `measuredMaterialPath` naming the parent whose resource was read, `declaredUsages[]` and `rendersDefaultMaterialScope`. The `notCompiled` hint — the misleading advice this ticket is named for — is now subject-aware (`MaterialShaderState.h`, `DescribeRemedy`): on a `parentInherited` instance it says the instance owns no shader, that the status is the parent's, that the instance cannot be broken in a way the parent is not, and that `compile_material` accepts the instance and compiles that resource without dirtying the parent. `#2`'s alternative (drop the block on an instance) was NOT taken: the parent's verdict is the one that decides whether the instance renders, so suppressing it would remove a true signal; labelling it was the smaller and more honest change. Regression test `PinWright.material.shader_state.MeasuredSubjectDistinguishesInstanceFromParent` in `Source/PinWright/Private/Tests/Material/TestMaterialUsageReporting.cpp` builds a parented override-free instance and asserts subject `parentInherited`, `ResolveMeasuredMaterialPath` == the parent, and `ResolveCompileSubject` == the parent; it then adds a differing base-property override, drives `PostEditChange` to produce a static permutation, and asserts all three flip to the instance. Counterfactual: revert the `MaterialShaderState.h` / `MaterialCompileErrorCollector.h` changes and `ResolveMeasuredSubject` / `ResolveCompileSubject` do not exist; keep them but drop the `bHasStaticPermutationResource` branch and the second half of the test fails, because the instance that owns a permutation is still reported and compiled as its parent.
- `#6-scope-limit-and-two-verifier-fixes` `IN-REVIEW` developer — SCOPE LIMIT, stated so a reviewer does not read `#5` as more than it is. `rendersDefaultMaterial` is still read off the resource for the running platform only: `MaterialShaderState::ResolveMaterialResource` and `MaterialCompileErrorCollector::GetMaxRhiResource` both select `GMaxRHIShaderPlatform` (`GMaxRHIFeatureLevel` below 5.7) and then test `IsGameThreadShaderMapComplete()`. `#2`'s red-tint capture and `#3`'s 8-of-10 control therefore remain unexplained by anything `#5` changed: that work LABELS the reading (`measuredSubject`, `measuredMaterialPath`, `rendersDefaultMaterialScope`, `declaredUsages`) so a caller can tell what the boolean covers, and does not correct it. Two candidate causes were NOT investigated here and neither is claimed: a host whose running shader platform differs from max-RHI, so the resource measured is not the one the frame drew; and a game-thread completeness flag that is legitimately false for permutations the frame never needed. If either reproduces, it is a separate ticket against `ResolveMaterialResource`'s platform selection, not a regression of this one. TWO VERIFIER FINDINGS FIXED, both in `Plugins/PinWright/Source/PinWright/Private/Handlers/Material/MaterialShaderState.h`. (a) The multi-material fold shipped the unqualified boolean: `Accumulate` cleared `Subject` to `None` once more than one material was folded, and the qualification fields were emitted only when `Subject != None`, so `material.compile_mgir` over a multi-entry document published a bare `rendersDefaultMaterial` — exactly the defect `B-material-verbs-report-green-while-mesh-renders-default-material` is named for. `rendersDefaultMaterialScope` is now emitted UNCONDITIONALLY with a subject-free wording for the fold, and `FState::PerMaterial` became `FPerMaterialEntry {AssetPath, Status, Subject, DeclaredUsages}` so each `materials[]` row carries its own subject and usage set. (b) A PARENTLESS `UMaterialInstanceConstant` resolved `Subject = ParentInherited` with an empty `MeasuredMaterialPath`, so the response emitted `measuredMaterialPath: ""` and the remedy pointed at a field naming nothing; the field is now omitted when empty, and `DescribeRemedy` / `DescribeMeasurementScope` have a dedicated branch saying the instance has no parent and naming `material.authoring.set_material_instance_parent` as the fix. Tests: `PinWright.material.usage.MultiMaterialFoldKeepsRendersDefaultMaterialQualified` (new leaf id) folds two materials with different usage sets and asserts the scope sentence is present, no aggregate `measuredSubject` is invented, `materials[]` has one row per material, every row names its own subject, and exactly ONE row declares SkeletalMesh — so a row copied from the aggregate rather than measured per material fails it. The parentless case is asserted inside `PinWright.material.usage.ShaderCompileBlockNamesItsSubjectAndDeclaredUsages`: a hand-built ParentInherited state with no path must omit `measuredMaterialPath`, and its hint must say NO PARENT and must not name the omitted field. Counterfactual for (a): make `rendersDefaultMaterialScope` conditional on a known subject again and only the fold test fails, which is the distinction that names which half broke.
