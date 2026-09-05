---
id: F-observe-blueprint-dispatcher-at-runtime
title: "No way to observe a Blueprint Event Dispatcher firing at runtime — Python cannot bind BP-declared multicast delegates, so gameplay-timed capture and probing has to poll"
status: OPEN
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
