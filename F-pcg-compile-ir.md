---
id: F-pcg-compile-ir
title: "PCGIR is decompile-only by an explicit deferral whose release condition — read-side demand surfacing compile-side friction — is now met: a Procedural Vegetation plant is 15-40 nodes of nested settings structs and the only route to one is per-node add_node plus per-property property.set"
status: OPEN
severity: Medium
category: feature
tags: [pcg, pcgir, ir, compile, round-trip, authoring, procedural-vegetation, call-count, deferral-released]
encounters: 1
lastSeen: 2026-08-29T17:30:00+05:00
---

# The deferral named its own release condition, and this is it

`F-pcg-decompile-ir` (DONE) shipped the read direction and deferred the write direction in terms
that were explicitly conditional — `:16-18`:

> compile direction (`pcg.compile_pcgir`) is **out of scope** for this ticket and is deferred to a
> future `F-pcg-compile-ir` ticket **if read-side demand surfaces compile-side friction**.

repeated at `:86-89` under *"Compile direction is out of scope"*. This ticket is filed under the
name that deferral reserved, and its whole content is the friction.

## Correction to the premise this was filed from, before anything else

The lead described `pcg.pcgir` as a decompile-only **verb**. It is not a verb at all.

- `pcg.pcgir` is a **wiki topic page**, `docs/wiki-src/pcg.pcgir.md`, auto-enrolled as a topic node
  and reachable as `call("pcg.pcgir")` for documentation only.
- The IR also has a **dump sidecar registration**, `REGISTER_DECOMPILE_IR(TEXT("pcgir.graph"),
  DumpFileNames::PcgIr, &GetPCGGraphClass, &BuildPCGIRGraphSidecar, 100)` at
  `Source/PinWrightPCG/Private/Handlers/PCG/PCGDecompileHandler.cpp:28-29` — a sidecar id, not a
  callable method.
- The **verb** is `pcg.decompile`, registered at `PCGDecompileHandler.cpp:31`, summary at `:32`:
  *"Decompile a UPCGGraph asset into PCGIR text."*

