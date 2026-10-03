---
id: B-animation-enum-tokens-fall-through-to-default
title: "Two animation creators treat unknown enum-like tokens as defaults and report successful creation of a different type"
status: DONE
severity: Medium
category: bug
tags: [animation, validation, enum, fallback, false-success, wrong-result]
---

# Unknown `assetType` and `blendType` values create defaults

Two creators in the animation handler family use an open-ended final branch for both omission and invalid input:

- `animation.create_animation_asset` documents `sequence` or `montage` (`Plugins/PinWright/Source/PinWright/Private/Handlers/Animation/AnimationHandler.cpp:842-850`), but every non-`montage` string enters the sequence factory branch (`:897-921`). The later “Unsupported assetType” error only covers factory allocation failure (`:924-927`).
- `animation.authoring.add_blend_node` documents `TwoWayBlend` or `LayeredBlend` (`Handlers/Animation/AnimationAuthoringHandler_AnimBlueprint.cpp:1345-1353`), but every unmatched value reaches the “Default fallback to TwoWayBlend” branch (`:1382-1419`), is marked dirty by default, and returns success naming the different created type (`:1422-1431`).

A typo such as `assetType="montages"` creates an AnimSequence; `blendType="Layered"` creates a TwoWayBlend. The response exposes the effective type only after mutation, so the caller cannot rely on validation to stop a wrong asset/node.

Normalize and validate the closed token sets before loading or mutating. Apply the default only when the field is absent/empty; reject a present unknown token with a typed error listing accepted values. Keep deliberate aliases explicit and add differential invalid-token tests.

**Workaround:** pass the exact documented spellings and compare the returned effective type before continuing.

## Related

- Catalog: `accepted-parameter-silently-dropped`, `accepted-parameter-silent-noop`, `coercion-slot-unit-direction-drift`

## History

- `#1-pattern-scan` `OPEN` reporter — Source-confirmed both unknown-token fallthrough branches and successful mismatched creation; no editor, build, or test was run.

- `#2-closed-token-sets` `IN-REVIEW` developer — Still reproducible before the fix (both open `else` branches present). `Handlers/Animation/AnimationHandler.cpp` `animation.create_animation_asset`: `assetType` validated right after `name`, before the skeleton load, `MakeDirectory` or factory; present unknown value → `INVALID_ASSET_TYPE` listing `sequence, montage` (case-insensitive; empty/absent = sequence). `Handlers/Animation/AnimationAuthoringHandler_AnimBlueprint.cpp` `animation.authoring.add_blend_node`: `blendType` validated before the blueprint load → `UNKNOWN_NODE_TYPE` listing `TwoWayBlend (alias Blend), LayeredBlend (alias LayeredBoneBlend)`; the TwoWayBlend fallback branch is gone (a build without a requested node header now returns `ANIMGRAPH_MODULE_UNAVAILABLE` instead of creating the other type). Param descriptions updated. Tests (new `Tests/Assets/TestAnimationClosedTokenInputs.cpp`): `PinWright.animation.CreateAnimationAssetAssetTypeClosedSet` (refusals assert code + no package created; `MONTAGE` still creates an AnimMontage), `PinWright.animation.authoring.AddBlendNodeBlendTypeClosedSet` (refusals assert code + unchanged node counts; `layeredblend` and omission still create). Docs: `docs/wiki-src/animation.md`, `docs/wiki-src/animation.authoring.md`, `CHANGELOG.md`. Fastcheck OK; not run in an editor yet.
- `#3-verified-linux` `DONE` tester — Passed non-skipped in run3/full: `PinWright.animation.CreateAnimationAssetAssetTypeClosedSet`, `PinWright.animation.authoring.AddBlendNodeBlendTypeClosedSet`. Acceptance met: a present unknown `assetType` (e.g. `montages`) is refused with `INVALID_ASSET_TYPE` and no package is created, and `MONTAGE` still creates an AnimMontage; a present unknown `blendType` (e.g. `Layered`) is refused with `UNKNOWN_NODE_TYPE` with node counts unchanged, while `layeredblend` and omission still create. The default applies only when the field is absent, and the error lists the accepted values and aliases.
