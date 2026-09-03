---
id: E-static-mesh-schema-aliases
title: "Static-mesh readback omits the compatibility field names triangleCount and uvChannels"
status: OPEN
severity: Low
category: ergonomic
tags: [static-mesh, schema, compatibility, aliases, readback]
encounters: 1
lastSeen: 2026-09-03T20:21:31+03:00
---

# Static-mesh readback lacks compatibility field aliases

## What happens

The shared static-mesh builder emits section triangle counts as `numTriangles` and
the per-LOD UV count array as `uvChannelsByLod`
(`Source/PinWright/Private/Handlers/Asset/StaticMeshDumpBuilder.cpp:55-97`). It
does not also emit the compatibility names `triangleCount` and `uvChannels`.
The text emitter mirrors only `numTriangles` for each section and
`uvChannelsByLod` at the root
(`StaticMeshTextEmitter.cpp:143-154,230-241`).

## Why it matters

Clients written to the requested or sibling mesh vocabulary must normalize the
fields themselves, and JSON/text consumers cannot use the compatibility names
directly. Severity is Low: the data is present and correct under another name, so
this is schema friction rather than missing information.

## What should happen

Either publish `triangleCount` beside each section's `numTriangles` and
`uvChannels` beside `uvChannelsByLod` with identical values, or declare a versioned
canonical-name migration. Keep JSON, text, RPC documentation, and cache/aspect
versions consistent, with parity tests preventing the aliases from drifting.

## Workaround

Normalize `numTriangles` to `triangleCount` and `uvChannelsByLod` to `uvChannels`
in the client.

## Related

- `F-static-mesh-section-material-map` — wave-6 ticket added `numTriangles` and
  explicitly omitted the original `triangleCount` wording as an alias.
- `F-static-mesh-uv-channel-readout` — wave-6 ticket added `uvChannelsByLod` and
  explicitly omitted the requested `uvChannels` spelling as an alias.

## History
- `#1-filed-wave-6-follow-up` `OPEN` reporter — Source-only verification confirmed `numTriangles` and `uvChannelsByLod` at `StaticMeshDumpBuilder.cpp:55-97` and matching text output at `StaticMeshTextEmitter.cpp:143-154,230-241`, with neither compatibility alias present. The two parent tickets explicitly document those alias omissions, so this ticket owns only the schema compatibility gap. No build, test, editor, or MCP call was run. Severity Low because callers can derive both aliases without losing information.
