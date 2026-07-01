---
id: E-describe-metasound-no-compact-mode
title: "audio.authoring.describe_metasound has no compact/header-only mode — full graph JSON overflows the 10000-char display limit and spills to disk even for tiny graphs"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [metasound, audio, authoring, describe_metasound, response-size, oversized-readback, pagination, compact, docs]
encounters: 3
lastSeen: 2026-06-30T20:45:21+03:00
---

# describe_metasound always returns the full graph JSON, so every readback spills to disk

`audio.authoring.describe_metasound` (added by `F-rpc-audio-describe-metasound`,
DONE) returns the full `MetaSoundDumpBuilder::BuildMetaSoundJson` snapshot every
call: `rootGraph` (classID/className/isPreset + interface inputs/outputs with
default literals), the complete `nodes` array (id, classID, name, **per-vertex
inputs/outputs with literal lookup**), `edges`, and `variables`. There is no
`compact`, `headerOnly`, `nodeIds`, or field-select parameter — it is
all-or-nothing.

The verification of `F-rpc-audio-describe-metasound #3` already measured the
output at **77 KB for a real asset**, and the per-vertex literal expansion makes
even trivial graphs large: in this task's build of `MS_AmbientDrone` (a
**2-node, 1-input, 1-output** graph — Sine oscillator + Multiply gain) the
`describe_metasound` response exceeded the **10000-char MCP display limit and
spilled to a `Saved/.../HttpResponses/<…>.json` file** that the agent then had
to `Read`. It did so **twice in the one task**: once mid-build to discover the
real pin names (the documented inspect step) and again for the final snapshot.

This is the same oversized-readback ergonomic shape the board has already fixed
for the two analogous structural-dump methods, and `describe_metasound` is the
MetaSound-graph member of that family with no such mode:

- [`E-widget-export-xml-token-limit`](E-widget-export-xml-token-limit.md) (DONE)
  — added `compact` / `omit_slot_chain` so `widget.export_xml` returns inline
  (2,321 bytes vs the 71,877-char default that spilled to a fallback file).
- [`E-graph-connections-pagination`](E-graph-connections-pagination.md) (DONE)
  — added `nodeIds` / `edgeType` / `maxEdges` + `truncated`/`totalMatched` so
  `blueprint.graph.get_graph_connections` no longer dumps every edge (was
  66,260 chars → fallback-file grep).

The DONE `E-http-response-spill` does NOT cover this: it explicitly scopes the
server-side file-reference fallback to **direct HTTP callers** and states "MCP
adapter calls bypass Unreal-side response spilling because MCP clients own
large-output behavior." Here the caller was on the MCP path, so the spill came
from the MCP client's own large-output handling — exactly the case the
per-method compact modes above were created to avoid, by keeping the common
readback small enough to stay inline.

## Why this is worth its own ticket now (cross-task aggregation)

The `describe_metasound` readback-size friction has now recurred in **at least
three independent clean/done MetaSound-build tasks** but has never been filed as
its own defect — it was only logged as a "Secondary note (not separately filed)"
inside `E-audio-create-metasound-name-path-vs-assetpath` (see its `#1` and the
`## Secondary note` block), deferred to a vague "broad oversized-readback class
… would benefit from a header-only / paginated mode." Three sightings of the
same friction with an already-shipped fix pattern is enough to track it
explicitly rather than leave it as a footnote on an unrelated param-naming
ticket.

## Distinct from neighbouring MetaSound tickets

- [`F-rpc-audio-describe-metasound`](F-rpc-audio-describe-metasound.md) (DONE) —
  *added* the RPC; this is the follow-on ergonomic that the always-full payload
  is awkward as an MCP readback. Not a regression of that feature, a size knob
  on top of it.
- [`E-metasound-shorthand-search-mismatch`](E-metasound-shorthand-search-mismatch.md)
  (OPEN) — the `gain`→`UE.Multiply.Audio` shorthand/pin-discovery gap. Related
  only in that this task used `describe_metasound` *as* the pin-discovery route
  (instead of the search route that ticket describes); the size of that
  describe output is the separate friction here.
- The `*-no-limit-spills` family (`E-actor-list-no-limit-spills`,
  `E-inspect-list-objects-no-limit-spills`, etc.) — same oversized-readback
  class, different namespaces; this is the MetaSound-describe instance.

## What it should do (downstream fix, not mine)

