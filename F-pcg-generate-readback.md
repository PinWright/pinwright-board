---
id: F-pcg-generate-readback
title: "pcg.generate: trigger UPCGComponent generation on a placed actor + read back point counts"
status: IN-REVIEW
severity: Medium
category: feature
tags: [pcg, generation, readback, async]
claimedBy: fuzz2
claimedAt: 2026-07-11T09:35:27.2903582+03:00
---

# pcg.generate — trigger generation on a placed actor and read back results

Split from `F-pcg-authoring-parity` (the highest-value slice of that former audit umbrella).

PinWright can author a `UPCGGraph` (nodes, edges, filters, subgraphs, and — after
`F-pcg-authoring-parity` — graph parameters) but has **no RPC to run a graph and observe the
result**. Today the only way an agent can trigger PCG generation and read back how many points
were produced is `python.execute` (`unreal.PCGComponent.generate_local(...)` + point-data
readback), which the board otherwise treats as an escape hatch to close (cf. the many
`*-no-completion-signal` tickets).

## Repro / gap
Grep `Handlers/PCG/` for `generate|regenerate|execute|PCGComponent`: no matches. The 13
registered `pcg.*` handlers are all graph-asset authoring/inspection; none places or drives a
`UPCGComponent`.

## Proposed scope
- `pcg.generate(actorPath | assign graph to a placed actor, ...)` — trigger generation on a
  placed actor's `UPCGComponent` (or an `APCGVolume`).
- Use the async completion-token seam (`Ctx.MakeAsyncToken()` → `Token->SendSuccess/SendError`;
  canonical consumer `Handlers/Environment/LandscapeHandler.cpp`) since generation completes over
  ticks on the PCG subsystem scheduler — do **not** block the game thread.
- Read back a result summary (at minimum output point counts) so the caller can confirm the graph
  produced data.

## Target notes
- Build against **UE 5.7** (this host's engine); the UE 5.8 `PCGToolset` `ExecuteGraphInstance`
  / `UPCGExecuteGraphInstanceAsyncResult` types are only a reference and may not exist unchanged
  on 5.7. The underlying primitives (`UPCGComponent` generation, generated-data readback) are
  present in 5.7's PCG plugin (`C:\UE_5.7\Engine\Plugins\PCG`).
- Verifying headlessly needs a real editor world + a placed actor + a deterministic graph
  (e.g. a `CreatePoints`/surface-sampler graph) and must pump the async completion to done — plan
  the test fixture accordingly (this is why it is its own ticket, not part of the CRUD slice).

## Acceptance
`pcg.generate` runs a scatter-style graph on a placed actor and returns a non-zero output point
count, driven to completion through the async token seam, without `python.execute`.

## History
- `#1-split-from-authoring-parity` `OPEN` reporter — Split off `F-pcg-authoring-parity` as the highest-value discrete capability. Gap verified: no generation/readback handler exists in `Handlers/PCG/` (grep for `generate|PCGComponent` returns nothing). Needs the async completion-token seam and a world-backed test fixture, hence a standalone ticket rather than being bundled with the synchronous graph-parameter CRUD slice.
- `#2-go-implement-generate-readback` `IN-REVIEW` developer — GO. Gap re-verified against current source: 13 registered `pcg.*` handlers, all graph-asset authoring/inspection; none drives a `UPCGComponent` (`grep generate|PCGComponent` in `Handlers/PCG/` = no matches). All cited UE 5.7 primitives confirmed present in `PCGComponent.h` (`GenerateLocal(bool)` :251, `GetGeneratedGraphOutput` :298, `OnPCGGraphGeneratedDelegate` :403) and `PCGBasePointData.h` (`GetNumPoints` :108). Implementing `pcg.generate`: resolve-or-add a `UPCGComponent` on a placed actor, optionally assign `graphPath`, `GenerateLocal(force)`, resolve the async token on the `bFired`-guarded `OnPCGGraphGeneratedDelegate`, and read back the produced point count. Corrected the stale "10 handlers" count to 13.
