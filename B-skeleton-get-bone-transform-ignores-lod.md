---
id: B-skeleton-get-bone-transform-ignores-lod
title: "skeleton.get_bone_transform accepts lodIndex but never reads or validates it"
status: OPEN
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
