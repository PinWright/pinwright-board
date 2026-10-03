---
id: F-sequencer-binding-convert
title: "Sequencer has no typed convert_binding verb (possessable <-> spawnable); the engine's ConvertToSpawnable/ConvertToPossessable work only on the focused Sequencer and fail silently otherwise"
status: OPEN
severity: Low
rice: [1, 1, 0.8, 2]
priority: 3
category: feature
tags: [sequencer, bindings, parity-ue58]
---

# Sequencer has no typed verb to convert an existing binding between possessable and spawnable

PinWright can create bindings either way: `sequencer.add_actor` / `add_actors` make possessables
and `sequencer.add_spawnable_from_class` makes spawnables (`Source/PinWright/Private/Handlers/Sequencer/SequenceHandler.cpp:876`,
`:1042`, `:1447`). There is no verb that converts an existing binding. Nothing under
`Handlers/Sequencer/` calls `ConvertToSpawnable` / `ConvertToPossessable` (grep at 7230b41d).

The engine already has the conversion. `ULevelSequenceEditorSubsystem::ConvertToSpawnable` and
`ConvertToPossessable` are `BlueprintCallable` (`LevelSequenceEditorSubsystem.h:168-173`) and
use the same `FSequencerUtilities` path as the Sequencer UI. They have two traps that make
calling them raw through `python.execute` error-prone:

- They act on the **focused sequence of the active Sequencer editor**, not on the sequence the
  binding proxy names (`LevelSequenceEditorSubsystem.cpp:1705-1715`, `GetActiveSequencer` at
  `:864`). If the sequence is not open, they return an empty array or proxy and report no error.
  If a different level sequence is focused, the binding GUID is looked up in that sequence.
- Converting a possessable with several bound objects to spawnable creates one spawnable per
  object and returns an array.

**Fix:** Add `sequencer.convert_binding {path, bindingGuid, to: "spawnable"|"possessable"}`.
It opens `path` in Sequencer, or refuses if a different sequence is focused. It calls the
subsystem method and refuses with a named error when the engine returns no new binding. The
response returns the new binding GUID(s) plus the `get_bindings` row (`kind`), and
`list_sections` on the new GUID shows the moved tracks. Document the
open-in-Sequencer requirement in `docs/wiki-src/sequencer.md`.

**Acceptance:** A keyed possessable converts to a spawnable with the same tracks, section
ranges and key counts in `list_sections`. `get_bindings` reports `kind:"spawnable"` for it, and
the reverse conversion works too. The verb refuses, rather than reporting success, when the
engine produces no binding.

**Workaround:** Call `editor.open_asset` on the sequence, then use `python.execute` with
`unreal.get_editor_subsystem(unreal.LevelSequenceEditorSubsystem).convert_to_spawnable(proxy)`
(or `convert_to_possessable`). Get the proxy from the sequence's binding with that GUID.
Confirm the result with `sequencer.get_bindings`.

## History
- `#1-no-binding-convert` `OPEN` reporter — Existing bindings cannot switch possessable/spawnable without losing tracks (grep verified). Epic 5.8 has conversion tools; file convert_binding preserving tracks and keys.
- `#2-rephrased` `OPEN` developer — Rephrased. The old text said that changing ownership means deleting the binding and losing its tracks. That is wrong: `ULevelSequenceEditorSubsystem::ConvertToSpawnable`/`ConvertToPossessable` are BlueprintCallable (`LevelSequenceEditorSubsystem.h:168-173`) and reachable through `python.execute` today. They act only on the active Sequencer's focused sequence and return an empty result with no error otherwise (`LevelSequenceEditorSubsystem.cpp:1705-1715`, `:864`). Reframed as a typed wrapper that opens the sequence or refuses, and refuses when no binding is produced. Added the Python workaround, the citations to the creation verbs (`SequenceHandler.cpp:876`, `:1042`, `:1447`), and Acceptance. Severity unchanged (Low). RICE E 1->2: a new verb with open/focus guarding plus a test is one handler plus tests, not a doc-sized change.
