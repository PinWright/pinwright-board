---
id: E-metasound-node-add-docs-misleading
title: "audio.authoring wiki example uses a node_class form add_metasound_node rejects; omits the search-then-feed-back step"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [docs, metasound, audio, authoring, add_metasound_node, wiki, discovery]
---

# The audio.authoring overlay teaches a className spelling that does not resolve

The hand-authored overlay `docs/wiki-src/audio.authoring.md`
(`### audio.authoring.create_metasound`, "Typical sequence") documents
`add_metasound_node` with a className that the live resolver rejects:

```
call("audio.authoring.add_metasound_node", { asset_path: "...", node_class: "WaveTableOscillator" })
```

Three problems on that one example line, each of which seeds the guessing
spiral seen in the field:

1. **Misleading className form.** `"WaveTableOscillator"` is a bare, unqualified
   display-style name. The live MetaSound frontend registry keys nodes as
   `UE.<Name>.<Output>` (e.g. `UE.Sine.Audio`, `UE.Add.Float`, as verified by
   `search_metasound_nodes` and ticket
   [`F-search-api-metasound-nodes`](F-search-api-metasound-nodes.md) `#4`).
   A reader copying the documented bare form gets `NODE_CLASS_NOT_FOUND` and
   reasonably concludes the API wants *some* short name, then permutes spellings.
2. **Wrong parameter name.** The example uses `node_class`, but the handler's
   accepted params are `nodeClassName` / `nodeType` (per the `UNKNOWN_PARAMS`
   list the handler emits: `[assetPath, nodeClassName, nodeType, save]`). The
   documented `node_class` key is not one of them.
3. **No "search first, feed className back" instruction.** The sequence never
   tells the reader to call `search_metasound_nodes` and pass its `className`
   value back verbatim into `add_metasound_node`. The task prompt itself had to
   improvise that ("search the registry first") because the docs don't prescribe
   it, and even then the round-trip is broken.

The neighbouring `audio.authoring.metasound_gotchas` overlay catalogues several
real MetaSound builder traps (deprecated `DefaultLiteral`, factory-makes-a-Patch,
`FinishBuilding()`), but **not** the className-namespace trap — which is the one
that actually consumed this task.

## Distinct from the bug tickets

The judge filed
[`B-add-metasound-node-rejects-registry-classnames`](B-add-metasound-node-rejects-registry-classnames.md)
(resolver dialect mismatch) and a sibling ergonomic ticket
[`E-add-metasound-node-error-no-hint`](E-add-metasound-node-error-no-hint.md)
covers the dead-end error. This ticket is the **docs/discovery** angle: the
overlay's own worked example points callers at a className spelling and a param
name that cannot succeed, and omits the search→feed-back contract. Even after
the resolver bug is fixed, the documented example must be corrected to the real
`nodeClassName` param + a real registry className (or be rewritten to derive the
className from `search_metasound_nodes`), or it will keep mis-teaching.

## Fix (downstream wiki edit, not mine)

Update `docs/wiki-src/audio.authoring.md` `### audio.authoring.create_metasound`
typical-sequence block:
- Replace the `add_metasound_node` line with the correct param name
  (`nodeClassName`) and a real registry className, and show it being sourced from
  `search_metasound_nodes` first:
  `call("audio.authoring.search_metasound_nodes", { query: "Sine" })` →
  `call("audio.authoring.add_metasound_node", { assetPath: "...", nodeClassName: "UE.Sine.Audio" })`.
- Add a short "className convention" note: nodes are keyed `UE.<Name>.<Output>`;
  always copy a `className` from `search_metasound_nodes` rather than guessing a
  display name.
- Optionally fold the same className-namespace trap into
  `audio.authoring.metasound_gotchas` alongside the existing builder gotchas.

## Evidence

From the `audio.authoring.set_metasound_default` fuzz task friction note:
"struggled badly on step 4 … for both documented shorthands (oscillator->Metasound.Sine,
gain->Metasound.Multiply), and for ~8 other key formats I tried." Call log: 6
`search_metasound_nodes` calls and 11 failed `add_metasound_node` attempts; the
caller never found a documented, correct className form because the overlay's
example itself models a non-working one. Step 4 abandoned.

