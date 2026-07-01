---
id: E-audio-dialogue-no-readback-get-info-unknown
title: "audio.authoring Dialogue assets have NO live readback — get_audio_info returns type:'Unknown' for DialogueWave/DialogueVoice (no gender/spokenText/context), and there is no describe_dialogue_* to match the describe_attenuation/sound_class/sound_mix siblings; the overlay even advertises a dialogue describe reader that doesn't exist"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [docs, audio, audio-authoring, dialogue, dialogue-wave, dialogue-voice, readback, describe, inspect-after-mutate, wiki]
---

# `audio.authoring` has no live readback for Dialogue assets — `get_audio_info` returns `type:"Unknown"`

This is the **Dialogue-family sibling** of the established `audio.authoring`
readback-thinness shape — `E-audio-authoring-attenuation-readback-undocumented`
(IN-REVIEW, shipped `describe_attenuation`) and
`E-audio-get-info-soundclass-mix-readback-thin` (IN-REVIEW, shipped
`describe_sound_class`/`describe_sound_mix`) — but the Dialogue case is **worse
than thin**: `get_audio_info` doesn't even *recognize* a `UDialogueWave` or
`UDialogueVoice`. Both fall through to the generic `else` branch and return
`type:"Unknown"` with no `spokenText`, no context mappings, no `gender`/
`plurality`. So the three Dialogue write verbs — `create_dialogue_voice`,
`create_dialogue_wave`, `set_dialogue_context` — are the **only** authoring-verb
family in the namespace with **zero** in-namespace readback: not a thin one, none.

The handler confirms it. `get_audio_info`
(`AudioAuthoringHandler.cpp:2600`, body branches at `:2620-2667`) casts to
`USoundCue` / `USoundWave` / `USoundClass` / `USoundMix` / `USoundSubmix` /
`USoundAttenuation` and, for anything else, hits the final `else` at the bottom:

```cpp
else
{
    Result->SetStringField(TEXT("type"), TEXT("Unknown"));
}
```

`UDialogueWave` / `UDialogueVoice` match no branch, so a readback of a dialogue
asset the namespace just created returns
`{"type":"Unknown","message":"Audio info retrieved"}` — a *successful* call that
confirms nothing. There is no `describe_dialogue_wave` / `describe_dialogue_voice`
(`rg "describe_dialogue" AudioAuthoringHandler.cpp` → no hits), and no dialogue
dump sidecar, so the only live way to confirm `spokenText`, the speaker/target
context wiring, or a voice's `gender`/`plurality` is to leave `audio.authoring`
for `asset.dump` → `properties.json` and/or `property.get` on individual fields.

## The overlay over-promises a dialogue describe reader

`docs/wiki-src/audio.authoring.md` makes this a docs gap on top of the
capability gap. The namespace summary (`audio.authoring.md:3`) advertises:

