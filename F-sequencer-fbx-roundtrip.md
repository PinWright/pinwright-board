---
id: F-sequencer-fbx-roundtrip
title: "Sequencer FBX import/export of animation"
status: IN-REVIEW
severity: Medium
category: feature
tags: [sequencer, fbx, import-export]
claimedBy: fuzz2
claimedAt: 2026-07-10T17:36:01.3738470+03:00
---

# Sequencer FBX import/export of animation

No FBX operations exist in the sequencer handlers (grep `Fbx|FBX` in `Source\PinWright\Private\Handlers\Sequencer\` = zero matches; ~55 `sequencer.*`/`sequence.*` RPCs, none FBX). Agents cannot round-trip sequence animation through external DCC tools (Maya/MotionBuilder) or ingest baked transform/property FBX onto sequence bindings. (Skeletal mocap is *partially* reachable today via `asset.import` -> `sequencer.add_animation_track`, but that is asset-reference-only and skeletal-only; there is no export path at all, and no baked transform/property-track import path.)

Engine API — present on THIS host's UE 5.7 **and identically on 5.3-5.6** (verified across all installed engines), so this is NOT a 5.8-only capability and needs NO version gate: `USequencerToolsFunctionLibrary::ExportLevelSequenceFBX` / `ImportLevelSequenceFBX` (module `SequencerScriptingEditor`, `SequencerTools.h`). This is the same canonical `SequencerTools` path Epic's Python FBX round-trip uses.

Proposed scope (signatures corrected to the real 5.7 API):
- `sequencer.export_fbx(path, filePath, bindings[]?)` — export bound-object animation to an `.fbx` file via `ExportLevelSequenceFBX(FSequencerExportFBXParams)`. NOTE: the engine exporter has **no** sub-range parameter (the ticket's original `range` arg does not exist in the API) — it always exports the whole playback range. `bindings[]` restricts which object bindings export; omit = all bindings.
- `sequencer.import_fbx(path, filePath, bindings[]?)` — import `.fbx` animation onto the sequence's object bindings via `ImportLevelSequenceFBX(World, Sequence, bindings[], UMovieSceneUserImportFBXSettings*, filename)` (baking transform + animated-property keys, matched to bindings by FBX node name). Import takes a bindings **array** + a typed settings object, NOT a single binding + free-form options.

Acceptance: `sequencer.export_fbx` on a keyed sequence writes a non-empty `.fbx` on disk and reports success; `sequencer.import_fbx` reads it back onto the bindings and reports success — both through the real `USequencerToolsFunctionLibrary` engine calls. Full transform-fidelity round-trip (clear keys -> re-import -> evaluated transform matches the original within tolerance) additionally holds when the bindings resolve to live world actors.

## History
- `#1-no-fbx-io` `OPEN` reporter — No FBX import/export in sequencer handlers (grep verified). Epic 5.8 ships FBX round-trip via SequencerTools; file export_fbx/import_fbx RPCs.
- `#2-reword-then-implement` `IN-REVIEW` fuzz2 — REWORD then implement (GO). Gap confirmed present; the engine FBX API (`ExportLevelSequenceFBX`/`ImportLevelSequenceFBX`) is verified present + stable across UE 5.3-5.7 (all installs), so dropped the misleading `parity-ue58` framing and confirmed NO version gate is required. Corrected the Fix signatures: export takes `FSequencerExportFBXParams` (no `range` param — full playback range only); import takes a `bindings[]` array + `UMovieSceneUserImportFBXSettings`. Severity/category unchanged (Medium/feature). Implementing `sequencer.export_fbx` + `sequencer.import_fbx` over `SequencerTools`.
