---
id: F-rpc-animation-list-curves
title: "Add live RPC `animation.authoring.list_curves` (Float + Transform, name + keyCount)"
status: DONE
severity: High
category: feature
tags: [animation, asset-dump, parity]
---

# Add live RPC `animation.authoring.list_curves` (Float + Transform, name + keyCount)

`AnimSequenceDumpBuilder::BuildCurvesArray` (`Source/EditorAutomationRpcGateway/Private/Handlers/Asset/AnimSequenceDumpBuilder.cpp:74-127`) emits a `curves[]` array into `anim_sequence.json` listing every Float and Transform curve with `{name, type, keyCount}`. There is no live RPC equivalent — none of the registered `animation.*` / `animation.authoring.*` methods returns the per-curve list. `animation.authoring.get_animation_info` (`AnimationAuthoringHandler.cpp:3320`) is the closest read endpoint and only reports `numNotifies`, duration, frame rate, and skeleton path; it never touches `Sequence->GetCurveData()`. Live consumers can only reach curve names by triggering a full `asset.dump` and reading the sidecar, which violates the policy that anything a dump emits must also be reachable via a live RPC.

`PropertyUtils::ExportRawCurveTracksSummary` (`PropertyUtils.cpp:213-255`, dispatched at `:587-590` for `FRawCurveTracks` struct properties) does parse the same data with richer per-curve detail (`curveTypeFlags`, `firstKey`/`lastKey`), but it is reachable only as a side-effect of generic property exports — it is not an animation-domain RPC and its `totalKeyCount` semantics differ from the dump builder's per-curve `keyCount` (transform curves are summed across XYZ × 3 channels in the dump builder vs. `Max3` of translation/rotation/scale key counts in the helper). Two independent code paths producing the same dump field is a drift hazard.

**Fix:** Extract a single `BuildCurvesArrayJson(const UAnimSequence*)` helper in `AnimSequenceDumpBuilder` that emits the canonical `[{name, type, keyCount}, ...]` shape. Call it from `BuildAnimSequenceJson` (replace the inline `BuildCurvesArray`) and from a new `REGISTER_RPC_HANDLER("animation.authoring.list_curves", "animation.authoring", ...)` that takes an `assetPath` to a `UAnimSequence` and returns `{ assetPath, curves: [...] }`. Update `docs/wiki/animation.authoring.md` to mention the new endpoint.

## History
- `#1-initial-repro` `OPEN` reporter — `AnimSequenceDumpBuilder::BuildCurvesArray` exposes Float + Transform curve names / keyCounts in `anim_sequence.json` but no live RPC returns the list. `animation.authoring.get_animation_info` reports duration / notifies / skeleton only and never reads `GetCurveData()`. `PropertyUtils::ExportRawCurveTracksSummary` parses the same data (richer: per-key first/last) but is only reachable through generic property exports and computes transform-curve key counts differently than the dump builder, so the two paths can drift. Proposal: extract `BuildCurvesArrayJson(UAnimSequence*)` and share it between the dump builder and a new `animation.list_curves` RPC.
- `#2-reformulated-namespace` `OPEN` reporter — The parity gap is real, but curve authoring already lives under `animation.authoring.set_curve_key`; reshape the endpoint target from `animation.list_curves` / `docs/wiki/animation.md` to `animation.authoring.list_curves` / `docs/wiki/animation.authoring.md`.
- `#3-added-authoring-list-curves` `IN-REVIEW` developer — Exported `AnimSequenceDumpBuilder::BuildCurvesArrayJson`, reused it in `BuildAnimSequenceJson`, added `animation.authoring.list_curves` with `assetPath` lookup and `SEQUENCE_NOT_FOUND` parity, documented it in `docs/wiki/animation.authoring.md`, and added regression coverage for registration plus dump-shaped Float / Transform curve output.
- `#4-verify-fix` `DONE` tester — Verified: wiki page for `animation.authoring.list_curves` returns schema with required `assetPath` (string); calling on `/Game/ScifiJungle/Demo/Characters/Mannequins/Animations/Shared/A_Steering_Straight` returned `{success:true, assetPath, curves:[{name:"pelvis", type:"Transform", keyCount:9}]}` — exact dump-builder shape with Transform type recognized.
