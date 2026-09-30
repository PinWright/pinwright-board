---
id: B-mgir-layout-grows-away-from-root
title: "MGIR layout grows the material graph rightwards from x=320 while the material output node stays at Material->EditorX (0 by default) — the final wires run backwards across the whole graph; lanes are ordered by GUID and sizes ignored"
status: OPEN
severity: Medium
category: bug
tags: [layout, material, mgir, gap-analysis-2026-09-30]
encounters: 1
lastSeen: 2026-09-30T12:00:00Z
---

# MGIR layout grows away from the material output node

This affects both `FMGIRLayoutEngine::LayoutExpressions`
(`Source/PinWright/Private/MGIR/MGIRLayoutEngine.cpp:51-70`) and the depth
computation it uses (`MGIR/MGIRExpressionUtils.h:86-173`).

**How it lays out:**
- Depth = longest path from the leaves (depth 0 = no inputs).
- X = `OriginX (320) + Depth × 320`; Y = `OriginY (180) + lane × 180`.
- Within a depth, lanes follow a stable key (GUID/name order).
- Only expressions still at (0,0) are moved.

**Why it looks wrong:**

1. **Wrong direction.** The material output node is drawn at
   `Material->EditorX/EditorY` (`UnrealEd/Private/MaterialGraphNode_Root.cpp:89`),
   which is 0 unless set. PinWright never writes it: there are no `EditorX` writes
   anywhere in `Source/`. The expressions therefore sit to the *right* of the
   output node, getting further right the closer they are to it, and every final
   wire runs back leftwards across the graph. The engine's own
   `UMaterialEditingLibrary::LayoutMaterialExpressions` does the opposite: it
   places column N at `-260 × (N+1)`, leftwards from the root
   (`MaterialEditingLibrary.cpp:145-268`).
2. **Arbitrary vertical order.** Sorting lanes by GUID ignores which input pin each
   node feeds, so wires cross for no reason.
3. **No sizes.** A fixed 180 px row pitch overlaps tall nodes, such as texture
   samples with previews and many-input functions. This part is already tracked by
   `F-material-layout-bounds-aware`, now re-pointed at `F-graph-layout-core`.

The same fixed-grid, GUID-ordered pattern is in `AGIR/AGIRLayoutEngine.cpp:193-273`
and `CRIR/CRIRLayoutEngine.cpp:71-156`.

Found by reading the code; not yet reproduced live.

**Fix:** use the material adapter of `F-graph-layout-core`:
- root fixed at `Material->EditorX/Y`;
- leftward growth;
- barycenter ordering;
- measured or estimated sizes.

A stop-gap is to mirror X around the root and order lanes by consumer pin index.

**Acceptance:** on an MGIR fixture compiled without authored positions (a texture
sample into a multiply into base colour, plus a scalar-parameter chain into
roughness):
- every expression is left of the output node;
- 0 backward edges;
- 0 overlaps.

## Severity justification

**Medium.** Every unpositioned MGIR compile, and every `material.authoring.auto_layout`
call on unpositioned nodes, produces a reversed graph. Readability only; nothing
is lost.

## History
- `#1-initial-report` `OPEN` reporter — Gap analysis 2026-09-30 (code reading): MGIR places depth-0 leaves at x=320 growing right, but the output node stays at Material->EditorX (never written by PinWright, default 0), so final wires run backwards; lanes sorted by GUID; fixed 320x180 grid ignores sizes. Engine layout grows leftwards from the root. Fix via F-graph-layout-core material adapter.
