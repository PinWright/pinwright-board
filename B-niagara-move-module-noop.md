---
id: B-niagara-move-module-noop
title: "niagara.move_module reorders the ParameterMap chain but the result was invisible to every NodePosY-sorted readback (fixed on both producer + reader side)"
status: IN-REVIEW
severity: High
category: bug
tags: [niagara, move-module, inspect, stack, reorder, nodeposy, relayout, readback]
---

# `niagara.move_module` reorders the chain; the readback could not see it (now fixed two ways)

`niagara.move_module` **does** reorder the stack: it rewires the ParameterMap
execution-link chain — `DisconnectStackGroup` (BreakAllPinLinks + bridge prev→next)
then `ConnectStackGroup` re-inserts the module's node group at the destination
(`NiagaraEditHandler.cpp:1012-1045`, helpers at `478-531`). That is the same
parameter-map group surgery the engine's `FNiagaraStackGraphUtilities` performs, and
the move resolves the right module (a wrong `entryId` correctly returns
`[MODULE_NOT_FOUND]`). The title "silent no-op / nothing moved" in the original repro
was **wrong**.

The real defect had two halves, and both are now fixed:

1. **Producer side** — the move **never relaid out the graph**, so no node's
   `NodePosY` changed. The engine's editor drag-drop reorder ends with
   `FNiagaraStackGraphUtilities::RelayoutGraph` (`NiagaraStackGraphUtilities.cpp:3414`),
   which writes `Node->NodePosY` per chain-traversal order; the plugin's move skipped
   that follow-on relayout.
2. **Reader side** — every order-bearing readback sorted modules by `NodePosY`, not
   by the chain. `niagara.inspect includeStack` → `NiagaraDumpBuilder::AddGraphStackModules`
   (`NiagaraDumpBuilder.cpp:938-949`) sorted function-call nodes by `A.NodePosY <
   B.NodePosY` (tiebreak `NodeGuid`) and assigned the reported `index` from that sorted
   position. Because the move rewired the chain but left `NodePosY` untouched, the
   `includeStack` payload was **byte-for-byte identical** (same sha256) before and after
   a real reorder — which the original repro mistook for the move never happening. It
   happened; the readback (and the editor canvas) just couldn't see it.

These two fixes are **complementary, not duplicate** (the ticket already noted this).
The producer fix makes the existing `NodePosY`-sorted readback and the editor canvas
reflect the move; the reader fix makes `inspect` derive order from the execution chain
directly, so a reorder is observable even before/independent of a relayout. Both landed
from parallel fix hosts on the same ticket and both are in the tree.

## Why it matters

Reordering a Particle Update stack so a force/gravity module runs in the right place
is the **core** of the common "fix the simulation order" authoring task, and the
documented order-bearing readback (`niagara.inspect includeStack`) could not confirm the
move landed — it reported byte-identity regardless. An agent had no in-band way to
verify a reorder short of a `niagara.graph.get` ParameterMap hand-trace. Other tickets
assume this works: `F-niagara-set-module-script` lists `move_module` as the way to
"restore stack position".

## Repro (replay-confirmed via mcp__editor-automation__call)

Asset: `/Game/ExampleContent/EnhancedInput/VFX/Confetti/NS_Confetti`, emitter
`ConfettiBurst`, Particle Update group. Four `niagara.move_module` calls
(GravityForce `555DA2BB…`→toIndex 0; SolveForcesAndVelocity `1EA68167…`→toIndex 4;
one with `compile:true`→`compiled:true`) each returned `{success:true,
toIndex:<requested>}`, and every post-move `niagara.inspect {includeStack:true}`
payload was byte-for-byte identical (all sha256 `1a3e06f9…`, 516220 chars). **Re-read
under the correct diagnosis:** that byte-identity is the expected NodePosY-sort
artifact of a successful chain reorder that never relaid out NodePosY — not evidence
the move failed. (The original repro used `includeGraphs:false`, so it never saw the
ParameterMap chain that did change.) The authoritative way to observe the landed move
is the `niagara.graph.get` ParameterMap link chain, as
`E-niagara-inspect-no-param-readback-projection` #5 established.

