---
id: F-observe-blueprint-dispatcher-at-runtime
title: "No way to observe a Blueprint Event Dispatcher firing at runtime — Python cannot bind BP-declared multicast delegates, so gameplay-timed capture and probing has to poll"
status: DONE
severity: Medium
category: feature
tags: [blueprint, dispatcher, delegate, multicast-delegate, pie, capture, probe]
---

# No way to observe a Blueprint Event Dispatcher firing at runtime

A Blueprint's Event Dispatchers are the natural "this just happened" signal in a running PIE session
— a weapon's `OnFired`, a pickup's `OnCollected`, a door's `OnOpened`. There is currently no way for
a caller to be notified when one fires. `unreal` Python exposes only native `FMulticastDelegate`
properties (`on_actor_hit`, `on_destroyed`, …); a dispatcher declared in a Blueprint is not exposed
as a bindable attribute, so `actor.on_fired.add_callable(fn)` fails and the dispatcher is invisible
from script.

## Reproduction (this checkout, 2026-09-05)

`/Game/FPS/Weapons/BP_WeaponBase` declares dispatchers `OnFired`, `OnImpact`, `OnReloadStarted`,
`OnReloadFinished`, `OnAmmoChanged` (confirmed by `blueprint.inspect`, which lists all five under
`events`). In PIE, against a live `BP_Weapon_AR_C`:

    [x for x in dir(weapon) if 'fired' in x.lower() or x.startswith('on_')]
    -> ['on_actor_begin_overlap', 'on_actor_end_overlap', 'on_actor_hit', 'on_actor_touch',
        'on_actor_un_touch', 'on_become_view_target', 'on_begin_cursor_over', 'on_clicked',
        'on_destroyed', 'on_end_cursor_over']

Every entry is a native `AActor` delegate. None of the five Blueprint-declared dispatchers appear,
and `weapon.on_fired` raises `AttributeError`.

## Why it matters beyond convenience

The concrete task was photographing a muzzle flash. A flash lives for a few tens of milliseconds of
game time; a screenshot issued from a separate RPC lands wherever it lands. Three consecutive builds
of this project failed to capture one by timing the shutter against an estimate of the fire interval.
The correct instrument is "freeze/capture when the weapon says it fired", which `OnFired` is exactly
for — and it is unreachable.

The workaround that finally worked is a poll: register a slate post-tick callback, read
`AmmoInMag` every tick, and act when it decrements.

    def watch(delta):
        m = weapon.get_editor_property('AmmoInMag')
        if m < state['last']:
            unreal.GameplayStatics.set_global_time_dilation(world, 0.0002)   # freeze inside the flash
        state['last'] = m
    handle = unreal.register_slate_post_tick_callback(watch)

That worked (`b05_muzzle.png` is the first flash this project has caught), but it is a proxy: it
infers the event from a side effect, needs a per-event proxy invented each time, ticks every frame,
and cannot observe a dispatcher with no observable side effect at all — `OnImpact` and
`OnReloadStarted` have none that is cheaply pollable.

## Ask

A verb to subscribe to a Blueprint dispatcher on a live object and deliver its firings, e.g.

    blueprint.watch_dispatcher { objectPath | actorName, dispatcherName, mode: "count" | "stream" }

returning a handle, with a matching `unwatch`. Minimum useful shape is a monotonically increasing
fire count plus the timestamp of the last firing, which is enough to drive "capture on the next
firing" and to assert in a probe that an event happened N times. Delivering the dispatcher's
parameters would be better, and would also make `OnImpact`-style events (which carry the hit result)
observable.

Adjacent, and cheaper if the above is too large: a `freezeOnDispatcher` option on the capture verbs,
so a caller can say "take this screenshot on the next `OnFired`" without owning the tick loop.

## Notes

- `F-blueprint-add-dispatcher` (DONE) covers *declaring* a dispatcher on a Blueprint and the BPIR
  `bind_dispatcher` / `call_dispatcher` instructions, which operate at author time inside a graph.
  This ticket is the runtime/observer half: watching one fire from outside the Blueprint, in PIE.
