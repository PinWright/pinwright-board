---
id: F-sequencer-evaluate-readback
title: "Sequencer force-evaluate and transform-at-frame readback"
status: OPEN
severity: High
category: feature
tags: [sequencer, readback, verification, parity-ue58]
---

# Sequencer force-evaluate and transform-at-frame readback

PinWright's sequencer readback is static asset structure only (`get_properties`, `list_sections`, `list_tracks`, `get_bindings`); there is no evaluated-state readback (grep `Evaluate|EvaluateAt` in `Source\PinWright\Private\Handlers\Sequencer\` = zero). After authoring keys, an agent cannot ask "where is this actor at frame N", so the cinematics edit-verify loop stays open: the only verification path today is screenshots.

UE 5.8 parity evidence: AnimationAssistantToolset ships `force_evaluate` and transform-at-frame readback as part of its getter-for-every-setter design (`C:\UE_5.8\Engine\Plugins\Experimental\Toolsets\AnimationAssistantToolset\Content\Python\animation_toolset\toolsets\sequencer.py`), which is what makes its 319-tool suite verifiable end to end.

Proposed scope:
- `sequencer.evaluate_at(sequence, frame)` - force evaluation of the open sequence at a frame (editor preview context).
- `sequencer.get_binding_transform(sequence, binding, frame)` - world/local transform of a bound actor/component at a frame.
- `sequencer.get_evaluated_property(sequence, binding, track, frame)` - evaluated value of any keyed property channel.

Acceptance: key a location track at two frames, evaluate at midpoint, returned transform matches expected interpolated value within tolerance; works for a CR control value once F-sequencer-controlrig-track lands.

## History
- `#1-no-evaluated-readback` `OPEN` reporter — Sequencer readback is structural only; no force-evaluate or transform-at-frame (grep verified). Epic 5.8 pairs every setter with an evaluated getter; file evaluate_at + binding-transform + evaluated-property RPCs to close the cinematics verify loop.
