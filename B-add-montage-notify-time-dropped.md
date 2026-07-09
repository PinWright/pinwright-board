---
id: B-add-montage-notify-time-dropped
title: "animation.authoring.add_montage_notify / add_notify / add_notify_state silently drop the requested time — notify always lands at t=0"
status: IN-REVIEW
severity: High
category: bug
tags: [animation, anim-montage, anim-notify, anim-notify-state, authoring, silent-noop]
encounters: 2
lastSeen: 2026-07-09T11:50:58.1866245+03:00
---

# animation.authoring.add_montage_notify (and add_notify) silently drop the requested time

`animation.authoring.add_montage_notify` accepts a `time` parameter ("Trigger time (default 0)") and returns `success: true`, but the requested time is never applied to the notify's timeline position. The notify is always created at t=0 regardless of the `time` value passed. The call is a silent success-with-wrong-effect: nothing errors, but the notify does not land where the caller asked.

Root cause is a write/read field mismatch on `FAnimNotifyEvent`:

- The writer `AnimationAuthoringHandler_Sequence.cpp:1365` sets only `NotifyEvent.TriggerTimeOffset = Time;`. It never calls `NotifyEvent.SetTime(Time)` / `NotifyEvent.Link(...)`, so the notify's `LinkValue` (the actual timeline position) stays 0.
- The reader `animation.authoring.list_notifies` reports each notify's time via `Event.GetTime()` (`AnimSequenceDumpBuilder.cpp:101`), and `FAnimNotifyEvent::GetTime()` returns the `FAnimLinkableElement` link time (`LinkValue`) — NOT `TriggerTimeOffset`. `TriggerTimeOffset` is a small runtime fudge offset, not the notify's display/trigger position; it's the wrong field to use for "where the notify sits on the montage."

So every notify added through this verb reports `time: 0` from `list_notifies`, the `asset.dump` `notifies[]` sidecar, and the editor's notify panel — the requested time is lost.

The sibling verb `animation.authoring.add_notify` (`AnimationAuthoringHandler_Sequence.cpp:643`) has the identical defect (sets only `TriggerTimeOffset`). Notably, the non-authoring `animation.add_notify` handler in `AnimationHandler.cpp` does it CORRECTLY — `NewEvent.Link(AnimSeq, (float)Time); NewEvent.TriggerTimeOffset = GetTriggerTimeOffsetForType(...);` — so the correct pattern already exists in this same plugin.

**Repro (live, replay-confirmed):**
1. `asset.search { query: "Skeleton", classFilter: ["Skeleton"], limit: 5 }` → pick `/Game/ExampleContent/IKRig/Mesh/DinoDragon/SK_DinoDragon_Skeleton`.
2. `animation.authoring.create_montage { name: "MTG_ReplayNotifyBug", path: "/Game/Animations", skeletonPath: "/Game/ExampleContent/IKRig/Mesh/DinoDragon/SK_DinoDragon_Skeleton", slotName: "UpperBody" }` → `success: true`.
3. `animation.authoring.add_montage_notify { assetPath: "/Game/Animations/MTG_ReplayNotifyBug", time: 0.5, notifyName: "ImpactFX" }` → `{"success":true,"message":"Montage notify added", ...}`.
4. `animation.authoring.list_notifies { assetPath: "/Game/Animations/MTG_ReplayNotifyBug" }` → `{"success":true, ... "notifies":[{"name":"ImpactFX","time":0,"duration":0,"branchingPoint":false}]}`.

Requested `time: 0.5`, reported `time: 0`. There is no way for a caller to place a notify at a non-zero time through this verb; the documented `time` parameter is effectively inert. (Workaround the attempt agent found: bypass the verb and rewrite the whole `Notifies` array via `property.set` with an explicit `LinkValue` — awkward and undiscoverable.)

**Fix:** In `add_montage_notify` (and `add_notify`) replace `NotifyEvent.TriggerTimeOffset = Time;` with the canonical link-then-offset pair used by the engine montage panel and by the existing `animation.add_notify` handler:
```cpp
NotifyEvent.Link(Montage, Time);  // or SetTime(Time); sets LinkValue → GetTime() == Time
NotifyEvent.TriggerTimeOffset = GetTriggerTimeOffsetForType(Montage->CalculateOffsetForNotify(NotifyEvent.GetTime()));
```
Then `RefreshCacheData()` as today. After the fix, `list_notifies`/dump should report `time: 0.5` for the repro above.

## Affected-methods update — `add_notify_state` sibling MISSED by the #2-fix

The `#2-fix` corrected the two POINT-notify writers (`add_montage_notify`, `add_notify`), which now route through `AnimationAuthoringHelpers::LinkNotifyAtTime` (it calls `Event.Link(Asset, Time)` so `GetTime()==Time`). But the notify-STATE sibling `animation.authoring.add_notify_state` has the IDENTICAL root cause and was NOT touched — it still silently drops its `startFrame`. At HEAD the writer `AnimationAuthoringHandler_Sequence.cpp:730` sets only:

