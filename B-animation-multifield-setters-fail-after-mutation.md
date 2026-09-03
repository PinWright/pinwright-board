---
id: B-animation-multifield-setters-fail-after-mutation
title: "Two animation multi-field setters return validation errors after already changing earlier fields"
status: OPEN
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

## History

- `#1-pattern-scan` `OPEN` reporter — Source-confirmed both late-error paths retain earlier writes with no rollback; no editor, build, or test was run.
