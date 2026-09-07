---
id: B-set-texture-sample-texture-resolves-one-of-many-same-name-params
title: "material.authoring.set_texture_sample_texture resolves a PARAMETER NAME to a single node and silently repoints only that one, leaving every sibling node with the same parameter name on the old texture"
status: OPEN
severity: High
category: bug
tags: [material, material-authoring, set-texture-sample-texture, texture-sample, parameter-name, silent-partial-success, triplanar, shared-editor]
encounters: 1
costly: 0
lastSeen: 2026-09-07T13:03:30Z
---

# `set_texture_sample_texture` repoints ONE node when a parameter name is shared, and reports plain success

**What I called.** `material.authoring.set_texture_sample_texture {assetPath: "/Game/FPS/Weapons/Materials/M_WPN_Master", nodeId: "AOAtlas", texturePath: ".../T_WPN_AOAtlas_M2"}` — `nodeId` given as a **parameter name**, which the wiki page explicitly permits ("`nodeId` (GUID, node name, or parameter name)").

**What happened.** Success, with `nodeId: "7AE613CD41D78E1ADA55CF8E6C3A7C0D"` and the new `texturePath` echoed back. No warning, no count, no hint that the name was ambiguous.

**What is actually in the graph.** `M_WPN_Master` has **two** `MaterialExpressionTextureSampleParameter2D` nodes that share the parameter name `AOAtlas` — `..._21` (`7AE613CD…`, the Y-band sample) and `..._22` (`9C09CA22…`, the Z-band sample). This is legal and deliberate: they are two samples of one atlas on different projection planes, and as a *parameter* they share one value at the material-instance level. Only `_21` was repointed. `_22` was left pointing at the old texture.

**Why that is worse than it sounds.** The material still compiled clean (`compileStatus: completed`, `compileSucceeded: true`, `compiledWithErrors: false`, `rendersDefaultMaterial: false`) and saved, because a graph with two different textures on two same-named parameter nodes is not a compile error. So every in-band signal said the repoint had landed. The two textures here were an old, effectively-empty ambient-occlusion atlas and its corrected replacement, so the shipped result would have been **half the AO chain sampling the correct map and half sampling the stale one**, with no error anywhere and a visible-only-at-certain-normals artefact. That is the "silent success with partial effect" class this board treats as the most important kind.

**How I caught it, and the only way I could have.** A byte scan of the saved `.uasset` counting the two texture names — old-exclusive count was still **2** after the call reported success. Then `get_material_node_details {nodeId: "AOAtlas"}` confirmed the same thing from the other side: it also resolves the name to `_21` only, so the read verb cannot reveal the sibling either. Nothing short of scanning the file or already knowing the second node existed would have shown it.

**Workaround.** Address the sibling by its **full node name**, not the parameter name: `nodeId: "MaterialExpressionTextureSampleParameter2D_22"` worked and returned `9C09CA22…`. Note the SHORT form `"TextureSampleParameter2D_22"` returns `NOT_FOUND`, so the caller must know the `MaterialExpression` prefix; the wiki's "node name" wording does not say which spelling is meant. After repointing both nodes the old-exclusive byte count went to **0**.

**What I expected.** Either (a) repoint every node sharing that parameter name and report how many were changed, or (b) refuse with an `AMBIGUOUS_PARAMETER_NAME` error listing the candidate node ids — the shape `actor.get`/`actor.set_transform` already use for ambiguous actor labels. Silently picking one is the one option that cannot be detected from the response. At minimum the response should carry a `matchedNodes`/`nodesChanged` count so `1` versus `2` is visible.

**Reach.** Multiple sample nodes sharing one texture parameter name is the normal shape for any triplanar or multi-band sampling material, which is exactly the kind of material this verb exists to edit — so this is not an exotic graph. Severity High rather than Medium because the failure is silent, survives a clean compile, and the caller's natural verification (the success payload, a compile, and even the matching read verb) all agree with the wrong answer.

**Not costly for me** (`costly: 0`): I found it in about four extra calls because disk verification is mandatory in this project, and no false conclusion reached a report. A caller who trusts the response instead of the file would pay for it, and that occurrence should be logged costly.

Related: `E-material-set-texture-on-existing-sample` is the ticket that requested this verb; this is a defect in the delivered implementation, not that ergonomic gap, so it is filed separately rather than appended there.

## History
- `#1-filed` `OPEN` reporter — Hit on `M_WPN_Master` while repointing a corrected AO atlas. Parameter name `AOAtlas` is shared by two `TextureSampleParameter2D` nodes; the verb repointed one, reported success with that node's id, and the material compiled and saved clean with the sibling still on the old texture. Caught only by a byte scan of the saved `.uasset` (old-exclusive name count still 2). Worked around by addressing the sibling as `MaterialExpressionTextureSampleParameter2D_22`; the short `TextureSampleParameter2D_22` spelling returns `NOT_FOUND`. Asked for: repoint-all with a count, or an ambiguity refusal listing candidates.
