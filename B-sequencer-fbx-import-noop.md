---
id: B-sequencer-fbx-import-noop
title: "sequencer.import_fbx reports success when no FBX node matches a target binding and no animation keys are written"
status: IN-REVIEW
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
- `#2-measured-import-outcome` `IN-REVIEW` developer — Changed `SequencerFbxHandler.cpp` to snapshot MovieScene track, section, key, and signed-object state before and after the engine import, cancel and return `NOTHING_IMPORTED` when unchanged, and report the measured counts on both failure and success. Added handler-level transient-sequence coverage in `PinWright.Sequencer.FbxImport.NoOpRejected`; no build, editor, MCP call, or test run was performed under the worker restriction.

## Fix

The engine's import helper returns true after warning that no FBX nodes matched, so the handler's boolean was not evidence of a mutation. The handler now measures all MovieScene root, camera-cut, and binding tracks around the call, including track/section counts, authored key counts, and signed-object identities; an unchanged result cancels the transaction and returns `NOTHING_IMPORTED` with before/after counts, while success carries the same measurements.

Files changed: `Source/PinWright/Private/Handlers/Sequencer/SequencerFbxHandler.cpp`, `Source/PinWright/Private/Handlers/ErrorCodes.h`, `Source/PinWright/Private/Tests/Sequencer/TestSequencerFbxOutcomeHonesty.cpp`, and `docs/wiki-src/sequencer.md`. Test id: `PinWright.Sequencer.FbxImport.NoOpRejected`.

Deliberately not changed: PinWright does not parse the FBX node map itself because that state is private to the engine importer; post-call MovieScene measurement covers exact-name and fallback matching without duplicating Unreal's FBX parser.
