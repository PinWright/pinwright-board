---
id: E-geometry-warp-extent-semantics
title: "geometry.twist/taper/bend 'extent' wiki entry says only 'Twist extent (default 50)' — units (world) and the symmetric-about-origin semantics (bSymmetricExtents=true) are undocumented, so sizing extent to a mesh's height requires reading the handler source"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [geometry, twist, taper, bend, warp, extent, docs, wiki, symmetric-extents, discoverability]
---

# The warp deformers' `extent` param is documented as a bare "extent (default 50)" with no units and no symmetry semantics

`geometry.twist`, `geometry.taper`, and `geometry.bend` each take an `extent`
number whose only documentation is the bare handler param description
`RPC_PARAM_OPT("extent", "number", "Bend extent (default 50)")`
(MeshOpsHandler.cpp:690 bend, :724 twist `"Twist extent (default 50)"`,
:759 taper `"Flare extent (default 50)"`) — which is what feeds the generated
`wiki-generated/geometry.{twist,taper,bend}.md` pages. Nothing states:
- the **units** — `extent` is in world units (the same units as the mesh's
  `height`/bounding box), not a 0..1 fraction of the mesh; and
- the **symmetry** — all three handlers set `bSymmetricExtents = true`
  (MeshOpsHandler.cpp:702 bend, :736 twist, :772 taper), so `extent` is a
  **half-extent measured symmetrically about the warp origin**
  (`LowerBoundsInterval = -Extent`, `UpperBoundsInterval = +Extent`). For a
  center-origin primitive of height H, the deform covers the whole mesh only
  when `extent ≈ H/2`, not `extent ≈ H`. Additionally, bend and twist set
  `bBidirectional = true` (MeshOpsHandler.cpp:703 bend, :737 twist) so the
  deform is centered on the origin; taper sets no such flag.

Without those two facts the natural guess ("extent = the column's full height")
over-covers by 2× and the result silently differs from intent. The wiki text
gives no way to compute the right value, so the only way to confirm coverage is
to read the C++ (`bSymmetricExtents`) or to twist-and-eyeball repeatedly.

## What it should do (docs only — works, just under-documented)

Improve the `extent` description in the **`docs/wiki-src/geometry.md`** overlay
(the source the generated `geometry.twist/taper/bend.md` pages are built from;
the overlay already carries per-method H3 notes for several geometry verbs
(create_box, create_arch, unwrap_uv, get_vertex_position, array_*, …) but none
for the twist/taper/bend warp family).
Add a short note for the warp family: `extent` is in **world units** and is a
**symmetric half-extent about the deform origin** (`bSymmetricExtents=true`), so
to cover a mesh of height H centered on the origin, pass `extent ≈ H/2`. One
shared sentence covers twist, taper, and bend since all three use the same
convention. The handler param descriptions (`RPC_PARAM_OPT("extent", ...,
"Twist extent (default 50)")`) could also be enriched, but the overlay is the
named target per the docs process.

## Friction evidence (this task — geometry.twist "TempleColumn_01", 11 calls, outcome clean)

The column was `height=300` centered at the origin. The agent passed
`extent=150` to both `geometry.twist` and `geometry.taper` (exactly H/2) and it
was correct — but per the self-report the agent only knew that was correct
because it **read the plugin source to confirm `bSymmetricExtents=true`**: "twist/
taper 'extent' units/semantics (symmetric half-extent about origin) are
undocumented in the wiki; I confirmed extent=150 covers the 300-tall centered
column by reading the plugin source (bSymmetricExtents=true) — a discoverability
gap." No call errored (outcome `clean`); this is pure discovery overhead — a
source dive substituting for a one-sentence wiki note, recurring for every
twist/taper/bend a prop-build chains.

Sibling of `E-geometry-deformer-echo-mesh-counts` (same warp family, response
shape) but orthogonal: that ticket is about the *response* omitting counts; this
is about the *request* param's meaning being undocumented. Per board policy this
is an E- tagged `docs`, with the named overlay to improve being
`docs/wiki-src/geometry.md`.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the `geometry.twist` "TempleColumn_01" twisted-column task (11 calls, outcome clean, judge filed nothing). PROCESS/docs finding: the warp deformers' `extent` param is documented as a bare "Twist extent (default 50)" (wiki-generated/geometry.twist.md:13, same for taper/bend) with no units and no symmetry note, yet all three handlers set `bSymmetricExtents=true` (MeshOpsHandler.cpp:702/736/772), making `extent` a world-unit half-extent about the origin (cover height H ⇒ extent ≈ H/2). The agent had to read the plugin source to confirm extent=150 covered the 300-tall centered column. Fix (downstream docs): add a one-sentence units+symmetry note for the twist/taper/bend `extent` in the `docs/wiki-src/geometry.md` overlay. No call errored; pure discoverability overhead.
- `#2-reword` `OPEN` developer — Reworded to fix stale source citations before implementing. The `bSymmetricExtents=true` lines are :702 (bend), :736 (twist), :772 (taper) in current source, NOT 602/636/672 (those now hold offset_faces/shell/chamfer); the bare param descriptions are at :690/:724/:759. Also added the omitted detail that bend and twist set `bBidirectional=true` (:703/:737) — taper does not. Defect is real and present; fix approach (geometry.md overlay note for the warp family) unchanged.
- `#2-reword-and-fix` `IN-REVIEW` developer — Reworded: corrected the stale `bSymmetricExtents` line cites `602/636/672` → actual `702/736/772` (verified against MeshOpsHandler.cpp — bend :702, twist :736, taper :772), and replaced the inaccurate "the overlay is a 5-line namespace blurb with no per-method param notes" with the true state (the overlay already has per-method H3 notes for several geometry verbs but none for the twist/taper/bend warp family). Fix (docs-only): added a `## Warp deformers (twist / taper / bend)` namespace section plus `### geometry.twist` / `### geometry.taper` / `### geometry.bend` H3 sections to `Docs/wiki-src/geometry.md`, documenting that `extent` is in **world units** and is a **symmetric half-extent about the mesh origin** (`bSymmetricExtents=true`, spans `[-extent,+extent]`) so covering a center-origin mesh of height H needs `extent ≈ H/2`; the taper H3 also flags that `flareX`/`flareY` are percentages while `extent` is a world-unit length. Regression test `Source/PinWright/Private/Tests/Infra/TestGeometryWarpExtentDocs.cpp` (4 IMPLEMENT_SIMPLE_AUTOMATION_TEST cases) renders the geometry namespace page and the twist/taper/bend method pages through the live `WikiHandler::RenderPage` and asserts the world-units + symmetric-half-extent + `extent ≈ H/2` markers survive — overlay-exclusive, so reverting the overlay sections fails the test. Follows the `E-geometry-primitive-orientation-axes-undocumented` precedent (same overlay, same WikiDocTestHelpers::RenderOrFail harness).
