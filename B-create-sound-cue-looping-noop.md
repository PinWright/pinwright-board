---
id: B-create-sound-cue-looping-noop
title: "audio.authoring.create_sound_cue silently ignores looping/volume/pitch when wavePath is omitted"
status: IN-REVIEW
severity: Medium
category: bug
tags: [audio, sound-cue, looping, silent-noop]
---

# audio.authoring.create_sound_cue silently ignores looping/volume/pitch when wavePath is omitted

The `audio.authoring.create_sound_cue` wiki page documents `looping`, `volume`,
and `pitch` as standalone optional parameters ("`looping` (boolean, optional):
Add looping node", "`volume` (number, optional): Volume multiplier", etc.) with
no stated precondition that they require `wavePath`. In the handler
(`Handlers/Audio/AudioAuthoringHandler.cpp`, the `create_sound_cue` body), the
**entire** node-graph build — wave player, looping node, and volume/pitch
modulator — is gated behind a single `if (!WavePath.IsEmpty())` guard
(lines ~386-415). So when `looping=true` (or non-default `volume`/`pitch`) is
passed **without** `wavePath`, those parameters are read and then silently
dropped: the cue is created with an empty graph and the call still returns a
clean success ("SoundCue '<name>' created", `existsAfter:true`). No error, no
warning, no field in the result indicating the looping/volume/pitch request was
ignored.

This is a silent success-with-no-effect on a documented input. The caller is
told the cue was created and has no signal that the looping it asked for never
materialized. Worse, the verification path makes it hard to notice:
`describe_sound_cue` reports `firstNode:""` / `nodes:[]` (no top-level
"looping" flag to contradict the request), and `get_audio_info` only reports
`nodeCount`. A fuzzing-attempt agent doing this exact task had to read the
handler C++ to discover the gating, then work around it with `add_cue_node`
(looping) + `property.set` (FirstNode wiring) to actually get a looping cue.

**Impact:** The single most natural way to author a looping ambient/music cue
("create a looping SoundCue") quietly produces a non-looping empty cue. The
looping/volume/pitch parameters are effectively dead unless the caller also
happens to supply a SoundWave at create time.

**Workaround:** After `create_sound_cue`, call `add_cue_node`
(`nodeType:"looping"`) and wire it as the cue's `FirstNode` (e.g. via
`property.set` on `FirstNode`, or `connect_cue_nodes`). For volume/pitch, add a
`modulator` node the same way.

**Fix (either):** (a) Make the looping/volume/pitch build path run even when
`wavePath` is empty — construct a `USoundNodeLooping` / `USoundNodeModulator`
chain rooted at `FirstNode` with no wave player child (the agent's manual
workaround proves a looping node with no wave child is a valid graph state for
later wiring); or (b) if the parameters genuinely cannot apply without a wave,
reject the combination with a clear error (e.g.
`LOOPING_REQUIRES_WAVE` "looping/volume/pitch require wavePath") instead of
fake-succeeding, and/or document the precondition on the wiki page. Silently
accepting-and-dropping is the failure mode to remove.

## Verbatim repro

`audio.authoring.create_sound_cue`
args:
```json
{"name":"SC_ReplayLoopNoWave","path":"/Game/Audio/ReplayTest","looping":true}
```
result (clean success — no hint looping was dropped):
```json
{"assetPath":"/Game/Audio/ReplayTest/SC_ReplayLoopNoWave","message":"SoundCue 'SC_ReplayLoopNoWave' created","assetName":"SC_ReplayLoopNoWave","existsAfter":true,"assetClass":"SoundCue"}
```

`audio.authoring.describe_sound_cue`
args:
```json
{"assetPath":"/Game/Audio/ReplayTest/SC_ReplayLoopNoWave"}
```
result (empty graph — the looping node was never created):
```json
{"assetKind":"SoundCue","path":"/Game/Audio/ReplayTest/SC_ReplayLoopNoWave.SC_ReplayLoopNoWave","firstNode":"","nodes":[]}
```

## History
- `#1-initial-repro` `OPEN` reporter — Replayed `create_sound_cue` with `looping:true` and no `wavePath`: returned clean success (`message:"SoundCue 'SC_ReplayLoopNoWave' created"`, `existsAfter:true`), but `describe_sound_cue` shows `firstNode:""` / `nodes:[]` — no looping node. Root cause confirmed in AudioAuthoringHandler.cpp: the looping/volume/pitch graph build is entirely inside `if (!WavePath.IsEmpty())`, so all three params are silently dropped when wavePath is omitted. Documented optional params with no stated wavePath precondition. Silent success-with-no-effect.
- `#2-fix-build-chain-without-wave` `IN-REVIEW` developer — Implemented Fix (a): `create_sound_cue` no longer gates the looping/volume/pitch nodes behind `if (!WavePath.IsEmpty())`. Refactored the graph build in `Source/EditorAutomationRpcGateway/Private/Handlers/Audio/AudioAuthoringHandler.cpp` so the wave player is the only piece requiring `wavePath`; the `USoundNodeLooping` (when `looping=true`) and `USoundNodeModulator` (when `volume`/`pitch` differ from 1.0) are now constructed whenever requested, chained onto the wave player if present and standing alone (root) otherwise, with `FirstNode` set + `LinkGraphNodesFromSoundNodes()` whenever any node was built — so `looping:true` with no `wavePath` now yields a cue rooted at a looping node (matching the `add_cue_node` standalone-node precedent). Also hardened the wave path: a supplied-but-unloadable `wavePath` now returns `SendError("WAVE_NOT_FOUND", ...)` instead of the prior silent empty-cue success. Regression test added: `Source/EditorAutomationRpcGateway/Private/Tests/Assets/TestCreateSoundCueLooping.cpp` (`EditorAutomationRpcGateway.Assets.SoundCue.CreateSoundCue.LoopingNoWaveBuildsLoopingNode` + `.VolumePitchNoWaveBuildsModulator`) drives the real registered handler via `InvokeHandlerWithCapture` with `looping:true`/`volume`/`pitch` and no `wavePath`, asserting `FirstNode` is a `USoundNodeLooping` / `USoundNodeModulator` — both fail against the pre-fix gated build (FirstNode null).
- `#3-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 2 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
