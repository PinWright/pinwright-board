---
id: F-sequencer-fbx-roundtrip
title: "Sequencer FBX import/export of animation"
status: OPEN
severity: Medium
category: feature
tags: [sequencer, fbx, import-export, parity-ue58]
---

# Sequencer FBX import/export of animation

No FBX operations exist in the sequencer handlers (grep `Fbx|FBX` in `Source\PinWright\Private\Handlers\Sequencer\` = zero matches). Agents cannot round-trip sequence animation through external DCC tools or ingest mocap/animation deliveries into a sequence.

UE 5.8 parity evidence: AnimationAssistantToolset import/export group (6 tools, `C:\UE_5.8\Engine\Plugins\Experimental\Toolsets\AnimationAssistantToolset\Content\Python\animation_toolset\toolsets\import_export.py`) does FBX round-trip with live anim-sequence linking, over `SequencerTools` (`ImportLevelSequenceFBX` / `ExportLevelSequenceFBX`).

Proposed scope:
- `sequencer.export_fbx(sequence, bindings[], filePath, range)` - export bound-actor animation to FBX.
- `sequencer.import_fbx(sequence, binding, filePath, options)` - import FBX animation onto a binding (transform + animated properties).

Acceptance: export a keyed transform track to FBX, clear keys, re-import, evaluated transforms match the original within tolerance.

## History
- `#1-no-fbx-io` `OPEN` reporter — No FBX import/export in sequencer handlers (grep verified). Epic 5.8 ships FBX round-trip via SequencerTools; file export_fbx/import_fbx RPCs.
