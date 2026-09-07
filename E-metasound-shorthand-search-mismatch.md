---
id: E-metasound-shorthand-search-mismatch
title: "nodeType shorthand keyword and the registry node it adds are disjoint — searching the shorthand word ('Gain') never surfaces the node's real pins"
status: OPEN
severity: Low
category: ergonomic
tags: [docs, metasound, audio, authoring, add_metasound_node, search_metasound_nodes, node-discovery, connect_metasound_nodes]
encounters: 2
costly: 1
lastSeen: 2026-06-24T19:46:41Z
---

# The `gain` shorthand adds `UE.Multiply.Audio`, but `search_metasound_nodes query="Gain"` can't find it — pin discovery for the wire step breaks

`audio.authoring.add_metasound_node` accepts friendly `nodeType` shorthands
(`oscillator`, `gain`, `add`, `waveplayer`) and — after the resolver fix in
[`B-add-metasound-node-rejects-registry-classnames`](B-add-metasound-node-rejects-registry-classnames.md)
`#2` — they now resolve correctly (`gain` → `UE.Multiply.Audio by Float`). The
add succeeds. The friction is **downstream of the add**: to wire that node with
`connect_metasound_nodes` the caller must know its pin names, and the natural
discovery move — search the registry for the same word they just used — fails:

- `nodeType: "gain"` resolves to **`UE.Multiply.Audio`** (a Multiply node, pins
  `PrimaryOperand` = audio in, `AdditionalOperands` = float gain, `Out` = audio
  out).
- `search_metasound_nodes { query: "Gain" }` returns **only dB-converter nodes**
  — the Multiply node the `gain` shorthand actually added is **not in that result
  set**. The searchable display/category text for `UE.Multiply.Audio` does not
  contain the word "gain".
- So the caller cannot learn the gain node's pin names by searching the shorthand
  keyword. They must already know that `gain` ≡ Multiply and re-search
  `query: "Multiply"` to surface `UE.Multiply.Audio` and read its
  `PrimaryOperand`/`AdditionalOperands`/`Out` pins before wiring.

The shorthand vocabulary (`gain`) and the registry vocabulary (`Multiply`) are
**disjoint**, and the wiki never publishes the shorthand→className mapping that
would bridge them. The overlay `docs/wiki-src/audio.authoring.md:37` lists the
shorthand *labels* (`oscillator`, `gain`, `add`, `waveplayer`) and the
`UE.<Name>.<Output>` convention, but does **not** say which className each
shorthand expands to — so a caller can use the shorthand to add the node yet has
no documented path from "I added the `gain` node" to "its pins are
PrimaryOperand/AdditionalOperands/Out", and the obvious search term is a dead
end.

## Distinct from the existing MetaSound tickets

This is purely the **post-add pin-discovery** angle on a fully *clean* task —
every call passed first try, `validate_metasound` returned `valid:true`,
`describe_metasound` round-tripped. It is not:

- [`B-add-metasound-node-rejects-registry-classnames`](B-add-metasound-node-rejects-registry-classnames.md)
  (IN-REVIEW) — that is the *add resolver/shorthand* rejecting valid classNames;
  here the add and the shorthand both **work**.
- [`E-add-metasound-node-error-no-hint`](E-add-metasound-node-error-no-hint.md)
  (OPEN, deferred) — that enriches the `NODE_CLASS_NOT_FOUND` *error*; here no
  call ever errored, so no error path is involved.
- [`E-metasound-node-add-docs-misleading`](E-metasound-node-add-docs-misleading.md)
  (IN-REVIEW) — that corrects the typical-sequence example's wrong param
  names/className spellings; this is the missing **shorthand→className→pins**
  bridge, a different docs gap on the now-correct page.

## What it should do (downstream wiki edit, not mine)

Make the shorthand-to-node mapping and its pins discoverable so a caller never
has to guess that `gain` ≡ Multiply:

- In `docs/wiki-src/audio.authoring.md` (the `### audio.authoring.create_metasound`
  className-convention note at line 37), turn the bare shorthand list into a small
  table that pairs each `nodeType` shorthand with the className it resolves to AND
  its key pin names, e.g.:
  `oscillator` → `UE.Sine.Audio` (in `Frequency`/out `Audio`);
  `gain` → `UE.Multiply.Audio by Float` (audio in `PrimaryOperand`, gain in
  `AdditionalOperands`, out `Out`);
  `add` → `UE.Add.Float`; `waveplayer` → `UE.Wave Player.Mono`.
  Add one line: "the shorthand keyword is not the registry name —
  `search_metasound_nodes { query: \"gain\" }` will not find the Multiply node;
  search the className word (`Multiply`) instead, or read the pins from this
  table."