Detailed replay steps:

1. Baseline `niagara.inspect`
   `{assetPath:".../NS_Confetti", includeProperties:false, includeStack:true, includeGraphs:false, includeCompile:false}`
   -> spills to file, 516220 chars, sha256 `1a3e06f9…`. `GravityForce` at `index:13`.
2. `niagara.move_module`
   `{assetPath:".../NS_Confetti", emitter:"ConfettiBurst", entryId:"555DA2BB4EE8A0A0D26F4E9C1BAE3F16", scriptUsage:"ParticleUpdateScript", toIndex:0, save:false}`
   -> `{success:true, operation:"move_module", entryId:"555DA2BB…", toIndex:0}`
3. Re-`niagara.inspect` (same args) -> **byte-identical**, 516220 chars, sha256
   `1a3e06f9…`. `GravityForce` **still at `index:13`** — did NOT move to the top
   of the Particle Update group (pre-fix symptom).
4. Independent repeat on a second module:
   `niagara.move_module {entryId:"1EA681674152F5B73C37F3AE70009629" (SolveForcesAndVelocity), scriptUsage:"ParticleUpdateScript", toIndex:4, save:false}`
   -> `{success:true, toIndex:4}`; re-inspect again byte-identical, sha256
   `1a3e06f9…`, `SolveForcesAndVelocity` **still at `index:10`**.
5. With `compile:true` to rule out a pending-recompile excuse:
   `niagara.move_module {entryId:"555DA2BB…", scriptUsage:"ParticleUpdateScript", toIndex:0, compile:true, save:false}`
   -> `{success:true, compileRequested:true, compiled:true, toIndex:0}`; re-inspect
   **still byte-identical** (sha256 `1a3e06f9…`), `GravityForce` still at `index:13`.

Four `move_module` calls (two distinct modules, two `toIndex` values, with and
without `compile`) all returned `success:true` with the requested `toIndex`, yet
every post-move `niagara.inspect` `includeStack` payload was byte-for-byte
identical (same sha256). The reporter read this as "no reorder," but the move
**did** land — the `NodePosY`-sorted `index` simply cannot see a connection-only
reorder (`move_module` rewires the ParameterMap chain and never touched
`NodePosY`), so the readback was structurally identical whether or not the move
worked. The `entryId`/`scriptUsage` resolve correctly (a wrong `entryId` returns
`[MODULE_NOT_FOUND]`, confirmed during replay).

**Workaround (pre-fix):** verify the reorder via `niagara.graph.get` and hand-trace
the ParameterMap `OutputMap -> InputMap` link chain (which `E-…-#5` records as the
fallback that proved the move landed).

## Fix (two complementary halves, both applied)

**Producer side (`NiagaraEditHandler.cpp`):** after the move's `ConnectStackGroup`
reconnect, inline the minimal relayout inside the move transaction. A vendored
`RelayoutModuleNodePositions(OutputNode)` helper (same anonymous namespace as
`GetOrderedModuleNodes`) walks the now-reordered module chain via
`GetOrderedModuleNodes` and assigns each module function-call node a strictly
increasing `NodePosY` (output-adjacent module smallest, matching the engine's
convention) — the exact key `AddGraphStackModules` sorts on, so the `includeStack` /
model-builder readbacks and the editor canvas now reflect the move. The chain reconnect
itself is already correct and must be left intact. A direct
`FNiagaraStackGraphUtilities::RelayoutGraph` call would not link — it is not
`NIAGARAEDITOR_API` (like `GetOrderedModuleNodes` and several other members the plugin
already reimplements inline).

