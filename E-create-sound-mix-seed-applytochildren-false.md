---
id: E-create-sound-mix-seed-applytochildren-false
title: "audio.authoring.create_sound_mix's classAdjusters seed path leaves bApplyToChildren at the struct default (false) and exposes no applyToChildren key, silently diverging from add_mix_modifier's documented Default: true for the same per-class adjuster"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [audio, soundmix, create_sound_mix, add_mix_modifier, classAdjusters, applyToChildren, default-divergence, seed-path]
---

# `create_sound_mix` `classAdjusters` seeds `applyToChildren=false`, but `add_mix_modifier` defaults it `true`

`audio.authoring` offers two paths to author the same per-class
`FSoundClassAdjuster` on a `USoundMix`: seed it at creation via
`create_sound_mix`'s `classAdjusters` array, or append it afterward via
`add_mix_modifier`. The wiki for `create_sound_mix` explicitly steers agents to
the seed path ("optionally seed adjusters at creation via `classAdjusters`";
"Append more adjusters post-create with `add_mix_modifier`"), so the two are
presented as interchangeable ways to express the same adjuster.

They are not interchangeable on one field: **`bApplyToChildren`**.

- `add_mix_modifier` registers `applyToChildren` as an optional boolean
  **`Default: true`** (wiki `add_mix_modifier.md:17`), and the handler honors it:
  `Adjuster.bApplyToChildren = Ctx.GetBool(TEXT("applyToChildren"), true);`
  (`AudioAuthoringHandler.cpp:1648`).
- The `create_sound_mix` seed loop (`AudioAuthoringHandler.cpp:1560-1568`) builds
  `FSoundClassAdjuster` setting only `SoundClassObject`, `VolumeAdjuster`, and
  `PitchAdjuster`. It **never assigns `bApplyToChildren`**, so the adjuster keeps
  the struct's zero-initialized default (`false`) — and the `classAdjusters` entry
  schema (`{soundClass, volumeAdjuster, pitchAdjuster}`, registered at `:1517`,
  documented identically in the wiki Notes) exposes **no `applyToChildren` key**,
  so there is no way to opt in either.

So the same logical operation — "duck SoundClass X to 0.4 in this mix" — applies
to child classes when authored via `add_mix_modifier` but **not** when seeded via
`create_sound_mix`, with no diagnostic and no knob to fix it. For a ducking mix
(the canonical use of a SoundMix, and the use named in the task that surfaced
this), the seeded duck silently won't propagate to the ducked class's children,
contradicting the default an agent reasonably carries over from the documented
sibling path.

This is the ergonomic default-divergence shape: both calls succeed, both persist
faithfully, but two documented siblings for one operation disagree on a field's
default and only one of them exposes the knob.

## Verbatim live repro (replayed against mcp__editor-automation__call)

1. `audio.authoring.create_sound_class { name: SC_A2CProbe,  path: /Game/AudioApplyChildrenProbe/Classes, volume: 1 }` → ok
2. `audio.authoring.create_sound_class { name: SC_A2CProbe2, path: /Game/AudioApplyChildrenProbe/Classes, volume: 1 }` → ok
3. `audio.authoring.create_sound_mix { name: Mix_A2CProbe, path: /Game/AudioApplyChildrenProbe/Mixes, classAdjusters: [{ soundClass: /Game/AudioApplyChildrenProbe/Classes/SC_A2CProbe, volumeAdjuster: 0.4, pitchAdjuster: 1 }] }` → ok (seed path; note no `applyToChildren` key is acceptable here — the schema has none)
4. `audio.authoring.add_mix_modifier { assetPath: /Game/AudioApplyChildrenProbe/Mixes/Mix_A2CProbe, soundClassPath: /Game/AudioApplyChildrenProbe/Classes/SC_A2CProbe2, volumeAdjuster: 0.4, pitchAdjuster: 1 }` → ok (modifier path, `applyToChildren` omitted → its documented `Default: true`)
5. `audio.authoring.describe_sound_mix { assetPath: /Game/AudioApplyChildrenProbe/Mixes/Mix_A2CProbe }` →

```json
{"type":"SoundMix","modifierCount":2,"adjusters":[
  {"soundClass":".../SC_A2CProbe.SC_A2CProbe","volumeAdjuster":0.4,"pitchAdjuster":1,"applyToChildren":false},
  {"soundClass":".../SC_A2CProbe2.SC_A2CProbe2","volumeAdjuster":0.4,"pitchAdjuster":1,"applyToChildren":true}],
 ...}
```