- Not filed as a bug: the absence is Unreal's Python exposure rule for BP-declared delegates, not a
  PinWright regression. It is filed as a feature because PinWright is the layer that can bridge it,
  and because the gap directly cost this project three builds of missed captures.

## History
- `#1-record-dispatcher-job` `IN-REVIEW` developer — Added the job verb `blueprint.record_dispatcher {objectPath, dispatcher, count=1, timeoutSeconds=30 (max 600), maxRecords=16, includeParams=true, pauseOnFire=false}`. The ask was a watch/unwatch handle pair; this ships one bounded job instead, so a forgotten unwatch cannot leave a binding behind. The job ticket is the handle and `system.job_cancel` is the unwatch. The verb binds a `UPinWrightDispatcherRecorder` into the delegate's own invocation list. Because that recorder overrides `ProcessEvent`, it receives the broadcaster's signature-shaped parameter buffer, which works for any signature and for native dynamic multicast delegates as well as BP dispatchers. It records `{seq, elapsedSeconds, frame, worldSeconds?, params}` per broadcast, with params exported like `property.get`. It unbinds on every exit (`count_reached`, `timed_out`, `target_destroyed`, cancel) and reports `bindingRemoved` as read back from the invocation list. `pauseOnFire` pauses PIE inside the broadcast that completes the count, using the same levers as `editor.pause` (resume with `editor.resume`), which is the shutter for the muzzle-flash case. It is refused with `NO_ACTIVE_SESSION` when the object is not in the PIE world. An unknown dispatcher is refused with the new `DISPATCHER_NOT_FOUND`, whose `available[]` lists the delegates the class has. Not done: a separate `freezeOnDispatcher` option on the capture verbs (`pauseOnFire` + a normal capture covers it), and a streaming per-fire progress frame. Files: `Source/PinWright/Private/Handlers/Blueprint/DispatcherRecorder.h` (new UCLASS), `Handlers/Blueprint/BlueprintRecordDispatcherHandler.cpp` (new), `Handlers/ErrorCodes.h` (+`ERR_DISPATCHER_NOT_FOUND`), `Tests/Blueprint/TestBlueprintRecordDispatcher.cpp` (new), `docs/wiki-src/blueprint.md` (`### blueprint.record_dispatcher`), `CHANGELOG.md`. Tests (`PinWright.blueprint.record_dispatcher.`): `CountReachedRecordsParamsAndUnbinds`, `CancelUnbinds`, `TimeoutCompletesUnmetAndUnbinds`, `RefusalsNameTheWayOut`, `PauseOnFireFreezesPie` (owned host-neutral PIE). Each test reads the invocation list directly and asserts the binding is present while the job runs before it asserts the binding is gone afterwards. Verify: run `PinWright.blueprint.record_dispatcher` offscreen. Live check: in PIE on a weapon BP, call `blueprint.record_dispatcher {objectPath:<PIE weapon>, dispatcher:"OnFired", pauseOnFire:true, wait:false}`, fire, poll `system.job_status`, then screenshot.
- `#2-review-fixes` `IN-REVIEW` developer — Applied the independent review. Blocker: `pauseOnFire` no longer pauses or freezes when the PIE world is tearing down or has not begun play. Before this, a dispatcher firing from `EndPlay`/`OnDestroyed` after EndPIE could leave the Slate fixed delta, motion blur and widget ticks stuck after the session. In that case `paused` reads `false`. Should-fix changes: unbind and the read-back now see through a garbage but not yet collected target (`Get(true)`), so `bindingRemoved` is measured. A class default object or archetype target is refused with `INVALID_PARAMS`. The single-PIE-world gate is now stated in the error message, the parameter description and the wiki. Nits: the result reports `uiFrozen`; broadcasts from off the game thread are ignored; the wiki states that `timeoutSeconds` is wall-clock time and runs while paused, and that copies made during the job carry the binding. New tests: `PinWright.blueprint.record_dispatcher.TargetDestroyedCompletesAndUnbinds` (MarkAsGarbage, latent; checks `target_destroyed` and that no binding is left on the garbage object) and `PinWright.blueprint.record_dispatcher.SparseNativeDelegateRecordsAndUnbinds` (`USceneComponent::OnComponentActivated.Broadcast`; checks the params and that the sparse delegate is unbound). Fixed the comment on the late-broadcast test step.
- `#3-review-fixes` `IN-REVIEW` developer — Applied the re-review. Blocker: after a recompile, unbind used the recompiled class's property offset on the old instance. A cached `TWeakFieldPtr` re-resolves by name to the NEW class's `OnFoo`, while the old instance sits on a `REINST_` class that keeps the old layout. The cached property is gone. Bind, unbind, the read-back and the params export now resolve `FindFProperty(Target->GetClass(), dispatcher)` on each use. `params` is skipped when the target is no longer live. The ticker exits on `!Target.IsValid()` alone. Should-fix: added a regression test for the teardown pause, and the wiki now says a copy's broadcasts are recorded as if the original fired, its binding is not removed, and it goes inert when the job ends. Nits: the CDO refusal is now tested. `pauseOnFire` also skips when a PIE end is queued (`GEditor->ShouldEndPlayMap()`), which closes the EndPIE/OnPIEEnded window before `bIsTearingDown` for the queued path; a direct `EndPlayMap()` call still bypasses that flag. README row and error-catalog entry not done. Tests: new `PinWright.blueprint.record_dispatcher.LayoutShiftingRecompileUnbindsOldInstance` (fixture int ahead of `OnFoo`, `blueprint.remove_variable` recompiles, latent wait, asserts `target_destroyed` and zero bindings read through the old instance's own class, with offset-shift and reinstancing preconditions); `PauseOnFireFreezesPie` extended (tearing-down and end-queued broadcasts: no pause, no freeze, `paused:false`); `RefusalsNameTheWayOut` extended (CDO → `INVALID_PARAMS`).
- `#4-review-fixes` `IN-REVIEW` developer — Re-review blocker: an object path that resolved to a destroyed object (garbage, not yet collected) was accepted. The job's weak pointer then read null and the bind dereferenced a null property after "started" was sent, which crashed the editor. Such an object is now refused with `OBJECT_NOT_FOUND`, and the bind null-checks the property and finishes the job with `target_destroyed`. `RefusalsNameTheWayOut` now covers the garbage-instance refusal, including that no binding is left. Wiki: records from a copy carry no `params` once the original is gone, and the `OBJECT_NOT_FOUND` refusal is documented.
- `#5-verified-linux` `DONE` tester — Fix commit 5256da40. All eight `PinWright.blueprint.record_dispatcher.*` tests passed non-skipped in run3/full: `CountReachedRecordsParamsAndUnbinds` (a Blueprint-declared dispatcher, added with blueprint.add_dispatcher, broadcast via ProcessMulticastDelegate; records seq, time, frame and params; unbinds), `CancelUnbinds`, `TimeoutCompletesUnmetAndUnbinds`, `TargetDestroyedCompletesAndUnbinds`, `LayoutShiftingRecompileUnbindsOldInstance`, `SparseNativeDelegateRecordsAndUnbinds`, `RefusalsNameTheWayOut` and `PauseOnFireFreezesPie` (owned PIE). Acceptance: subscribing to a BP dispatcher on a live object works. The job ticket is the handle and `system.job_cancel` the unwatch. The ask's minimum (fire count plus last-fire timestamp) and its preferred params are delivered. "Capture on the next firing" is `pauseOnFire` plus a normal capture, which also covers the adjacent freezeOnDispatcher ask. Limits: a bounded job (`count`, `timeoutSeconds` max 600), not an open-ended watch, and no streaming mode; a single PIE world only; the reporter's live weapon `OnFired` muzzle-flash check was not run (no such BP on this host).
