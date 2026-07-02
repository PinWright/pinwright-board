---
id: E-connect-metasound-graph-input-pin-docs
title: "connect_metasound_nodes: a graph-input vertex's output pin name (sourceOutputName) equals the input's own name — undocumented, so callers guess"
status: OPEN
severity: Low
category: ergonomic
tags: [docs, metasound, audio, authoring, connect_metasound_nodes, add_metasound_input, node-discovery]
encounters: 1
lastSeen: 2026-07-01T18:40:28.0956489+03:00
---

# Wiring a graph input into a node requires knowing the input vertex's output pin name — which is the input name itself, but that convention is undocumented

To wire a graph **input** (created by `add_metasound_input`) into a node with
`connect_metasound_nodes`, the caller passes `sourceNodeId` = the input's
returned `nodeId` and `sourceOutputName` = the pin the input value comes out on.
That pin name **equals the input's own name** (e.g. input `RPM` → source pin
`"RPM"`, input `Volume` → source pin `"Volume"`), but nothing documents this:

- The generated `connect_metasound_nodes.md` describes `sourceOutputName` only as
  "Output pin name on source" — with no statement that a **graph-input vertex**
  exposes its value on an output pin named identically to the input.
- `docs/wiki-src/audio.authoring.md`'s rewritten typical-sequence block (see
  [`E-metasound-node-add-docs-misleading`](E-metasound-node-add-docs-misleading.md)
  `#4`, IN-REVIEW) shows node→node `sourceOutputName` values (e.g. `Sine.Audio`)
  but never the **input-vertex→node** case, where the source is a graph input
  rather than a DSP node.
- `audio.authoring.metasound_gotchas.md` says nothing about how graph inputs
  surface as source vertices.

So the caller must **infer** the source pin name. In this task the agent narrated
the guess up front ("the input vertex's output pin name is likely the input name
itself") and it happened to be right on the first try — but only by luck; there
is no documented contract to rely on.

## Distinct from neighbouring MetaSound tickets

- [`E-metasound-node-add-docs-misleading`](E-metasound-node-add-docs-misleading.md)
  (IN-REVIEW) — corrects the typical-sequence example's **param spellings /
  className** (`node_class`→`nodeClassName`, `asset_path`→`assetPath`, bare names
  → `UE.<Name>.<Output>`) and adds the search→feed-back step. It documents the
  node→node connect shape but **not** the graph-input-vertex output-pin-name
  convention this ticket names — a different, narrower connect semantic.
- [`E-metasound-shorthand-search-mismatch`](E-metasound-shorthand-search-mismatch.md)
  (OPEN) — pin discovery for the `gain`/Multiply **node** (`PrimaryOperand` etc.).
  This is the pin-name convention for a graph **input**, not a node.

## What it should do (downstream wiki edit, not mine)

Add one line to `docs/wiki-src/audio.authoring.md` (the `connect_metasound_nodes`
overlay, and/or the typical-sequence block) — and optionally
`audio.authoring.metasound_gotchas.md` — stating the graph-input-vertex naming
convention:

> When the source of a `connect_metasound_nodes` edge is a graph **input** (from
> `add_metasound_input`), `sourceNodeId` is the input's returned `nodeId` and
> `sourceOutputName` **is the input's own name** (e.g. input `RPM` → source pin
> `"RPM"`). Graph inputs expose their value on a single output pin named
> identically to the input.

A cheaper code-side alternative (note only, out of scope for this docs ticket):
have `add_metasound_input` echo the created vertex's output pin name in its
success response, so the caller never has to infer it.

## Evidence

Clean/done fuzz task (focus `audio.authoring.add_metasound_input`, built
`/Game/Audio/MS_VehicleEngine` end-to-end — every call first-try, judge filed
nothing on the outcome). CallAnalyzer call-trace finding, verbatim:

> SAY before wiring the graph inputs: 'the input vertex's output pin name is
> likely the input name itself' — the agent had to GUESS the sourceOutputName for
> a graph-input vertex. CALL connect sourceNodeId=RPM-vertex sourceOutputName="RPM"
> targetInputName="Frequency" and sourceOutputName="Volume" -> both returned
> edgesCreated:1 first try. The connect_metasound_nodes wiki page documents
> sourceOutputName only as 'Output pin name on source' with no statement that a
> graph-input vertex exposes its value on an output pin named identically to the
> input.

Agent friction note, verbatim:

> "I had to infer the graph-input vertex's output pin name (turned out to equal
> the input name) for the RPM/Volume connections — both connected first try."

It worked on the first guess here, so Low severity — but only by luck; a caller
who guesses a different source-pin spelling gets a failed edge with no documented
recovery.

severity rationale: impact=docs/discoverability gap (works but undocumented) × reach=metasound-authoring input-wiring (not every editor session) -> Low

## History
- `#1-initial-audit` `OPEN` reporter — PROCESS/docs friction from the clean `audio.authoring.add_metasound_input` fuzz task (built MS_VehicleEngine end-to-end; every call first-try, judge filed nothing on outcome). Wiring a graph input into a node with `connect_metasound_nodes` requires `sourceOutputName` = the input vertex's output pin name, which equals the input's own name (`RPM`→`"RPM"`, `Volume`→`"Volume"`), but this convention is undocumented: the generated `connect_metasound_nodes.md` describes `sourceOutputName` only as "Output pin name on source", the `docs/wiki-src/audio.authoring.md` typical-sequence shows only node→node connects, and `metasound_gotchas.md` is silent. The agent had to infer the pin name ("likely the input name itself") — right on the first try, but by luck. Distinct from `E-metasound-node-add-docs-misleading` (connect param spellings/className, node→node shape) and `E-metasound-shorthand-search-mismatch` (pin discovery for the gain node). Proposes one documenting line in the `connect_metasound_nodes` overlay of `docs/wiki-src/audio.authoring.md` (and optionally the gotchas overlay) stating the graph-input-vertex output-pin naming convention; cheaper code-side alternative noted only (echo the input's output pin name in the `add_metasound_input` success response). Low severity — worked first-guess this time but undocumented.
