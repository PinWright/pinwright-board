---
id: B-decompile-orphan-pure-nodes-grafted
title: "blueprint.decompile grafts an entry-less subgraph's orphan pure-value nodes into an unrelated reachable event's last block (and drops that subgraph's orphan exec nodes from the body), producing misleading, non-round-trippable BPIR"
status: IN-REVIEW
severity: Medium
category: bug
tags: [bpir, decompile, orphan, pure-node, entryless, timeline, round-trip, reachability]
encounters: 1
lastSeen: 2026-06-28T23:22:29Z
---

# Orphan pure-value nodes are emitted as bindings inside an unrelated event's block

When a Blueprint EventGraph contains an **entry-less but fully-wired value-producing
subgraph** (e.g. an auto-play Timeline whose `Play` pin is never wired from any event,
driving `Set Relative Location`/`Set Relative Scale 3D`/`Spawn Emitter` via `Lerp(Vector)`
data nodes) **alongside** at least one reachable entry point, `blueprint.decompile`
mis-emits the orphan subgraph:

- The orphan **pure / value-producing** nodes (the Timeline value-reader, the two
  `Lerp(Vector)` calls, `GetWorldLocation`) are emitted as `%nK = call ...` **bindings
  inside the last block of the last reachable entry** — here the new custom event's
  `@merge:` block — even though nothing in that block consumes them and they belong to a
  completely different subgraph.
- The orphan **exec** nodes of that same subgraph (`Set Relative Location`,
  `Set Relative Scale 3D`, `Spawn Emitter at Location`) are **dropped from the body
  entirely** and appear only as `# orphan` warnings.

Net effect: the decompiled BPIR is (1) **misleading** — the body falsely shows the
reachable custom event owning a Timeline + two VLerps + a GetWorldLocation that are not
part of it; (2) **lossy** — the entry-less Timeline→Set/Spawn exec flow is absent from the
emitted body; and (3) **not round-trippable** — recompiling this text would graft the
Timeline/VLerps/GetWorldLocation into the custom event's merge block and would NOT
reconstruct the original auto-play-Timeline subgraph, corrupting the asset's logic.

Note the inconsistency: the four orphan **exec** nodes ARE warned, but the orphan **pure**
nodes that actually land in the body (`Lerp (Vector)` ×2, `GetWorldLocation`) carry **no**
warning — so a consumer reading the body has no signal that those three lines are
misplaced.

This is a decompiler defect independent of `blueprint.compile_bpir`: the BPIR upsert that
added the custom event was clean and idempotent (the pre-existing graph's wiring is fully
intact in `get_nodes`); the upsert merely supplied the reachable entry whose last block the
decompiler then dumped the unrelated orphans into.

## Repro (mcp__pinwright__call, verbatim)