**Reader side (`NiagaraDumpBuilder.cpp`):** make the `niagara.inspect` stack readback
derive module order (and the `index` field) from the ParameterMap execution chain
instead of sorting by visual `NodePosY`. `AddGraphStackModules` now calls a new
`CollectStackModuleNodesInExecutionOrder` that walks backward from each output node's
ParameterMap input pin through the linked `UNiagaraNodeFunctionCall` chain (mirroring
`GetOrderedModuleNodes`, reusing the exported `PinWrightNiagara::FindParameterMapPin`),
assigning `index` from that order, then appends any chain-unreachable function nodes in
`NodePosY` order so every module is still listed. This covers both the emitter
(`BuildEmitterStackJson`) and system (`BuildStackJson`) paths of `niagara.inspect`.

No asset-dump aspect-version bump is needed: the dumper format is unchanged — the
stack JSON feeds the live `niagara.inspect` RPC (it is not a cached dump sidecar; see
TestNiagaraDumpBuilder `WritesNiagaraAspectFiles`), and the producer change only alters
in-asset `NodePosY` data, and only when a move is actually performed. Note: the sibling
`NiagaraModelBuilder` `AddGraphStackModules` (the separate `niagara.model.get` RPC, not
`niagara.inspect`) still has the same posY sort and is out of scope for this ticket.

## Distinct from

- `B-niagara-save-no-disk-write` (OPEN) — that's about `save` not writing to disk;
  this is the relayout/readback gap, verified with `save:false` against the live
  in-editor graph.
- `B-niagara-module-input-stack-infer` (IN-REVIEW) — `set_module_input` wrongly
  *rejects* with `INVALID_STACK`; this `move_module` call *accepts* and applies, but
  the apply was invisible to the NodePosY readback.
- `E-niagara-inspect-no-param-readback-projection` (OPEN), `#5` — sibling from the same
  test session: it tracks the inspect *response size / projection* (no `emitter`
  filter, ~500KB spill) and asks for a per-group `moduleOrder`. This ticket is the
  narrower **correctness** half (the `index` value is faithful to execution order, plus
  the producer relayout); that ticket can still narrow the spill. Complementary, not
  duplicate.

