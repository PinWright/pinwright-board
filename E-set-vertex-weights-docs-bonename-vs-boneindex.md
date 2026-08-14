---
id: E-set-vertex-weights-docs-bonename-vs-boneindex
title: "skeleton.set_vertex_weights docstring says influences are (boneName, weight) but the accepted shape is {boneIndex, weight}"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [skeleton, skin-weights, set-vertex-weights, docs, param-shape-contradiction]
encounters: 1
lastSeen: 2026-07-02T13:14:11.6193939+03:00
---

# set_vertex_weights docs contradict themselves on the influence shape (boneName vs boneIndex)

The `skeleton.set_vertex_weights` documentation is internally inconsistent about how a
per-vertex influence is keyed:

- The method **summary** (handler registration docstring,
  `Source/PinWright/Private/Handlers/Animation/SkeletalMeshHandler.cpp:526`) says: *"Each
  vertex entry lists its **(boneName, weight)** influences."*
- The **`weights` param description** on the very same handler (line 530) says: *"Array of
  `{vertexIndex, influences:[{boneIndex, weight}]}`"* — i.e. **boneIndex**, not boneName.

The working call in the Attempt used `boneIndex` (e.g. v100 `{2:0.5, 3:0.5}` with
hip_L=2/knee_L=3), so the **param description is correct and the summary is wrong**. Because
these two lines flow into the generated method wiki page (`skeleton.set_vertex_weights.md`,
reached via `call("skeleton.set_vertex_weights")`), an agent reading the page sees the
summary promise bone *names* and the param require bone *indices*.

## Impact (the friction)

The contradiction on the exact focus method forced defensive work: the agent's narration
explicitly flagged it ("the parameter description says `influences:[{boneIndex, weight}]` but
the method summary says `(boneName, weight)`"), then hedged by calling `skeleton.list_bones`
to obtain **both** the names and the indices for all 61 bones before committing to the
`boneIndex` form. The `boneIndex` call worked first try, so this is discoverability friction,
not a hard blocker — but it is a real wobble on a skin-weight-authoring entry point, and the
extra `list_bones` on an 85540-vertex / 61-bone mesh spilled to an `HttpResponses` file.

## Fix

Make the summary and the param agree on the accepted influence key. Since the code accepts
`{boneIndex, weight}`, correct the **summary** to say `(boneIndex, weight)` (or explicitly
document that `boneName` is *also* accepted, if it is). Two touch points:

- Handler docstring: `Source/PinWright/Private/Handlers/Animation/SkeletalMeshHandler.cpp:526`
  — change "(boneName, weight)" to "(boneIndex, weight)".
- Wiki overlay `docs/wiki-src/skeleton.md`: there is no dedicated
  `### skeleton.set_vertex_weights` overlay section today (only cross-references to it from
  `describe_mesh` / `describe_skin_weights`). Add one that states the accepted influence
  shape `{vertexIndex, influences:[{boneIndex, weight}]}` unambiguously, and note
  `skeleton.list_bones` as the boneIndex source — so callers don't have to reverse-engineer
  the key from the contradictory summary.

severity rationale: impact=docs/discoverability wobble (works first try once the param is read; costs one extra list_bones + a hedge) × reach=rare (skin-weight authoring) -> Low

## History
- `#1-initial-audit` `OPEN` reporter — PROCESS/struggle audit of the `skeleton.set_vertex_weights` focus task (SK_DinoDragon). The method summary (`SkeletalMeshHandler.cpp:526`) says influences are "(boneName, weight)" while the `weights` param on the same handler (line 530) says "Array of {vertexIndex, influences:[{boneIndex, weight}]}"; the working call used boneIndex, so the summary is the wrong one. Source-confirmed both lines. The agent (SAY) flagged the contradiction and pre-fetched all 61 bones via `skeleton.list_bones` to hold both names and indices before committing to boneIndex (which worked first try). Fix: correct the handler summary to "(boneIndex, weight)" and add a `### skeleton.set_vertex_weights` section to `docs/wiki-src/skeleton.md` pinning the `{vertexIndex, influences:[{boneIndex, weight}]}` shape + naming `list_bones` as the index source. Low — a discoverability wobble on the focus method, not an execution failure.
- `#2-docs-follow-the-new-contract` `IN-REVIEW` developer — Resolved, and the resolution went the OTHER way from this ticket's proposed fix. The ticket proposed changing the summary to say (boneIndex, weight) to match the param. Instead the verb now accepts BOTH `boneName` and `boneIndex`, with `boneName` preferred and winning when both are given — see `B-set-vertex-weights-boneindex-unvalidated-index-space`, which had to specify the index space anyway. So the original summary's `boneName` was not wrong so much as unimplemented. Both the handler summary and the `weights` param description in `Source/PinWright/Private/Handlers/Animation/SkeletalMeshHandler.cpp` now state the accepted shape `{vertexIndex, influences:[{boneName | boneIndex, weight}]}`, that `boneIndex` is a REFERENCE-SKELETON index (never a section-local slot), and that `vertexIndex` is a flat LOD render-vertex index matching `describe_skin_weights` / `audit_skin_weights`. The requested `### skeleton.set_vertex_weights` section now exists in `Docs/wiki-src/skeleton.md` and names `skeleton.list_bones` as the `boneIndex` source, alongside a namespace-level `## Skin weights` section that defines both bone index spaces. **NOT COMPILED, NOT RUN.**
