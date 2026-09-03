---
id: B-bpir-key-released-entry-silently-dropped
title: "entry key_released <Key>() reports compiled:true and createdNodes, then wires nothing — the body is left as orphaned nodes and the handler simply does not exist"
status: OPEN
severity: High
category: bug
tags: [bpir, compile-bpir, input-key, key-released, silent-noop, orphan-nodes, entry-points]
encounters: 1
lastSeen: 2026-09-02T23:20:00+03:00
---

# `entry key_released` produces orphans, not a handler

`blueprint.compile_bpir` accepts an `entry key_released <Key>()` block, answers `compiled: true`,
`status: "UpToDate"`, `errors: []`, `warnings: []`, and returns a `createdNodes` list — and the
handler is not in the graph afterwards. The body's nodes are created and left unwired; nothing is
attached to the `UK2Node_InputKey` node's **Released** exec pin.

`bpir.entry-points` lists `entry key_released SpaceBar()` beside `entry key_pressed SpaceBar()` with
no caveat, so there is nothing to warn a caller off it.

## Repro

`/Game/FPS/Weapons/Test/BP_WeaponTestPawn` (a `DefaultPawn` child). One `compile_bpir` document
containing, among other entries, a matched pair:

```
entry key_pressed LeftMouseButton() {
    %ok = call IsValid(Object: $CurrentWeapon)
    %b = branch(%ok) [true -> @go, false -> @done]
@go:
    call StartFire(Target: $CurrentWeapon)
@done:
}

entry key_released LeftMouseButton() {
    %ok = call IsValid(Object: $CurrentWeapon)
    %b = branch(%ok) [true -> @go, false -> @done]
@go:
    call StopFire(Target: $CurrentWeapon)
@done:
}
```

Response: `compiled: true, status: "UpToDate", errors: [], warnings: [], nodeCount: 44`.

`blueprint.decompile {graphName: "EventGraph"}` immediately afterwards emits
`entry key_pressed LeftMouseButton()` and `entry key_pressed RightMouseButton()` — and **no
`key_released` entry of any kind** — plus:

```
warnings: [
  "Orphaned node not reachable from any entry point: EventGraph :: K2Node_CallFunction 'Is Valid' nodeId=9E079B47-… @(0,2424)",
  "Orphaned node not reachable from any entry point: EventGraph :: K2Node_CallFunction 'Is Valid' nodeId=9CC0E756-… @(0,3328)"
]
```

Those two y-coordinates sit exactly between the surviving `key_pressed` entries, i.e. where each
`key_released` body was laid out. The bodies were built; only the exec wire into the Released pin
was not made, and the branch that would have carried it is gone too.

**Re-compiling the released entry on its own does not help, and the shape of the failure is stable.**
A second, separate call containing only the `key_released LeftMouseButton()` block returned
`compiled: true, nodeCount: 5, errors: [], warnings: []`, and the decompile afterwards still shows
no `key_released` entry — plus a **third** orphan `Is Valid` at `@(240, 5000)`, the new body. So each
attempt silently accretes dead nodes into the graph.

## Why this is High and not cosmetic

The graph looks right in every readout an agent has short of a decompile: the compile succeeded, the
node count went up, and no diagnostic fired. The consequence in this case is a test pawn whose
trigger never releases — hold-to-fire starts and never stops — which is exactly the sort of defect
that reads as a *game-logic* bug and gets hunted in the wrong place. It also poisons the asset: the
orphan sweep now reports nodes the author did not knowingly create, so a later
`blueprint.graph.delete_orphaned_nodes` becomes a judgement call rather than a cleanup.

## Relationship to `B-bpir-upsert-skips-inputkey-entries` (DONE)

Same node class, opposite symptom, and worth reading together. That ticket was *"upsert
(append/replace) silently **duplicates** key_pressed/key_released InputKey entries instead of
replacing them"* — re-authoring an existing `key_pressed Tab()` left the old `UK2Node_InputKey` in
place and added a second. It is `DONE`.

This is not that bug and not a regression report against its fix, because the mechanism differs:
nothing here is duplicated. One `UK2Node_InputKey` exists per key, the Pressed pin is wired, and the
Released pin is not. The connection worth flagging to whoever picks this up is that
`UK2Node_InputKey` carries **both** Pressed and Released exec outputs on a single node, so
"one entry per node" and "two BPIR entries per node" are in tension — an upsert keyed on the node
rather than on the pin would explain both tickets. That is a guess offered as a lead; no source was
read.

