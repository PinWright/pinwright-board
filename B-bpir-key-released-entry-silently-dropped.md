---
id: B-bpir-key-released-entry-silently-dropped
title: "Decompile starts key_released InputKey bodies from the Pressed exec pin and silently drops them"
status: IN-REVIEW
severity: High
category: bug
tags: [bpir, decompile, input-key, key-released, silent-data-loss, round-trip, entry-points]
encounters: 1
lastSeen: 2026-09-02T23:20:00+03:00
---

# `key_released` decompile drops its entry body

A Blueprint with a released-only `UK2Node_InputKey` decompiles successfully and emits the correct
`entry key_released <Key>()` signature, but the emitted block omits the body wired to `Released`.
That is silent data loss in BPIR output.

Current compiler source refutes the original diagnosis that `key_released` silently fails to wire.
`SetupKeyEvent(..., true)` resolves the named `Released` pin, the body wire uses that exact pin,
connection failures accumulate as compile errors, created nodes are deleted on compiler failure,
and the handler also has snapshot rollback. The original runtime orphan evidence was not reproduced
in this verification-constrained pass and is retained below and in `#1-filed` as report history.

## Original runtime report

The original reporter used `/Game/FPS/Weapons/Test/BP_WeaponTestPawn` (a `DefaultPawn` child) and
submitted one `compile_bpir` document containing, among other entries, a matched pair:

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

The report recorded: `compiled: true, status: "UpToDate", errors: [], warnings: [], nodeCount: 44`.

The report's immediate `blueprint.decompile {graphName: "EventGraph"}` output emitted
`entry key_pressed LeftMouseButton()` and `entry key_pressed RightMouseButton()` — and **no
`key_released` entry of any kind** — plus:

```
warnings: [
  "Orphaned node not reachable from any entry point: EventGraph :: K2Node_CallFunction 'Is Valid' nodeId=9E079B47-… @(0,2424)",
  "Orphaned node not reachable from any entry point: EventGraph :: K2Node_CallFunction 'Is Valid' nodeId=9CC0E756-… @(0,3328)"
]
```

Those two y-coordinates were reported between the surviving `key_pressed` entries. Current compiler
source does not support the original conclusion that the Released wire was never made; it does
explain why decompile omitted a correctly wired Released body.

The reporter also made a second call containing only the `key_released LeftMouseButton()` block. It returned
`compiled: true, nodeCount: 5, errors: [], warnings: []`, and the decompile afterwards still shows
no `key_released` entry and a third orphan warning. That runtime orphan accumulation was not
reproduced during this static, no-editor implementation pass.

## Diagnosis and ask

`FBpirTextEmitter::EmitEntrySignature` recognizes an active `Released` output and emits a
`key_released` signature. `FBpirDecompiler::DecompileGraphInternal` nevertheless started every
entry-body walk at `GetExecOutputPin(EntryNode, 0)`. UE allocates InputKey outputs in `Pressed`, then
`Released` order, so a released-only entry walked the unlinked `Pressed` pin and silently emitted an
empty body.

The fix must make the body walk use the same active Released sense as the emitted signature without
changing compiler wiring, node creation, or rollback. The InputKey decompiler still represents an
entry by node rather than by node plus exec pin; supporting simultaneous pressed and released bodies
on one physical node is a broader model change and remains outside this ticket.

`B-bpir-upsert-skips-inputkey-entries` (DONE) concerns duplicate InputKey nodes during upsert and is
not the same defect. The orphan-warning tickets concern diagnostics rather than entry-body traversal.

## Severity

**High** because a successful decompile silently omits authored behavior on a documented entry type.
Reach is limited to legacy InputKey entries, so no upward reach adjustment applies.

## Fix

The root cause was the decompiler's unconditional first-output traversal even when the entry
signature identified the active `Released` sense. `BpirDecompiler.cpp` now uses the existing
`FBpirInputKeyHelpers` production helpers to select the active `Released` exec pin before walking
that InputKey body, while retaining first-output traversal for other entry types.

Files changed:

- `Source/PinWright/Private/Decompiler/BpirDecompiler.cpp`
- `Source/PinWright/Private/Tests/Bpir/TestCompilerGapCoverage.cpp`
- `docs/wiki-src/bpir.examples.key-input.md`
- `Docs/wiki-src/bpir.examples.md`

Regression test: `PinWright.bpir.compiler.integration.KeyReleasedEntry`
(`FCompilerIntegrationKeyReleasedEntryTest`) calls the production compiler, orphan finder, and
decompiler and verifies graph topology plus same-entry body preservation.

