---
id: B-animation-multifield-setters-fail-after-mutation
title: "Two animation multi-field setters return validation errors after already changing earlier fields"
status: IN-REVIEW
severity: High
category: bug
tags: [animation, validation, rollback, atomicity, partial-mutation, state-machine]
---

# Validation is interleaved with writes in two Anim Blueprint setters

`animation.authoring.set_transition_settings` assigns `TransNode->LogicType` immediately after parsing it (`Plugins/PinWright/Source/PinWright/Private/Handlers/Animation/AnimationAuthoringHandler_AnimBlueprint.cpp:1162-1171`), then can return `INVALID_BLEND_MODE` (`:1173-1180`) or `BLEND_CURVE_NOT_FOUND` (`:1184-1191`). The earlier logic-type change remains in memory even though the RPC reports an error.

`animation.authoring.set_layered_blend_layers` similarly writes `Node->Node.BlendMode` (`:1777-1794`) before loading `blendMaskPath`; a missing mask returns `BONE_MASK_NOT_FOUND` (`:1796-1804`) with the mode already changed. It can then fail layer validation at `WriteLayeredBlendLayers` (`:1807-1813`) after that same early write. Neither handler owns a transaction or rollback.

Concrete calls are a valid `logicType` plus invalid `blendMode`, or `blendMode="BlendMask"` plus a nonexistent mask path. Both errors look like refusal, but a later unrelated save can persist the partial field change.

Parse and resolve every optional value into local candidates before writing the first field. Then apply the complete set inside the established scoped transaction/snapshot rollback shape; if any engine-side write fails, restore the node and cancel the transaction. Tests must compare all fields before and after each late validation failure.

**Workaround:** validate all enum tokens and asset paths separately before calling; reload the package after any setter error.

## Related

- Catalog: `partial-mutation-without-complete-rollback`, `partial-nonatomic-success`
- `E-set-transition-settings-no-echo` — response ergonomics, not failure atomicity

## Fix

The report was TRUE: both handlers wrote their first enum field before resolving later values that could return an error. `set_transition_settings` now parses logic type and blend mode, resolves the curve, and reads the disabled candidate before writing any transition field. `set_layered_blend_layers` now resolves the requested mode and blend-mask asset, validates the complete layer array through a shared non-mutating helper, and preflights every supplied reflected option (`meshSpaceRotationBlend`, `meshSpaceScaleBlend`, and `curveBlendOption`) with the same import semantics against scratch storage before changing mode or layers. Invalid reflected values return `PROPERTY_SET_FAILED` with no partial mutation. The validated scalar writes are guarded with restoration, and the mutation-only `ApplyValidatedLayeredBlendLayers` path removes the post-mode handled-error branch; the defensive public writer still validates for its other callers. No transaction or snapshot is needed because all known validation failures now precede the first live mutation.

Files changed:

- `Source/PinWright/Private/Handlers/Animation/AnimationAuthoringHandler_AnimBlueprint.cpp`
- `Source/PinWright/Private/Handlers/Animation/AnimGraphConstructionUtils.cpp`
- `Source/PinWright/Private/Handlers/Animation/AnimGraphConstructionUtils.h`
- `Source/PinWright/Private/Tests/Assets/TestAnimationSetterAtomicity.cpp`
- `Docs/wiki-src/animation.authoring.md`

Regression tests: `PinWright.animation.authoring.SetTransitionSettingsAtomicValidation` covers invalid blend mode and missing curve failures; `PinWright.animation.authoring.SetLayeredBlendLayersAtomicValidation` covers missing blend-mask, malformed-layer, and invalid `curveBlendOption` failures. Each compares the complete reflected node-property snapshot before and after the refused call and asserts the required `LogicType`/`BlendMode`/nested `Node` keys are present before equality. `Docs/wiki-src/animation.authoring.md` documents the new `PROPERTY_SET_FAILED` rejection for invalid reflected values. No Unreal editor, build, or automation run was performed under the worker brief.

## History

- `#1-pattern-scan` `OPEN` reporter — Source-confirmed both late-error paths retain earlier writes with no rollback; no editor, build, or test was run.
- `#2-animation-setter-preflight` `IN-REVIEW` developer — Preflighted all transition candidates and layered mode/mask/layer inputs before mutation, extracted shared layer validation, and added late-failure byte-snapshot regressions. Static diff checks only; Unreal/build/automation execution is deferred to aggregate verification.
- `#3-curve-field-preflight` `IN-REVIEW` developer — Preflighted every supplied reflected option through the exact import path on scratch storage, restored scalar values on an unexpected write error, moved those writes before mode/layer mutation, removed the setter's post-preflight handled-error branch, and added invalid-curve regression coverage plus non-vacuous snapshot-key assertions. Static checks only; Unreal/build/automation execution remains deferred.
