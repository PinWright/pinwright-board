---
id: F-rpc-sequencer-list-sections
title: "Add live RPC `sequencer.list_sections` returning per-section range/blendType/rowIndex/channels"
status: DONE
severity: High
category: feature
tags: [sequencer, asset-dump, parity, policy]
---

# Add live RPC `sequencer.list_sections` returning per-section range/blendType/rowIndex/channels

`LevelSequenceDumpBuilder::BuildSectionJson` (in `Source/PinWright/Private/Handlers/Asset/LevelSequenceDumpBuilder.cpp`) emits per-section payload into `level_sequence.json` containing `range{start,end}` (frame numbers), `blendType` (string from `EMovieSceneBlendType`), `rowIndex`, and `channels[{type, keyCount}]` derived from `Section->GetChannelProxy()`. None of this is reachable via a live RPC. `sequencer.list_tracks` (`SequenceHandler.cpp:2379`) returns per-track `sectionCount` only — it never iterates `Track->GetAllSections()` to surface section state, and no other registered `sequencer.*` method (38 enumerated via `REGISTER_RPC_HANDLER("sequencer.`) returns channel-level data.

This violates the "asset dumps must not have exclusive functionality" policy: any caller that wants to verify section frame ranges, blend types, row layout, or channel/key counts without re-dumping the asset has no live path today.

**Fix:** Add a new handler `sequencer.list_sections` registered alongside `sequencer.list_tracks` in `SequenceHandler.cpp`. Required param `path` (sequence asset path), optional filters `bindingGuid` and `trackName` to narrow the result. Response shape mirrors the dump exactly so call-site assertions can compare directly against `level_sequence.json`:

```
{
  "sequencePath": "...",
  "sections": [
    {
      "trackName": "...",
      "trackClass": "...",
      "bindingGuid": "..."  // empty for master tracks
      "rowIndex": 0,
      "blendType": "Absolute" | "Additive" | "Relative" | "" (when invalid),
      "range": { "start": <frameNum>, "end": <frameNum> },  // omit bound when unbounded
      "channels": [ { "type": "MovieSceneFloatChannel", "keyCount": 12 }, ... ]
    },
    ...
  ],
  "sectionCount": N
}
```

Reuse the dump's helpers — extract `MakeFrameRangeObject`, `BlendTypeToString`, and `BuildChannelEntriesJson` from the anonymous namespace in `LevelSequenceDumpBuilder.cpp` into a shared header (e.g. `Utils/MovieSceneJsonUtils.h`, where `MakeFrameRateObject` already lives) so the handler and the dump emit byte-identical JSON. Without that extraction the live RPC will drift from the dump and the parity guarantee breaks.

Channels could alternatively split into a separate `sequencer.list_channels` (params: section identity by `bindingGuid` + `trackName` + `rowIndex`), but nesting them under `list_sections` matches the dump's shape and keeps the round-trip count down. Defer the split until a real consumer needs per-channel detail beyond `{type, keyCount}` (e.g. enumerated key times/values), at which point `list_channels` becomes the natural home for the deeper payload.

## History
- `#1-initial-repro` `OPEN` reporter — `LevelSequenceDumpBuilder` writes per-section `range`, `blendType`, `rowIndex`, and `channels[{type,keyCount}]` to `level_sequence.json`, but `sequencer.list_tracks` exposes only `sectionCount`. Enumerated all 38 `sequencer.*` registrations via grep — no live RPC returns per-section or channel state. Violates the no-exclusive-dump-functionality policy.
- `#2-implemented-list-sections` `IN-REVIEW` developer — Added `sequencer.list_sections` in `SequenceHandler.cpp`, shared MovieScene section JSON helpers through `MovieSceneJsonUtils.h`, and covered live section range/blend/row/channel parity with an automation test.
- `#3-verify-fix` `DONE` tester — Verified: `sequencer.list_sections` on `/Game/Meadow_Environment_Set/Maps/LS_DroneRotation` returned `sectionCount=3` with full per-section payload (`trackName`, `trackClass`, `bindingGuid`, `rowIndex`, `blendType="Absolute"`, `range={}` for unbounded, `channels[]` with `type`+`keyCount`). `bindingGuid` filter narrowed result to 1 section as expected. Response shape matches dump spec.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place; every rewritten path was confirmed to exist at plugin HEAD `ef8a1f1b`. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
