---
id: E-geometry-auto-uv-redundant-with-unwrap-uv
title: "geometry.auto_uv duplicates geometry.unwrap_uv (both XAtlas) but lacks uvChannel — forces a comparison-read to pick the non-channel-0 variant"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [geometry, uv, xatlas, auto_uv, unwrap_uv, duplicate-method, discovery, docs]
---

# Two near-identical XAtlas auto-unwrap methods, distinguishable only by reading both param lists

`geometry.auto_uv` and `geometry.unwrap_uv` both perform the *same* operation —
an XAtlas auto-unwrap via
`UGeometryScriptLibrary_MeshUVFunctions::AutoGenerateXAtlasMeshUVs(...)`. Their
summaries are near-synonyms ("Auto-generate UV coordinates using XAtlas" vs
"Auto-generate UV unwrapping for a dynamic mesh using XAtlas"), so neither
distinguishes itself from the other at the discovery surface. The only real
difference is the parameter list:

- `geometry.auto_uv` (MeshOpsHandler.cpp:341-361) — params: `actorName` only.
  Calls `AutoGenerateXAtlasMeshUVs(Mesh, **0**, ...)` — UV channel is
  **hardcoded to 0**, so it can never target another channel.
- `geometry.unwrap_uv` (MeshInfoHandler.cpp:589-625) — params: `actorName` +
  `uvChannel` (default 0). Calls `AutoGenerateXAtlasMeshUVs(Mesh, **UVChannel**, ...)`.

`unwrap_uv` is a strict superset of `auto_uv`: passing `uvChannel:0` to
`unwrap_uv` reproduces `auto_uv` exactly. The two even live in different
handler files (MeshOpsHandler vs MeshInfoHandler), which is likely how the
duplication crept in. There is no functional reason to keep `auto_uv` as a
distinct method, and nothing in either method page, the `geometry` namespace
overlay, or a `## See also` tells an agent which to reach for.

**Scope note (3-way, not 2-way):** a *third* call site —
`geometry.pack_uv_islands` (MeshInfoHandler.cpp:630-669) — also calls
`AutoGenerateXAtlasMeshUVs(Mesh, UVChannel, FGeometryScriptXAtlasOptions(), ...)`,
i.e. it performs another full XAtlas auto-unwrap. (As a side note, it reads a
`textureResolution` param and echoes it in the result but never passes it to any
pack/atlas-size call — a latent no-op param worth its own ticket; not in scope
here.) So the real XAtlas redundancy is a triplet — `auto_uv` / `unwrap_uv` /
`pack_uv_islands` — and `unwrap_uv` is the channel-aware canonical form. The
disambiguation below covers all three.

## Why this is friction (not just a code smell)

An agent that wants an atlas unwrap on a **non-zero** UV channel — a normal
ask when keeping a clean wrap on channel 0 and an atlas unwrap on channel 1 —
has to open *both* method pages, notice that only `unwrap_uv` exposes
`uvChannel`, and infer that `auto_uv` is `unwrap_uv` frozen at channel 0. That
inference is invisible from the summaries; the agent only gets it by
comparison-reading the param tables. The cost is a discovery tax paid on the
first contact with the UV-unwrap pair, recurring for any agent that needs a
channel other than 0.

## Fix (chosen: ergonomic convergence + docs)

1. **Ergonomic (done):** add the `uvChannel` param to `geometry.auto_uv` so it
   converges with `unwrap_uv` (auto_uv no longer hardcodes channel 0), and make
   each verb's summary point at the other — `auto_uv` declares itself an alias of
   `unwrap_uv`, and `unwrap_uv` names `auto_uv` as its alias — so the discovery
   surface stops presenting two indistinguishable XAtlas verbs. This follows the
   `E-create-material-instance-duplicate-divergent-shape` precedent (make the
   canonical verb the capable superset, keep the legacy verb as a converged
   alias).
2. **Docs (done):** disambiguate on the `geometry` overlay
   (`docs/wiki-src/geometry.md`) — a `## UV unwrap verbs` note that `unwrap_uv`
   is the canonical channel-aware XAtlas unwrap, `auto_uv` is its alias, and
   `pack_uv_islands` is the third XAtlas site — plus reciprocal
   `### geometry.auto_uv` / `### geometry.unwrap_uv` overlay sections
   cross-referencing the sibling, so an agent reaching either method page learns
   which to use without opening the other.

## Friction evidence (this task — geometry.project_uv "StonePillar" build, 14 calls, outcome clean)

The story (step 4) asked for an XAtlas auto-unwrap on **UV channel 1**. The
agent's self-reported friction: *"chose unwrap_uv (has uvChannel param) over
auto_uv (no channel param) for the ch1 XAtlas unwrap."* That choice was only
reachable by comparison-reading both methods' params — exactly the discovery
tax above. The call itself succeeded (`geometry.unwrap_uv {XAtlas ch1}` →
`ok:true`); nothing errored, so this is pure PROCESS overhead, not an outcome
bug. The seed `geometry.project_uv` landed `clean` in the ledger.

## Docs page to improve

`docs/wiki-src/geometry.md` — add a disambiguation note for the
`auto_uv` / `unwrap_uv` pair (and reciprocal `### method` cross-references), so
the channel-aware vs channel-0 distinction is visible at the point either
method is discovered.

## Fix

