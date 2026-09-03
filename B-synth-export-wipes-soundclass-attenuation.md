---
id: B-synth-export-wipes-soundclass-attenuation
title: "audio.synth.export updated_in_place clears SoundClassObject and AttenuationSettings, which its docs never mention — re-synthesising a wave silently unroutes it from the mix"
status: OPEN
severity: High
category: bug
tags: [audio, synth, export, soundwave, soundclass, attenuation, data-loss, silent-failure, docs-mismatch, updated-in-place]
encounters: 4
lastSeen: 2026-09-03T01:30:00Z
---

# `audio.synth.export` drops a wave's mix routing and does not say so

## Symptom

Re-exporting a candidate over an existing `USoundWave` (`mode: "updated_in_place"`)
resets `SoundClassObject` and `AttenuationSettings` to null. Both are `USoundBase`
properties, not payload, and nothing in the response reports the loss —
`verification.pass:true`, `existsAfter:true`, `existsOnDisk:true`.

Measured on `/Game/FPS/Audio/Waves/Foley/SW_Step_Concrete_A`:

```
before: SoundClassObject = /Game/FPS/Audio/Mix/SC_Foley.SC_Foley
        AttenuationSettings = /Game/FPS/Audio/Attenuation/ATT_Foley.ATT_Foley
after:  both keys absent from asset.dump properties.json
disk:   grep -ac ATT_Foley SW_Step_Concrete_A.uasset -> 0   (untouched siblings -> 2)
        grep -ac SC_Foley  SW_Step_Concrete_A.uasset -> 0   (untouched siblings -> 3)
```

## Why it is damaging

An unrouted wave leaves its SoundClass, so it escapes every SoundMix, duck and class
volume, and loses its attenuation curve so it plays at full level at any distance.
Nothing in the editor flags it; the asset still opens, still previews, still passes
`audio.synth.export`'s own decode-back verification. On this project the FPS audio
build had all 72 playable assets carrying a SoundClass and an attenuation; twelve
footstep re-syntheses would have silently removed that from twelve of them.

## Docs mismatch

`audio.synth.export` and `audio.synth` both scope the reset to **per-wave** state:

> "re-exporting the same candidate to the same path rewrites that wave in place
> (which also resets that wave's per-wave properties (looping, volume, sound group)
> to defaults), so set those after the last export rather than before."

`SoundClassObject` and `AttenuationSettings` are neither in that list nor per-wave —
they are the `USoundBase` routing every playable asset needs. A caller who read the
warning and planned to re-set `bLooping` afterwards still loses the routing, because
they were never told it was at risk. `audio.authoring.create_sound_wave_from_pcm`
carries the same three-item wording and presumably the same behaviour.

Compounding it: `audio.authoring.set_sound_wave_properties` covers `bLooping`,
`volume`, `pitch`, `soundGroup`, `compressionQuality`, `bMature`, `bSingleLine` — so
every property the docs *do* warn about has a repair verb, and the two they omit have
none. There is no RPC anywhere in `audio.authoring` that assigns a SoundClass or an
attenuation to an existing wave; repair requires `python.execute`.

## What it should do

Preferred: preserve `SoundClassObject` and `AttenuationSettings` across an
`updated_in_place` export. The verb is documented as rewriting the *payload*, and the
routing is not payload — an in-place update that keeps the asset's identity should
keep its wiring.

If the reset is deliberate, then both:

- Name them in the reset list on `audio.synth.export`, `audio.synth` and
  `audio.authoring.create_sound_wave_from_pcm`, and say there is no verb to restore
  them.
- Report the cleared references in the export response (e.g. a `resetProperties`
  array), so the loss is visible rather than inferred from a later dump.

Either way, add a `set_sound_wave_routing` (or extend `set_sound_wave_properties`
with `soundClass` / `attenuationSettings`) so repair does not require raw Python.

## Workaround

**Corrected after filing (see History `#2`): repair does NOT need `python.execute`.** Two
`property.set` calls do it through the typed reflected writer — `propertyName` `SoundClassObject`
and `AttenuationSettings`, `value` a full object path such as
`/Game/FPS/Audio/Mix/SC_Weapons.SC_Weapons` — then `asset.save {force:true}`. Re-verified by the
reporter: `applied:true, markedDirty:true`. That also narrows the fix ask: the fields are already
reachable, so adding `soundClass`/`attenuationSettings` to `set_sound_wave_properties` is a thin
wrapper over a working path, and the wiki sentence can be corrected before any code lands. The
Python form below still works and is convenient for bulk repair across many assets.