- Optionally fold the same gotcha into
  `docs/wiki-src/audio.authoring.metasound_gotchas.md`: "the friendly `nodeType`
  shorthands do not match `search_metasound_nodes` query text — a shorthand adds
  a node whose registry display name is a different word."

A cheaper, code-side alternative (out of scope for this docs ticket, note only):
have the `add_metasound_node` success response echo the resolved node's pin names,
so the caller never needs a second search to wire it.

## Evidence

From this task's friction note (`audio.authoring.describe_metasound` focus —
authoring `MS_AmbientDrone` end-to-end, outcome **clean**), verbatim:

> "Minor: the \"gain\" shorthand resolves to UE.Multiply.Audio by Float, but
> searching query=\"Gain\" returns only dB converter nodes (not that node), so I
> had to search \"Multiply\" to discover the gain node's real pin names
> (PrimaryOperand=audio in, AdditionalOperands=float gain, Out=audio out) before
> wiring; otherwise smooth, no retries or fallbacks."

Call log corroborates the extra discovery hop: three `search_metasound_nodes`
calls — `query=Sine` (found `UE.Sine.Audio`), `query=Gain` (only converters; the
gain DSP node not found), then `query=Multiply cat=Math` (found
`UE.Multiply.Audio` and its pins) — i.e. one search was spent solely because the
shorthand keyword does not match the searchable node name. Fully recoverable (the
task completed clean), hence Low severity, but every MetaSound author wiring a
`gain` node pays the same extra-search tax until the mapping is documented.

## History
- `#1-initial-audit` `OPEN` reporter — PROCESS/docs friction from the clean `audio.authoring.describe_metasound` fuzz task (authored MS_AmbientDrone end-to-end; every call first-try, validate clean, describe round-trips). The `nodeType: "gain"` shorthand correctly adds `UE.Multiply.Audio by Float` (resolver fix from B-add-metasound-node-rejects-registry-classnames), but `search_metasound_nodes { query: "Gain" }` returns only dB-converter nodes — NOT the Multiply node the shorthand added — so the caller could not discover that node's pins (PrimaryOperand/AdditionalOperands/Out) by searching the shorthand word, and had to know `gain` ≡ Multiply and re-search `query: "Multiply"` (one extra search hop in the call log). The shorthand vocabulary and registry vocabulary are disjoint and the wiki (`docs/wiki-src/audio.authoring.md:37`) lists the shorthand labels but never the className each maps to or its pins. Distinct from B-add-metasound-node-rejects-registry-classnames (add works here), E-add-metasound-node-error-no-hint (no error here), and E-metasound-node-add-docs-misleading (typical-sequence param spellings) — this is the post-add pin-discovery / shorthand→className→pins bridge gap. Proposes a shorthand→className→pins table in `docs/wiki-src/audio.authoring.md` (and optionally the gotchas overlay), with a note that the shorthand keyword is not a `search_metasound_nodes` query term; cheaper code-side alternative noted only (echo resolved pins in the add response).
- `#2-recurs-on-patch-gain-task` `OPEN` reporter — Cross-task recurrence (independent corroboration) from the `audio.authoring.create_metasound_patch` "GainStage_Patch" fuzz task (outcome tool_bug — the populate was blocked by B-metasound-patch-mutators-reject, but this search friction is orthogonal and was hit first). The caller needed the gain node's className/pins to wire it; searched the shorthand word and adjacent terms across SIX `search_metasound_nodes` calls — `query=gain`, `query=Audio Gain`, `category=Dynamics`, `query=gain incl-deprecated`, `category=Mix`, and a converter-only result — none surfaced the gain DSP node. Friction note verbatim: "'gain' returns only dB/linear converter utilities in search (the real gain node is 'UE.Multiply.Audio by Float', discoverable only by reading the handler's shorthand map)." Same disjoint-vocabulary gap as `#1`, but worse than the one-extra-hop seen there: the caller burned five fruitless searches and only confirmed `gain` ≡ `UE.Multiply.Audio by Float` by reading the plugin C++ shorthand map (last resort) — there is no documented or searchable path from the `gain` keyword to the node. Reinforces the proposed shorthand→className→pins table fix; this second task makes the "every MetaSound author wiring gain pays this tax" claim concrete (6 wasted searches + a source dive).
