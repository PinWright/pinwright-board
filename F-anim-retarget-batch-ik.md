---
id: F-anim-retarget-batch-ik
title: "No verb runs an IK Retargeter batch export, so retargeting animation clips across skeletons needs python.execute"
status: OPEN
severity: High
category: feature
tags: [animation, retargeting, ik-retargeter, batch, gap-analysis-2026-09-30]
encounters: 1
lastSeen: 2026-09-30T12:00:00Z
---

# No IK Retargeter batch export verb

PinWright can author IK Rigs and IK Retargeters (`animation.authoring.create_ik_rig`, `add_ik_chain`,
`create_ik_retargeter`, `set_retarget_chain_mapping`) but nothing runs a retarget export.
`animation.setup_retargeting` is now an honest skeleton-swap copier (`retargeted:false`), and
`B-animation-setup-retargeting-does-not-retarget` `#2` deliberately added no mesh or retargeter
parameters. Moving clips from one character to another (e.g. Mannequin to a custom rig), one of the
most common animation tasks, is only reachable through `python.execute`.

Competitors (pinwright.com/compare, "Batch retarget animations"): ue-mcp `batch_retarget_animations`
(`RunBatchRetarget`, pre-flight `FIKRetargetProcessor::Initialize`, unmapped-chain report,
`requireCompleteMapping`, deletes created outputs on a partial result; UE 5.8 only,
`AnimationHandlers_StateMachine.cpp:2873-3088`); ChiR24 `setup_retargeting` (auto-builds rigs and
retargeter, `RunBatchRetarget`, 5.8 only); Monolith `batch_retarget_animations`
(`UIKRetargetBatchOperation::RunRetarget(Context)`, no version gate, seeds default ops when none).

## Engine API

`C:/UE_5.8/Engine/Plugins/Animation/IKRig/Source/IKRigEditor/Public/RetargetEditor/IKRetargetBatchOperation.h`:
- `RunBatchRetarget(const FIKRetargetBatchOperationInputs&)` `:194` — UE 5.8 only.
- `DuplicateAndRetarget(...)` `:206` — `UE_DEPRECATED(5.8)`, present 5.3-5.7 (checked in local engine trees).
- `RunRetarget(FIKRetargetBatchOperationContext&)` `:221` — present 5.3-5.8; shows an `FScopedSlowTask`.

Chain inspection: `UIKRetargeterController::GetSourceChain(tgt, op)` `IKRetargeterController.h:300`,
`AutoMapChains` `:286`, `AddDefaultOps` `:161`, `GetNumRetargetOps` `:183` (op-name params 5.6+ only;
5.3-5.5 chains live on the asset). `IKRigEditor` is already linked (`Source/PinWright/PinWright.Build.cs:186-187`).

## Proposed scope

New verb `animation.retarget_animations` (leave `setup_retargeting`'s contract as is; its wiki points here):
- Required: `retargeter`, `sourceMesh`, `targetMesh`, `assets[]`, `outputPath` (no safe default mesh exists).
- Optional: `prefix`, `suffix`, `overwrite` (default false), `requireCompleteMapping` (default true).
- Pre-checks: refuse `RETARGETER_NO_OPS` on 5.6+ when the op count is 0 (name `AddDefaultOps` as the
  remedy, or accept an explicit `seedDefaultOps:true`); refuse `RETARGET_CHAINS_UNMAPPED` listing the
  unmapped target chains; refuse source-mesh / asset skeleton incompatibility.
- Run `DuplicateAndRetarget` below 5.8 and `RunBatchRetarget` on 5.8, guarded, with rows in
  `docs/engine-version-support.md`.
- Verify by reading each output back: Skeleton equals the target mesh's skeleton, bone-track count > 0.
  Report `created[] {path, boneTracks, frames}`, `mappingComplete`, `unmappedTargetChains[]`, save state
  via `AddAssetSaveReport`.
- On a partial result delete every output this call created, and say so.

**Risk:** the slow-task UI under `-unattended` is untested; a retargeter created by
`create_ik_retargeter` on 5.6+ probably has zero ops (inferred from the `set_retarget_chain_mapping`
wiki note, not tested).

## Acceptance

- Mannequin to a test character retargets 3 clips; outputs carry the target skeleton and have bone tracks > 0.
- Unmapped chains are refused with a list naming them.
- A retargeter with 0 ops is refused on 5.6+.
- A forced mid-batch failure leaves no outputs behind.
- Passes on 5.3 and 5.8 in the version matrix.

**Effort:** M.

**Related:** `B-animation-setup-retargeting-does-not-retarget`, `F-ik-rig-retargeter-family-not-compiled`.

## History
- `#1-no-batch-retarget` `OPEN` reporter — Filed from the 2026-09-30 animation gap analysis against competitor MCP servers. Source-confirmed at plugin HEAD `2580e7f4`: no handler calls `RunBatchRetarget`, `DuplicateAndRetarget` or `RunRetarget`. Severity High: hard blocker on a common task with only `python.execute` as a workaround.
