---
id: B-sequencer-display-rate-invalid
title: "sequencer.set_display_rate accepts malformed or non-positive rates and writes an invalid FFrameRate"
status: OPEN
severity: Medium
category: bug
tags: [sequencer, frame-rate, validation, coercion, false-success]
encounters: 1
lastSeen: 2026-09-03T23:08:58+03:00
---

# Display-rate parsing turns malformed text into a successful invalid setting

## What happens

`sequencer.set_display_rate` parses strings ending in `fps` and rational strings
with `FCString::Atoi`, then sets `bRateFound=true` without proving either token was
numeric or that the resulting rate is valid
(`X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Sequencer\SequenceHandler.cpp:565-596`).
It writes and reports the value at `:599-606`. Consequently `"oopsfps"` becomes
`0/1`, while `"24/not-a-number"` becomes `24/0` and is still accepted.

The engine's `FFrameRate::IsValid()` requires a positive denominator
(`C:\UE_5.8\Engine\Source\Runtime\Core\Public\Misc\FrameRate.h:48-51`), but
`UMovieScene::SetDisplayRate` only assigns the value
(`C:\UE_5.8\Engine\Source\Runtime\MovieScene\Public\MovieScene.h:832-835`).

## Why it matters

The RPC returns success after storing a timeline rate that downstream frame/time
conversion and rendering code cannot interpret correctly. The malformed-input
route is uncommon and has a direct correction workaround, so the default High
false-success impact is reduced to Medium for reach.

## What should happen

Parse the whole token strictly, reject zero/negative numerator or denominator, and
require `NewRate.IsValid()` before mutation. Follow the strict lexical-validation
shape requested by `B-sequencer-tick-resolution-substring-parse`, and return the
measured `MovieScene->GetDisplayRate()` value after the write.

## Workaround

Pass a known positive integer or rational such as `30fps` or `24000/1001`, then
read the sequence rate back before relying on it.

## Related

- False-success pattern `coercion-slot-unit-direction-drift`.
- Data-loss pattern `accepted-parameter-silently-dropped` (validation sibling).
- `B-sequencer-tick-resolution-substring-parse` — strict rate-parser fix shape for a different verb.
- `E-sequencer-add-camera-track-no-transaction` — separately covers transaction gaps.

## History
- `#1-filed-pattern-scan` `OPEN` reporter — Source-only scan confirmed permissive `Atoi` parsing at `SequenceHandler.cpp:570-596`, mutation and success at `:599-606`, the engine validity rule, and the assignment-only setter. The existing tick-resolution ticket does not cover `set_display_rate`. No build, test, editor, MCP call, plugin edit, commit, or repro was performed.