Snapshot the routing before exporting, restore it after, and verify against the
`.uasset` bytes rather than an in-memory readback:

```python
w = unreal.load_object(None, '/Game/.../SW_X.SW_X')
w.modify(True)
w.set_editor_property('sound_class_object', sc)
w.set_editor_property('attenuation_settings', att)
unreal.PinWrightPackageLibrary.mark_package_dirty(w)
unreal.EditorLoadingAndSavingUtils.save_packages([w.get_outermost()], False)
```

## History

- `#1-initial-repro` `OPEN` reporter — Found while re-synthesising the 12 FPS
  footsteps (`/Game/FPS/Audio/Waves/Foley/SW_Step_*`) against audio review 01 rubric
  row 7. First export (`SW_Step_Concrete_A`, `mode:"updated_in_place"`,
  `verification.pass:true`, `saved:true`) reported nothing unusual; an `asset.dump`
  taken before and after showed `SoundClassObject` and `AttenuationSettings` present
  before and absent after, and `grep -a` on the `.uasset` confirmed the references
  gone from disk while eleven untouched siblings still carried them. Snapshotting the
  routing for all 18 assets in scope before continuing, then restoring it through
  `python.execute` afterwards, cost the task an extra round trip but caught the loss;
  an agent that trusted `verification.pass` would have shipped 17 unrouted waves.
  Dedup: ripgrep over the board for `synth.export` / `SoundClassObject` / `sound_class`
  found the four synth tickets (`B-synth-layer-peak-excludes-layer-gain`,
  `B-synth-patch-canonical-omits-layer-fx`, `B-synth-schema-advertises-unrenderable-ranges`,
  `B-synth-targets-never-scored`) and `E-synth-cookbook-no-ambience-loops` — that last
  one covers `bLooping` surviving export, which is the *documented* half of this
  reset; none covers the undocumented routing half.

- `#2-second-repro-weapons-impacts-and-a-cheaper-repair` `OPEN` reporter — Independent
  repro on the other half of the same audio build: the ten weapon/impact waves
  (`SW_Tail_{Outdoor,Indoor}`, `SW_Fire_{AR,Pistol}_Body`, `SW_Impact_Metal_{A,B}`,
  `SW_Impact_Glass_{A,B}`, `SW_Explosion_{Close,Distant}`), re-synthesised against review 01
  §3-4 rubric rows 1/4/5/8 and defects 8/9/12. **All ten lost both references**, so the
  failure is not specific to Foley, to `SC_Foley`/`ATT_Foley`, or to a particular
  SoundClass — these carry `SC_Weapons` and `SC_SFX`, all ten against `ATT_Weapon`, and
  every one came back empty from
  `grep -a -o -E "/Game/FPS/Audio/(Attenuation|Mix)/[A-Za-z_]+"` after an export that
  answered `verification.pass:true, saved:true, mode:"updated_in_place"`.
  **Correction to this ticket's Workaround: repair does not require `python.execute`.**
  Two `property.set` calls per wave do it through the typed reflected writer —
  `property.set {objectPath:"<wave>.<wave>", propertyName:"SoundClassObject", value:"/Game/FPS/Audio/Mix/SC_Weapons.SC_Weapons"}`
  and the same for `AttenuationSettings` — followed by `asset.save {assetPath, force:true}`.
  Both answered `applied:true, markedDirty:true`, and all twenty writes verified against the
  `.uasset` bytes afterwards (`grep -ac 'SC_SFX\|SC_Weapons'` ≥ 2 and `grep -ac ATT_Weapon`
  ≥ 1 on every file, against 0 immediately after export). That matters for the ask: the
  missing piece is a *documented* routing verb in `audio.authoring`, not a missing
  capability — `property.set` already reaches these fields, so `set_sound_wave_properties`
  gaining `soundClass` / `attenuationSettings` is a thin wrapper over a path that works
  today, and the wiki's repair sentence can be corrected before the code is.
  Also confirms the docs-mismatch section from the caller's side: I read the
  "looping, volume, sound group" sentence *before* exporting, planned no repair because
  none of the three mattered for one-shots, and would have shipped ten unrouted waves had
  the post-export `grep -a` been for payload only. `asset.reload` was not used at any point.

