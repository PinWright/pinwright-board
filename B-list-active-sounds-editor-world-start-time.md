---
id: B-list-active-sounds-editor-world-start-time
title: "audio.list_active_sounds in a non-ticking editor world: playbackTimeSeconds stays 0 and startWorldTimeSeconds reports 'now', so two dispatches seconds apart show the same start time"
status: OPEN
severity: Medium
category: bug
tags: [audio, runtime, active-sound, list_active_sounds, timing, editor-world, silent-wrong-data]
encounters: 1
lastSeen: 2026-10-02T18:45:00Z
rice: [1, 3, 1, 1]
priority: 33
---

# startWorldTimeSeconds is not a start time when the game is not ticking

`audio.list_active_sounds` emits `startWorldTimeSeconds = World->GetTimeSeconds() -
ActiveSound.PlaybackTimeUnscaled` (`Handlers/Audio/AudioActiveSoundsHandler.cpp:100-101`) and the
wiki calls it "the engine's own WorldTimeWhenPlayed", recommending it for spotting double
dispatches ("two rows with the same soundPath and close startWorldTimeSeconds mean two
dispatches").

The engine advances `PlaybackTime` / `PlaybackTimeUnscaled` only when the game is ticking:
`FAudioDevice::UpdateActiveSoundPlaybackTime(bool bIsGameTicking)` (`AudioDevice.cpp:4294-4312`,
called at `:4901`). In the editor world outside PIE nothing ticks, so both stay 0 and
`startWorldTimeSeconds` equals the current world time on every call.

## Observed (wt2, Linux, UE 5.8, offscreen editor, editor world, no PIE)

`SFX_FireMedium_L` played at editor world time ~12.25 s; `SFX_FireSmall_L` played ~3 s later.
The next snapshot:

```
{"soundPath":".../SFX_FireMedium_L...","playbackTimeSeconds":0,"startWorldTimeSeconds":15.265856180340052, ...}
{"soundPath":".../SFX_FireSmall_L...", "playbackTimeSeconds":0,"startWorldTimeSeconds":15.265856180340052, ...}
```

Five seconds later both rows read `startWorldTimeSeconds: 29.01476393453777`. The first snapshot,
taken right after the first play, had read `12.252577489241958` for the same sound. Two dispatches
3 s apart look simultaneous, and a single sound's "start" moves with every call.

In PIE the values are correct: two `PlaySoundAtLocation` calls issued at world time 7.397 s and
15.450 s report `startWorldTimeSeconds` 7.297 and 15.450, with `playbackTimeSeconds` advancing.

## Why it matters

The editor world is where the MCP audio play verbs land (see `B-audio-play-verbs-ignore-pie-world`),
so a caller testing with them gets the wrong timing on the path most likely to be tried first,
with nothing in the response flagging it.

## Ask

When the owning world is not game-ticking, omit `startWorldTimeSeconds` (as virtualized rows
already do) or add a per-row/top-level flag saying playback time is frozen; document that
`playbackTimeSeconds` does not advance in a non-PIE editor world.

## Severity

Silent wrong data is High by impact class; timing claims on editor-world audio are a narrow path,
so one level down: Medium.

## History
- `#1-filed` `OPEN` G16-live-tester — Found during the live check of `audio.list_active_sounds` (`F-audio-no-active-sound-enumeration`). In the editor world (no PIE) `playbackTimeSeconds` stays 0 and `startWorldTimeSeconds` tracks the current world time (12.25 -> 15.27 -> 29.01 for one looping sound across three snapshots; two sounds dispatched ~3 s apart both read 15.2659). Root cause is engine behaviour (`UpdateActiveSoundPlaybackTime(bIsGameTicking)`), but the verb presents the value as WorldTimeWhenPlayed without qualification. In PIE the timing is correct. Dedup: no other ticket mentions `startWorldTimeSeconds` or `playbackTimeSeconds`.
