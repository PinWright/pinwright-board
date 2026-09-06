---
id: B-static-switch-input-connection-lost-across-save-and-reload
title: "A StaticSwitchParameter input bound AFTER construction is lost across save and reload - it binds, reads back, compiles, renders and saves written, then comes back NULL; the same input set during construction by compile_mgir survives"
status: OPEN
severity: High
category: bug
tags: [material, static-switch, serialisation, save, reload, default-material, silent-revert, connect-nodes, compile-mgir, post-hoc-connection]
encounters: 2
lastSeen: 2026-09-06T06:40:00Z
---

# The connection is live, rendered and saved — and gone after a restart

## What happened, in order, all measured

On `/Game/FPS/Player/M_FPSArms`, a `MaterialExpressionStaticSwitchParameter` named
`EnableViewmodelFOV` gating a `MaterialFunctionCall` into World Position Offset.

1. Bound input 0 through the engine API — `MaterialEditingLibrary.connect_material_expressions(call,
   '', switch, 'True')` returned **True**.
2. Read it back: `get_material_node_details` reported
   `A: {Expression: MaterialExpressionMaterialFunctionCall_0, OutputIndex: 0}`. **Connected.**
3. `compile_material`: `shaderCompile.succeeded: true`, **`rendersDefaultMaterial: false`**.
4. `asset.generate_thumbnail` on the instance: `usingDefaultMaterial: false`,
   `meanLuminance 0.132` — dark, correct.
5. **Rendered correctly in a PIE frame** (`Docs/fps/evidence/player/b09-01_idle.png`): the arms read
   as dark fabric where every previous build had shown pale grey.
6. `asset.save {force: true}` -> `saveState: "written"`, `sizeBytes: 18041`.

Then the editor restarted (twice, for unrelated crashes). On the next session, with the `.uasset`
unchanged on disk at 18 041 bytes:

```
get_material_info M_FPSArms
-> shaderCompile: {status: "failed", errors: ["(Node StaticSwitchParameter) Missing A input"],
                   rendersDefaultMaterial: true}
get_material_node_details <switch>
-> A: {Expression: null, OutputIndex: -1}
   B: {Expression: MaterialExpressionConstant3Vector_0}     <- input 1 survived
```

Input 0 is null again. Input 1, connected in the same session by the same means, survived. The next
PIE frame had pale arms again.

## Why this is worse than the sibling ticket

`B-connect-nodes-accepts-true-false-pin-names-on-static-switch-and-wires-nothing` covers
`connect_nodes` failing to bind input 0 while reporting success. This is a different failure and it
defeats the workaround for that one: the connection **was** made (by the engine API, not the verb),
**was** verified by read-back, **was** proven by a shader compile and by a rendered frame, and
**was** saved with the strongest success signal the save verb has. Every check available to a caller
passed. It still did not survive serialisation.

That makes it unfalsifiable in-session: there is no measurement I can take before a restart that
distinguishes "this will persist" from "this will silently revert". The only test is to restart the
editor and look, which is not a check anyone can run per-edit.

Cost here: three builds of a first-person viewmodel rendering as the engine Default Material. Twice
I reported the material fixed on evidence that was, at the time, complete and correct.

## Asked for

1. Find why input 0 of `MaterialExpressionStaticSwitchParameter` is not serialised, or is dropped on
   load, when input 1 on the same node is. `A` and `B` are both `FExpressionInput` on the same
   class, so an asymmetry between them is the place to look.
2. If the write path is at fault rather than the load path, `asset.save` should not report
   `saveState: "written"` over a graph it did not fully serialise.
3. A post-save verification hook would make this class of defect visible: re-read the graph from the
   saved package rather than from memory and diff. `asset.save`'s own docs already warn that reading
   back from `load_asset` returns the in-memory object — this is exactly the case that warning
   exists for, and nothing currently acts on it.

## Workaround, and why I took it

Removed the static switch entirely. `M_FPSArms` is used by one component — the first-person
viewmodel — so the switch had no purpose there; it was mirroring a pattern from a weapon master
shared with world and AI weapons. The `MaterialFunctionCall` now drives World Position Offset
directly, so there is no input-0 to lose.

After: `M_FPSArms.uasset` 16 898 bytes, `EnableViewmodelFOV` **0 occurrences** in the bytes,
`MF_ViewmodelFOV` still 2, `compile_material` -> `succeeded: true, rendersDefaultMaterial: false`,
`generate_thumbnail` -> `usingDefaultMaterial: false, meanLuminance 0.131`. The stale
`EnableViewmodelFOV` override on `MI_FPSArms` was cleared with
`clear_parameter_override` so the instance does not carry an override for a parameter that no longer
exists.

Anyone who needs the switch (a material shared between viewmodel and world) cannot take this
workaround, which is why the ticket stands.

severity rationale: impact=silently reverts a material to the engine Default Material across a
restart, with every in-session check passing x reach=any material using a static switch, the
standard way to gate an optional feature -> High

## History
- `#1-filed` `OPEN` reporter — Found on the FPS PLAYER stream after the arms viewmodel rendered pale in a frame that followed two editor restarts, having rendered dark before them. The measurement chain above is the whole story; the earlier ticket's `connect_nodes` defect is real but separate, and this one survives its workaround.
- `#2-narrowed-by-a-control-it-is-the-POST-HOC-binding-path` `OPEN` reporter — **The WEAPONS stream ran the restart test on an independently authored switch and input 0 SURVIVED, which narrows this from a serialisation bug to a connection-path bug.** Their control, on `/Game/FPS/Weapons/Materials/M_WPN_OpticLens`, same node class (`MaterialExpressionStaticSwitchParameter`), after a full save -> process death -> reload:

```
properties.A = {Expression: ".../M_WPN_OpticLens:MaterialExpressionTransform_1", OutputIndex: 0}
properties.B = {Expression: ".../MaterialExpressionConstant3Vector_0"}
compile_material -> compileSucceeded true, shaderCompile.rendersDefaultMaterial FALSE
```

  **The difference is how the input was set, not what node it is.** Theirs was authored in a single `material.compile_mgir` `Extend` call, which emits `True:` / `False:` as part of the expression's construction — the input is populated while the expression is being created. Mine was bound afterwards with `MaterialEditingLibrary.connect_material_expressions(call, '', switch, 'True')` on an expression that already existed. That post-hoc binding is the half that did not survive.

  So the search is much smaller than `#1` implied: not "`FExpressionInput` A does not serialise" but "an `FExpressionInput` written after the expression exists is not marked dirty, or is not written through to the serialised input, while the same field set at construction is". Retitled accordingly.

  It also makes this and `B-connect-nodes-accepts-true-false-pin-names-on-static-switch-and-wires-nothing` two depths of one problem rather than two problems: `connect_nodes` reports success and binds nothing at all; `connect_material_expressions` binds something that passes every in-session check — read-back, shader compile, rendered frame, `saveState: "written"` — and then does not serialise. Both are the post-hoc connection path being unreliable on this node. **Practical guidance until it is fixed: author expression inputs at construction (`material.compile_mgir` `Extend`) rather than connecting them afterwards**, which is a working route past both tickets and is what the other stream did without hitting either.

  Credit where due: the control was theirs, run at my request after I warned them their hook might have the same defect. It did not, and their negative result is worth more to the fix than my positive one.
