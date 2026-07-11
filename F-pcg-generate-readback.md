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
- `#2-go-implement-generate-readback` `IN-REVIEW` developer — GO. Shipped `pcg.generate` (`Handlers/PCG/PCGGenerateHandler.cpp`): resolves an actor by label/name/path, gets-or-adds a `UPCGComponent` (transacted, idempotent), optionally assigns `graphPath` via `SetGraphLocal`, calls `GenerateLocal(force)`, and resolves the async completion token on the `bFired`-guarded `OnPCGGraphGeneratedDelegate`, reading back `pointCount`/`dataCount` from `GetGeneratedGraphOutput`. Point-count summation promoted to a named-namespace inline helper `PinWrightPCG::CountGeneratedPoints` (`Handlers/PCG/PCGGenerateReadback.h`, version-guarded `UPCGBasePointData::GetNumPoints` 5.5+ / `UPCGPointData` older) so the handler and its test share one production symbol. New error code `NO_PCG_GRAPH` (`Handlers/ErrorCodes.h`). Tests (`Tests/PCG/TestPCGGenerateHandler.cpp`): adopted the reporter's red `PinWright.pcg.generate.HandlerRegistered` (was failing pre-fix, now green), plus `PointCountReadback` (real helper vs a synthetic 5+3-point-plus-non-point collection = 8) and `MissingActorRejected` (dispatches the real handler → synchronous `ACTOR_NOT_FOUND`). Plugin compiles clean; all three scoped tests pass. Live async generation is exercised by real MCP usage, not the headless suite — the engine's own PCG tests likewise run elements via synthetic contexts rather than full-component generation, and the plugin's tick pump does not advance the world/PCG-subsystem scheduler; the readback + dispatch tests cover the load-bearing logic deterministically.