```cpp
NotifyEvent.TriggerTimeOffset = StartTime;
```

It never calls `Link()`/`SetTime()`, so `LinkValue` stays 0 and the reader `GetTime()` (`AnimSequenceDumpBuilder.cpp:119`, `Entry.Time = Event.GetTime();`) reports 0. The duration IS applied correctly via `SetDuration(EndTime - StartTime)` at line 731 — only the START is lost. The fix must extend to the notify-state start (Link the start; keep `SetDuration` for the end).

Affected methods (family): `animation.authoring.add_montage_notify` (fixed), `animation.authoring.add_notify` (fixed), `animation.authoring.add_notify_state` (STILL BROKEN at HEAD).

**Repro (live, replay-confirmed at HEAD):**
1. `animation.authoring.add_notify_state { assetPath: "/Game/Characters/Mannequins/Animations/Manny/MM_Run_Fwd", notifyName: "ReplayProbe_Dust", startFrame: 6, endFrame: 12 }` -> `{"success":true,"message":"Notify state added", ...}`.
2. `animation.authoring.list_notifies { assetPath: "/Game/Characters/Mannequins/Animations/Manny/MM_Run_Fwd" }` -> the probe entry reads `{"name":"ReplayProbe_Dust","time":0,"duration":0.2, ...}`.

Requested `startFrame: 6` (=0.2s on the 30fps clip), reported `time: 0`. The sibling point notifies placed via the fixed `add_notify` in the same clip (`Footstep_L` @ 0.2s, `Footstep_R` @ 1.167s) landed correctly, isolating the defect to the notify-STATE path. There is no reposition/set-notify-time verb to work around it.

## History
- `#1-initial-repro` `OPEN` reporter — Replay-confirmed live: `add_montage_notify` with `time: 0.5` returns `success: true` but `list_notifies` reports `time: 0` (requested time silently dropped). Root cause: writer sets only `FAnimNotifyEvent::TriggerTimeOffset` (`AnimationAuthoringHandler_Sequence.cpp:1365`), while reader uses `GetTime()` = `LinkValue` (`AnimSequenceDumpBuilder.cpp:101`); `LinkValue` is never set so the notify stays at t=0. Same defect in `animation.authoring.add_notify` (line 643). Correct pattern (`Link()` + `GetTriggerTimeOffsetForType(CalculateOffsetForNotify())`) is already used by the engine montage panel and the non-authoring `animation.add_notify` handler. Repro asset `/Game/Animations/MTG_ReplayNotifyBug` on skeleton `SK_DinoDragon_Skeleton`.
- `#2-fix` `IN-REVIEW` developer — Defect confirmed live at HEAD (plugin clone in sync with origin/master `0f992a3`; not already-fixed). Fixed both writers in `Handlers/Animation/AnimationAuthoringHandler_Sequence.cpp`: `add_montage_notify` (was :1374) and the sibling `add_notify` (:643) now call `NotifyEvent.Link(Montage/AnimAsset, Time)` to set the `FAnimLinkableElement` `LinkValue` (so `GetTime()==Time`, which is what the reader at `AnimSequenceDumpBuilder.cpp:101` returns) and set `TriggerTimeOffset = GetTriggerTimeOffsetForType(EAnimEventTriggerOffsets::OffsetBefore)` — the exact canonical pattern the non-authoring `animation.add_notify` (`AnimationHandler.cpp:1173-1175`) already uses (used the in-repo constant-offset form rather than the more elaborate `CalculateOffsetForNotify`, which is `WITH_EDITOR`-gated; the offset is a minor fudge, `Link()` setting `LinkValue` is the load-bearing fix). Regression test added: `animation.authoring.add_montage_notify.AppliesRequestedTime` in `Tests/Gameplay/TestAnimationHandlers.cpp` — drives the real handler with `time: 0.5` on a created montage, then asserts both the live notify's `GetTime()` and the production `list_notifies` reader report 0.5 (pre-fix both were 0). Not compiled/run here (later phase).
- `#3-additional-add-notify-state` `IN-REVIEW` reporter — Additional evidence (new angle): the #2-fix MISSED the notify-STATE sibling `animation.authoring.add_notify_state`, which shares the identical root cause and still silently drops `startFrame`. Replay-confirmed live at HEAD: `add_notify_state { startFrame: 6, endFrame: 12, notifyName: "ReplayProbe_Dust" }` on `/Game/Characters/Mannequins/Animations/Manny/MM_Run_Fwd` returns `success:true` but `list_notifies` reports the state at `time: 0` (duration 0.2 IS correct). Guilty line `AnimationAuthoringHandler_Sequence.cpp:730` sets only `NotifyEvent.TriggerTimeOffset = StartTime;` — never `Link()`/`SetTime()` — so `LinkValue`/`GetTime()` stay 0, while the two now-fixed point verbs route through `LinkNotifyAtTime`. Fix must extend to the notify-state start (Link the start; keep `SetDuration` for the end). Added `add_notify_state` to the affected-methods list; bumped encounters 1->2.
