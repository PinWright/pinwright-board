---
id: B-source-effect-save-ignored
title: "Source-effect chain authoring ignores save and reports success after only marking the package dirty"
status: IN-REVIEW
severity: High
category: bug
tags: [audio, source-effect, persistence, save, false-success]
encounters: 1
lastSeen: 2026-09-03T23:08:58+03:00
---

# Source-effect chain writes are not saved to disk

## What happens

`audio.authoring.create_source_effect_chain` advertises `save:true` and reads it
into `bSave`, but never uses the value. It always calls `McpSafeAssetSave` and
returns success
(`X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Audio\AudioAuthoringHandler.cpp:2777-2818`).
`audio.authoring.add_source_effect` repeats the same mechanism after mutating the
chain at `:2834-2882`.

`McpSafeAssetSave` explicitly performs no disk write; it only marks the package
dirty and notifies the registry
(`X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Utils\AssetUtils.h:131-143`).
The response's `AddAssetVerification` can therefore disclose
`existsOnDisk:false` or `pendingSave:true` while the RPC itself remains green.

## Why it matters

The default contract says these authoring operations save. A caller can close the
editor after a successful response and lose the new chain or appended effect. It
also cannot request the documented `save:false` behavior because both values take
the same path. Severity is High for a silently dropped persistence parameter on
normal asset-authoring verbs.

## What should happen

Route both verbs through this file's existing `SaveAudioAsset(Asset, bSave)`
helper, which uses `SaveAssetToDiskReportingPresence`, and publish the result with
`AddAssetSaveReport`. Fail when `save:true` cannot establish disk presence; for
`save:false`, report the intentionally pending package without claiming a save.

## Workaround

Call `asset.save` for the returned asset path, or use the editor's Save All, before
closing the editor.

## Related

- Data-loss patterns `memory-state-masquerades-as-durable-save` and `accepted-parameter-silently-dropped`.
- False-success patterns `persistence-without-disk-proof` and `accepted-parameter-silent-noop`.
- `B-audio-create-save-no-disk-write` — fixed sibling routes through `SaveAudioAsset`.
- `F-source-effect-preset-authoring` — tracks the separate inert `effectType` input.

## Fix

The defect was TRUE: both handlers parsed `bSave` but unconditionally stopped at `McpSafeAssetSave`, so neither value selected a disk-write path. Both routes now mark the package dirty, thread `EAssetSaveState` through `SaveAudioAsset` / `SaveAssetToDiskReportingPresence`, emit the shared measured save report, return typed `SAVE_FAILED` with the mutation receipt when `save:true` is not durable, and leave `save:false` as an explicit dirty in-memory edit with `saveState:notRequested` and `pendingSave:true`.

Files changed: `Source/PinWright/Private/Handlers/Audio/AudioAuthoringHandler.cpp`, `Source/PinWright/Private/Tests/Media/TestSourceEffectSaveContract.cpp`, and `docs/wiki-src/audio.authoring.md`. Test ids: `PinWright.Audio.SourceEffectSave.CreateChainMeasuredPersistence` and `PinWright.Audio.SourceEffectSave.AddEffectRespectsSaveFlag`. Deliberately not changed: the separate informational `effectType` parameter or other audio-authoring callers of `SaveAudioAsset`; no build, editor, MCP, or automation run was performed under the worker brief.

## History
- `#1-filed-pattern-scan` `OPEN` reporter — Source-only scan confirmed both handlers read and ignore `bSave`, use the memory-only `McpSafeAssetSave`, and send success. The earlier audio-save ticket covers routes through `SaveAudioAsset`; these source-effect routes bypass that helper and were not covered. No build, test, editor, MCP call, plugin edit, commit, or repro was performed.
- `#2-honor-measured-save` `IN-REVIEW` developer — Routed both source-effect writes through measured disk persistence with `EAssetSaveState`, made non-durable requested saves typed failures, preserved explicit dirty-only behavior for `save:false`, documented the contract, and added handler-level disk/package-state tests. Source/static checks only; compile and automation remain for the checkpoint agent.