Decompile-only is confirmed in the docs (`pcg.pcgir.md:3`: *"It is decompile-only via
`pcg.decompile`; compile is deferred"*, and `:54`: *"There is no round-trip yet"*) and enforced in
code — `includeReferencedSubgraphs` is documented *"Reserved for future use; subgraphs are always
flattened in the current decompile direction"* (`PCGDecompileHandler.cpp:35`). There is no
`pcg.compile` in the namespace's 14 registered verbs.

## The friction, measured

Building a Procedural Vegetation plant through PinWright became possible for the first time this
session (`F-pcg-create-graph-class-parameter`, DONE). What it costs:

- A real PV plant graph is **15-40 nodes**, and PV settings are deep nested structs — distribution
  parameters, hormone/phyllotaxy blocks, grower settings.
- The only authoring route is `pcg.add_node` per node and `property.set` per property. There is no
  batch property write: `F-property-set-batch` (OPEN, Medium) measured **one PCG graph at ~110
  individual `property.set` calls**, and its title names PCG as the case that provoked it. A PV
  plant is larger than that graph, and its properties are nested rather than flat.
- Nested is the expensive word. `B-property-set-object-hop-notification-noop` (OPEN) records that
  `property.set` through an object hop is a **silent no-op** — the write lands, reads back, and the
  subsystem's inner `PostEditChangeProperty` override never runs — so the per-property route is not
  merely long, it has a failure mode that reports success. Authoring a 40-node plant one property at
  a time means paying ~100+ calls per graph on a path where some fraction of them silently do
  nothing.
- And the per-node build pattern is exactly what triggers
  `B-pcg-connect-pins-silently-replaces-edge`: every PV node's `In` pin is single-connection, so
  wiring incrementally can silently unwire earlier work. A compile-from-text path declares the whole
  edge set at once instead of accumulating it.

That is D-8's "no batch property write" at its worst, on the one graph family where the alternative
already exists in one direction.

## The round trip is one direction away, and the emitter is not the blocker

Verified this session against a live `ProceduralVegetationGraph` containing a PV node.
`pcg.decompile` returned `warnings: []` and emitted the PV node by its full class path alongside the
stock ones:

```
entry pcg `/Game/PinWrightScratch/PVTest/PCG_PwVerdictSubclass.PCG_PwVerdictSubclass` {
    node N_Input = `/Script/PCG.PCGGraphInputOutputSettings` @(0, 0) { ... }
    node N_Output = `/Script/PCG.PCGGraphInputOutputSettings` @(200, 0) { ... }
    node N1 = `/Script/ProceduralVegetation.PVSeedGeneratorSettings` @(400, 0) {
        Seed = 2067415478
    }
    node N2 = `/Script/PCG.PCGCreatePointsSettings` @(200, 200) { ... }
    connect N2.Out -> N1.In
}
```

So the emitter has **no PV-specific blind spot at the node level**: it resolves and names
`/Script/ProceduralVegetation.PVSeedGeneratorSettings` with no warning, and positions and edges come
through. **Bounded honestly:** that node was default-constructed, so this measurement does *not*
establish that the emitter round-trips PV's deep nested settings structs — only that PV nodes are
emitted as first-class nodes. Whether nested PV structs survive a round trip is the first thing a
compile implementation would have to test, and it is untested here.

(Incidental, and recorded because it corroborates a separate ticket from a third code path: the
decompiled text shows a single `connect N2.Out -> N1.In`. Two edges had been authored into that pin.
See `B-pcg-connect-pins-silently-replaces-edge`.)

## The ask

`pcg.compile_pcgir(text, savePath)` — the write direction, matching the shape the sibling IRs
already ship (`blueprint.compile_bpir`, `material.compile_mgir`, `animation.compile_agir`), and
reusing the `IrCore` foundations `F-pcg-decompile-ir` chose over a forked lexer.

Two constraints the deferral already identified and this ticket does not relitigate: the parser,
type-spec resolver and layout handling are real work, and the flattening of referenced subgraphs
(`PCGDecompileHandler.cpp:35`) means a compile has to decide what a flattened emit means on the way
back. Neither is a reason not to file; both are why this is its own ticket rather than a follow-on
edit to the decompiler.

**Acceptance:** a PV graph authored by `pcg.add_node` + `property.set`, decompiled, compiled back to
a second asset, and the two decompiles compared — with at least one explicit content assertion
beyond byte-equality, per `agent-conventions.md`'s rule that symmetric loss defeats a pure
round-trip test.

**Not asked for here:** a PV-specific authoring verb. The gap is the generic compile direction; PV is
the workload that made it hurt, not a special case, and a PV-shaped verb would be the umbrella this
board's policy declines.

## Related

- `F-pcg-decompile-ir` (DONE, Medium) — the deferral this releases. Its scope decision was correct
  and is not being reopened; this is the follow-on it named.
- `F-property-set-batch` (OPEN, Medium) — the generic form of the per-property cost. If it lands
  first, this ticket's friction drops substantially but does not vanish, because a batch property
  write still leaves per-node `add_node` and per-edge `connect_pins` calls. The two are
  complementary, not alternatives, and either can land alone.
- `B-property-set-object-hop-notification-noop` (OPEN) — why the per-property route is not merely
  long. Referenced, not restated.
- `B-pcg-connect-pins-silently-replaces-edge` (OPEN, High, filed this session) — why the per-node
  route is hazardous as well as slow.
- `F-pcg-create-graph-class-parameter` (DONE this session) — what made PV graphs authorable at all,
  and therefore what turned this deferral's condition from hypothetical into measured.

## Dedup

Board-wide search for `pcgir`, `compile_pcgir`, `pcg.compile` and PCG round-trip tickets: the only
files mentioning PCGIR are `F-pcg-decompile-ir` (which defers this by name),
`B-pcg-inspect-edge-direction-reversed`, `F-pcg-core-graph` and
`F-foliage-namespace-has-no-behavioural-tests`, none of which asks for the compile direction. No
ticket named `F-pcg-compile-ir` existed; this file claims the id the deferral reserved.

## Severity

**Medium, argued.** Impact class is the rubric's Medium verbatim — *"Doable, but only via a
documented workaround, a source dive, or **many extra calls**"*. Bulk PV authoring is not blocked;
it is ~150+ calls per plant across `add_node`, `connect_pins` and `property.set`, on documented
verbs that work.

**The High reading, and why it is refused.** *"A missing verb, so a reasonable task is impossible"*
is the High-or-Medium band, and one could argue a 40-node nested-struct graph is impossible in
practice rather than merely expensive. It is refused because the practical impossibility is not
this ticket's to claim: the call-count half is owned by `F-property-set-batch` and the
silently-failing half by `B-property-set-object-hop-notification-noop`, both already filed, and
charging their impact here would triple-count one problem on a board whose picker ranks by severity.
What is left for this ticket alone is the absence of a bulk path, which is a capability ask.

**Reach modifier declined in both directions, and named.** No bump up: text-IR authoring is a
per-namespace convenience, not a verb that runs in almost every session, and the sibling compile
directions were each filed and shipped at ordinary priority. No bump down: this is not a rare edge
path either — it is the normal way anyone would build a plant, and the reason the decompiler was
built first. Medium stands unmodified.

## History
- `#1-deferral-condition-met` `OPEN` reporter — Filed under the id `F-pcg-decompile-ir:16-18` reserved, whose deferral was explicitly conditional on *"read-side demand surfac[ing] compile-side friction"*; this ticket is that friction. **Corrects the premise it was filed from:** `pcg.pcgir` is not a verb — it is a wiki topic page (`docs/wiki-src/pcg.pcgir.md`) plus a dump-sidecar registration (`REGISTER_DECOMPILE_IR(TEXT("pcgir.graph"), ...)`, `PCGDecompileHandler.cpp:28-29`); the verb is `pcg.decompile` (`:31`, summary `:32`). Decompile-only confirmed at `pcg.pcgir.md:3` and `:54` and enforced in code, where `includeReferencedSubgraphs` is *"Reserved for future use; subgraphs are always flattened in the current decompile direction"* (`PCGDecompileHandler.cpp:35`); there is no `pcg.compile` among the namespace's 14 verbs. Friction: a PV plant is 15-40 nodes of deep nested settings structs and the only route is per-node `add_node` plus per-property `property.set`, with no batch write — `F-property-set-batch` measured one PCG graph at ~110 `property.set` calls and names PCG as its provoking case — on a path where `B-property-set-object-hop-notification-noop` records that a write through an object hop silently no-ops, and with a per-node wiring pattern that triggers `B-pcg-connect-pins-silently-replaces-edge` because every PV node's In pin is single-connection. **Verified live that the emitter is not the blocker:** `pcg.decompile` on a `ProceduralVegetationGraph` containing a `PVSeedGeneratorSettings` node returned `warnings: []` and emitted the PV node by full class path alongside the stock ones, with positions and the edge. **Bounded honestly** — that node was default-constructed, so this does NOT establish that PV's nested settings structs survive a round trip, which is the first thing a compile implementation must test. Recorded incidentally that the decompiled text shows one edge where two had been authored, corroborating `B-pcg-connect-pins-silently-replaces-edge` from a third code path. Ask: `pcg.compile_pcgir(text, savePath)` matching the shipped `blueprint.compile_bpir` / `material.compile_mgir` / `animation.compile_agir` shape and reusing `IrCore`, with acceptance as a decompile → compile → decompile comparison carrying at least one explicit content assertion beyond byte-equality per `agent-conventions.md` on symmetric loss. Explicitly **not** asking for a PV-specific authoring verb — the gap is the generic compile direction and a PV-shaped verb would be the umbrella board policy declines. Dedup: the only PCGIR files on the board are `F-pcg-decompile-ir` (which defers this by name), `B-pcg-inspect-edge-direction-reversed`, `F-pcg-core-graph` and `F-foliage-namespace-has-no-behavioural-tests`; none asks for the compile direction and no `F-pcg-compile-ir` existed. Severity Medium with the High reading named and refused, because the call-count and silent-no-op halves are already owned by `F-property-set-batch` and `B-property-set-object-hop-notification-noop` and charging them here would triple-count one problem; reach declined in both directions.
