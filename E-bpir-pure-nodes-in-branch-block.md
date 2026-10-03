---
id: E-bpir-pure-nodes-in-branch-block
title: "blueprint.decompile prints a pure node textually inside the @then: block of the branch that first uses it, while later @merge: blocks consume its outputs — the body reads as 'this value only exists when the debug branch is taken', and disproving it costs a get_node_details call"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [bpir, decompile, pure-node, branch, block-placement, false-data-flow-read, get_node_details, blueprint]
encounters: 1
lastSeen: 2026-09-03T00:00:00Z
---

# A pure node placed in a branch block reads as conditionally-computed data

## What was called

```
blueprint.decompile {assetPath: "/Game/FPS/Weapons/BP_WeaponBase"}
```

## What happened

In the decompiled `ProcessHit` body, the `Break Hit Result` **pure** node is emitted under the
`@then:` block of a `bDebugPlotHits` branch — the first place one of its outputs is consumed —
while its other outputs are consumed further down, in later `@merge:` blocks.

Shape of what the body prints (elided to the structure that matters):

```
%b = branch(bDebugPlotHits) [true -> @then, false -> @merge]

@then:
    %n2 = break Hit Result(%hit)        # emitted here: first use
    call DrawDebugPoint(... %n2.<field> ...)
    exec -> @merge

@merge:
    ... consumes %n2.<other field> ...  # but %n2 was "computed" inside @then
```

Read as control flow — which is how a block-structured IR is meant to be read — that says the
break is only evaluated when debug plotting is on, and that the `@merge:` consumers read a
value that was never produced on the `false` path. That is a live data-flow bug, and it is the
first thing a reviewer will chase.

It is not one. The node is pure: it has no exec pins, so its placement in the text carries no
scheduling meaning at all and the value is available on both paths.

## The cost

Disproving the apparent bug required a separate call whose only purpose was to check for the
absence of exec pins:

```
blueprint.graph.get_node_details {assetPath: "/Game/FPS/Weapons/BP_WeaponBase",
                                  graphName: "EventGraph", nodeId: <break node>}
```

Every pure node that first appears inside a conditional block costs the same detour, and a
reviewer who does *not* make the call either files a phantom bug or — worse — dismisses a real
one nearby as "probably the same printing artefact".

## What was expected

Either of these removes the ambiguity, and neither changes emitted semantics:

- **A `pure` marker on the binding** — `%n2 = pure break Hit Result(%hit)` — so the reader
  knows the placement is textual, not scheduled. Cheapest, and it composes with existing text.
- **Hoist pure nodes to the top of the function body** (or to the innermost block that
  dominates *all* their consumers, rather than the block of their first consumer), so a value
  read in `@merge:` is never bound inside `@then:`.

The hoist is the stronger fix because it also makes the body round-trip-honest to a reader
diffing two revisions; the marker is the minimal stopgap and could ship first.

## Severity note

Rated **Medium**, not Low: this is not naming or docs friction. The body actively asserts a
control-flow relationship that does not exist, and the reader's only recourse is an extra RPC
per suspect node. It stops short of High because nothing downstream *consumes* the wrong
reading — the emitted graph is correct, and `blueprint.graph.get_node_details` does answer
truthfully when asked.

## Related

