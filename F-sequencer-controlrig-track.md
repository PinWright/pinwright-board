---
id: F-sequencer-controlrig-track
title: "Sequencer Control Rig track: add, key controls at frames, read back"
status: OPEN
severity: High
category: feature
tags: [sequencer, controlrig, animation, parity-ue58]
---

# Sequencer Control Rig track: add, key controls at frames, read back

PinWright's sequencer namespace (37 methods) has zero Control Rig support: no way to add a Control Rig track to a binding, key CR controls, or read control values. Verified: no `ControlRig` reference anywhere in `Source\PinWright\Private\Handlers\Sequencer\` (SequencerHandler.cpp, SequenceHandler.cpp). The `controlrig` namespace is CRIR asset round-trip only; no sequencer integration.

UE 5.8's built-in AnimationAssistantToolset ships a 72-tool ControlRig-in-Sequencer suite (`C:\UE_5.8\Engine\Plugins\Experimental\Toolsets\AnimationAssistantToolset\Content\Python\animation_toolset\toolsets\controlrig_sequencer.py`), including `key_controls_at_frames`, control listing, and per-control transform readback, built over `ControlRigSequencerLibrary`. This is the entry point to the whole cinematics workflow (bake, layers, spaces depend on CR tracks existing).

Proposed scope:
- `sequencer.add_controlrig_track(sequence, binding, rigClass)` - add CR track for a skeletal binding (FK fallback when no rig asset given).
- `sequencer.list_controls(sequence, binding)` - control names, types, current values.
- `sequencer.key_controls(sequence, binding, frame, controls{name: value})` - key one or more controls at explicit frames; supports transform/float/bool/vector control types.
- `sequencer.get_control_value(sequence, binding, control, frame)` - evaluated value readback.

Acceptance: on a skeletal-mesh binding, add a CR track, key two controls at two frames, read values back at both frames, values match within tolerance; decompiled `list_sections` shows the CR section.

## History
- `#1-cr-track-gap` `OPEN` reporter — Sequencer namespace has no Control Rig track support (zero ControlRig references in Handlers\Sequencer). UE 5.8 built-in toolset ships 72 CR-in-Sequencer tools; parity requires track add + key controls + readback.
