---
id: F-audio-dialogue-remove-clear-context-verb
title: "audio.authoring has no verb to remove or clear a DialogueWave context mapping — only add/replace-by-speaker exist, so an over-added context can't be pruned"
status: OPEN
severity: Low
category: feature
tags: [audio, audio-authoring, dialogue, dialogue-wave, set-dialogue-context, missing-verb]
encounters: 1
lastSeen: 2026-07-11T12:09:17+03:00
---

# `audio.authoring` has no `remove_dialogue_context` / `clear_dialogue_contexts` verb

The `audio.authoring` dialogue surface can create a `DialogueWave` and *add*
context mappings (`set_dialogue_context`, default append; `replace:true` does an
upsert keyed on the speaker voice), but there is **no verb to remove or clear a
context mapping** once it has been added. The five dialogue verbs
(`create_dialogue_voice`, `describe_dialogue_voice`, `create_dialogue_wave`,
`describe_dialogue_wave`, `set_dialogue_context`) are the entire surface — none
deletes.

So a caller who adds the wrong context, or adds one too many, cannot prune it
through the RPC surface: `replace:true` can only overwrite an existing mapping
that shares the same speaker, never delete one, and `ContextMappings` is a
`TArray<FDialogueContextMapping>` of nested UObject-pointer structs that a
generic `property.set` cannot realistically author. The only real recourse is to
delete and recreate the whole `DialogueWave` and re-add just the wanted contexts.

This is the symmetric-remove capability gap in the dialogue family, analogous to
`B-create-montage-duplicate-default-slot`'s note that there is no
`remove_montage_slot` verb.

## Context / provenance

Split out of `E-dialogue-context-default-not-reclaimed` (the off-by-one where the
first `set_dialogue_context` never reclaimed the engine-seeded `ContextMappings[0]`).
That ticket's primary defect — the off-by-one and the *unremovable engine-seeded
orphan* — was fixed by reclaiming the pristine seed on the first authored set, so
there is no longer an engine orphan to prune. This ticket carries the remaining,
separable capability the E ticket listed as optional: a typed verb to remove a
*caller-added* context after the fact. It is deliberately NOT gold-plated into the
reclaim fix (net-new API surface warrants its own validity pass).

## What it should do

Add a typed remove/clear verb to `audio.authoring`, e.g.:

- `audio.authoring.remove_dialogue_context` — remove a single mapping, addressed
  by index or by speaker voice path (and optionally target set), returning the new
  `contextCount`.
- and/or `audio.authoring.clear_dialogue_contexts` — drop all mappings (or reset to
  a single empty seed) on a wave.

Register both unconditionally (namespace must appear in the wiki) and gate the body
on `MCP_HAS_DIALOGUE`, mirroring `set_dialogue_context`.

## Acceptance

- Create a `DialogueWave`, add two contexts via `set_dialogue_context`, then remove
  one via the new verb — `describe_dialogue_wave` reports `contextCount:1` and the
  surviving mapping is the one that was kept (correct speaker/targets).
- A regression test drives the production remove/clear handler against real
  DialogueWave/DialogueVoice fixtures (EAContentExamples57 content or in-code
  fixtures — never Lyra) and asserts the live `ContextMappings` array after removal.

## History
- `#1-split-from-e-reclaim` `OPEN` reporter — Carved out of `E-dialogue-context-default-not-reclaimed` when that ticket's reclaim-on-first-set fix shipped. The E fix removed the engine-seeded orphan and the off-by-one; the remaining separable item is a typed verb to remove/clear a *caller-added* dialogue context, which the E ticket listed as an optional add-on. No such verb exists in `audio.authoring` today (`set_dialogue_context` appends or upserts-by-speaker only; `replace:true` cannot delete; `ContextMappings` is a nested-struct TArray a generic `property.set` cannot author). Low severity: rare edge path (only needed to undo an over-add) with a heavy recreate-the-asset workaround. Symmetric with the `remove_montage_slot` gap noted in `B-create-montage-duplicate-default-slot`.
</content>
</invoke>