> "Authoring API for all UE sound assets: SoundCues, MetaSounds, SoundClasses &
> Mixes, attenuation/**effects/dialogue**, **plus dump-parity describe readers
> for inspect-after-mutate workflows**."

…but the `## Inspect-after-mutate` section (`:7`) enumerates describe readers for
SoundCue / MetaSound / SoundWave / SoundAttenuation **and names none for
Dialogue**, and never states the `get_audio_info` `type:"Unknown"` behavior for
dialogue assets. An agent that trusts the summary's "describe readers for
…dialogue" line will hunt for a `describe_dialogue_*` that doesn't exist, then
fall back — the same discover-the-absence-then-pivot cost
`E-audio-authoring-attenuation-readback-undocumented` documented for
SoundAttenuation before its reader shipped.

## Evidence (this task)

Struggle-audit of an `audio.authoring.create_dialogue_wave` tavern-dialogue build
(seed `audio.authoring.create_dialogue_wave`, outcome tool_bug — the judge filed
the stray-null-target corruption separately as
`B-dialogue-context-null-target-prepended`; this is the distinct PROCESS angle).
The story built two `DialogueVoice`s (DV_Innkeeper Masculine, DV_Player Neuter),
two `DialogueWave`s with spoken text, wired speaker/target contexts on both, and
step 7 asked to "read back both DialogueWave assets (and confirm the voices
exist) so I can verify the speaker/target wiring and spoken text landed
correctly." The build ran clean (all `create_*`/`set_dialogue_context` `ok:true`),
but every readback through the documented reader was empty: the call log shows
**four** `audio.authoring.get_audio_info` readbacks (DW_InnkeeperGreeting,
DW_InnkeeperFarewell, DV_Innkeeper, DV_Player) that all returned `type:"Unknown"`,
forcing the agent to pivot out of the namespace. Friction note (verbatim):

> "get_audio_info is too thin for dialogue assets (returns type:\"Unknown\", no
> gender/spokenText/context), and there is no
> audio.authoring.describe_dialogue_wave/voice nor a dump sidecar, so I had to
> fall back to asset.dump + property.get to verify."

The recovery cost is concrete in the log: after the four dead `get_audio_info`
calls, the agent emitted `call("asset")` (namespace index nav), four `asset.dump`
executes (one per asset, to read `properties.json` for `spokenText` + context
mappings), then **six** `property.get` calls to confirm the two voices' `Gender`
— of which the **first four `property.get` calls hard-failed** on param-name
guesses (`path`/`name` → `[MISSING_REQUIRED_PARAM] objectPath`; then
`objectPath`/`name` → `[MISSING_REQUIRED_PARAM] propertyName`) before the
`objectPath`/`propertyName` pair worked. So a single "did my dialogue wiring
land?" intent that one `describe_dialogue_wave` could answer cost ~15 recovery
calls (4 dead reads + 1 index nav + 4 dumps + 6 property.get, 4 of them errored).
(The `Gender` value `Neuter` for DV_Player was *only* confirmable via
`property.get` — it was omitted from the `asset.dump` too.)

## What it should do / how to fix

Mirror the describe-verb pattern the SoundAttenuation / SoundClass / SoundMix
siblings shipped (`AudioAuthoringHandler.cpp:2518` `describe_attenuation` is the
template), so the fields the dialogue write verbs persist are confirmable
in-namespace without an `asset.dump` / `property.get` pivot:

- **`audio.authoring.describe_dialogue_voice { assetPath }`** — echo
  `gender` and `plurality` (the two params `create_dialogue_voice` writes,
  `:1938`).
- **`audio.authoring.describe_dialogue_wave { assetPath }`** — echo
  `spokenText` plus a `contexts` array, one object per
  `FDialogueContextMapping` with `speaker` and a `targets[]` voice-path list
  (the wiring `set_dialogue_context` writes, `:2050`+). This reader would *also*
  make the `B-dialogue-context-null-target-prepended` stray-null observable
  from the RPC surface (today the null only surfaces on an `asset.dump`).

Until a reader ships, do the **cheap docs win now**: in
`docs/wiki-src/audio.authoring.md`, (1) correct the `:3` summary so it does not
claim a dialogue describe reader that doesn't exist, and (2) add a Dialogue line
to `## Inspect-after-mutate` stating that `get_audio_info` returns `type:"Unknown"`
for `DialogueWave`/`DialogueVoice` and that the spoken text / context wiring /
voice gender are confirmed via `asset.dump { assetPath }` → `properties.json`
(`spokenText`, `ContextMappings[].Context.{Speaker,Targets}`) plus `property.get
{ objectPath, propertyName:"Gender" }` for the voice gender (note: a voice's
gender is omitted from the dump and is only confirmable via `property.get`).

## Workaround (until a describe verb lands)

Confirm a `DialogueWave` via `asset.dump { assetPath }` → `properties.json`
(`SpokenText`, `ContextMappings[].Context.{Speaker,Targets}`); confirm a
`DialogueVoice`'s gender via `property.get { objectPath:"<voice path>",
propertyName:"Gender" }` — `get_audio_info` returns `type:"Unknown"` and confirms
nothing for either.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle-audit of an `audio.authoring`
  tavern-dialogue build (seed `audio.authoring.create_dialogue_wave`, outcome
  tool_bug; the stray-null-target data corruption filed separately by the judge
  as `B-dialogue-context-null-target-prepended`). Distinct PROCESS angle: the
  Dialogue asset family is the only `audio.authoring` write-verb family with NO
  live readback — `get_audio_info` (`AudioAuthoringHandler.cpp:2600`,
  `else`-branch `type:"Unknown"` at the body tail) doesn't recognize
  `UDialogueWave`/`UDialogueVoice`, so the four readback calls in the task all
  returned `type:"Unknown"` and confirmed nothing. There is no
  `describe_dialogue_wave`/`describe_dialogue_voice` (the other audio types each
  got a `describe_*` — `E-audio-authoring-attenuation-readback-undocumented`,
  `E-audio-get-info-soundclass-mix-readback-thin`), and no dump sidecar, forcing
  a pivot to `asset.dump` + `property.get`. The overlay over-promises: the
  `audio.authoring.md:3` summary advertises "describe readers for
  …dialogue" but `## Inspect-after-mutate` (`:7`) names none and never states
  the `type:"Unknown"` behavior. Friction note (verbatim): "get_audio_info is too
  thin for dialogue assets (returns type:\"Unknown\", no gender/spokenText/
  context), and there is no audio.authoring.describe_dialogue_wave/voice nor a
  dump sidecar, so I had to fall back to asset.dump + property.get to verify."
  Call log confirms 4 dead `get_audio_info` reads + ~11 recovery calls (1 index
  nav, 4 `asset.dump`, 6 `property.get` of which 4 errored on param-name guesses).
  Proposed: ship `describe_dialogue_voice` (echo gender/plurality) +
  `describe_dialogue_wave` (echo spokenText + contexts[].{speaker,targets}) per
  the `describe_attenuation` template; and the cheap interim win — fix the
  `docs/wiki-src/audio.authoring.md:3` summary's false dialogue-describe promise
  and add a Dialogue line to `## Inspect-after-mutate` naming the
  `asset.dump`/`property.get` fallback (voice gender is omitted from the dump and
  only confirmable via `property.get`).
- `#2-describe-verbs-shipped` `IN-REVIEW` developer — Shipped the two Dialogue
  describe readers (the Dialogue member of the `describe_attenuation` /
  `describe_sound_class` / `describe_sound_mix` family) plus a thin recognition
  fix in `get_audio_info`, and documented them. Code
  (`Source/PinWright/Private/Handlers/Audio/AudioAuthoringHandler.cpp`): added
  `audio.authoring.describe_dialogue_voice` (echoes `gender` /
  `plurality` — the inverse of `create_dialogue_voice`'s parse ladder) and
  `audio.authoring.describe_dialogue_wave` (echoes `spokenText`, `contextCount`,
  and a `contexts[]` array of per-`FDialogueContextMapping` `speaker` +
  `targets[]` voice paths + `soundWave` + `localizationKeyFormat` — the wiring
  `set_dialogue_context` writes); both `#if MCP_HAS_DIALOGUE`-gated like the
  write verbs, keys mirroring the authoring params. The wave reader emits the raw
  `targets` list (a null shows as an empty-string path), so the
  `B-dialogue-context-null-target-prepended` stray-null is now observable from
  the RPC surface. Also added `UDialogueWave` / `UDialogueVoice` branches to
  `get_audio_info` so it returns `type:"DialogueWave"` (with `contextCount`) /
  `type:"DialogueVoice"` instead of `type:"Unknown"`. Docs
  (`docs/wiki-src/audio.authoring.md`): corrected the `:3` summary, added the two
  verbs to `## Inspect-after-mutate`, and added a `### ` method section for each.
  Regression test
  (`Source/PinWright/Private/Tests/Media/TestAudioHandlers.cpp`):
  `PinWright.audio.authoring.describe_dialogue.EchoesWrittenFields` builds two
  transient `DialogueVoice`s + one `DialogueWave`, sets Gender/Plurality/SpokenText,
  drives the PRODUCTION `set_dialogue_context` handler to wire speaker+target, then
  invokes both describe verbs and asserts every field round-trips (it fails if
  either verb is reverted — handler-not-found — or any field is dropped); plus a
  `describe_dialogue_wave.MissingRequiredParam` smoke test. Both `#if
  MCP_TEST_HAS_DIALOGUE`-gated.
