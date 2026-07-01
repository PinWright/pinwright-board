---
id: E-rpc-sequencer-extend-get-properties
title: "Extend `sequencer.get_properties` with tickResolution / spawnableCount / possessableCount"
status: DONE
severity: Low
category: ergonomic
tags: [sequencer, asset-dump, parity]
---

# Extend `sequencer.get_properties` with tickResolution / spawnableCount / possessableCount

`LevelSequenceDumpBuilder::BuildLevelSequenceJson` emits `tickResolution{numerator,denominator}`, `displayRate`, `playbackRange`, `bindingCount`, `spawnableCount`, and `possessableCount` at the root of the level-sequence dump (`LevelSequenceDumpBuilder.cpp:113-120`). The matching live RPCs do not cover the same surface:

- `sequencer.get_properties` (`SequenceHandler.cpp:1029-1077`) returns only `frameRate` (= `MovieScene->GetDisplayRate()`), `playbackStart`, `playbackEnd`, `duration`. There is no way to read the tick resolution at runtime even though `sequencer.set_tick_resolution` already exists as a setter (`SequenceHandler.cpp:1808`).
- `sequencer.get_bindings` (`SequenceHandler.cpp:971-1024`) returns the binding array with `id` + `name` only — it never tells the caller whether each binding is a `Spawnable` or a `Possessable`, and exposes no counts.

This violates the "asset dumps must not have exclusive functionality" policy: three pieces of state (`tickResolution`, `spawnableCount`, `possessableCount`) are observable only through `asset.dump`.

**Fix:** Extend `sequencer.get_properties` to also return `tickResolution{numerator,denominator}` (from `MovieScene->GetTickResolution()`), `bindingCount`, `spawnableCount`, and `possessableCount` (from the same `GetBindings()` / `GetSpawnableCount()` / `GetPossessableCount()` calls the dump builder uses). Optionally also add a `kind: "spawnable" | "possessable"` field per entry in `sequencer.get_bindings` by branching on `MovieScene->FindPossessable` / `FindSpawnable` (those calls already happen in the handler — just promote the discriminator into the response). Existing fields remain unchanged for back-compat.

## History
- `#1-initial-repro` `OPEN` reporter — `sequencer.get_properties` omits tickResolution / spawnable+possessable counts that `LevelSequenceDumpBuilder` already emits. `sequencer.get_bindings` doesn't distinguish spawnable vs possessable. Result: three pieces of MovieScene state are reachable only via `asset.dump`, breaking the dump-vs-live parity policy.
- `#2-extended-get-properties` `IN-REVIEW` developer — Changed `sequencer.get_properties` in `SequenceHandler.cpp` to return `tickResolution`, `bindingCount`, `spawnableCount`, and `possessableCount` from `UMovieScene` while preserving existing fields; added a captured handler regression test in `TestSequencerHandlers.cpp` for the new response fields.
- `#3-verify-fix` `DONE` tester — Verified: `sequencer.get_properties` on `/App/Sequences/FlythroughSequence01` returned `tickResolution:{numerator:24000,denominator:1}`, `bindingCount:8`, `spawnableCount:0`, `possessableCount:8` alongside the preserved `frameRate`, `playbackStart`, `playbackEnd`, `duration`.
