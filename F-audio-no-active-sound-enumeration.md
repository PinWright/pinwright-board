---
id: F-audio-no-active-sound-enumeration
title: "No verb can observe an active sound — PlaySoundAtLocation creates an FActiveSound and no UAudioComponent, so find_objects_by_class returns 0 forever and no runtime audio claim is checkable"
status: OPEN
severity: Medium
category: feature
tags: [audio, runtime, active-sound, FActiveSound, audio-device, find_objects_by_class, observability, verification, weapons]
encounters: 1
lastSeen: 2026-09-06T00:00:00Z
---

# The published surface can inspect every audio asset and no audio playback

Every audio verb on the board reads or writes **assets** — Sound Cues, MetaSounds, attenuation,
concurrency, sound classes. Nothing reads the **audio device**. So a whole class of runtime claim
about a game that is actually making noise is unverifiable through MCP.

## Why the obvious route returns zero and always will

`UGameplayStatics::PlaySoundAtLocation` — which this project's weapon uses for fire, dry-fire,
reload and impact — creates an `FActiveSound` inside the audio device and **does not create a
`UAudioComponent`**. The component-based discovery route therefore cannot see it:

```
system.inspect.find_objects_by_class {"className": "AudioComponent"}   ->  0 instances
```

That zero is correct and permanent. It is returned while sounds are audibly playing, and no
argument, world scope or filter changes it, because there is no UObject to enumerate — the playing
sound lives in the device's active-sound list, not in the object graph. A caller who does not know
the engine's split reads the zero as "nothing is playing", which is the worst shape the answer can
take: a clean, well-formed, confident lie.

## The claims this blocks — concrete, from this round

- **Did the dry-fire click fire once or twice?** A double-dispatch on an empty magazine is audible
  and is exactly the kind of defect a review is meant to catch. There is no way to count dispatches.
- **Did the impact sound dispatch per surface type?** The weapon selects an impact cue by surface;
  confirming the selection means observing which asset actually played, at which world position.
- **Did the reload sound get cut off by the next state transition?** Requires a start time and a
  still-playing flag.
- **Is a sound virtualised rather than audible?** A sound past the concurrency or distance limit is
  "playing" in every sense the game can see and inaudible to the player. Nothing distinguishes them.

Each of these is answerable by looking at the device for one frame, and none is answerable today.

## What is asked for

A **read-only** verb enumerating the audio device's active sounds. Minimum row:

- `soundPath` — the asset actually playing (the thing a per-surface or per-state claim is about)
- `location` — world position, so a dispatch can be tied to the hit it came from
- `startTime` — so overlaps, cut-offs and double-dispatches are visible as timing rather than counts
- `virtualized` — audible vs virtualised, the distinction concurrency limits create and nothing else
  reports

Useful next, in rough value order: `owningComponent` (null for the `PlaySoundAtLocation` case, which
is itself the signal), the concurrency group and whether the sound is being limited by it, current
volume/pitch multipliers, and the attenuation settings resolved for this instance.

Read-only is the whole scope. No stop, no mute, no fade — those are separate asks, and keeping this
one observational keeps it cheap and keeps it safe to call during PIE.

Note this is not a discovery-verb gap that a filter fix could close. `system.inspect.*` enumerates
UObjects; `FActiveSound` is not one. The device has to be asked directly.

## Severity

**Medium**, the hard-blocker-with-no-workaround band, adjusted down for reach. There is no
workaround at all — not a slower one, not an uglier one: the object simply is not in the object
graph, so no combination of published verbs answers the question, and `python.execute` reaching into
`FAudioDevice` is not a published surface either. That argues High. Reach pulls it back to Medium:
runtime audio verification is not an every-session path, and a project that routes all its audio
through `UAudioComponent` (spawned via `audio.create_audio_component` / `audio.spawn_sound_at_location`)
can already inspect components today. It is projects using the engine's ordinary fire-and-forget
call — the documented, recommended one for one-shot SFX — that have nothing.

## Related

- `E-spawned-audio-component-not-actor-readable` (IN-REVIEW, docs) — the **component** case: audio
  components the MCP audio verbs spawn are parented to WorldSettings and are readable via the
  returned `componentPath`. That path exists precisely because those verbs create a component;
  `PlaySoundAtLocation` creates none, so nothing in that ticket's steer applies here.
- `B-play-sound-attached-not-attached` (WONTFIX) — its whole disposition turns on reading a
  component's `AttachParent` **after playback finished**, and the developer's closing argument is
  about auto-destroy timing. A verb that reported the live active sound at the moment of the call
  would have settled that ticket from observation instead of from engine source, which is a second
  argument for this capability.
- `F-metasound-no-render-to-pcm` (OPEN) — the offline analogue: render a MetaSound to samples and
  measure them. Complementary, not overlapping: that measures what an asset *would* sound like, this
  reports what the device is *currently* doing.
- `F-sound-concurrency-asset-authoring` (OPEN) — authors the concurrency rules whose effect
  (`virtualized`) this verb would make observable. Neither implies the other; together they close a
  loop.
- `B-get-components-cannot-resolve-worldsettings` — filed the same round, the other case where a
  component-parented-to-WorldSettings read dead-ends; different mechanism (a resolvable actor
  excluded from a candidate set) but the same practical shape, that the object route is not
  available.

## History
- `#1-filed` `OPEN` WEAPONS-critic — Filed during a WEAPONS critic review round 4. `UGameplayStatics::PlaySoundAtLocation` — the call this project's weapon uses for fire, dry-fire, reload and impact — creates an `FActiveSound` in the audio device and no `UAudioComponent`, so `system.inspect.find_objects_by_class {className:"AudioComponent"}` returns 0 instances while sounds are audibly playing, permanently and for every argument combination, because there is no UObject to enumerate. The zero is well-formed and confident and reads as "nothing is playing". Unverifiable as a result: whether the dry-fire click dispatched once or twice, whether the impact sound dispatched per surface type, whether a reload sound was cut off by the next state transition, and whether a sound is audible or virtualised past a concurrency or distance limit — all answerable by looking at the device for one frame, none answerable through any published verb. Ask: a read-only verb enumerating the audio device's active sounds, minimum row `{soundPath, location, startTime, virtualized}`, with owning component (null for this case, itself the signal), concurrency group and limiting state, volume/pitch multipliers and resolved attenuation as useful follow-ons; read-only scope deliberately, no stop/mute/fade, so it stays cheap and safe to call during PIE. Explicitly not closable by a discovery-verb filter fix — `system.inspect.*` enumerates UObjects and `FActiveSound` is not one, so the device must be asked directly. Dedup: exact ripgrep for `FActiveSound` across the board returns zero hits and no ticket owns runtime audio observation; the closest neighbours are `E-spawned-audio-component-not-actor-readable` (the component case, whose `componentPath` steer exists only because those verbs create a component), `B-play-sound-attached-not-attached` (WONTFIX, decided on post-playback `AttachParent` state and auto-destroy timing — a live active-sound readout would have settled it by observation rather than engine source, which is a second argument for this verb), `F-metasound-no-render-to-pcm` (the offline analogue: what an asset would sound like, not what the device is doing), and `F-sound-concurrency-asset-authoring` (authors the rules whose `virtualized` effect this would make observable). Severity Medium: impact is the no-workaround band — the object is not in the object graph, so no combination of published verbs answers the question and `python.execute` into `FAudioDevice` is not a published surface either — pulled down from High by reach, since runtime audio verification is not an every-session path and projects routing audio through `UAudioComponent` can inspect components today; the gap falls on projects using the engine's ordinary fire-and-forget one-shot call.