- `B-decompile-orphan-pure-nodes-grafted` (IN-REVIEW) — orphan (entry-less) pure nodes emitted
  as bindings inside an unrelated reachable event's last block. **Distinct:** there the nodes
  genuinely do not belong to the block and the body is lossy/non-round-trippable; here the node
  is legitimately reachable and the emit is semantically correct — only the *placement* misleads.
  Same underlying question, though (which block owns a pure node's binding), so a fixer taking
  either should look at both.
- `B-bpir-decompile-shared-tail-absorbed-into-branch` (DONE) — the exec-side analogue: a shared
  join absorbed into one branch's block, there with real corruption on recompile.
- `B-bpir-exec-target-leads-pure-node` (OPEN) — the compile-side counterpart, where a pure node
  leading a labeled block breaks exec-target resolution. Together these three say block
  membership of pure nodes is under-specified on both directions of the round trip.
- `E-bpir-pure-fn-name-undiscoverable` (IN-REVIEW) — pure-function *discovery*, orthogonal.

## History
- `#1-filed` `OPEN` WEAPONS-critic — Found during a WEAPONS critic review of `/Game/FPS/Weapons/BP_WeaponBase`. `blueprint.decompile` emits the `Break Hit Result` pure node inside the `@then:` block of a `bDebugPlotHits` branch — the block of its first consumer — while other outputs of the same node are consumed in later `@merge:` blocks, so the body reads as "this value is only computed when debug plotting is on" and the merge consumers read a value never produced on the `false` path. That reading is wrong: the node is pure and has no exec pins, so its textual placement carries no scheduling meaning. Confirming that cost a dedicated `blueprint.graph.get_node_details` call on the break node purely to check for absent exec pins, and the same detour is owed by every pure node that first appears inside a conditional block; a reviewer who skips it either files a phantom bug or dismisses a real neighbouring one as the same artefact. Ask: emit a `pure` marker on the binding (`%n2 = pure break Hit Result(...)`), or hoist pure nodes to the top of the function body / to the block dominating all their consumers rather than the block of the first consumer. Rated Medium rather than Low because the body asserts a control-flow relationship that does not exist, not merely awkward naming. Distinct from `B-decompile-orphan-pure-nodes-grafted` (orphan, entry-less, lossy) — this node is reachable and the emit is semantically correct.
- `#2-hoist-cross-block-pure-bindings` `IN-REVIEW` developer - Confirmed in current source: `EmitPureDependencies` binds a pure node at its first consumer and later blocks reuse the `%name`. Took the hoist option. New `FBpirDecompiler::HoistCrossBlockPureBindings` (`Source/PinWright/Private/Decompiler/BpirDecompiler.cpp/.h`), run per entry after its blocks are walked: a pure binding (no exec pins) printed inside a labeled block whose `%name` is read on a line of a different block moves, with every pure binding it reads, to the top of the entry body in emission order. Bindings in the entry block (above the first label) already dominate every consumer and stay. Scope ceiling (marked `ponytail:` in source): a pure value whose inputs reach an impure node's output (other than entry outputs, self, variable gets) keeps its first-consumer placement, since placing it after its producer needs a dominator walk over labels. `AppendNodeLine` now also records each node's emitted line count (`FEntryState::NodeToLineCount`). `bpir.txt` aspect 11 -> 12. Wiki: `docs/wiki-src/bpir.pure-impure.md` documents the placement rule. Tests (`Tests/Bpir/TestBpirPureBindingHoist.cpp`): `PinWright.bpir.pure_binding_hoist.CrossBlockReadHoistsToEntryTop` (function with branch, `Conv_IntToString($Base)` read in `@then` and `@merge`; asserts the binding is in the entry block before `branch(`, then compiles the output into a fresh Blueprint, checks one shared conversion node feeds both PrintStrings and the second decompile is byte-identical; fails with the hoist removed) and `PinWright.bpir.pure_binding_hoist.SameBlockReadStaysInPlace` (guard: a binding read only in its own block stays under its label).
- `#3-linux-verification` `IN-REVIEW` tester — Run3 on PinWright 7230b41d, UE 5.8 Linux, both non-skipped in run3/full. `PinWright.bpir.pure_binding_hoist.CrossBlockReadHoistsToEntryTop`: a pure `Conv_IntToString($Base)` read in `@then` and `@merge` is bound in the entry block before `branch(`, recompiles to one shared node, and re-decompiles byte-identical. `PinWright.bpir.pure_binding_hoist.SameBlockReadStaysInPlace` passed too. Remaining: the hoist has a deliberate ceiling (`ponytail:` in BpirDecompiler.cpp, also stated in `bpir.pure-impure.md`). A pure value that reads an impure node's output stays in its first consumer's block. That is exactly the misleading shape this ticket reports, and a Break Hit Result fed by a trace call is the common case. No `pure` marker was added for the residual case. The reporter's BP_WeaponBase ProcessHit was not replayed, so it is unknown which side it falls on. A human must accept the ceiling or ask for the marker (or a dominator-based hoist) on non-hoisted cross-block bindings.
