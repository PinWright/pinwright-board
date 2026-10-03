---
id: B-sequencer-display-rate-invalid
title: "sequencer.set_display_rate accepts malformed or non-positive rates and writes an invalid FFrameRate"
status: DONE
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
- `#2-in-review` `IN-REVIEW` developer — Still reproducible on HEAD before the fix (source: `FCString::Atoi` on every token, `bRateFound=true` without validation). Additionally found the slot was declared `number`, so over the wire the param gate refused the documented `'30fps'`/`'24000/1001'` forms while still passing `0`, `-24`, `"0"` and `29.97` (rounded to 30) to the write. Fix in `Handlers/Sequencer/SequenceHandler.cpp`: new `SequenceHandlerDisplayRate::TryParse` — JSON integer or string `N`, `Nfps`, `N/D`, `N/Dfps`, every token through `TryParseStrictJsonInteger` with range [1, int32 max], plus `FFrameRate::IsValid()`; any failure refuses `INVALID_ARGUMENT` before `SetDisplayRate`. Slot retyped `string` (gate's string accepts numbers too). `displayRate` echo now reads `MovieScene->GetDisplayRate()` after the write. Behaviour change: fractional numbers refused instead of rounded (CHANGELOG). Tests (dispatcher-routed, param gate included, read the stored rate off the MovieScene): `PinWright.sequencer.set_display_rate.MalformedRateRefusedBeforeWrite` (15 bad values incl. `"oopsfps"`, `"24/not-a-number"`, `"24/0"`, `0`, `-24`, `29.97`, `true` -> INVALID_ARGUMENT, rate stays 25/1; fails pre-fix on the numeric cases and on the gate's error code) and `PinWright.sequencer.set_display_rate.DocumentedFormsWriteExactRate` (`"30fps"`, `"24000/1001"`, `"48"`, `60` stored exactly + echo; fails pre-fix because the gate refused the string forms). Files: `Source/PinWright/Private/Handlers/Sequencer/SequenceHandler.cpp`, `Source/PinWright/Private/Tests/Sequencer/TestSequencerSetDisplayRateStrictParse.cpp` (new), `docs/wiki-src/sequencer.md` (new `### sequencer.set_display_rate`), `CHANGELOG.md`. Sibling coercion in `sequencer.set_properties` filed separately as `B-sequencer-set-properties-frame-rate-coerced`. fastcheck clean on both TUs; not yet run in the editor.
- `#3-verified-linux` `DONE` tester — Fix commit de96be3e. Passed non-skipped in run3/full: `PinWright.sequencer.set_display_rate.MalformedRateRefusedBeforeWrite` (15 bad values, including `"oopsfps"`, `"24/not-a-number"`, `"24/0"`, `0`, `-24`, `29.97` and `true`, each `INVALID_ARGUMENT` through the dispatcher, with the stored rate unchanged at 25/1) and `PinWright.sequencer.set_display_rate.DocumentedFormsWriteExactRate` (`"30fps"`, `"24000/1001"`, `"48"` and `60` stored exactly and echoed from `MovieScene->GetDisplayRate()`), plus the existing `ValidParamsNoCrash`/`ValidPathResponds`. Every "What should happen" item is met: strict whole-token parse, zero or negative values refused, `IsValid()` checked before the mutation, and the echo measured. Behaviour change (fractional rates refused, not rounded) is in the CHANGELOG.