## The ask

1. Wire `entry key_released <Key>()` to the existing `UK2Node_InputKey`'s Released pin, creating the
   node only when no node for that key exists.
2. If (1) is not immediately available, **refuse the entry** with a typed error naming
   `key_released` as unsupported. A refusal costs one compile; a silent success costs a debugging
   session in the wrong file, and leaves orphans behind either way.
3. Whatever the outcome, do not leave the body's nodes in the graph when the entry they belong to
   was not created — the rollback path that `compile_bpir` already uses for `COMPILE_FAILED` is the
   right behaviour here.
4. Until fixed, `bpir.entry-points` should stop listing `entry key_released SpaceBar()` as an
   available entry shape, or mark it non-functional.

## Dedup

Board-wide search for `key_released`, `InputKey`, `key_pressed`, `orphaned node`.
`B-bpir-upsert-skips-inputkey-entries` (DONE, High) is the nearest neighbour and is analysed above —
different mechanism, not a duplicate, not a regression claim.
`B-bpir-input-event-entry-signatures-unknown` concerns what an input entry's *signature* is, not
whether its body gets wired. `B-bpir-orphan-warning-diagnostics-lossy` and
`B-orphan-finder-vs-decompiler-disagree` are about how orphans are *reported*; this is about a verb
creating them and calling it success. `B-bpir-statement-cast-success-unwired-replace-shared-topology`
is the same family of silent-unwired-on-success but on cast exec pins under replace mode. No ticket
covers `key_released` failing to wire.

## Severity

**High**, per *"silent false-success … the caller trusts a result that is a lie and builds on it"*.
Every response field said the handler existed. Reach modifier declined: input-key entries are common
in test harnesses and prototypes but are not in almost every session, and projects on Enhanced Input
will not touch this path at all.

## History
- `#1-filed` `OPEN` reporter — `entry key_released <Key>()` returns `compiled: true, status: "UpToDate", errors: [], warnings: []` with a populated `createdNodes` list, and creates no handler. Repro on `/Game/FPS/Weapons/Test/BP_WeaponTestPawn` (`DefaultPawn` child): one document with matched `entry key_pressed LeftMouseButton()` / `entry key_released LeftMouseButton()` blocks (each `IsValid` -> `branch` -> `StartFire`/`StopFire`) compiled with `nodeCount: 44` and no diagnostics; `blueprint.decompile {graphName:"EventGraph"}` immediately after emits both `key_pressed` entries and **no `key_released` entry at all**, plus two `Orphaned node not reachable from any entry point … K2Node_CallFunction 'Is Valid'` warnings at `@(0,2424)` and `@(0,3328)` — exactly the y-coordinates where each released body was laid out. So the bodies were built and only the exec wire into the `UK2Node_InputKey` Released pin was never made. Re-authoring the released entry alone in a separate call returned `compiled: true, nodeCount: 5` with no diagnostics and still produced no entry, adding a **third** orphan `Is Valid` at `@(240,5000)` — each attempt accretes dead nodes. Consequence in situ: a test pawn whose trigger never releases, which reads as a game-logic bug and gets hunted in the wrong file, and an asset whose orphan sweep now lists nodes the author never knowingly created. **Relationship to `B-bpir-upsert-skips-inputkey-entries` (DONE)**, stated so it is not mistaken for a regression: that ticket was silent *duplication* of InputKey entries on re-author; nothing here duplicates — one node per key exists, Pressed is wired, Released is not. Lead offered as a guess with no source read: `UK2Node_InputKey` carries both Pressed and Released exec outputs on one node, so an upsert keyed on the node rather than the pin would explain both tickets. Asked for: wire the released entry to the existing node's Released pin; failing that refuse `key_released` with a typed error rather than reporting success; in either case roll the body's nodes back instead of leaving orphans, using the path `COMPILE_FAILED` already takes; and until fixed stop listing `entry key_released SpaceBar()` on `bpir.entry-points` as an available shape. Severity High per the silent-false-success band; reach modifier declined because Enhanced Input projects never reach this path.
