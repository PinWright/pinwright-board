---
id: B-dialogue-context-null-target-prepended
title: "audio.authoring.set_dialogue_context prepends a stray null into the context's Targets array"
status: IN-REVIEW
severity: Medium
category: bug
tags: [audio, audio-authoring, dialogue, dialogue-wave, dialogue-voice, data-corruption]
encounters: 2
lastSeen: 2026-07-11T10:59:05.8727190+03:00
---

# `audio.authoring.set_dialogue_context` prepends a stray `null` into the Targets array

`audio.authoring.set_dialogue_context` reports success and adds a
`FDialogueContextMapping` whose `Context.Speaker` is set correctly, but the
`Context.Targets` array it writes has an extra `null` element inserted at the
**front**, before the requested target voices. The requested targets all land
(in order), so this is an off-by-one: exactly **one** stray null is prepended
regardless of how many targets were supplied.

A `null` DialogueVoice in a context's target list is invalid wiring — the
localization key hash and per-context resolution iterate the target voices, so
a leading null is at best dead data and at worst a resolution hazard. The RPC
returns a clean success (`{"contextCount":N,"message":"Dialogue context mapping
added", ...}`), so callers have no signal that the persisted data is malformed;
it only surfaces on a later `asset.dump` / property read.

## Repro (verbatim, replayed live)

Setup — two DialogueVoices and a fresh DialogueWave:

```
call("audio.authoring.create_dialogue_voice",
     {name:"DV_Innkeeper_R", path:"/Game/Audio/Dialogue/TavernReplay",
      gender:"Masculine", plurality:"Singular"})
call("audio.authoring.create_dialogue_voice",
     {name:"DV_Player_R", path:"/Game/Audio/Dialogue/TavernReplay",
      gender:"Neuter", plurality:"Singular"})
call("audio.authoring.create_dialogue_wave",
     {name:"DW_GreetingReplay", path:"/Game/Audio/Dialogue/TavernReplay",
      spokenText:"Welcome to the Drunken Dragon, traveler. What'll it be?"})
```

Add one context with exactly ONE target voice:

```
call("audio.authoring.set_dialogue_context",
     {assetPath:"/Game/Audio/Dialogue/TavernReplay/DW_GreetingReplay",
      speakerPath:"/Game/Audio/Dialogue/TavernReplay/DV_Innkeeper_R",
      targetVoices:["/Game/Audio/Dialogue/TavernReplay/DV_Player_R"]})
-> {"contextCount":2, "message":"Dialogue context mapping added", ...}
```

`asset.dump` of the wave -> `properties.json`, `ContextMappings[1]` (the
one added by the call):

```json
"Context": {
    "Speaker": "/Game/Audio/Dialogue/TavernReplay/DV_Innkeeper_R.DV_Innkeeper_R",
    "Targets": [
        null,
        "/Game/Audio/Dialogue/TavernReplay/DV_Player_R.DV_Player_R"
    ]
}
```

The leading `null` was never requested — only `DV_Player_R` was passed.

**Deterministic / off-by-one confirmed with two targets** — requesting
`["DV_Player_R","DV_Bard_R"]` produces `Targets: [null, DV_Player_R, DV_Bard_R]`
(exactly one prepended null, requested voices intact and in order). Reproduced
3/3 times, including with `replace:true`.

## Root cause

The null is **engine-seeded**, not produced by a handler `AddDefaulted`/`SetNum`
pattern. `FDialogueContext`'s constructor
(`Engine/Private/DialogueTypes.cpp`: `FDialogueContext::FDialogueContext()` →
`Targets.AddZeroed();`) always pre-seeds `Targets` with exactly one zeroed
`TObjectPtr<UDialogueVoice>` (the leading null). `set_dialogue_context`
default-constructs `FDialogueContextMapping NewMapping;`
(`AudioAuthoringHandler.cpp`), so `NewMapping.Context.Targets` already holds
`[null]`, and the handler's clean `Targets.Add(...)` loop appends the resolved
voices on top → `[null, V1, V2…]`. This is why exactly one null is prepended
regardless of target count and why `replace:true` is equally affected (it
assigns the same default-seeded `NewMapping`).

**Fix:** clear the engine-seeded placeholder before populating — call
`NewMapping.Context.Targets.Reset()` (or `Empty()`) immediately before the
loop that adds the resolved `targetVoices`, then `Modify()` +
`MarkPackageDirty()` + save as today.

## Workaround

None from the RPC surface — the null is written into the asset. A consumer
would have to hand-edit the DialogueWave in-editor to strip the leading null
target, defeating headless authoring.

## Note (separate, not this ticket)

