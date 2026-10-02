---
id: E-drive-target-match-no-widget-name
title: "drive.expect / drive.wait_for target matches only the whole handle or the whole visible label — a widget name like BT_CreateRoom never matches, so the caller must know the localized label"
status: WONTFIX
severity: Low
category: ergonomic
tags: [drive, drive.expect, drive.wait_for, target, handle, widget-name, localization, matching]
encounters: 2
lastSeen: 2026-09-29T12:21:45Z
---

# `target` accepts a full handle or a full label, never a widget name

`FDriveConditionEval::ElementMatchesTarget`
(`Plugins/PinWright/Source/PinWright/Private/Handlers/Drive/DriveConditionEval.cpp:80-93`)
matches an element two ways only:

```cpp
if (!Element.Handle.IsEmpty() && Element.Handle.Equals(Target, ESearchCase::CaseSensitive))  return true;
if (!Element.Label.IsEmpty()  && Element.Label.Equals(Target, ESearchCase::IgnoreCase))      return true;
```

Both are **whole-string equality**. Every condition type routes through it — `FindMatch` /
`CountMatches` at `:95-119`, consumed by `drive.expect` (`DriveConditionEval.cpp:130-200`),
`drive.wait_for` (`DriveWaitHandler.cpp:105`) and the web variant
(`DriveWebHandlers.cpp:248`) — so the limitation is uniform across the assertion surface.

The consequence in practice: the designer-facing widget **name** does not work as a target.
In a real-click PIE run on `W_CreateMultiplayerRoom` (UE 5.8, `X:\src\unreal\unreal-fpv-dev`,
plugin `fa755a4f`), `BT_CreateRoom` and `BT_ResetRoomName` never matched anything, because the
reported leaf is the inner `SCommonButton` whose handle is the ancestor chain **containing**
that name as one segment (`ComputeHandleBaseKey`, `DriveLiveResolver.cpp:377-395`: the key is
the chain of named ancestors plus the leaf's Slate type, e.g.
`WBP_RootLayout_C_0/.../BT_CreateRoom/SCommonButton`), and a segment is not the whole string.

So the caller is pushed onto the **visible label** — which on this project is localized
Russian text. Asserting on a UI string means the expectation breaks on any copy edit or
language switch, and the agent has to first learn the exact glyphs (an extra `drive.observe`,
whose element list is the 200-450 KB spill described in
`E-drive-observe-element-list-no-projection-spills`).

## What it should do

Accept a widget name as a third target form: match `Target` against the element's own backing
`UWidget` name, or equivalently against any single `/`-separated segment of `Handle`
(`GetLeafName` / the anchor chain already exist in `DriveLiveResolver.cpp:377-395`). A name is
stable across localization and copy edits, and it is what the caller reads off the Blueprint,
so it is the natural thing to type.

Two design points for whoever implements it:
- Keep whole-handle and whole-label matching first, so no existing expectation changes meaning;
  name matching is an additional fallback.
- A name segment can repeat across sibling widgets, so the count-based conditions
  (`CountMatches`) may match more than one element — either document that or reuse the same
  `[k]` disambiguator handles already carry.

Document the accepted target forms on `docs/wiki-src/drive.md`, which today does not state that
matching is whole-string.

**Workaround:** take the exact `handle` from a preceding `drive.observe` and pass that, or pass
the exact localized label.

severity rationale: impact = pure friction; the expectation is reachable via the handle the
caller can read off `drive.observe`, nothing is wrong or hidden -> Low. Reach = every drive
assertion goes through this matcher, which argues for a bump to Medium, but the workaround is
one field of a call the caller has usually already made, so it stays Low.

## History
- `#1-widget-name-never-matches` `OPEN` reporter — Filed from a real-click PIE verification (UE 5.8, `X:\src\unreal\unreal-fpv-dev`, plugin `fa755a4f`, `L_Core` -> `W_Multiplayer` -> `W_CreateMultiplayerRoom`). `drive.expect` / `drive.wait_for` targets named `BT_CreateRoom` and `BT_ResetRoomName` matched nothing; `ElementMatchesTarget` (`DriveConditionEval.cpp:80-93`) does whole-string handle (case-sensitive) or whole-string label (case-insensitive) equality only, and the widget name appears merely as one segment of the composed handle (`ComputeHandleBaseKey`, `DriveLiveResolver.cpp:377-395`). Callers are therefore forced onto the localized Russian label, which is unstable across copy edits and costs an extra spilling `drive.observe` to learn. Ask: accept a widget name / handle segment as a third target form, after the existing exact matches, and document the accepted forms in `docs/wiki-src/drive.md`.
- `#2-count-condition-segment-timeout` `OPEN` reporter - Second encounter, UE 5.8 Linux, host `/sdb-disk/src/unreal/unreal-fpv-wt2`, plugin clone `61c243f5`, PIE on `L_Core`. `drive.click {handle:"W_OverallUILayout_C_0/W_TrackSelect_C_0/BT_ALLTracks/SCommonButton", os_input:true, instance_name:"W_OverallUILayout", wait_for:{type:"count", target:"TrackListItem2", expected_count:8, count_op:"gte"}, timeout_ms:10000}` returned `outcome:"timeout"`, `condition_met:false`, `elapsed_ms:10028`, although the click worked and the list showed four `W_TrackListItem2_C_n` rows (handles `W_OverallUILayout_C_0/W_TrackSelect_C_0/W_TrackListItem2_C_2/SCommonButton` and so on). I expected a segment or substring match; the target is a handle segment, and `ElementMatchesTarget` only does whole-string equality, so the count was 0. Two things made this slower to diagnose than it needed to be: the generated `drive.md` says a target is "matched against element handle/label/type", but type is not matched at all and matching is whole-string; and the action verb's timeout response carries no `actual`/`detail` (`count=0`, "0 element(s) match target") even though `FDriveConditionEval` computes it, so the caller cannot tell a wrong target from a slow UI. Cheap: one 10 s timeout, then `drive.observe` plus a filter script.
- `#3-stale-sweep-stale` `WONTFIX` developer — Fixed by plugin `ec169d19` ("drive: widget_absent no longer passes on a target that never matched"): `ElementMatchesTarget` now also matches a whole `/`-delimited run of the handle's segments, so `BT_CreateRoom` and `TrackListItem2`-style widget names match (`DriveConditionEval.cpp:100-104`, `DriveHandleHasSubPath`), and `docs/wiki-src/drive.md:47` documents the three target forms. The `#2` side note (an action verb's `wait_for` timeout carries no `actual`/`detail`) is still true at `DriveActionCommon.cpp:159-168` and is reported to the triage manager for a separate ticket.