Asset: `/Game/ExampleContent/Blueprints/Blueprints/BP_Timeline_Ball.BP_Timeline_Ball`
(Content Examples; its `Bounce` Timeline is configured "set to play and loop
automatically", so no event node drives it — the whole Timeline subgraph is entry-less).
After upserting any reachable `custom_event` (the new event lives at `@(2000..3200, …)`,
the pre-existing Timeline subgraph at `@(560..1504, …)`):

`blueprint.decompile {assetPath:"/Game/ExampleContent/Blueprints/Blueprints/BP_Timeline_Ball.BP_Timeline_Ball", graphName:"EventGraph"}` →

```
@merge:
    call PrintString(InString: "Done", ...) @(3200, 2000)
    %n3: enum<ETimelineDirection> = call K2Node_Timeline(NewTime: 0.0) @(560, 400)
    %n2: struct<Vector> = call VLerp(A: 0.5,0.5,0.5, B: 0.8,0.8,0.2, Alpha: %n3.Scale) @(1008, 704)
    %n4: struct<Vector> = call VLerp(A: 0,0,0, B: 0,0,500, Alpha: %n3.Movement) @(1008, 480)
    %n5: struct<Vector> = call K2_GetComponentLocation(Target: $Ball) @(768, 1008)
}
```
with warnings (note: only the EXEC orphans are listed; the VLerps/GetWorldLocation that
landed in the body above are NOT warned):
```
Orphaned node not reachable from any entry point: EventGraph :: K2Node_Timeline 'Bounce' nodeId=AFB6...C6BA @(560,400)
Orphaned node not reachable from any entry point: EventGraph :: K2Node_CallFunction 'Set Relative Location' nodeId=62B3...7360 @(1504,384)
Orphaned node not reachable from any entry point: EventGraph :: K2Node_CallFunction 'Set Relative Scale 3D' nodeId=59FA...245C @(1504,608)
Orphaned node not reachable from any entry point: EventGraph :: K2Node_CallFunction 'Spawn Emitter at Location' nodeId=56A0...24DF @(1008,944)
```

Ground truth from `blueprint.graph.get_nodes` on the same graph: the Timeline's `Update`→
`Set Relative Location`→`Set Relative Scale 3D`, `Impact`→`Spawn Emitter`, and
`Movement`/`Scale`→the two `Lerp (Vector)` data edges are all present and wired — i.e. a
real entry-less subgraph, not junk, that the decompile body neither represents faithfully
nor confines to its own section.

## Expected

The reachable event's decompiled body must contain only nodes that belong to it, so
`decompile → edit → recompile` does not graft foreign nodes into the event. Concretely, the
orphan-pure injection pass must distinguish two kinds of pure orphan:

- **Genuinely standalone** (output feeds nothing — e.g. an authored side-effect
  `%ss = subsystem<X>()`): keep hoisting it into the host entry so it round-trips. This
  behavior is load-bearing and locked by `B-bpir-subsystem-getter-roundtrip` (DONE) — a fix
  must NOT regress it.
- **A data feeder of an entry-less subgraph** (output IS consumed by other, unreachable
  nodes — the auto-play Timeline's two `Lerp(Vector)` and `GetWorldLocation`): it belongs to
  that subgraph, not to the host entry. It must NOT be grafted into the body; instead it must
  be reported by the orphan-warning sweep, symmetrically with the exec orphans (which are
  already warned). This removes both the misleading body lines and the silent-loss asymmetry.

(The original Expected proposed either (a) relocating the whole entry-less subgraph under its
own non-entry section — a sizable new grammar/decompiler/compiler feature, out of scope for a
Medium bug — or (b) "omit the orphan body nodes" wholesale, which applied verbatim would also
suppress the standalone subsystem getter and regress its round-trip. The discriminator above
is the achievable, non-regressing scope.)

**Fix:** In the decompiler's orphan-pure injection pass (`BpirDecompiler.cpp`), before
hoisting a pure orphan into the host entry, check whether any of its output data pins are
linked; if so it has a downstream consumer — skip the graft and record the node. Extend the
orphan-warning sweep (which currently `continue`s on every pure node) to warn on those
recorded pure orphans, guarded by a body-presence check so a pure node legitimately pulled in
as another hoisted orphan's dependency is not falsely warned. Pure orphans with no consumer
keep being hoisted exactly as before.

## History
- `#1-initial-repro` `OPEN` reporter — Found via mcp-test-workflow seed `blueprint.compile_bpir` (idempotent-re-apply angle) on `/Game/ExampleContent/Blueprints/Blueprints/BP_Timeline_Ball`; culprit is `blueprint.decompile`, not the seed. The compile_bpir upsert was clean/idempotent (new custom event appears once, pre-existing Timeline subgraph fully intact per `get_nodes`). Replay-confirmed by the oracle: `blueprint.decompile` of the saved EventGraph emits the entry-less `Bounce` Timeline's pure value nodes (`%n3` Timeline, `%n2`/`%n4` `Lerp (Vector)`, `%n5` `GetWorldLocation`) as bindings inside the unrelated custom event's `@merge:` block (none consumed there), drops that subgraph's three orphan exec setters from the body (warned only), and warns on the exec orphans but NOT on the pure nodes that landed in the body. Body is misleading + lossy on exec + non-round-trippable. Distinct from `B-orphan-finder-vs-decompiler-disagree` / `B-bpir-decompile-warnings-skip-composite-bridge` (those are about the WARNINGS array's reachability false-positives) and from `E-decompile-unreachable-output-pin-labels` DONE (disconnected exec-pin LABELS on AsyncAction) — this is about orphan PURE nodes wrongly placed in a reachable block's BODY. Severity Medium: `decompile` is an every-session method, but the trigger needs an entry-less value-producing subgraph coexisting with a reachable entry (auto-play timelines / externally-driven event-less graphs), and the orphan warnings give a partial signal, with `get_nodes`/`get_graph_connections` as the ground-truth workaround.
- `#2-reword+fix` `IN-REVIEW` developer — REWORDED then fixed. The original Expected's two proposed paths were mis-scoped: path (a) (relocate the entry-less subgraph under its own non-entry section) is a new grammar+decompiler+compiler feature, over-scope for a Medium bug; path (b) ("omit the orphan body nodes" wholesale) applied verbatim would also drop the *standalone* unconsumed subsystem getter and regress `B-bpir-subsystem-getter-roundtrip` (DONE), which relies on the injection pass hoisting `%ss = subsystem<X>()` into BeginPlay. Reworded Expected to the achievable discriminator + added a **Fix:** line: a pure orphan with NO downstream consumer keeps round-tripping (hoisted); a pure orphan whose output IS consumed (an entry-less subgraph's data feeder) is omitted from the body and warned instead. Implemented in `Plugins/PinWright/Source/PinWright/Private/Decompiler/BpirDecompiler.cpp`: (1) the orphan-pure injection loop now skips a pure node when any output data pin has `LinkedTo.Num() > 0`, collecting it into a new `OmittedPureOrphans` set instead of emitting it as a `%nK = …` binding; (2) the orphan-warning sweep no longer blanket-`continue`s on pure nodes — it now warns nodes in that set, guarded by a body-presence check across `RenderedEntries[*].State.NodeToValueName` so a dependency that was nonetheless emitted into a body isn't falsely warned. Exec-orphan warnings and standalone-pure hoisting are unchanged. Regression test `PinWright.bpir.decompiler.OrphanPureNodeNotGrafted` (`Plugins/PinWright/Source/PinWright/Private/Tests/Bpir/TestBpirDecompilerOrphanPureGrafting.cpp`) builds BeginPlay→PrintString (reachable) plus a `MakeLiteralString` pure orphan feeding an orphan PrintString consumer and a standalone unconsumed `MakeLiteralInt`; asserts the feeder's GUID appears in the orphan warnings and `MakeLiteralString` is absent from the body (both fail pre-fix, when the feeder was grafted into the body and pure nodes were never warned), while the standalone's GUID is NOT warned (guards the subsystem-getter behavior). Did not compile/run (later phase).