- `#3-third-encounter-nine-waves-build-03` `OPEN` reporter — Hit again fixing the Build 03
  audio wave defects (`Docs/fps/reviews/audio-review-02.md` §5). Nine waves re-exported over
  existing assets: `SW_Explosion_Close`, `SW_Explosion_Distant`, `SW_Impact_{Concrete,Wood}_{A,B}`
  (`/Game/FPS/Audio/Waves/Impacts/`), `SW_Reload_Bolt` (`/Waves/Weapons/`), `SW_Amb_RopeSlap_01`
  and `SW_Amb_Sea_Loop` (`/Waves/Ambience/`). Every export answered
  `verification.pass:true, mode:"updated_in_place"`, and immediately afterwards
  `grep -ac 'SC_SFX\|SC_Weapons\|SC_Ambience'` and `grep -ac 'ATT_'` both returned **0 on all
  six waves that had reached disk** — measured on the `.uasset` bytes, not in memory. Nothing in
  any of the nine responses mentioned routing. Repaired with the two `property.set` calls this
  ticket's `#2` documents; that path still works and remains the correct workaround.
  Two additions from this encounter:
  **(a) The wipe re-fires on every export, including a corrective re-export.** `SW_Impact_Wood_B`
  was exported, repaired with `property.set`, then re-exported a second time to fix an unrelated
  DC-offset regression — and the second export cleared `SoundClassObject` / `AttenuationSettings`
  again, discarding the repair. Anyone following an export -> repair -> re-measure -> re-export
  loop must re-repair after *each* export, not once at the end. The docs' "set those after the
  last export rather than before" sentence covers looping/volume/sound group but never names
  routing, so a caller who has read it still loses these two.
  **(b) It is invisible to the subsystem's own audit.** `audio.analysis.audit_folder` over
  `/Game/FPS/Audio/Waves` (63 waves) reported `flagged: 1` for a DC offset and nothing else —
  it has no routing check, so six freshly unrouted waves swept clean in the same call that
  found an unrelated 0.0116 DC offset. Worth pairing with this ticket's ask: an
  `audit_folder` finding for a playable wave with a null `SoundClassObject` would have caught
  all three encounters of this bug without the caller knowing to look.

- `#4-fourth-encounter-build-04-round-robin` `OPEN` reporter — Hit again building the Build 04
  round-robin variants and the wood-footstep repair (`Docs/fps/reviews/audio-review-03.md` §5
  items 3 and 4). Six exports: four **new** waves `SW_Fire_{AR,Pistol}_{Mech,Body}_B`
  (`/Game/FPS/Audio/Waves/Weapons/`, `mode:"created"`) and two **in-place** rewrites
  `SW_Step_Wood_{A,B}` (`/Game/FPS/Audio/Waves/Foley/`, `mode:"updated_in_place"`). All six
  answered `verification.pass:true` and said nothing about routing.
  One addition this encounter narrows the ask:
  **`mode:"created"` has the same end state as `mode:"updated_in_place"`, for a different
  reason, and the ticket title only names the second.** A newly created SoundWave has null
  `SoundClassObject` / `AttenuationSettings` because nothing ever set them — that is not a wipe,
  it is the CDO default — yet the consequence for the caller is identical: an asset that will
  be cooked, referenced by a cue or MetaSound, and play unrouted through the master bus. On this
  build the two failure modes were indistinguishable from the response: the same
  `verification.pass:true`, the same silence about routing, the same
  `grep -ac 'SC_Weapons' = 0` on the bytes afterwards. So the fix this ticket asks for should be
  stated as **export must never leave a playable wave with a null `SoundClassObject`** —
  carry the existing values across an in-place rewrite, and either take routing arguments or
  disclose the null on a create. Repairing one and not the other still ships silent unrouted
  waves.
  All six repaired with the two `property.set` calls + `asset.save`, then verified on the bytes:
  `grep -ac` returns 2-3 for the class and 1-2 for the attenuation on every one.
