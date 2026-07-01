---
id: B-agir-missing-anim-sequence-refs
title: "AGIR omits anim-sequence references (Pose nodes, notifies, sync groups, montage data)"
status: DONE
severity: Medium
category: bug
tags: [agir, anim-graph, decompiler]
---

# AGIR omits anim-sequence references

`agir.txt` files emit AnimGraph state machines but never include:

- Animation clips referenced in Pose nodes (just node placeholder, no asset link)
- Play rate / play rate curves
- Notifies (timestamp + event name)
- Sync group membership
- Additive layer metadata
- Montage-linked data

All 49 AGIR files in the dump tree show this pattern.

## Sample

`App/Meshes/Mannequin/Animations/ThirdPerson_AnimBP/agir.txt` — state machine present, no anim asset links.

## Fix

The missing references come from state-machine state body graphs not being walked. `state_machine` emission created `state Name { ... }` shells but never visited each `UAnimStateNode::BoundGraph`, so asset-player nodes inside states were invisible to AGIR.

Do not add `anim_clip` syntax. State bodies should use existing canonical AGIR graph instructions inside the existing state block:

```
state Idle {
    %sequence_player_0 = call `/Script/AnimGraph.AnimGraphNode_SequencePlayer`(Sequence: "/Game/...", PlayRate: 1.25, GroupName: "Locomotion")
    output %sequence_player_0
}
```

Animation asset internals such as notifies and montage data stay in existing asset sidecars. AGIR should reference the assets and reflected player fields, not duplicate each asset schema.

## History
- `#1-agir-anim-refs` `OPEN` reporter — all 49 files affected. State machine context is incomplete without clip references.
- `#2-state-body-asset-refs` `IN-REVIEW` developer — Emitted and compiled AGIR state body call/output instructions so state-machine asset player nodes expose sequence references and reflected fields, with a regression test for sequence player state bodies.
- `#3-verify-fix` `DONE` tester — Verified: ran asset.dump on /App/Meshes/Mannequin/Animations/ThirdPerson_AnimBP; agir.txt state bodies now contain call AnimGraphNode_SequencePlayer/BlendSpacePlayer with Sequence/BlendSpace asset paths and reflected fields (PlayRate: 0.75, bLoopAnimation: false), matching the canonical form in the ticket.
