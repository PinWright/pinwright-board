---
id: B-sequencer-tick-resolution-substring-parse
title: "sequencer.set_tick_resolution substring-matches '24000'/'60000' in the resolution arg — 240000 / 600000 silently clamp to 24000 / 60000 (10x error), no error, no string workaround"
status: IN-REVIEW
severity: Medium
category: bug
tags: [substring-parse-wrong-value, sequencer, set_tick_resolution, silent-wrong-data, tick-resolution]
encounters: 1
lastSeen: 2026-07-11T11:48:18.1474738+03:00
claimedBy: fuzz2
claimedAt: 2026-07-11T22:34:04.5806110+03:00
---

# `sequencer.set_tick_resolution` clamps any resolution containing the substring "24000"/"60000" to 24000/60000

The handler parses the `resolution` string with loose `Contains` substring fast-paths
BEFORE the rational and numeric branches. Any value whose decimal string *contains*
`24000` or `60000` as a substring is silently clamped to `24000/1` or `60000/1` — a
10x (or worse) error — with a bare `{}` success and no error. Because the two substring
checks run first, there is also **no string workaround**: the rational form (e.g.
`"240000/1"`) contains `24000` too, so it hits the same fast-path.

Legitimate high-precision film tick resolutions that trip this:

- `240000` -> clamped to `24000/1` (10x low)
- `600000` -> clamped to `60000/1` (10x low)
- also `124000`, `160000`, `240000/1`, `600000/1`, `1024000`, etc. — anything containing the digit runs.

Exact `24000` / `60000` and unrelated values (e.g. `48000`, `120000`) parse correctly,
which is why the bug hides: the common defaults work, only a caller asking for a value
that happens to embed those digit runs gets the silent downgrade. The caller "trusts a
lie" — the write reports success, and only a separate `get_properties` read reveals the
wrong value (the setter itself returns a bare `{}` with no echo, so nothing in the
response flags it).

## What it should do

Parse the resolution to a number/rational first, then compare — do not `Contains`-match
digit substrings. Accept an exact numeric string (`FCString::Atoi` / `Atod`) or the
`num/den` rational form and build `FFrameRate` from the parsed value; reject or echo the
applied value so a mismatch is visible. (Sibling `sequencer.set_display_rate` already
parses cleanly — `EndsWith("fps")` / `Contains("/")` split / `IsNumeric` — and even
echoes `displayRate` in its response; `set_tick_resolution` is the outlier on both the
parse bug and the missing echo.)

## Guilty source line

`Plugins/PinWright/Source/PinWright/Private/Handlers/Sequencer/SequenceHandler.cpp:2302-2306`:

```
if (ResolutionStr.Contains(TEXT("24000")))
    TickResolution = FFrameRate(24000, 1);
else if (ResolutionStr.Contains(TEXT("60000")))
    TickResolution = FFrameRate(60000, 1);
else if (ResolutionStr.Contains(TEXT("/")))
```

The `Contains` substring tests (should be exact/parse-then-compare) precede the `/`
rational branch, so both `"240000"` and `"240000/1"` never reach a correct parse.

## Verbatim repro (replay against mcp__pinwright__call, HEAD)

1. `sequencer.create` args `{name:"OracleTickProbe", path:"/Game"}` -> `/Game/OracleTickProbe`.
2. `sequencer.set_tick_resolution` args `{path:"/Game/OracleTickProbe", resolution:"240000"}` -> `{}` (bare success).
   `sequencer.get_properties` -> `tickResolution:{numerator:24000,denominator:1}`  (asked 240000, got 24000).
3. `sequencer.set_tick_resolution` args `{path:"/Game/OracleTickProbe", resolution:"600000"}` -> `{}`.
   `sequencer.get_properties` -> `tickResolution:{numerator:60000,denominator:1}`  (asked 600000, got 60000).
4. Control `sequencer.set_tick_resolution` args `{resolution:"48000"}` -> `get_properties` `tickResolution:{numerator:48000,denominator:1}`  (correct).
5. No-workaround check `sequencer.set_tick_resolution` args `{resolution:"240000/1"}` -> `{}`;
   `get_properties` -> `tickResolution:{numerator:24000,denominator:1}`  (rational form also clamped — no string reaches 240000).

severity rationale: impact=silent-wrong-data (caller trusts a 10x-off value, no in-RPC workaround) = High-class x reach=rare (only resolution strings embedding the 24000/60000 digit runs, on the cinematics tick-resolution setter, not every-session) -> Medium.

## History
- `#2-in-review` `IN-REVIEW` developer — GO. Independently confirmed the substring-parse defect present in synced source (`SequenceHandler.cpp:2355-2358`; the cited `:2302-2306` drifted but is byte-identical). Decision: parse-first fix — drop the `Contains("24000")`/`Contains("60000")` fast-paths so resolution strings flow through the existing rational/numeric branches (which already parse `240000`/`600000`/`240000/1` correctly), reject a non-empty unparseable resolution with `INVALID_ARGUMENT`, and echo the applied `tickResolution` (mirroring sibling `set_display_rate`/`get_properties`) so the write is verifiable rather than a bare `{}`. Severity Medium unchanged; single-method scope, no follow-ons. Adopting the red test `PinWright.Sequencer.SetTickResolution.NumericStringNotSubstringClamped` as the regression gate (strengthened to also assert the echo). Compile + test verification to follow.
- `#1-initial-repro` `OPEN` reporter — Found via SEED task on `sequencer.set_tick_resolution` (film-precision IntroCutscene; the task used `resolution=60000`, an exact match, so it worked and the task passed clean). Source read of `SequenceHandler.cpp:2302-2306` showed the `Contains("24000")`/`Contains("60000")` substring fast-paths precede the rational/numeric branches. Replay-confirmed on HEAD: `240000`->24000/1, `600000`->60000/1, `240000/1`->24000/1 (rational escape also broken), control `48000`->48000/1 correct. Silent wrong data with a bare `{}` and no echo. Dedup: ripgrep across the board — only `E-rpc-sequencer-extend-get-properties` (DONE) names `set_tick_resolution` (as the existing setter whose readback it added); no ticket covers this parse bug. Sibling `set_display_rate` parses cleanly and echoes, so the defect is confined to this one method (single-method, not a family).
