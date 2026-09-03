---
id: B-sequencer-fbx-import-noop
title: "sequencer.import_fbx reports success when no FBX node matches a target binding and no animation keys are written"
status: OPEN
severity: High
category: bug
tags: [sequencer, fbx, import, bindings, no-op, false-success]
encounters: 1
lastSeen: 2026-09-03T23:08:58+03:00
---

# A valid FBX with no matching bindings is a successful no-op

## What happens

`sequencer.import_fbx` validates only that the file is non-empty and that target
bindings exist, then returns success whenever the engine call returns true
(`X:\src\unreal\unreal-fpv-dev\Plugins\PinWright\Source\PinWright\Private\Handlers\Sequencer\SequencerFbxHandler.cpp:275-329`).
With the default `matchByNameOnly:true`, the engine imports a node only on an exact
name match, logs unmatched nodes, and returns true unconditionally
(`C:\UE_5.8\Engine\Source\Editor\MovieSceneTools\Private\MovieSceneToolHelpers.cpp:3839-3943`).

A valid FBX whose animated node names differ from the selected binding display
names therefore writes no keys but produces an RPC response containing
`success:true` and only the requested binding count.

## Why it matters

Automation cannot distinguish a completed import from a no-op without separately
enumerating every affected track and key. It may save or render the unchanged
sequence believing animation was imported. This is High because the default name
matching path silently returns the wrong result.

## What should happen

Precompute and publish the node-to-binding match set, or snapshot relevant key
counts before and after the engine call. Return `NO_BINDINGS_MATCHED` when the
match set is empty and `NO_KEYS_WRITTEN` when execution produces no new or replaced
keys. On success, report matched bindings, unmatched FBX nodes, and measured key
changes; do not use the request's binding count as proof.

## Workaround

Make FBX node names exactly match sequence binding display names, then read the
target tracks and keys after import instead of trusting the success flag.

## Related

- False-success patterns `partial-nonatomic-success`, `request-echo-not-result-readback`, and `artifact-structure-not-content`.
- Data-loss pattern `terminal-success-before-completion-or-invariant`.
- `F-sequencer-fbx-roundtrip` — feature ticket that introduced the route, not no-op detection.

## History
- `#1-filed-pattern-scan` `OPEN` reporter — Source-only scan followed the handler through the engine's exact-name loop, unmatched-node warning, and unconditional true return, confirming a valid no-match FBX produces no mutation but a green RPC. Board search found no ticket for this mechanism. No build, test, editor, MCP call, plugin edit, commit, or repro was performed.
