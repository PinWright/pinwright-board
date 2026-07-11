---
id: E-describe-sound-cue-omits-cue-level-fields
title: "audio.authoring.describe_sound_cue omits cue-level ConcurrencySet/Attenuation/SoundClass/Volume/Pitch — the named inspect-after-mutate readback can't verify cue-level writes"
status: OPEN
severity: Medium
category: ergonomic
tags: [audio, sound-cue, describe_sound_cue, readback, cue-level-fields, inspect-after-mutate, describe-omits-field]
encounters: 1
lastSeen: 2026-07-11T03:02:19.4116967+03:00
---

# `audio.authoring.describe_sound_cue` omits every cue-level field, so it can't confirm concurrency / attenuation writes

`audio.authoring.describe_sound_cue` is the readback the `audio.authoring`
"Inspect-after-mutate" overlay names for SoundCue verification ("dump-parity live
read ... callers can verify graph edits"), and it is the readback the shared-
concurrency task's success check is built around. But it delegates to
`SoundCueDumpBuilder::BuildSoundCueJson`, which emits only the node **graph** —
`assetKind`, `path`, `firstNode`, and the `nodes[]` array (per-node
`className` / `properties` / `edges` / `soundWave`). It emits **no cue-level
fields at all**: not `ConcurrencySet`, not `AttenuationSettings`, not
`SoundClass`, `VolumeMultiplier`, or `PitchMultiplier`.

The practical consequence: `set_cue_concurrency` writes the cue's
`ConcurrencySet`, but `describe_sound_cue` returns byte-identical output whether
the cue has a shared concurrency group, no concurrency, or just had one cleared.
It can neither confirm the shared reference nor distinguish a lifted cue from a
still-throttled one — the exact verification the task needed. The only live
reader that surfaces these cue-level fields is `audio.authoring.decompile_sound_cue`
(SCIR), whose cue-level block covers `attenuation`, `concurrency`, `sound_class`,
`volume`, and `pitch` (see `F-sound-cue-decompile-scir`). So an agent that
follows the overlay's own guidance to `describe_sound_cue` gets nothing back and
must pivot to a second reader (SCIR) to prove the write took.

This is the same readback-omits-an-authored-field class as
`E-texture-describe-omits-lodbias-wrap` (describe surfaces the graph but drops
the setter's own target field) — here the sibling setter `set_cue_concurrency`
writes a field the paired `describe` never reports.

## Evidence (this task)

Focus `audio.authoring.set_cue_concurrency`; the story built three cues
(`SC_Manip_Pickup` / `SnapToActor` / `Scale`), wired all three to a shared
`CG_ManipulationSFX` (max 3, StopOldest), then lifted the limit off `Scale`.
The call log shows the readback failing to verify at every checkpoint:

- Baseline `describe_sound_cue` on the pickup cue: no concurrency field present.
- After all three `set_cue_concurrency` saves, `describe_sound_cue` on pickup,
  snap, and scale: still no concurrency field on any of the three.
- After lifting `Scale`, `describe_sound_cue` on scale post-lift: still no
  concurrency field — indistinguishable from the still-limited pickup/snap.

The agent recovered only by cross-checking `decompile_sound_cue` (SCIR), which
showed `concurrency [/Game/Audio/Concurrency/CG_ManipulationSFX...]` on all three
pre-lift and its absence on scale after the clear — five extra decompile calls to
do the verification the named `describe` readback should have done in one. Root
cause confirmed in a last-resort source read: `SoundCueDumpBuilder.cpp` emits no
cue-level properties, while `SCIRDecompiler.cpp` emits the `concurrency [...]`
block.

## What it should do / how to fix

Extend `SoundCueDumpBuilder::BuildSoundCueJson` (the single builder both
`describe_sound_cue` and the `sound_cue.json` asset-dump sidecar share, so the
fix lands on both surfaces at once) to emit a cue-level block alongside the node
graph: `concurrency` (the `ConcurrencySet` paths), `attenuation`
(`AttenuationSettings`), `soundClass`, `volume` (`VolumeMultiplier`), and `pitch`
(`PitchMultiplier`) — the same cue-level surface `decompile_sound_cue` already
produces. Then the inspect-after-mutate flow for `set_cue_concurrency` /
`set_cue_attenuation` verifies through the overlay's own named readback without
falling back to SCIR.

**Workaround:** verify cue-level concurrency / attenuation / soundClass / volume
/ pitch with `audio.authoring.decompile_sound_cue` (SCIR), not
`describe_sound_cue`.

severity rationale: impact=readback omits a field and forces a fallback (Medium) x reach=cue-level concurrency/attenuation authored on a non-every-session audio path (no bump) -> Medium

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of the shared-manipulation-SFX concurrency task (focus `audio.authoring.set_cue_concurrency`). Distinct PROCESS angle from the judge's silent-no-op ticket `B-cue-setter-missing-asset-silent-success`: `describe_sound_cue` — the readback the audio.authoring overlay names for cue verification and the tool the success check relied on — emits only the node graph (firstNode/nodes[]) via `SoundCueDumpBuilder::BuildSoundCueJson` and omits ALL cue-level fields, so it returned no concurrency field for any of the 3 cues both before and after the lift and could not confirm or distinguish the writes. Agent recovered with 5 extra `decompile_sound_cue` (SCIR) calls, which DO surface `concurrency`/`attenuation`/`sound_class`/`volume`/`pitch` (per `F-sound-cue-decompile-scir`); root-caused in source (`SoundCueDumpBuilder.cpp` no cue-level props vs `SCIRDecompiler.cpp` emits them). Proposed: extend the shared `BuildSoundCueJson` with a cue-level block (concurrency/attenuation/soundClass/volume/pitch), fixing both `describe_sound_cue` and the `sound_cue.json` sidecar in one place. Same readback-omits-authored-field class as `E-texture-describe-omits-lodbias-wrap`.
