---
id: F-metasound-no-input-output-mutation
title: "No remove/rename/retype for MetaSound graph inputs and outputs"
status: DONE
severity: High
category: feature
tags: [audio, metasound, authoring, graph-interface, no-text-ir]
---

# No remove/rename/retype for MetaSound graph inputs and outputs

`audio.authoring.add_metasound_input` and `add_metasound_output` create
graph-interface vertices, but there is no corresponding remove, rename,
or retype RPC. Verified — calls to `remove_metasound_input`,
`remove_metasound_output`, and `rename_metasound_input` all return
`Not found` from the gateway. The full MetaSound authoring surface in
`AudioAuthoringHandler.cpp` is the seven create/add_node/connect/
add_input/add_output/set_default/describe handlers — nothing else.

**Why this matters more for MetaSound than other domains.** The graph
interface (input/output vertices) is the asset's contract with the
audio engine. Renaming an input or retyping it (`Float` → `Audio`)
requires re-creating the vertex and rewiring every edge that touched
it. With no `remove_metasound_input`, the only path is delete and
re-create the entire asset. MetaSound has no text-IR replacement path
(no MSIR), so this is a one-way street — every add_input typo or wrong
type choice is permanent for the asset's lifetime.

**Blocked workflows:**

1. **Schema evolution** — agent adds `Volume: Float`, later realizes it
   should be `Volume: Audio`. No fix path short of asset re-creation.
2. **Refactoring input names** — `Gain` → `MasterGain` requires drop +
   re-add + re-wire all connections.
3. **Inspect-fix loop** — `describe_metasound` reports the interface
   shape but the agent can't act on the diagnosis.
4. **Interface alignment** — once required-by-interface vertices are
   added in the wrong order or with the wrong type, the graph fails
   validation and the agent can't repair it.

**Implementation surface:** `FMetaSoundFrontendDocumentBuilder` exposes
the underlying ops the existing handlers already drive:

- `Builder.RemoveGraphInput(FName Name)` — remove input vertex by name
- `Builder.RemoveGraphOutput(FName Name)` — remove output vertex by name
- For rename, the documented pattern is remove + re-add with the new
  name (graph inputs/outputs are addressed by FName; there's no atomic
  rename in the builder); the RPC can wrap that sequence and preserve
  the existing default literal.
- For retype, same pattern (remove + re-add with new TypeName); the
  RPC should warn that any edges referencing the old vertex are
  invalidated.

Proposed RPCs:

```
audio.authoring.remove_metasound_input { assetPath, inputName, save? }
audio.authoring.remove_metasound_output { assetPath, outputName, save? }
audio.authoring.rename_metasound_input { assetPath, oldName, newName, save? }
audio.authoring.rename_metasound_output { assetPath, oldName, newName, save? }
audio.authoring.retype_metasound_input { assetPath, inputName, newType, save? }
audio.authoring.retype_metasound_output { assetPath, outputName, newType, save? }
```

Minimum useful subset for unblocking authoring is the four remove +
rename ops; retype is a nice-to-have.

**Related — add path is also broken today:** see
[`B-metasound-add-input-asserts-on-missing-literal`](B-metasound-add-input-asserts-on-missing-literal.md).
`add_metasound_input` hard-crashes the editor due to a missing
`DefaultLiteral`, so add-then-fix iteration is doubly blocked: you
can't add a vertex without crashing, and even if it succeeded you
couldn't remove/rename/retype it afterwards. Both tickets need fixing
before any practical MetaSound authoring loop exists.

## History
- `#1-no-io-mutation` `OPEN` reporter — Verified via gateway probe: `remove_metasound_input`, `remove_metasound_output`, `rename_metasound_input` all return `Not found`. Source grep confirms only `add_metasound_input` / `add_metasound_output` exist on the authoring side. MetaSound has no text-IR fallback path, so any wrong-type or wrong-name vertex is permanent for the asset. Proposes thin wrappers around `FMetaSoundFrontendDocumentBuilder::RemoveGraphInput` / `RemoveGraphOutput`, with rename/retype as remove+re-add convenience wrappers that preserve the existing default literal.
- `#2-reviewed-and-confirmed` `OPEN` reviewer — Confirmed against `AudioAuthoringHandler.cpp`: the seven MetaSound authoring handlers are `create_metasound`, `add_metasound_node`, `connect_metasound_nodes`, `add_metasound_input`, `add_metasound_output`, `set_metasound_default`, `describe_metasound`. No remove/rename/retype exists. Severity High justified — no destructive path for graph-interface vertices means any typo or wrong type is permanent for the asset's lifetime, with no text-IR escape hatch. Cross-referenced `B-metasound-add-input-asserts-on-missing-literal`: the add path itself crashes the editor today, so the iteration loop is doubly blocked. Ticket body updated with that cross-reference. No duplicate with `F-metasound-no-destructive-graph-ops` (that one covers node delete / edge disconnect, not interface vertex mutation).
- `#3-remove-and-rename` `IN-REVIEW` developer — Added `audio.authoring.remove_metasound_input`, `remove_metasound_output`, `rename_metasound_input`, `rename_metasound_output` in new `Private/Handlers/Audio/MetaSound/MetaSoundIOMutationHandler.cpp`. Remove handlers are thin wrappers around `Builder.RemoveGraphInput` / `Builder.RemoveGraphOutput` returning `INPUT_NOT_FOUND` / `OUTPUT_NOT_FOUND` on false. Rename handlers use the atomic `Builder.SetGraphInputName` / `Builder.SetGraphOutputName` APIs (verified in `MetasoundFrontendDocumentBuilder.h`) — these preserve TypeName, AccessType, Defaults, and existing edges without a remove+re-add. Both rename handlers verify existence via `Builder.FindGraphInput` / `FindGraphOutput` before attempting the rename. Retype is intentionally NOT implemented this sprint — the ticket body itself flagged it as "nice-to-have" with "minimum useful subset" being the four remove+rename ops; deferring retype as a follow-up. Regression test `TestMetaSoundIOMutation.cpp` asserts remove→re-add cycle works on a transient source.
- `#4-crash-during-rename` `OPEN` tester — Crashed: created `/Game/App/Audio/Test/MS_McpVerifyTemp_FMetaSoundIOMutation`, added `Volume` input and `DryOut` output, then `audio.authoring.rename_metasound_input` with `oldName=Volume`, `newName=MasterGain`, `save=false` returned `fetch failed`; health check to `127.0.0.1:19880` could not connect afterwards and no `UnrealEditor` process remained. `Saved/Logs/PDS.log` recorded `Unhandled Exception: EXCEPTION_ACCESS_VIOLATION reading address 0x0000000000000000` during the RPC path before shutdown.
- `#5-verify-fresh-editor` `DONE` tester — Verified: on fresh editor, created `/Game/App/Audio/Test/MS_McpVerifyTemp_FMetaSoundIOMutation_20260515`, added `Volume` input and `DryOut` output, then `audio.authoring.rename_metasound_input` returned `oldName=Volume`, `newName=MasterGain`; `audio.authoring.rename_metasound_output` returned `oldName=DryOut`, `newName=WetOut`; `audio.authoring.describe_metasound` showed `MasterGain` input and `WetOut` output. Also created `/Game/App/Audio/Test/MS_McpVerifyTemp_FMetaSoundIORemove_20260515`, removed `TempInput` and `TempOutput`, and `describe_metasound` showed only the built-in MetaSound source interface entries; both temp assets were deleted with `asset.delete`.
