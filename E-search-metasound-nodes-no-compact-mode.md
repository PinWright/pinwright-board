---
id: E-search-metasound-nodes-no-compact-mode
title: "audio.authoring.search_metasound_nodes has no compact mode — enum-heavy generator queries overflow the 10000-char display limit and spill to disk"
status: OPEN
severity: Low
category: ergonomic
tags: [metasound, audio, authoring, search_metasound_nodes, response-size, oversized-readback, compact, node-discovery, docs]
encounters: 1
lastSeen: 2026-07-01T18:40:28.0956489+03:00
---

# search_metasound_nodes returns full per-node vertex lists with fully-expanded enum variants, so a common generator query overflows the inline limit

`audio.authoring.search_metasound_nodes` (added by
[`F-search-api-metasound-nodes`](F-search-api-metasound-nodes.md), DONE) emits,
for every matched class, the full input/output vertex lists with each pin's type
fully expanded — including enum variants such as
`"type":"Enum:SineGenerationType:Variable"`. There is **no `compact`/summary
mode** and the default `limit` is 50, so a common discovery query fans out to a
large payload:

- `query="Sine"` matched 12 classes and produced **16902 chars**, blowing the
  **10000-char MCP display threshold**; the full payload was written to a
  `Saved/.../HttpResponses/<…>.json` file that the agent then had to `Read` to
  recover the `className`/pin names it needed to add and wire the oscillator.
- The sibling `query="Multiply"` in the same task returned only 5 matches and
  fit inline fine — so the overflow is driven purely by result-set size ×
  per-node verbosity, not by anything the caller can pre-narrow.

This is the same oversized-readback ergonomic shape the board already fixed for
its neighbouring structural-dump methods with an opt-in size knob:

- [`E-describe-metasound-no-compact-mode`](E-describe-metasound-no-compact-mode.md)
  (IN-REVIEW) — added `compact`/`nodeIds` to `describe_metasound` so the routine
  confirm-the-graph readback stays inline instead of spilling to disk.
- [`E-widget-export-xml-token-limit`](E-widget-export-xml-token-limit.md) (DONE)
  — `compact`/`omit_slot_chain` on `widget.export_xml`.
- [`E-graph-connections-pagination`](E-graph-connections-pagination.md) (DONE) —
  `nodeIds`/`maxEdges` on `blueprint.graph.get_graph_connections`.

`search_metasound_nodes` is the **node-discovery member** of that family with no
such mode: node discovery is the first move of nearly every MetaSound-authoring
task, and any generator/oscillator/math query with a dozen matches will overflow.

## Distinct from neighbouring MetaSound tickets

- [`E-describe-metasound-no-compact-mode`](E-describe-metasound-no-compact-mode.md)
  — same oversized-readback shape but a **different method** (`describe_metasound`
  dumps one graph's nodes; this dumps a registry-search result set). The board
  files this per-method (cf. the `*-no-limit-spills` family), so this is the
  `search_metasound_nodes` instance, not a dup.
- [`E-metasound-shorthand-search-mismatch`](E-metasound-shorthand-search-mismatch.md)
  (OPEN) — the *content* gap (the `gain` shorthand keyword doesn't match the
  Multiply node's searchable text). This ticket is the orthogonal *size* gap: the
  result you do get back is too big to display inline.
- [`F-search-api-metasound-nodes`](F-search-api-metasound-nodes.md) (DONE) —
  *added* the RPC; this is the follow-on ergonomic that its always-full,
  enum-expanded result set is an awkward MCP readback.

## What it should do (downstream fix, not mine)

Give `search_metasound_nodes` an opt-in size knob mirroring the DONE precedents,
one or more of:

- `compact: true` (default `false`, no change for existing callers) — per result
  emit only `className` + `displayName` + a flat `name:type` pin list, dropping
  the full enum-variant expansion (`Enum:SineGenerationType:Variable` → `Enum`),
  so the typical discovery search stays inline.
- A smaller default `limit` (e.g. 10) for the interactive/MCP path so a broad
  query doesn't return 50 fully-expanded classes by default.
- A count/summary header so a caller can decide whether to fetch full detail.

Implementation surface: `Private/Handlers/Audio/MetaSound/MetaSoundSearchHandler.cpp`
(the handler `F-search-api-metasound-nodes` `#3` added). Default behavior stays
the full result for parity.

## Evidence

Clean/done fuzz task (focus `audio.authoring.add_metasound_input`, built
`/Game/Audio/MS_VehicleEngine` end-to-end — every call first-try, judge filed
nothing on the outcome). CallAnalyzer call-trace finding, verbatim:

> CALL search query="Sine" -> RESULT {"outputTooLong":true,"message":"Response
> exceeds display limit (16902 chars, threshold 10000); full payload written to
> ...HttpResponses/...json"}. … Contrast: the sibling CALL search query="Multiply"
> returned only 5 matches and fit inline fine. The Sine overflow is driven by
> verbose per-node input/output lists that fully expand enum type variants (e.g.
> "type":"Enum:SineGenerationType:Variable"), inflating 12 matches to 16.9 KB.

Agent friction note, verbatim:

> "the Sine search response overflowed the 10k display limit and had to be read
> from the written HttpResponses file"

Fully recoverable (read off disk), hence Low severity — but every MetaSound
author whose discovery query returns a dozen enum-heavy nodes pays a disk
round-trip on the very first search of the build.

severity rationale: impact=response-spill (recovers via Read) × reach=metasound-authoring node-discovery (not every editor session) -> Low

## History
- `#1-initial-audit` `OPEN` reporter — PROCESS friction from the clean `audio.authoring.add_metasound_input` fuzz task (built MS_VehicleEngine end-to-end; every call first-try, judge filed nothing on outcome). `audio.authoring.search_metasound_nodes` has no compact/summary mode and a default `limit` of 50, and emits full per-node vertex lists with fully-expanded enum type variants (`Enum:SineGenerationType:Variable`), so `query="Sine"` (12 matches) produced 16902 chars > the 10000-char MCP display threshold and spilled to a Saved/.../HttpResponses JSON file the agent had to Read to recover the className/pins; the sibling `query="Multiply"` (5 matches) fit inline. Same oversized-readback shape already given an opt-in `compact`/`nodeIds` knob for `describe_metasound` (`E-describe-metasound-no-compact-mode`, IN-REVIEW) and DONE for `widget.export_xml` (`E-widget-export-xml-token-limit`) and `get_graph_connections` (`E-graph-connections-pagination`); `search_metasound_nodes` is the node-discovery member of the family with no such mode. Distinct from `E-metasound-shorthand-search-mismatch` (search *content* gap) and `F-search-api-metasound-nodes` (added the RPC). Proposes `compact:true` (className+displayName+flat name:type pins, dropping enum-variant expansion) and/or a smaller default `limit` on `Private/Handlers/Audio/MetaSound/MetaSoundSearchHandler.cpp`. Low severity — recovers via disk Read but taxes the first discovery search of every enum-heavy MetaSound build.