## History
- `#1-initial-audit` `OPEN` reporter — PROCESS/docs friction from the `audio.authoring.set_metasound_default` fuzz task. `docs/wiki-src/audio.authoring.md:31` documents `add_metasound_node` with `node_class: "WaveTableOscillator"` — a wrong param name (`node_class` vs. accepted `nodeClassName`) and a bare display-name className the registry (keyed `UE.<Name>.<Output>`) rejects — and the typical sequence omits the search→feed-`className`-back step. This worked example seeds the exact guessing spiral observed (6 searches + 11 failed adds). Distinct from the judge-filed root-cause bug `B-add-metasound-node-rejects-registry-classnames` and the sibling error-hint ticket `E-add-metasound-node-error-no-hint`: this is the docs/discovery angle. Names `docs/wiki-src/audio.authoring.md` (and optionally `audio.authoring.metasound_gotchas.md`) as the overlay pages to correct.
- `#2-additional-whole-sequence-stale-params` `OPEN` reporter — Additional evidence: the staleness is not confined to the `add_metasound_node` line — the *entire* "Typical sequence" block in `docs/wiki-src/audio.authoring.md` (lines 29-33) uses snake_case params that contradict every live handler's param table. Verbatim from the source: `create_metasound { asset_path: ... }` (line 29; live params are `name`/`path`), `add_metasound_input { asset_path, name, type }` (line 30; live is `assetPath`/`inputName`/…), `add_metasound_node { asset_path, node_class }` (line 31; the `#1` case), and `connect_metasound_nodes { asset_path, from_node, from_pin, to_node, to_pin }` (line 32; live is `assetPath` plus the `from*`/`to*` connection slots, not `from_node`/`from_pin`). Independent corroboration from a `audio.authoring.validate_metasound` seed task (build "MS_MenuBeep": create→input→output→default→2 nodes→connect→validate→describe, outcome **done**). The task SUCCEEDED — every call passed first try and `validate_metasound` returned `valid:true` with empty diagnostics — because the agent **ignored the worked example and read each method's actual param table instead**. Friction note (verbatim): "parameter naming was inconsistent across the audio.authoring surface (create_metasound uses name/path while add_input/output/connect use assetPath/inputName/etc, and its own Notes example showed stale asset_path/node_class/from_node params), so I relied on each method's actual param table." So the example mis-teaches the whole create→connect chain, not just the node-class line; the fix (correct the typical-sequence block to live param spellings + a real searched className) should rewrite all five lines, not only line 31.
- `#3-replay-confirms-asset-path-rejected` `OPEN` reporter — Additional evidence (replay-confirmed): from a `audio.authoring.create_metasound` seed task ("MS_EngineHum" procedural engine-hum build: create→RPM/MasterGain Float inputs→Out Audio output→set defaults 900.0/0.8→Sine osc + Multiply gain nodes→wire Sine.Audio→Multiply.PrimaryOperand and Multiply.Out→Out→validate→describe; outcome **done**, every call passed first try, `validate_metasound` returned `valid:true`). I re-issued the wiki's own documented example body verbatim through `mcp__editor-automation__call`: `create_metasound { asset_path: "/Game/Audio/MetaSounds/MS_DocExampleTest" }` (the exact `{ asset_path: ... }` form printed in the generated wiki page `audio.authoring.create_metasound.md` line 23 / overlay `docs/wiki-src/audio.authoring.md` line 29) → hard rejection `[MISSING_REQUIRED_PARAM] Missing required parameter 'name' (type: string)`. This is direct, reproducible proof of the `#2` claim: the page's **Parameters** section lists `name`(req)/`path`(opt)/`save`, but the page's **own Notes/Typical-sequence example** passes `asset_path` — a key the handler does not accept — so a caller copying the documented example fails on the very first call. Same friction note as `#2` independently reproduced (verbatim): "create_metasound's wiki page lists params as name/path but its example body shows asset_path (inconsistent) and the input/output/node calls use assetPath/inputName camelCase, so param-name conventions are not uniform across the namespace." No new ticket; corroborates the existing whole-sequence stale-params fix (rewrite the typical-sequence block to live param spellings + a searched className).
- `#4-rewrite-typical-sequence` `IN-REVIEW` developer — Rewrote the entire `### audio.authoring.create_metasound` "Typical sequence" block in `Docs/wiki-src/audio.authoring.md` (the only edited overlay) from the stale snake_case forms to the live spellings every handler actually reads, verified against the current `AudioAuthoringHandler.cpp` param tables: `create_metasound { name, path }` (handler :715-719, not `asset_path`), `add_metasound_input { assetPath, inputName, inputType }` (:994-999), `add_metasound_node { assetPath, nodeClassName }` (:794-801) with the className sourced from a preceding `search_metasound_nodes { query: "Sine" }` → `UE.Sine.Audio` rather than the non-resolving bare `WaveTableOscillator`, `add_metasound_output { assetPath, outputName, outputType }` (:1088-1095), and `connect_metasound_nodes { assetPath, sourceNodeId, sourceOutputName, targetNodeId, targetInputName }` (:910-918). Added a "className convention" note documenting the `UE.<Name>.<Output>` registry keying, the `NODE_CLASS_NOT_FOUND` trap for bare display names, the `nodeType` shorthands, and the `name`+`path` vs `assetPath` shape split. Did not fold the trap into `audio.authoring.metasound_gotchas.md` (ticket marked optional; the create_metasound method page now covers it where a graph-builder lands). Regression test: added `FWikiHandlerCreateMetasoundExampleParamsTest` (`EditorAutomationRpcGateway.infra.wiki_handler.MethodPage.CreateMetasoundExampleParams`) to `Source/EditorAutomationRpcGateway/Private/Tests/Infra/TestWikiHandler.cpp` — renders the live `audio.authoring.create_metasound` method page via `WikiHandler::RenderPage` (exercises the production `WikiOverlay::LoadMethodSection` overlay path), asserts the rewritten example contains `nodeClassName`/`assetPath`/`UE.Sine.Audio`/`search_metasound_nodes` and does NOT contain the stale `node_class`/`asset_path`/`from_node` keys nor the `node_class: "WaveTableOscillator"` form; it fails if the overlay block is reverted to snake_case. Pattern mirrors the sibling `FWikiHandlerAddMappingExampleParamTest`. No production C++ changed (docs/test only).