Deliberately not changed: compiler dispatch or wiring, InputKey node creation/reuse, compiler or
handler rollback, the `key_pressed` / `key_released` language contract, and `bpir.entry-points`.

### Unexplained by this fix / tester must check

The original missing-signature result and orphan accumulation are not explained by this decompiler
fix. Before moving the ticket to `DONE`, create a fresh Blueprint and use `blueprint.compile_bpir`
to compile one `entry key_released <Key>()` block with an observable body twice, unchanged. After
each compile, read back the EventGraph with `blueprint.decompile`, count the `UK2Node_InputKey`
nodes for that key, and record every orphan warning. Reopen this ticket if either successful compile
is missing the `key_released` signature or body on read-back, if the second compile increases the
InputKey-node count, or if either read-back reports a new orphan.

One conditional accumulation mechanism is plausible from current source but not confirmed:
`BpirCompiler.cpp:3032-3038` recognizes an existing InputKey entry for upsert deletion only when
the requested exec sense is active. If `Released` is already disconnected, that node is skipped and
`SetupKeyEvent` creates a new InputKey node at `BpirCompiler.cpp:5007` on the retry. Only the tester
reproduction above can establish whether that path explains the original report.

## History
- `#1-filed` `OPEN` reporter — `entry key_released <Key>()` returns `compiled: true, status: "UpToDate", errors: [], warnings: []` with a populated `createdNodes` list, and creates no handler. Repro on `/Game/FPS/Weapons/Test/BP_WeaponTestPawn` (`DefaultPawn` child): one document with matched `entry key_pressed LeftMouseButton()` / `entry key_released LeftMouseButton()` blocks (each `IsValid` -> `branch` -> `StartFire`/`StopFire`) compiled with `nodeCount: 44` and no diagnostics; `blueprint.decompile {graphName:"EventGraph"}` immediately after emits both `key_pressed` entries and **no `key_released` entry at all**, plus two `Orphaned node not reachable from any entry point … K2Node_CallFunction 'Is Valid'` warnings at `@(0,2424)` and `@(0,3328)` — exactly the y-coordinates where each released body was laid out. So the bodies were built and only the exec wire into the `UK2Node_InputKey` Released pin was never made. Re-authoring the released entry alone in a separate call returned `compiled: true, nodeCount: 5` with no diagnostics and still produced no entry, adding a **third** orphan `Is Valid` at `@(240,5000)` — each attempt accretes dead nodes. Consequence in situ: a test pawn whose trigger never releases, which reads as a game-logic bug and gets hunted in the wrong file, and an asset whose orphan sweep now lists nodes the author never knowingly created. **Relationship to `B-bpir-upsert-skips-inputkey-entries` (DONE)**, stated so it is not mistaken for a regression: that ticket was silent *duplication* of InputKey entries on re-author; nothing here duplicates — one node per key exists, Pressed is wired, Released is not. Lead offered as a guess with no source read: `UK2Node_InputKey` carries both Pressed and Released exec outputs on one node, so an upsert keyed on the node rather than the pin would explain both tickets. Asked for: wire the released entry to the existing node's Released pin; failing that refuse `key_released` with a typed error rather than reporting success; in either case roll the body's nodes back instead of leaving orphans, using the path `COMPILE_FAILED` already takes; and until fixed stop listing `entry key_released SpaceBar()` on `bpir.entry-points` as an available shape. Severity High per the silent-false-success band; reach modifier declined because Enhanced Input projects never reach this path.
- `#2-walk-released-pin` `IN-REVIEW` developer — Reformulated the ticket around the confirmed decompiler defect; changed `BpirDecompiler.cpp` to walk the active InputKey `Released` output, strengthened `FCompilerIntegrationKeyReleasedEntryTest` to cover compiler success, graph topology, orphan absence, and released-body round-trip through production code, and corrected the key-input wiki coverage note. Compiler wiring, node creation/reuse, rollback, and the entry-point language contract were deliberately unchanged; Unreal compile, tests, editor, and MCP were not run under the sprint constraint.
- `#3-tester-reopen-gate` `IN-REVIEW` developer — Added the exact fresh-Blueprint double-compile/read-back check and reopen criteria for the still-unexplained missing-signature and orphan-accumulation evidence, documented the conditional upsert mechanism that could accrete InputKey nodes when `Released` is disconnected, and recorded the omitted `Docs/wiki-src/bpir.examples.md` change.