## History
- `#1-initial-repro` `OPEN` reporter — Replay-confirmed via mcp__editor-automation__call on `/Game/ExampleContent/EnhancedInput/VFX/Confetti/NS_Confetti` emitter `ConfettiBurst`. Four `niagara.move_module` calls (GravityForce `555DA2BB…`→toIndex 0; SolveForcesAndVelocity `1EA68167…`→toIndex 4; both with `scriptUsage:"ParticleUpdateScript"`; one repeat with `compile:true`→`compiled:true`) each returned `{success:true, toIndex:<requested>}`, but every post-move `niagara.inspect {includeStack:true}` payload was byte-for-byte identical to the pre-move baseline (all sha256 `1a3e06f9…`, 516220 chars), with `GravityForce` still at `index:13` and `SolveForcesAndVelocity` still at `index:10`. Originally diagnosed as zero reordering; rework shows the move lands but the `NodePosY`-sorted `index` cannot observe a connection-only reorder. A wrong `entryId` returns `[MODULE_NOT_FOUND]`, so the target resolves correctly. Dedup: ripgrep across OPEN/DONE/WONTFIX found no existing `move_module` ticket (`F-niagara-set-module-script` only references it as a stack-restore tool, assuming it works; `B-niagara-save-no-disk-write` is the disk-write defect, not the in-memory reorder; `B-niagara-module-input-stack-infer` is a wrongful *rejection*, the opposite symptom).
- `#2-reword-and-fix-producer` `IN-REVIEW` developer — REWORDED: three validity lenses (correctness, adversarial, board-historian) all found the "silent no-op" diagnosis false. The move DOES reorder the ParameterMap chain (`NiagaraEditHandler.cpp:1012-1045`, `DisconnectStackGroup`/`ConnectStackGroup` at `478-531`); the byte-identical `includeStack` readback is a NodePosY-sort artifact (`NiagaraDumpBuilder.cpp:938-949` sorts by `NodePosY`, which the move never touches) of a reorder that never got a follow-on relayout. Title/body/Fix rewritten to "reorders chain but never relayouts NodePosY". FIXED in `Source/PinWright/Private/Handlers/Niagara/NiagaraEditHandler.cpp`: added a vendored `RelayoutModuleNodePositions(OutputNode)` helper (in the same anonymous namespace as `GetOrderedModuleNodes`) and call it after the successful `ConnectStackGroup` in `ApplyModuleMutation`'s MoveModule branch. It walks the reordered module chain via `GetOrderedModuleNodes` and assigns each module node a strictly increasing `NodePosY` (output-adjacent smallest), the exact key `AddGraphStackModules` sorts on. A direct `FNiagaraStackGraphUtilities::RelayoutGraph` call was rejected because that engine helper is NOT `NIAGARAEDITOR_API` and would not link cross-module (same reason the plugin already reimplements `GetOrderedModuleNodes`/`GetStackFunctionOverrideNode` inline). No aspect-version bump (dumper format unchanged; only per-asset `NodePosY` data changes, only on an actual move). TEST: `Source/PinWright/Private/Tests/Niagara/TestNiagaraMoveModule.cpp` — `PinWright.niagara.move_module.RelayoutsNodePosY` builds a transient system, adds three ParticleUpdate modules, forces their `NodePosY` equal, invokes `niagara.move_module` to move the tail module to index 0, then asserts (a) the moved module is now first in the ParameterMap chain — walked via the plugin's linkable `PinWrightNiagara::FindParameterMapPin` — guarding a real no-op regression, and (b) the modules' `NodePosY` values are no longer all-equal and their ascending-`NodePosY` order matches the new chain order reversed (the `includeStack` readback order); both fail if the relayout call is reverted. Plus a registration test.
- `#3-fix-inspect-stack-execution-order` `IN-REVIEW` developer — Independently (parallel fix host) confirmed the same REWORD via correctness + adversarial + board-history lenses, and fixed the **reader** half: the inspect stack readback now orders modules (and assigns `index`) by the ParameterMap execution chain instead of the visual `NodePosY` sort. In `Source/PinWright/Private/Handlers/Niagara/NiagaraDumpBuilder.cpp`, `AddGraphStackModules` now calls a new `CollectStackModuleNodesInExecutionOrder` that walks backward from each output node's ParameterMap input pin through the linked `UNiagaraNodeFunctionCall` chain (mirroring `GetOrderedModuleNodes`, NiagaraEditHandler.cpp), then appends any chain-unreachable function nodes in `NodePosY` order so every module is still listed; it reuses the exported `PinWrightNiagara::FindParameterMapPin`. This fixes both the `niagara.inspect includeStack` emitter path (`BuildEmitterStackJson`) and the system path (`BuildStackJson`). No aspect-version bump: the stack JSON feeds the live `niagara.inspect` RPC (NiagaraInspectHandler.cpp:26/48) and is not a cached dump sidecar (`niagara_stack.json` is never written — see TestNiagaraDumpBuilder `WritesNiagaraAspectFiles`). Regression test `PinWright.Assets.Niagara.DumpBuilder.StackOrderFollowsExecutionChain` in `Source/PinWright/Private/Tests/Assets/TestNiagaraDumpBuilder.cpp` builds a transient emitter ParticleUpdate graph chained C→B→A→Output with `NodePosY` set in the OPPOSITE order, then asserts the readback `entryId` order is C, B, A (execution chain) while `posY` descends — which fails if the code reverts to the NodePosY sort (it would emit A, B, C). Note: the sibling `NiagaraModelBuilder` `AddGraphStackModules` (the separate `niagara.model.get` RPC) still has the same posY sort and is out of scope. This reader fix and `#2`'s producer fix are complementary and both present in the tree.
