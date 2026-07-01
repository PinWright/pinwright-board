---
id: B-asset-dump-mgir-empty-graph-ambiguous
title: "asset.dump mgir.txt for engine-generated materials emits bare entry material '...' { } with no comment indicating intentional emptiness"
status: DONE
severity: Low
category: bug
tags: [asset-dump, material, mgir]
---

# asset.dump mgir.txt for engine-generated materials emits bare entry material '...' { } with no comment indicating intentional emptiness

For HLOD/Flatten-style auto-generated materials with no expression graph, the MGIR dump emits just the `entry material "..." { }` shell. There's no comment indicating "no expression graph (auto-generated material)" vs "graph dump failed" — they look identical.

Consumer can't tell whether this is an empty graph or a failed extraction.

**Repro:**
1. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/Game/Materials/HLOD/MiHLODMaterial/mgir.txt` — 71 bytes: `entry material "..." {\n}`.
2. Inspect `C:/Unity/unreal-fpv/.editor-automation/asset-dumps/Game/Materials/Special/RemoveSurfaceMaterial/mgir.txt` — 88 bytes.
3. Observe: bare shell with no comment to disambiguate intentional empty vs failed extract.

**Fix:** MGIRDecompiler's `DecompileMaterial` and `DecompileFunction` track whether any body line was emitted between the `{` header and `}` close. When the body is empty, emit a single canonical marker `    # no expression graph` (4-space indent, inside the block). One marker covers all empty cases — engine-generated, `bUseMaterialAttributes` fallthrough, runtime-stand-in — without the decompiler needing to introspect material state.

## History
- `#1-initial-repro` `OPEN` reporter — Engine-generated materials produce a bare `entry material "..." { }` MGIR shell with no commentary. Sample paths: `Game/Materials/HLOD/MiHLODMaterial/mgir.txt` (71 bytes), `Game/Materials/Special/RemoveSurfaceMaterial/mgir.txt` (88 bytes). Indistinguishable from a failed graph extraction.
- `#2-emit-no-graph-marker` `IN-REVIEW` developer — Decompiler now emits `    # no expression graph` inside the body when no expression/output lines were appended (symmetric in `DecompileMaterial` and `DecompileFunction`, `MGIRDecompiler.cpp`). Verified MGIRParser tolerates `#`-prefix comment lines for round-trip safety. Added regression test `FMGIRDecompileEmptyMaterial_EmitsNoGraphMarker` constructing a transient `UMaterial` with no expressions and asserting the marker appears inside the block.
- `#3-verify-fix` `DONE` tester — Verified: re-ran `asset.dump` on both repro assets. `Game/Materials/HLOD/MiHLODMaterial/mgir.txt` and `Game/Materials/Special/RemoveSurfaceMaterial/mgir.txt` both now contain `    # no expression graph` on line 2 between the `{` and `}`, matching the canonical marker spec.
