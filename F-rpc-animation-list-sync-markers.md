---
id: F-rpc-animation-list-sync-markers
title: "Add live RPC `animation.authoring.list_sync_markers`"
status: DONE
severity: Medium
category: feature
tags: [rpc, animation, asset-dump, parity]
---

# Add live RPC `animation.authoring.list_sync_markers`

`AnimSequenceDumpBuilder::BuildAnimSequenceJson` emits a `syncMarkers[]` array
(each entry `{name, time}`, sourced from `UAnimSequence::AuthoredSyncMarkers`)
into the asset-dump `anim_sequence.json` sidecar. There is no equivalent live
RPC: the only sync-marker handler is the writer `animation.authoring.add_sync_marker`
(AnimationAuthoringHandler.cpp:772), and `animation.authoring.get_animation_info`
(line 3320) only reports counts, not the marker list. This violates the policy
that asset dumps must not have exclusive functionality — anything a dump emits
should also be reachable via a live MCP RPC.

Place the reader next to the writer in `animation.authoring`, matching the
existing reader-next-to-writer pattern (`skeleton.list_morph_targets`,
`skeleton.list_virtual_bones`, `skeleton.list_sockets`). Output should mirror
the dump's shape: sorted by `time` ascending then `name`, each entry
`{name: string, time: number}`. Skip the `#if WITH_EDITORONLY_DATA` content
on non-editor builds (return empty array), as the dump builder does.

**Fix:** Add `REGISTER_RPC_HANDLER("animation.authoring.list_sync_markers", ...)`
in `AnimationAuthoringHandler.cpp` taking `assetPath` (string, required).
Resolve the sequence via `LoadAnimSequenceFromPath`, iterate
`Sequence->AuthoredSyncMarkers`, build the same `{name, time}` JSON entries
`BuildSyncMarkersArray` in `AnimSequenceDumpBuilder.cpp` produces (sorted by
time then name), and return `{ syncMarkers: [...] }`. Consider extracting the
shared sort-and-emit helper so both the dump builder and the live handler call
the same code, preventing drift.

## History
- `#1-initial-repro` `OPEN` reporter — `anim_sequence.json` emits `syncMarkers[]` but no live RPC reads them; only `animation.authoring.add_sync_marker` (writer) and `get_animation_info` (counts only) exist. Add `animation.authoring.list_sync_markers` next to the writer with output shape matching the dump's `{name, time}` entries (sorted by time then name).
- `#2-added-live-reader` `IN-REVIEW` developer — Exported the shared sync-marker dump serializer, added `animation.authoring.list_sync_markers` beside the writer, and covered registration plus sorted `{name,time}` shape with a regression test.
- `#3-verify-fix` `DONE` tester — Verified: `animation.authoring.list_sync_markers` wiki page shows required `assetPath` schema; live call on `/Game/Meshes/Clutch/SK_Clutch02_Anim` returned `{success, assetPath, syncMarkers: []}` initially, then after adding ZMarker (frame 15) and AMarker (frame 5) via `add_sync_marker`, returned `syncMarkers: [{name:"AMarker", time:0.1667}, {name:"ZMarker", time:0.5}]` — sorted by time ascending with the expected `{name,time}` shape.
