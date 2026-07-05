---
id: F-sequencer-binding-convert
title: "Sequencer convert existing binding possessable <-> spawnable"
status: OPEN
severity: Low
category: feature
tags: [sequencer, bindings, parity-ue58]
---

# Sequencer convert existing binding possessable <-> spawnable

Bindings can be created either way (`AddPossessable`, `AddSpawnable` via `sequencer.add_spawnable_from_class`) but an existing binding cannot be converted: no `ConvertToSpawnable`/`ConvertToPossessable` anywhere in `Source\PinWright\Private\Handlers\Sequencer\` (grep verified). Changing your mind about ownership currently means deleting the binding and losing all its tracks/keys.

UE 5.8 parity evidence: AnimationAssistantToolset binding model includes possessable/spawnable/custom binding conversions (`...\animation_toolset\toolsets\custom_bindings.py`, 8 tools), over `LevelSequenceEditorSubsystem` conversion APIs.

Proposed scope:
- `sequencer.convert_binding(sequence, binding, to: possessable|spawnable)` preserving tracks, sections, and keys.

Acceptance: keyed possessable converted to spawnable retains all tracks/keys and evaluates identically; reverse direction too.

## History
- `#1-no-binding-convert` `OPEN` reporter — Existing bindings cannot switch possessable/spawnable without losing tracks (grep verified). Epic 5.8 has conversion tools; file convert_binding preserving tracks and keys.
