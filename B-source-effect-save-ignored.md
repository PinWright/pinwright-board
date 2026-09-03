---
id: B-source-effect-save-ignored
title: "Source-effect chain authoring ignores save and reports success after only marking the package dirty"
status: OPEN
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

## History
- `#1-filed-pattern-scan` `OPEN` reporter — Source-only scan confirmed both handlers read and ignore `bSave`, use the memory-only `McpSafeAssetSave`, and send success. The earlier audio-save ticket covers routes through `SaveAudioAsset`; these source-effect routes bypass that helper and were not covered. No build, test, editor, MCP call, plugin edit, commit, or repro was performed.