Two adjusters in one mix, authored identically (vol 0.4, pitch 1), differ only by
which path created them: the `create_sound_mix`-seeded one is
**`applyToChildren:false`**, the `add_mix_modifier`-added one is
**`applyToChildren:true`**. The divergence is real and read back through the
in-namespace `describe_sound_mix` reader.

## What it should do

Make the two adjuster-creation paths agree on the field, and let the seed path
control it:

- In the `create_sound_mix` `classAdjusters` loop
  (`AudioAuthoringHandler.cpp:1560-1568`), read an optional per-entry
  `applyToChildren` and default it to **`true`** to match `add_mix_modifier`:
  ```cpp
  bool bApplyChildren = true;
  AdjObj->TryGetBoolField(TEXT("applyToChildren"), bApplyChildren);
  Adjuster.bApplyToChildren = bApplyChildren;
  ```
- Update the `classAdjusters` param description (`:1517`) and the wiki Notes
  (`docs/wiki-src/audio.authoring.md` / the `create_sound_mix` overlay) to document
  the `applyToChildren` key on each entry and its `true` default, so the seed
  schema matches `add_mix_modifier`'s documented surface.

If the project deliberately wants the seed default to stay `false`, the minimal
ergonomic fix is at least to document the divergence on both pages so an agent
seeding a duck adjuster isn't surprised — but matching `add_mix_modifier`'s
`true` default is the consistent resolution.

## History
- `#1-initial-repro` `OPEN` reporter — REALISM-mode audio-mixing-bus build
  (Master/Dialogue/Ambience SoundClass hierarchy + `DialogueDuck` SoundMix seeded
  via `create_sound_mix` `classAdjusters`). Attempt ran fully clean (every call
  `ok:true`, zero retries); the agent's own friction note flagged it verbatim:
  "the `DialogueDuck` adjuster seeded via `create_sound_mix`'s `classAdjusters`
  came back `applyToChildren=false`, whereas `add_mix_modifier` documents that flag
  defaulting to `true` - the create-time seed path doesn't expose `applyToChildren`,
  a small inconsistency between the two adjuster-creation paths." Replay-confirmed
  live: a single `Mix_A2CProbe` with one seeded adjuster (`create_sound_mix`) and
  one appended adjuster (`add_mix_modifier`, `applyToChildren` omitted) read back
  via `describe_sound_mix` as `applyToChildren:false` vs `applyToChildren:true`
  respectively — same vol/pitch, divergent only by creation path. Source-confirmed:
  the seed loop (`AudioAuthoringHandler.cpp:1560-1568`) never assigns
  `bApplyToChildren` (struct default `false`) and the `classAdjusters` schema
  (`:1517`) has no `applyToChildren` key, while `add_mix_modifier` (`:1648`) defaults
  it `true`. Distinct from `B-add-mix-modifier-fade-params-dropped` (a write-side
  fade silent-drop) and `E-audio-get-info-soundclass-mix-readback-thin` (readback
  thinness, since closed by the `describe_sound_mix` reader used here to confirm) —
  this is a default-divergence + missing-knob on the seed path itself.
- `#2-fix` `IN-REVIEW` developer — aligned the `create_sound_mix` `classAdjusters`
  seed path with `add_mix_modifier`'s documented `applyToChildren` default. In the
  seed loop (`Source/PinWright/Private/Handlers/Audio/AudioAuthoringHandler.cpp`,
  the `classAdjusters` `for` block ~:1554-1562) the per-entry adjuster now reads an
  optional `applyToChildren` via `TryGetBoolField` defaulting to `true`
  (`Adjuster.bApplyToChildren = bApplyChildren;`), mirroring `add_mix_modifier`'s
  `Ctx.GetBool("applyToChildren", true)`. The `classAdjusters` param schema
  (`RPC_PARAM_OPT` at ~:1511) now documents the `{… applyToChildren}` key + `true`
  default, and the wiki overlay (`Docs/wiki-src/audio.authoring.md` create_sound_mix
  section) documents the key, shows an `applyToChildren: false` example, and states
  the two paths now agree on the default. Regression test added alongside the
  existing audio handler tests:
  `Source/PinWright/Private/Tests/Media/TestAudioHandlers.cpp` →
  `PinWright.audio.authoring.create_sound_mix.SeedApplyToChildrenDefaultsTrue` — it
  seeds two adjusters (one omitting the key, one explicit `false`), then reads the
  live `USoundMix::SoundClassEffects` back and asserts the omitted one is now `true`
  (fails pre-fix when the seed loop left it `false`) while the explicit `false` is
  honored (guards against a fix that blanket-forces `true`). Line numbers in the
  body/`#1` are pre-fix and drifted a few lines; the defect and fix are unchanged.