Converged the two verbs (option 1, ergonomic):
- `geometry.auto_uv` (real location **MeshOpsHandler.cpp:341-361**, not 263-283
  — that range is `geometry.subdivide`) now exposes the same optional `uvChannel`
  param and passes it through to `AutoGenerateXAtlasMeshUVs` instead of hardcoding
  channel 0, so it is no longer a channel-0-only dead end.
- Its summary now reads "XAtlas auto-unwrap; same op as geometry.unwrap_uv (use that
  for the channel-aware form)" so the discovery surface stops presenting two
  indistinguishable XAtlas verbs.
- `docs/wiki-src/geometry.md` gains reciprocal `### geometry.auto_uv` /
  `### geometry.unwrap_uv` overlay sections that name each other as the same op and
  point at `pack_uv_islands` for island packing.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the `geometry.project_uv` "StonePillar" task (14 calls, outcome clean, judge filed nothing). PROCESS finding: `geometry.auto_uv` (MeshOpsHandler.cpp:263-283, `actorName` only, `AutoGenerateXAtlasMeshUVs(Mesh,0,...)`) and `geometry.unwrap_uv` (MeshInfoHandler.cpp:589-625, `actorName`+`uvChannel`, `AutoGenerateXAtlasMeshUVs(Mesh,UVChannel,...)`) are the same XAtlas op with near-identical summaries; `unwrap_uv` is a strict superset, yet neither method page nor the `geometry` overlay disambiguates them. The agent had to comparison-read both param lists to pick `unwrap_uv` for the channel-1 unwrap (self-report: *"chose unwrap_uv (has uvChannel param) over auto_uv (no channel param)"*). No error — pure discovery tax. Fix: alias `auto_uv`→`unwrap_uv` (or add `uvChannel` to `auto_uv`) and cross-reference; at minimum disambiguate on `docs/wiki-src/geometry.md`. Distinct from `E-geometry-deformer-echo-mesh-counts` (that's about deformers omitting count echoes; this is about two redundant XAtlas verbs).
- `#2-reword+fix` `IN-REVIEW` developer — Reworded: corrected stale line numbers (auto_uv is MeshOpsHandler.cpp:341-361, not :263-283 which is now subdivide) and added the 3-way scope note (`pack_uv_islands` is a third XAtlas auto-unwrap call site; its `textureResolution` is a latent no-op param, flagged for a separate ticket). Implemented option-1 convergence (matches the `E-create-material-instance-duplicate-divergent-shape` precedent): `geometry.auto_uv` now takes the same `uvChannel` param (default 0) as `unwrap_uv` instead of hardcoding channel 0, and its summary declares it an alias of `unwrap_uv`; `unwrap_uv`'s summary reciprocally names `auto_uv` as its alias — so the discovery surface disambiguates from either method without comparison-reading both param tables. Files: `Source/PinWright/Private/Handlers/Geometry/MeshOpsHandler.cpp` (auto_uv: add uvChannel param, route it to AutoGenerateXAtlasMeshUVs, echo it, alias summary), `Source/PinWright/Private/Handlers/Geometry/MeshInfoHandler.cpp` (unwrap_uv: reciprocal alias summary), `docs/wiki-src/geometry.md` (new `## UV unwrap verbs` editorial note + reciprocal `### geometry.auto_uv` / `### geometry.unwrap_uv` H3 cross-refs covering pack_uv_islands too). Regression test: `Source/PinWright/Private/Tests/Infra/TestGeometryUvUnwrapDisambiguation.cpp` (`PinWright.infra.geometry.AutoUvConvergesWithUnwrapUv`) — asserts against the live registration records that auto_uv has a uvChannel param and that both summaries cross-reference the sibling; fails if either handler edit is reverted.
- `#2-reword-and-fix` `IN-REVIEW` developer (parallel host) — Reword: corrected the load-bearing citation for `geometry.auto_uv` — `#1` cited `MeshOpsHandler.cpp:263-283`, which is actually the `geometry.subdivide` handler; the real `auto_uv` registration is **MeshOpsHandler.cpp:341-361** (the `unwrap_uv` cite `MeshInfoHandler.cpp:589-625` was correct). Fix (option 1, ergonomic): converged the two verbs so the discovery tax is gone — `geometry.auto_uv` now exposes an optional `uvChannel` (default 0) and (in the merged tree) delegates to the shared `GeometryUtils::ApplyXAtlasUnwrap` helper that `unwrap_uv`/`pack_uv_islands` use, so the op cannot drift; its summary declares it an alias of `unwrap_uv`. Added reciprocal `### geometry.auto_uv` / `### geometry.unwrap_uv` overlay sections to `docs/wiki-src/geometry.md` that name each other as the same op and point at `pack_uv_islands`. Files: `Source/PinWright/Private/Handlers/Geometry/MeshOpsHandler.cpp`, `docs/wiki-src/geometry.md`. Regression test `PinWright.geometry.auto_uv.ExposesChannelAndCrossRef` (in `Source/PinWright/Private/Tests/World/TestGeometryHandlers.cpp`) introspects the production registration and asserts `auto_uv` exposes `uvChannel` and that its summary references `unwrap_uv` — fails if the convergence is reverted. (Adversarial lens noted the docs `### See also` work overlaps `F-geometry-uv-prep-pipeline-batch`; the reciprocal H3 sections here are the auto_uv/unwrap_uv pair only and leave the 4-verb pipeline cross-refs to that ticket. `geometry.pack_uv_islands` also wraps XAtlas but adds island packing — a distinct op, left as-is.) Merge note: the two parallel `#2` fixes were reconciled — the shared-helper delegation (HEAD) was kept as the more robust implementation; both regression tests are retained.
