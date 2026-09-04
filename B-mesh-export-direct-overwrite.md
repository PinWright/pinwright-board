---
id: B-mesh-export-direct-overwrite
title: "geometry.export_obj and export_stl write directly over the final path, so a retry or failed write can destroy the previous export"
status: IN-REVIEW
severity: High
category: bug
tags: [geometry, export-obj, export-stl, file-output, overwrite, atomic-write, data-loss, collision]
encounters: 1
lastSeen: 2026-09-03T00:00:00Z
---

# Mesh exports replace the destination in place with no collision policy

`MeshIOHandler.cpp:646-657` resolves the OBJ destination and calls
`FFileHelper::SaveStringToFile` on the final path. The STL branches do the same with
`SaveArrayToFile` or `SaveStringToFile` at `:698-722`. Neither verb stages a sibling temporary file,
atomically moves it into place, rejects an existing destination, nor exposes an `overwrite` choice.
The default path is derived only from actor label, so re-exporting the same actor necessarily
collides.

A normal retry overwrites the previous artifact without warning. More importantly, an I/O failure
after the destination has been opened can leave the old good export truncated while the RPC merely
returns `WRITE_FAILED`. This is the catalog's unsafe-output-replacement-or-collision shape.

## What should happen

Write to a unique sibling temporary path, verify the completed byte count/content, then atomically
replace the destination. Reuse `AssetDumpWriter::WriteFileAtomic` or extract its binary-capable
equivalent. Add an explicit `overwrite` policy (default refusal) or choose a collision-safe default
name, and report whether a prior file was replaced.

**Workaround:** provide a fresh `filePath` for every export and keep a backup before replacing an
important OBJ/STL.

## Fix

The root cause was that all three export branches wrote directly to the final path and declared no
collision policy. Both RPCs now default `overwrite` to false, report `ALREADY_EXISTS` without
altering an occupied destination, and route OBJ, ASCII STL, and binary STL through the shared
`AtomicFileWriter`. Opted-in replacement is staged, verified, and atomically published; successful
responses report whether an existing file was `replaced`. A non-authoritative exists probe avoids
serializing a mesh for an obvious refused collision, while the writer's `FailIfExists` publication
policy remains the race-safe authority.

Changed files:
- `Source/PinWrightGeometry/Private/Handlers/Geometry/MeshIOHandler.cpp`
- `Source/PinWrightGeometry/Private/Tests/Geometry/TestMeshIOHandler.cpp`
- `docs/wiki-src/geometry.md`
- `X:/src/unreal/.pinwright-board/B-mesh-export-direct-overwrite.md`

Regression coverage: `FGeometryMeshExportAtomicOverwriteContractTest`
(`PinWright.geometry.MeshExport.AtomicOverwriteContract`) calls the production handlers for OBJ,
ASCII STL, and binary STL. It covers default collision refusal and byte preservation, opted-in
replacement with `replaced=true`, handler propagation of a locked-destination publish failure while
preserving the sentinel and removing the temporary file, fresh output with `replaced=false`, and the
text/binary byte contracts. The shared `FAtomicFileWriterFailurePreservesOriginalTest` separately
locks the staged source while exercising the real publisher, excluding unsafe delete-before-move
inside the writer called by these handlers. This test covers the handlers' collision policy,
explicit-overwrite integration, and error mapping; the production call sites show all three branches
use that writer. No handler-local injection seam was added because it would duplicate the shared
writer's existing call-local seam and expose a second abstraction solely for tests.

Unrelated writers were deliberately not migrated. `AssetDumpWriter`, gateway-port output, image
output, the shared writer, Build.cs files, and `ErrorCodes.h` are outside this ticket's implementation
scope.

## History
- `#1-source-pattern-scan` `OPEN` reporter — OBJ, ASCII STL, and binary STL all write directly to the final destination and have no overwrite/collision contract. Source-only; no files were exported.
- `#2-atomic-export-policy` `IN-REVIEW` implementer — Added default collision refusal, atomic opted-in replacement, `replaced` results, production-calling coverage for all three formats, and updated the geometry contract. Static inspection only; no build, Unreal run, automation, MCP, or Git operation was performed.