A freshly-created `DialogueWave` already carries one default
`FDialogueContextMapping` with `Speaker:null` / `Targets:[]` (this is engine
`UDialogueWave` construction behavior), which is why `set_dialogue_context`
reports `contextCount:2` after a single add and `replace:true` does not collapse
it. That default-empty mapping is engine-side and is NOT the subject of this
ticket — this ticket is strictly the stray null **inside the Targets array of
the mapping the RPC writes**.

## History
- `#1-initial-repro` `OPEN` reporter — Replayed live against `mcp__editor-automation__call` on UE 5.7. `set_dialogue_context` returns success but the persisted `FDialogueContextMapping.Context.Targets` has a stray `null` prepended before the requested target voices. Off-by-one confirmed: exactly one leading null regardless of target count (`[null, V1]` for one target, `[null, V1, V2]` for two), reproduced 3/3 incl. with `replace:true`. Verified via `asset.dump` -> `properties.json` (`ContextMappings[].Context.Targets`). No board ticket covers dialogue context mappings (closest, F-sound-wave-property-edit, is about USoundWave property edits and merely mentions the dialogue-context table as a wave consumer).
- `#2-fix` `IN-REVIEW` developer — Reworded the cause section: the stray null is engine-seeded, not a handler `AddDefaulted`/`i+1` pattern (`FDialogueContext::FDialogueContext()` does `Targets.AddZeroed()` in `Engine/Private/DialogueTypes.cpp`, so `FDialogueContextMapping NewMapping;` already carries `[null]` before the handler's clean `Targets.Add()` loop appends the resolved voices). Fix: added `NewMapping.Context.Targets.Reset();` before the populate loop in `set_dialogue_context` (`Source/EditorAutomationRpcGateway/Private/Handlers/Audio/AudioAuthoringHandler.cpp`), clearing the engine placeholder so the persisted Targets is exactly the requested voices in order. Regression test `FAudioAuthoringSetDialogueContextNoLeadingNullTest` ("…set_dialogue_context.TargetsHasNoLeadingNull") in `Source/EditorAutomationRpcGateway/Private/Tests/Media/TestAudioHandlers.cpp` drives the production handler against real DialogueWave/DialogueVoice fixtures and asserts `Context.Targets` == `[V1]` for a 1-target add and `[V1, V2]` for a 2-target `replace:true` add (no leading null, correct order) — it would fail if the `Targets.Reset()` were reverted (Targets would be `[null, V1]` / `[null, V1, V2]`).
- `#3-additional-fix-absent-at-head-observable-via-describe` `IN-REVIEW` reporter — Additional evidence: STILL REPRODUCES at HEAD and the `#2-fix` `Targets.Reset()` is NOT present in the current source. The module was renamed `EditorAutomationRpcGateway` -> `PinWright`; the shipped describe verbs (`E-audio-dialogue-no-readback-get-info-unknown` `#2`) survived the rename but this ticket's fix did not — the live `set_dialogue_context` handler at `Plugins/PinWright/Source/PinWright/Private/Handlers/Audio/AudioAuthoringHandler.cpp:2367-2372` reads `FDialogueContextMapping NewMapping; NewMapping.Context.Speaker = SpeakerVoice; for (UDialogueVoice* TargetVoice : TargetVoices) { NewMapping.Context.Targets.Add(TargetVoice); }` with NO `Targets.Reset()` before the populate loop, so the engine-seeded leading null is never cleared. New repro angle: the bug is now directly observable via the shipped `audio.authoring.describe_dialogue_wave` reader (no `asset.dump` pivot needed), exactly as `#2-fix` predicted. Live replay on UE 5.7 via `mcp__pinwright__call`: created `DV_Hero_OR` (Masculine) + `DV_Companion_OR` (Feminine) + `DW_Greeting_OR` under `/Game/Audio/Dialogue/OracleReplay`; `describe_dialogue_wave` on the FRESH wave returned `contextCount:1` with one engine-default mapping `{speaker:"",targets:[],localizationKeyFormat:"{ContextHash}"}`; after ONE `set_dialogue_context` (speaker `DV_Hero_OR`, targetVoices `[DV_Companion_OR]`) the readback returned `contextCount:2` with the written mapping `targets:["","/Game/Audio/Dialogue/OracleReplay/DV_Companion_OR.DV_Companion_OR"]` — the requested target intact but with the stray leading empty-string (null) target prepended. Confirms both the subject bug (stray null in the written Targets) and the separate engine-default-empty-context off-by-one from the Note are unfixed at HEAD.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. The body needed no edit. 2 citations sit in history rows and are left verbatim per the append-only rule, mapping by the same rule; the mapped paths was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
