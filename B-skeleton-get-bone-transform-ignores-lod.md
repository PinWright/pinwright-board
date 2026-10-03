---
id: B-skeleton-get-bone-transform-ignores-lod
title: "skeleton.get_bone_transform accepts lodIndex but never reads or validates it"
status: DONE
severity: Medium
category: bug
tags: [skeleton, animation, lod, ignored-parameter, false-success, readback]
---

# `lodIndex` has no effect on `skeleton.get_bone_transform`

The registered contract advertises optional `lodIndex` (`Plugins/PinWright/Source/PinWright/Private/Handlers/Animation/SkeletonHandler.cpp:350-357`). The handler reads only the mesh/skeleton paths and `boneName` (`:359-361`), resolves the asset-wide `FReferenceSkeleton` (`:369-399`), and returns `GetRefBonePose()[BoneIndex]` (`:401-441`). There is no read or validation of `lodIndex` anywhere in the body.

Thus `lodIndex=999` succeeds exactly like `lodIndex=0`, and a bone excluded from a mesh LOD is still reported from the full reference skeleton. A caller can treat the successful response as LOD-scoped evidence even though the result is LOD-independent.

Either remove `lodIndex` from the public contract and state that this is an asset-wide reference-pose read, or implement real mesh-LOD validation/inclusion semantics and echo the applied LOD. Do not accept an unsupported selector. Add a differential test where two LOD selections must either produce deliberately distinct scope or one is rejected.

**Workaround:** omit `lodIndex` and interpret the result only as the skeleton's global local-space bind pose.

## Related

- Catalog: `accepted-parameter-silently-dropped`, `accepted-parameter-silent-noop`, `wrong-target-scope-or-identity`

## History

- `#1-pattern-scan` `OPEN` reporter — Source-confirmed the declared parameter has no body read and the response comes from the global reference skeleton; no editor, build, or test was run.
- `#2-lodindex-removed` `IN-REVIEW` developer — Took the ticket's first option: `lodIndex` removed from `skeleton.get_bone_transform`'s `RPC_PARAMS` (`Source/PinWright/Private/Handlers/Animation/SkeletonHandler.cpp`) and the summary now says the read is asset-wide and LOD-independent. The verb reads the reference skeleton's bind pose, which no mesh LOD changes, so a LOD selector had nothing honest to select; the dispatcher's unknown-param gate now refuses `lodIndex` (any value) with `UNKNOWN_PARAMS` instead of succeeding. Behaviour change noted in CHANGELOG. Test: `PinWright.skeleton.get_bone_transform.RefusesLodIndex` (`Tests/Gameplay/TestAnimationHandlers.cpp`) asserts via `ParamSpecTestHelpers::IsParamAccepted` that `boneName` is accepted and `lodIndex` / `lod_index` are not (fails if the declaration is restored). Wiki: new `### skeleton.get_bone_transform` section in `docs/wiki-src/skeleton.md`. Per-LOD bone inclusion ("does LOD n keep this bone") is a different question this verb never answered; not added.
- `#3-review-nit-wire-refusal` `IN-REVIEW` developer — Re-review NIT: `PinWright.skeleton.get_bone_transform.RefusesLodIndex` now also dispatches `{skeletonPath, boneName, lodIndex: 0}` through `DispatcherTestHelpers::Dispatch` and asserts the wire refusal `UNKNOWN_PARAMS` that the CHANGELOG and wiki promise. The expected dispatcher warning is declared. If the declaration is restored, the call reaches the handler, which returns a different error, and the test fails.
- `#4-verified-linux` `DONE` tester — Passed non-skipped in run3/full: `PinWright.skeleton.get_bone_transform.RefusesLodIndex`. Acceptance (the ticket's first option: remove `lodIndex` and state the read is asset-wide) met. The test asserts that `lodIndex` / `lod_index` are no longer accepted params and that a real dispatch with `lodIndex: 0` is refused `UNKNOWN_PARAMS` on the wire, so an unsupported selector can no longer succeed. Doc verified: `docs/wiki-src/skeleton.md` has the asset-wide, LOD-independent note. Per-LOD bone inclusion was not added; the ticket did not require it under this option.
