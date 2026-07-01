---
id: E-sequencer-playback-control-bare-empty-no-state
title: "sequencer.play / pause / stop / set_playback_speed all return a bare {} — caller can't tell a real pause from a stop or no-op without reading C++"
status: OPEN
severity: Low
category: ergonomic
tags: [docs, sequencer, playback, pause, success-field, readback]
encounters: 1
lastSeen: 2026-06-23T10:08:23Z
---

# `sequencer.play` / `pause` / `stop` / `set_playback_speed` return a bare `{}` — pause-vs-no-op is indistinguishable from the wire response

All four editor-preview playback-control verbs return an **empty JSON object `{}`**
on success — the handlers call `Ctx.SendSuccess(nullptr)` with no echo of the
resulting transport state:

- `sequencer.set_playback_speed` → `SetPlaybackSpeed(...)` then `Ctx.SendSuccess(nullptr)` (`SequenceHandler.cpp:1275-1277`)
- `sequencer.pause` → `ULevelSequenceEditorBlueprintLibrary::Pause()` then `Ctx.SendSuccess(nullptr)` (`SequenceHandler.cpp:1309-1311`)
- `sequencer.stop` → `Pause()` + scrub-to-frame-0 then `Ctx.SendSuccess(nullptr)` (`SequenceHandler.cpp:1340-1353`)
- `sequencer.play` → same bare-`{}` shape

Because the success payload carries no `paused` / `playing` / `speed` / `frame`
field, the wire response alone **cannot distinguish a genuine pause from a stop,
from a silent no-op**. For a task whose whole point is "did pause actually *hold*
playback (vs. stop or do nothing)?", the `{}` answers nothing. The only way the
audited run could confirm the pause hit the real `Pause()` path was to **read the
plugin C++ (`SequenceHandler.cpp`)** and reason about the guard
(`GetCurrentLevelSequence() == LevelSeq` → real `Pause()`; otherwise an
`EXECUTION_ERROR`). That C++ read should not be the verification surface for a
clean-outcome task.

This is the playback-control sibling of the keyframe-writer gap
`E-sequence-add-keyframe-bare-empty-no-success` (frame-numbered `add_keyframe`
also returns a bare `{}`) and of the track-state read-gap
`E-sequencer-set-track-state-no-readback-doc` / `F-sequencer-track-state-readback`
(set_track_* writes are unreadable). It is distinct from both: those cover
keyframe writes and per-track mute/solo/lock; this covers the **transport verbs**
(play/pause/stop/speed), for which there is no read surface or echo at all.

There is also **no transport read-back RPC** in the namespace — `get_properties`
exposes frame rate, playback range, tick resolution and binding counts, but not
the current play head / paused flag / active playback speed — so a caller can't
even confirm the transport state via a separate read. (A transport-state reader
would be a larger `F-` feature; this ticket is the cheap docs/echo mitigation.)

## What to do

1. **Echo the resulting transport state (primary, ergonomic).** Have each verb
   return at least `{ success: true }`, ideally with a self-describing echo:
   `pause`/`play` → `{ "paused": true|false }` (or `"state": "paused"|"playing"`),
   `set_playback_speed` → `{ "speed": <applied> }`, `stop` →
   `{ "paused": true, "frame": 0 }`. Then a caller distinguishes a real pause from
   a no-op from the response, with no C++ read and no separate probe. This matches
   the board's authoring-response convention (a `success` flag plus an echo of
   what changed) that the bare `{}` currently violates.
2. **Wiki note (downstream wiki process, cheap interim).** `docs/wiki-src/sequencer.md`
   advertises "editor-preview playback" and "playback speed" in the namespace
   intro but documents none of `play` / `pause` / `stop` / `set_playback_speed`
   nor their return shapes. Add short H3 entries (or a "playback controls" note)
   stating: each returns a bare `{}` on success today (no `paused`/`speed` echo);
   the verbs only act when the target sequence is the **currently-open** one
   (otherwise `EXECUTION_ERROR` "Sequence not currently open in editor" /
   `EDITOR_NOT_OPEN`), so a non-error `{}` *is* the confirmation the genuine
   `Pause()`/`Play()`/`Stop()` path ran; and there is no transport read-back RPC,
   so callers should not budget a verification round-trip. Tighten/remove once (1)
   lands.

**Workaround (today):** treat a non-error `{}` from these verbs as success — the
handler guard (`GetCurrentLevelSequence() == LevelSeq`) means a `{}` can only come
back from the real transport call on the open sequence; any other case is an
error, not a silent `{}`.

## Evidence (this task)

Struggle-audit (PROCESS) of the `sequencer.pause` SEED task (13 calls, outcome
`clean`). The full preview pass — `play`, `pause`, `set_playback_speed 2.0`,
`play` (resume), `set_playback_speed 0.5`, `pause`, `stop` — every call
`ok=true`, no retries, no `is_error`. Friction note verbatim: "every
playback-control call (play/pause/stop/set_playback_speed) returns an empty `{}`
on success with no self-describing field (e.g. no `{"paused":true}`), so the wire
response alone can't distinguish a real pause from a stop or no-op -- I had to read
the plugin C++ (SequenceHandler.cpp) to confirm pause only returns `{}` via the
genuine Pause() path when the sequence is the currently-open one (error
otherwise). The wiki documents params but not return shapes, so confirming
'paused vs no-op' wasn't possible from the wiki/MCP responses alone." Method
discovery and execution were smooth; the cost was confirming the *meaning* of the
empty success response.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit (PROCESS) of the `sequencer.pause` SEED task (13 calls, all `ok=true`, outcome `clean`; per-finding judge filed nothing). The four transport verbs (`play`/`pause`/`stop`/`set_playback_speed`) all `Ctx.SendSuccess(nullptr)` → bare `{}` (`SequenceHandler.cpp:1275-1277,1309-1311,1340-1353`), so the wire response can't distinguish a real pause from a stop or no-op; the run had to read the C++ guard (`GetCurrentLevelSequence()==LevelSeq`) to confirm the genuine `Pause()` path ran, and `get_properties` has no transport-state field. Dedup: ripgrep across OPEN/IN-REVIEW/DONE/WONTFIX (`paused|playback.control|set_playback_speed|SendSuccess(nullptr)|genuine Pause`) — distinct from `E-sequence-add-keyframe-bare-empty-no-success` (keyframe writer's bare `{}`), `E-sequencer-set-track-state-no-readback-doc` / `F-sequencer-track-state-readback` (per-track mute/solo/lock read-gap), and `F-editor-status` (`pieIsPaused`, a PIE flag, not sequencer transport). Asks: echo `paused`/`speed`/`frame` from the verbs (primary), plus a `docs/wiki-src/sequencer.md` note that these return a bare `{}` and only act on the currently-open sequence so a non-error `{}` is the confirmation (interim).
