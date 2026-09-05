---
id: B-sequencer-fbx-stale-export
title: "sequencer.export_fbx can report success for a stale destination file after the engine ignores a failed write"
status: IN-REVIEW
severity: High
category: bug
tags: [sequencer, fbx, export, overwrite, artifact, false-success]
encounters: 1
lastSeen: 2026-09-03T23:08:58+03:00
---

# A failed FBX write can be mistaken for a successful export

## What happens

`sequencer.export_fbx` writes directly to the caller's final path, then accepts
the export when the engine returned true and any non-empty file exists there
(`X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Sequencer\SequencerFbxHandler.cpp:188-238`).
The engine implementation calls the void `Exporter->WriteToFile` API, has no
write result to observe, and returns true unconditionally
(`C:\UE_5.8\Engine\Source\Editor\MovieSceneTools\Private\MovieSceneToolHelpers.cpp:4078-4111`);
`USequencerToolsFunctionLibrary` relays that result
(`C:\UE_5.8\Engine\Plugins\MovieScene\SequencerScripting\Source\SequencerScriptingEditor\Private\SequencerTools.cpp:329-352`).

If a non-empty destination already exists and the replacement write fails, for
example because the file is read-only or locked, the stale file satisfies the
handler's size check and the RPC reports a fresh successful export.

## Why it matters

The caller can publish or import an old FBX while being told the current sequence
was exported. Direct writes also give no explicit overwrite policy or rollback
boundary. This is High: it is a silent wrong artifact on a normal export path.

## What should happen

Use the catalog's safe output-replacement shape: require an explicit overwrite
decision for an existing destination, export to a unique sibling temporary path,
verify that new file's existence and non-zero size, and atomically replace the
final path only after verification. Report measured final size and freshness (or
hash), and clean up only the task-owned temporary file on failure.

## Workaround

Export to a new, nonexistent filename and independently verify its modification
time or content before using it.

## Related

- Data-loss pattern `unsafe-output-replacement-or-collision`.
- False-success patterns `artifact-structure-not-content` and `request-echo-not-result-readback`.
- `F-sequencer-fbx-roundtrip` — feature ticket for these verbs, not stale-file detection.

## History
- `#1-filed-pattern-scan` `OPEN` reporter — Source-only scan traced the handler's direct final-path write and post-call size probe to the engine exporter, where the void `WriteToFile` API can return early after initialization failure and the wrapper still returns true. Board search found no ticket for stale FBX output or overwrite safety. No build, test, editor, MCP call, plugin edit, commit, or repro was performed.
- `#2-staged-atomic-export` `IN-REVIEW` developer — Changed `SequencerFbxHandler.cpp` to refuse existing outputs by default, export to a unique sibling stage, verify its bytes, and publish through `AtomicFileWriter`; a failed publish leaves the destination untouched and returns an error. Added handler-level transient-sequence coverage in `PinWright.Sequencer.FbxExport.AtomicStaleDestination`; no build, editor, MCP call, or test run was performed under the worker restriction.

## Fix

The engine's export helper reports true after a void file-write call, and the handler previously validated the caller's final path, allowing old bytes to masquerade as the new export. The handler now exports only to a task-owned sibling path, loads and verifies the staged bytes, then publishes them through `AtomicFileWriter`; existing destinations require `overwrite:true`, publication failures preserve the old file, and success reports final size plus `replaced`.

Files changed: `Source/PinWright/Private/Handlers/Sequencer/SequencerFbxHandler.cpp`, `Source/PinWright/Private/Tests/Sequencer/TestSequencerFbxOutcomeHonesty.cpp`, and `docs/wiki-src/sequencer.md`. Test id: `PinWright.Sequencer.FbxExport.AtomicStaleDestination`.

Deliberately not changed: `AtomicFileWriter` itself and Unreal's FBX exporter remain untouched; the handler composes the shared stage-verify-publish contract around the engine API.