Add an opt-in size knob to `audio.authoring.describe_metasound`, mirroring the
two DONE precedents, e.g. one or more of:

- `compact: true` (default `false`, no change for existing callers) — drop the
  per-vertex literal-lookup expansion and keep node id/className/name + edges +
  interface input defaults. This alone should keep the common discover-pins /
  confirm-graph readback well under the display threshold.
- `nodeIds: [...]` — return only the named node(s)' vertices for the
  "I just added node X, what are its pin names?" use case (the exact reason
  describe was called mid-build here), so the wire step no longer needs the
  whole graph.
- A node-count summary header so a caller can decide whether to fetch the full
  payload.

Default behavior stays the full snapshot (parity with `F-rpc-audio-describe-metasound`
and `asset.dump`'s `metasound.json`); only the inline-readback path opts into the
compact form.

## Evidence

This task (focus `audio.authoring.add_metasound_node`, build `MS_AmbientDrone`,
outcome **clean** — every call first-try, judge filed nothing on the outcome).
Friction note, verbatim:

> "describe_metasound exceeded the 10000-char display limit both times and dumped
> to a Saved/PinWright/HttpResponses JSON file that I had to Read; I had to inspect
> that graph dump (a documented step) to discover the real pin names …"

Call log corroborates the two spilling readbacks: `describe_metasound`
"(written to disk, read)" mid-build to inspect pins, and `describe_metasound`
"final snapshot (written to disk, read)" — both forced a `Read` off disk for a
2-node graph.

Prior independent sightings of the same friction (logged but never filed):
`E-audio-create-metasound-name-path-vs-assetpath` `#1` (MS_MenuBeep validate
task: "describe_metasound readback exceeded the display limit and was read off
disk twice") and `#2` (MS_EngineHum create task: a pre-wire `describe_metasound`
"to read real GUIDs/pins"). `F-rpc-audio-describe-metasound #3` measured the
real-asset payload at ~77 KB.

Fully recoverable every time (read off disk), hence Low severity — but every
MetaSound graph author pays a disk round-trip on the routine confirm-the-graph
and discover-the-pins readbacks until a compact mode exists.

## History
- `#3-add-compact-and-nodeids` `IN-REVIEW` developer — Added the opt-in size knobs mirroring the two DONE precedents (`E-widget-export-xml-token-limit` `compact`, `E-graph-connections-pagination` `nodeIds`). `audio.authoring.describe_metasound` now takes `compact` (boolean) and `nodeIds` (array): `compact` drops each node's per-vertex `inputs`/`outputs` (the vertex-GUID + literal-lookup bulk that overflows the 10000-char inline limit even on a 2-node graph) while keeping node id/classID/name + edges + rootGraph interface defaults; `nodeIds` restricts `nodes[]` to the named node(s) at full vertex detail for the "what are node X's pins?" mid-build case. Any reduced view adds a `compact`/`nodeCount`/`returnedNodeCount` summary header. Default behavior is unchanged — the full snapshot — so `asset.dump`'s `metasound.json` sidecar (which calls the no-options builder overload) is byte-identical and needs no aspect bump. Files: `Source/PinWright/Private/Handlers/Asset/MetaSoundDumpBuilder.h` (new `FMetaSoundDumpOptions` struct + options overload), `Source/PinWright/Private/Handlers/Asset/MetaSoundDumpBuilder.cpp` (compact node trimming, nodeIds filter, summary header, default-path delegation), `Source/PinWright/Private/Handlers/Audio/AudioAuthoringHandler.cpp` (parse `compact`/`nodeIds` → options). Test: `Source/PinWright/Private/Tests/Assets/TestMetaSoundDumpBuilder.cpp` `PinWright.Assets.MetaSound.DumpBuilder.CompactAndNodeIds` builds a transient source with two graph-output nodes and asserts default carries full per-node vertices + no header, `compact` strips the vertices + adds the header with `nodeCount`==full, and `nodeIds` returns exactly the requested node with `returnedNodeCount`==1 while `nodeCount` still reports the full graph size.
- `#2-fourth-sighting-remove-output-task` `OPEN` reporter — Fourth independent clean/done sighting, this time on the `audio.authoring.remove_metasound_output` fuzz task (built `MS_AmbientWind`: 2 Float inputs + 3 Audio outputs, removed two outputs, renamed the survivor, validated). The task ran `describe_metasound` **three times** (after-adds verify, post-remove verify, final-state confirm) and the friction note states the payload "exceeded the 10000-char display threshold **every time** and were spilled to HttpResponses files I had to Read" — so a 5-vertex user graph (2 in / 1 out + 2 auto interface outputs) spills on every readback, same as the 2-node MS_AmbientDrone in `#1`. Reinforces the `compact`/`nodeIds`/count-header proposal: the routine confirm-the-graph readback that a MetaSound author calls 3× per edit session pays 3 disk round-trips. No change to severity (Low — recovers via Read) or the proposed fix.
- `#4-additional-compact-inline-confirmed-docs-recommend` `IN-REVIEW` reporter — Fifth independent clean/done sighting of the readback-size friction, this time on the `audio.authoring.compile_metasound` fuzz task (built `MS_EngineHum`: RPM+MasterGain Float inputs, Sine oscillator + Multiply gain, 4 edges, RPM default 800 / MasterGain 0.5; compile `{compiled:true,valid:true}`, validate `{valid:true,diagnostics:[]}` — every call first-try, judge filed nothing on the outcome). Adds TWO things on top of `#1`/`#2`: (a) **direct confirmation the IN-REVIEW `#3` compact knob works inline** — the agent's mid-build `describe_metasound` ran with the *default* (non-compact) options and overflowed at **14640 chars > 10000 threshold**, spilling to `Saved/PinWright/HttpResponses` and forcing an extra out-of-band `Read`, but its FINAL `describe_metasound {compact:true}` returned **inline with no overflow** — first observed proof that `compact` keeps the routine confirm-the-graph readback under the display limit even though the default still spills on a trivial 4-node graph. (b) A **docs/discoverability angle** that survives the shipped structural fix (default stays full by design): name `docs/wiki-src/audio.authoring.md` (the `describe_metasound` section + the `audio.authoring` index) to prominently document the `compact: true` parameter and **recommend it for round-trip wiring readbacks**, noting the default verbose form overflows the 10000-char display threshold even for tiny MetaSounds (14640 chars here on 4 nodes + Mono output). Without that recommendation a caller who does not already know to pass `compact:true` pays the disk round-trip on the very first mid-build pin-discovery describe (exactly what happened here). No severity change (Low — recovers via `Read`); added the `docs` tag for the discoverability half. CallAnalyzer call-trace finding corroborates: first describe overflowed (14640 > 10000, written to disk, extra `Read` required); final describe with `compact:true` returned inline cleanly.
- `#1-initial-audit` `OPEN` reporter — PROCESS friction from the clean `audio.authoring.add_metasound_node` fuzz task (built MS_AmbientDrone end-to-end; every call first-try, judge filed nothing on outcome). `describe_metasound` always returns the full `MetaSoundDumpBuilder::BuildMetaSoundJson` payload (rootGraph + interface + all nodes with per-vertex literal lookup + edges + variables) with no compact/headerOnly/nodeIds knob, so even a 2-node graph (MS_AmbientDrone) overflowed the 10000-char MCP display limit and spilled to a Saved/.../HttpResponses JSON file the agent had to Read — twice in the one task (mid-build pin discovery + final snapshot). Same oversized-readback shape already fixed for `widget.export_xml` (`E-widget-export-xml-token-limit`, DONE — `compact`/`omit_slot_chain`) and `blueprint.graph.get_graph_connections` (`E-graph-connections-pagination`, DONE — `nodeIds`/`maxEdges`); `describe_metasound` is the MetaSound-graph member of the family with no such mode. NOT covered by `E-http-response-spill` (DONE), which explicitly scopes the file-reference fallback to direct-HTTP callers and bypasses MCP-adapter calls. Cross-task aggregation: the same readback-size friction was logged-but-never-filed as a "Secondary note (not separately filed)" in `E-audio-create-metasound-name-path-vs-assetpath` `#1`+`#2` (MS_MenuBeep validate, MS_EngineHum create) and measured ~77 KB on a real asset in `F-rpc-audio-describe-metasound #3` — three+ independent sightings, hence filed explicitly. Distinct from `F-rpc-audio-describe-metasound` (added the RPC) and `E-metasound-shorthand-search-mismatch` (shorthand→pins discovery; describe was merely the route used here). Proposes an opt-in `compact: true` / `nodeIds` / count-header knob, default unchanged. Low severity — recovers cleanly via disk Read every time but taxes every graph author's routine readbacks.
