---
id: F-sequencer-bake-controlrig
title: "Sequencer bake: anim sequence to Control Rig track and back, space-switch bake"
status: OPEN
severity: High
category: feature
tags: [sequencer, controlrig, bake, animation, parity-ue58]
---

# Sequencer bake: anim sequence to Control Rig track and back, space-switch bake

No bake operations exist in PinWright's sequencer handlers (grep `bake|Bake` in `Source\PinWright\Private\Handlers\Sequencer\` = zero matches). An agent can neither convert an existing animation into an editable Control Rig track nor export a keyed CR performance back to an AnimSequence asset, which makes the CR cinematics loop a dead end even once F-sequencer-controlrig-track lands.

UE 5.8 parity evidence (`C:\UE_5.8\Engine\Plugins\Experimental\Toolsets\AnimationAssistantToolset\Content\Python\animation_toolset\toolsets\controlrig_sequencer.py` and `sequencer.py`): `bake_to_control_rig`, `export_anim_sequence`, and `bake_space` (space switching with baked transition), over `ControlRigSequencerLibrary` / `SequencerTools`.

Proposed scope:
- `sequencer.bake_to_controlrig(sequence, binding, rigClass, options)` - convert existing anim section keys into a CR track.
- `sequencer.export_anim_sequence(sequence, binding, outAssetPath, range)` - bake the evaluated performance to a new/existing AnimSequence.
- `sequencer.bake_control_space(sequence, binding, control, newSpace, range)` - space switch with baked keys.

Depends on: F-sequencer-controlrig-track.

Acceptance: round trip an AnimSequence -> CR track -> edited key -> exported AnimSequence; exported asset plays with the edit; space bake produces identical world-space pose across the switch frame.

## History
- `#1-no-bake-ops` `OPEN` reporter — No bake in sequencer handlers (grep verified). Epic 5.8 ships bake_to_control_rig / export_anim_sequence / bake_space; both directions plus space-switch bake needed for a usable CR cinematics loop.
