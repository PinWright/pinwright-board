---
id: B-mesh-export-direct-overwrite
title: "geometry.export_obj and export_stl write directly over the final path, so a retry or failed write can destroy the previous export"
status: OPEN
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

## History
- `#1-source-pattern-scan` `OPEN` reporter — OBJ, ASCII STL, and binary STL all write directly to the final destination and have no overwrite/collision contract. Source-only; no files were exported.
