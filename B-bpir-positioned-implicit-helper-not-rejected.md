---
id: B-bpir-positioned-implicit-helper-not-rejected
title: "compile_bpir authored-position guard's blanket IsNodePure() exemption admits visible Break* projection helpers (silent false-success)"
status: IN-REVIEW
severity: High
category: bug
tags: [bpir, compiler, authored-position, implicit-helper, silent-success, layout]
encounters: 6
lastSeen: 2026-06-29T02:57:23Z
---

# compile_bpir authored-position guard exempts visible Break* helpers via a too-broad IsNodePure() skip

`blueprint.compile_bpir`'s documented authored-position contract says that when
**every** visible node-backed instruction in an entry body carries `@(x, y)`,
the body is in manual-placement mode and any **implicit visible helper /
generated node** must be **rejected** — because those helpers have no BPIR line
to carry a coordinate and would silently land on top of the author's manual
layout. The contract is documented three ways:

- `wiki/bpir.entry-points.md` (layout-mode table): *"Every primary-node
  instruction has @(x, y) → Authored coordinates are preserved for the primary
  nodes. **Implicit visible helper/generated nodes are rejected.**"*
- `wiki/bpir.entry-points.md` — the canonical failing example, complete with the
  diagnostic the compiler is supposed to emit:
  ```
  entry function UsesStruct(struct<FHitResult> Hit) -> float {
      %x = call Conv_DoubleToFloat(InDouble: $Hit.Location.X) @(300, 0)
      return %x @(620, 0)
  }
  # Compile error: authored-position mode would create an implicit visible helper node.
  ```
- `wiki/blueprint.compile_bpir.md` — *"If every primary node-backed instruction
  in a body has `@(x, y)`, those absolute coordinates are preserved and implicit
  visible helpers are rejected."*

**The guard exists and runs — but its pure-node exemption is too broad.** The
guard is `RejectImplicitVisibleHelpersForAuthoredBlock`
(`Source/PinWright/Private/Compiler/BpirCompiler.cpp:2041`), invoked after
data/exec wiring at all three authored-position body-compile call sites
(`BpirCompiler.cpp:3141`, `:3567`, `:3917`) and only when placement mode is
enabled (every visible node-backed instruction positioned —
`BpirCompiler.cpp:1879-1902`). It is *not* an unwired branch. The defect is the
**blanket `IsNodePure()` exemption** at `BpirCompiler.cpp:2087-2093` (an earlier
revision): the guard `continue`d past **every** pure `UK2Node`, with a comment
naming `K2Node_Self`, pure `K2Node_VariableGet`, "pure function calls" and knots
as inlined operands. Member access `$Hit.Location.X` / `%fwd.X` synthesises pure
native-break helpers (`BreakHitResult` / `BreakVector`, both pure
`K2Node_CallFunction`) or pure `K2Node_BreakStruct`, and `IsValid(self)`
synthesises a pure `K2Node_Self` — so the blanket pure skip exempted *all* of
them and the compile returned `success: true`.

The exemption is correct for **truly-inline operands** but wrong for the
**Break\* projection helpers**. The decompiler re-inlines exactly three pure node
classes as bare operands with no standalone line — `K2Node_Self` (`self`), pure
`K2Node_VariableGet` (`$Var`) and `K2Node_Knot` (reroute), per the
standalone-statement pass in `Decompiler/BpirDecompiler.cpp:817-821`. The
`Break*` helpers are *also* pure, but the decompiler emits each as its own
positioned `%n = call Break...` line — so they are exactly the implicit
**visible** helpers the contract targets. The bug is that the compile-side
exemption keyed on `IsNodePure()` instead of the decompile-side inline set,
admitting visible helpers the round-trip then surfaces.

This is a **silent false-success on the seed method's documented contract**: a
caller who fully positions a body trusts the documented guarantee that the graph
will contain *only* their nodes at *their* coordinates. Instead the compiler
quietly adds an unpositioned `Break*` helper (here stacked at `x = 0`,
overlapping the entry node), and the guard stays silent. The **sibling guard
works**: a *mixed* positioned/unpositioned body is correctly rejected
(`BpirCompiler.cpp:1884-1900`).

**Self / VariableGet are a separate, benign sub-case (not a defect to reject).**
Encounters #2/#4/#5/#6 are the `K2Node_Self` leak from `IsValid(Object: self)` —
the wiki's own §1b *good* example. Those round-trip cleanly (the decompiler
re-inlines `self`, emits no visible line, no orphan, no layout collision); only
the helper's auto-assigned position is lost. Per the contract's "visible"
qualifier these must stay **accepted** — rejecting them would break the wiki's
own §1b example. (An optional cosmetic follow-up, *don't-spawn* `K2Node_Self`
for a self-context object pin so no node is created at all, is out of scope for
this contract fix.)

## Repro (replay-confirmed this session, stock asset)

`mcp__pinwright__call` `blueprint.compile_bpir` on
`/Game/ExampleContent/Blueprint_Communication/Blueprints/BP_Light_Bulb_Basic`:

```json
{
  "assetPath": "/Game/ExampleContent/Blueprint_Communication/Blueprints/BP_Light_Bulb_Basic",
  "code": "entry function OracleUsesStruct(struct<FHitResult> Hit) -> float {\n    %x = call Conv_DoubleToFloat(InDouble: $Hit.Location.X) @(300, 0)\n    return %x @(620, 0)\n}"
}
```

Expected (per docs): `COMPILE_FAILED` — *"authored-position mode would create an
implicit visible helper node."*

Observed:
```json
{"nodeCount":4,"createdNodes":[...],"errors":[],
 "warnings":["Conv Double to Float : ... deprecated ..."],
 "compiled":true,"status":"UpToDateWithWarnings","success":true}
```

`blueprint.decompile` of the created function shows the two implicit helpers the
contract says must be rejected, silently auto-positioned over the entry/authored
region:
```
entry function OracleUsesStruct(struct<HitResult> Hit) -> float @(0, 0) {
    %n0: bool   = call BreakHitResult(Hit: $Hit)        @(0, 80)
    %n1: double = call BreakVector(InVec: %n0.Location) @(0, 208)
    %n2: float  = call Conv_DoubleToFloat(InDouble: %n1.X) @(300, 0)
    return %n2 @(620, 0)
}
```

No crash, no hang; editor stayed responsive. In the well-formed function shape
above the helpers are wired (0 orphans) but mis-laid-out; in an
**event-shaped** all-positioned body whose forced helper output is not consumed
on the exec chain, the same missing guard leaves the helper chain **orphaned**
(reporter's parallel Case B: `find_orphaned_nodes` → `orphanedCount: 4` —
`Self, Get Actor Location, Conv Double to Float, Break Vector`), an asset-quality
regression on top of the contract violation.

## Expected / Fix

Narrow the existing guard's exemption (`RejectImplicitVisibleHelpersForAuthoredBlock`,
`BpirCompiler.cpp`) from a blanket `IsNodePure()` skip to **the decompiler's
inline-operand set** — `UK2Node_Self`, `UK2Node_VariableGet`, `UK2Node_Knot`
(the same three classes `BpirDecompiler.cpp:817-821` re-inlines without a
standalone line). Any other generated node — notably the pure `BreakHitResult` /
`BreakVector` / `BreakStruct` projection helpers that decompile to their own
positioned `%n` line — is then no longer exempt and is rejected with the
guard's existing diagnostic ("authored-position mode forbids implicit visible
helper/generated node ..."), rolling back cleanly. This reconciles compile-side
with decompile-side: anything that gets a visible line is rejected, anything
inlined is accepted — so it rejects the canonical `$Hit.Location.X` /
member-access must-fail example while keeping the wiki's §1b `IsValid(Object:
self)` good example (and `$Var` operands) compiling.

The wiki is already consistent with this narrowing (the §1b `IsValid(self)`
example is accepted because `self` is inlined; the `BreakHitResult` must-fail
example is rejected because it is visible) — no docs change is required, only the
"visible" qualifier in the layout-mode table is now load-bearing.

Regression: compile an all-positioned body whose `$param.Member` access forces a
generated visible `BreakVector` and assert it is rejected with the
`forbids implicit visible helper` diagnostic (before the fix it returned
`success: true`).

## History
- `#1-initial-repro` `OPEN` reporter — Replay-confirmed live on stock
  `BP_Light_Bulb_Basic`: the wiki's own canonical must-fail example
  (`bpir.entry-points.md:143`, all-positioned body forcing `BreakHitResult` +
  `BreakVector` via `$Hit.Location.X`) compiles `success: true / compiled: true /
  nodeCount: 4` instead of the documented `COMPILE_FAILED` "authored-position
  mode would create an implicit visible helper node". Decompile shows the two
  implicit helpers silently injected at `@(0, 80)` / `@(0, 208)`, overlapping the
  entry node. The sibling mixed-position guard IS enforced (a partially-positioned
  body returns a clean manual-placement error), so the implicit-helper branch of
  the guard is simply not wired up. No crash/hang; editor responsive; cleanup via
  `remove_function` restored the graph (0 orphans).
- `#2-self-pin-helper-leaks-wiki-1b-example` `OPEN` reporter — Additional
  trigger confirming the guard gap is broad: an authored-position
  `event`+`custom_event` body in a fresh `/Game/BP_BpirAuthoredPos2` Actor BP
  (every node carries `@(x,y)`) that wires `self` into a function object pin —
  `%valid = call IsValid(Object: self) @(337, 41)`, **the wiki's own §1b
  authored-position example** (`bpir.entry-points.md:81-82`) — compiles
  `{compiled:true, status:UpToDate, errors:[], warnings:[], success:true}` with
  `nodeCount:9` for only **8** authored node-backed lines. `blueprint.graph.get_nodes`
  pins the unauthored 9th node: `K2Node_Self_0` nodeType `K2Node_Self`
  "Self-Reference" at auto-assigned `@(16, 104)` — an implicit visible helper with
  no BPIR line to carry a coordinate, exactly what §1b lines 113/136/143 promise is
  **rejected**. Not rejected, not warned. Wrinkle vs. the Break* helpers in #1: this
  one round-trips *cleanly* — `blueprint.decompile` re-inlines it as `self`
  (`call IsValid(Object: self)`) with no positioned line, so it produces no visible
  `n0`/`n1` line and no orphan/layout collision, BUT its `@(16, 104)` position is
  simply lost (unauthorable). So same missing guard, milder surface: the §1b path
  silently injects an unpositionable helper rather than honoring the documented
  reject (preferred fix here is arguably (a) treat a `self`-context object pin as the
  editor's implicit self default so NO `K2Node_Self` is spawned, rather than (b)
  reject the wiki's own example). The 8 genuinely authored coords (incl. non-grid
  `337` and negative `-103`) and both entry-signature positions round-tripped
  byte-exact; only the injected Self node is unauthored. Same focus method
  `blueprint.compile_bpir`; surfaced by an authored-position round-trip-equivalence
  probe. Docs to fix alongside the code: `bpir.entry-points.md` §1b (its own
  `IsValid(Object: self)` example leaks) and `blueprint.compile_bpir.md`.
- `#3-vector-component-projection-trigger` `OPEN` reporter — Cross-task evidence
  (malformed/edge-positioning probe iteration, fresh `/Game/BP_BpirPosProbe`
  Actor BP): the guard gap reproduces via a **third, distinct trigger** —
  vector-component projection on a positioned `return`, not a `$param` member
  chain. Authored-position function `entry function UsesHelper() -> float {\n
  %fwd = call GetActorForwardVector(Target: self) @(300, 0)\n  return %fwd.X
  @(620, 0)\n}` — both primary node-backed lines carry `@(x,y)`, and `return
  %fwd.X` forces an implicit `BreakVector` projection helper with no BPIR line to
  carry a coordinate. Compiled `{nodeCount:4, compiled:true, status:UpToDate,
  errors:[], success:true}` — silent false-success, no `COMPILE_FAILED`.
  `blueprint.decompile` of `UsesHelper` confirms the unauthored helper:
  `%n1: double = call BreakVector(InVec: %n0) @(0, 344)` auto-placed at the
  unauthored `@(0, 344)`, wedged between the authored `@(300, 0)` and `@(620, 0)`.
  Same missing guard as `#1` (`BreakHitResult`/`BreakVector` via `$Hit.Location.X`)
  and `#2` (`K2Node_Self` via `IsValid(self)`); this confirms the gap also covers a
  bare vector-component access (`.X`) on a function-call result, so the guard is
  branch-blind to *how* the implicit `Break*` helper gets synthesized. Same
  iteration whose mixed-position rollback corruption fed
  `B-compile-bpir-transaction-ensure` (`#11`). Gateway stayed responsive (no
  hang/crash) and a root-namespace liveness probe answered afterward.
- `#4-additional-event-beginplay-branch-trigger` `OPEN` reporter — Additional
  evidence: the `K2Node_Self` leak (same surface as `#2`) also reproduces from an
  **`entry event BeginPlay` body with a `branch`** (not a function/custom_event),
  using the wiki's own §1b `IsValid(Object: self)` example verbatim with
  adversarial off-grid/negative/large coords. Fresh `/Game/BP_OracleSelfHelper`
  Actor BP, `blueprint.compile_bpir`:
  `entry event BeginPlay() {\n  %valid = call IsValid(Object: self) @(-417, 13337)\n  %branch = branch(%valid) [true -> @ok, false -> @done] @(2049, -88)\n@ok:\n  call PrintString(InString: "Valid") @(9941, -417)\n@done:\n  call PrintString(InString: "Done") @(13337, 2049)\n}`
  → `{nodeCount:6, compiled:true, status:UpToDate, errors:[], warnings:[], success:true}`
  for only **5** authored node-backed lines (no `COMPILE_FAILED`). `get_nodes` pins the
  unauthored 6th node: `K2Node_Self` "Self-Reference" auto-placed at `@(0, 944)`
  (note: a *different* auto-layout coordinate than `#2`'s `@(16, 104)` and the
  reporting probe's `@(-416, 13416)` — the leaked Self node's placement is
  context-dependent, so it can land anywhere relative to the authored layout).
  `blueprint.decompile` re-inlines it as bare `self`
  (`call IsValid(Object: self)`) with NO positioned line → its `@(0, 944)` is lost
  (unauthorable, non-round-tripping). All 5 authored coords (incl. negative `-417`,
  `-88`, `-417`, large `13337`/`9941`/`13337`/`2049`, off-grid `2049`) round-tripped
  byte-exact in both `get_nodes` and `decompile`; only the injected Self node fails
  the round-trip. Same focus method `blueprint.compile_bpir`; surfaced by an
  authored-position FORMATTING round-trip-equivalence probe. Gateway responsive,
  no crash/hang.
- `#5-cross-task-custom-event-reconverge-body-liveness` `OPEN` reporter — Cross-task
  liveness reconfirm (same `K2Node_Self` leak surface as `#2`/`#4`; struggle-auditor process
  angle). Authored-position round-trip-equivalence probe (focus `blueprint.compile_bpir`,
  namespace `blueprint`, transcript `agent-a4f4f72eb42e0d70c.jsonl`), fresh Actor BP
  `/Game/BP_AuthoredPosStress`: a fully-positioned `custom_event` body (Start → `IsValid` →
  `branch` fanning true/false to two PrintString arms reconverging at `Finished`) where the
  condition `%ok = call IsValid(Object: self)` again silently materialized an unauthored
  `UK2Node_Self` (Object: self) helper carrying NO BPIR line and NO `@(x,y)` — admitted, not
  rejected, despite the authored-position implicit-visible-helper reject contract. Extends the
  known reproduction shape: prior `IsValid(self)` triggers were `event`+`custom_event` (`#2`)
  and `event BeginPlay`+`branch` (`#4`); this is a `custom_event` with a full
  branch/fan-out/reconvergence body. Round-trips cleanly (the Self node gets no position, so it
  produced no visible line and no orphan/layout collision) — same milder surface as `#2`/`#4`,
  benign in-run, but contradicts the stated contract. All 7 primary nodes + entry round-tripped
  at their exact authored coords; only the injected Self node is unauthored. The attempt's
  self-report independently flagged it: "authored-position mode admitted an undocumented implicit
  UK2Node_Self helper (Object: self) without rejecting it." Confirms the preferred-fix-(a) angle
  on `#2` (treat a `self`-context object pin as the editor's implicit self default so NO
  `K2Node_Self` is spawned). Dedup: matched this OPEN ticket on rg `UK2Node_Self`/`implicit
  helper`/`IsValid`; appended rather than re-filed. Severity unchanged High.
- `#6-cross-task-multi-self-arg-misread-as-exemption` `OPEN` reporter — Cross-task evidence (struggle-auditor process angle; authored-position node-FORMATTING round-trip-equivalence probe, focus `blueprint.compile_bpir`, namespace `blueprint`, transcript `agent-a4dab0f116a59dd4c.jsonl`, fresh Actor BP `/Game/BP_BpirFmtRoundTrip`). Same `K2Node_Self` leak as #2/#4/#5, now with **multiple inline `self` args in one body**: `compile_bpir` returned `nodeCount:15` (15 createdNodes) for only **13** authored node-backed lines — the 2 extras are unauthored `K2Node_Self` provider nodes synthesized for the inline `self` on `IsValid(Object: self)`, `GetActorLocation(Target: self)`, `SetActorLocation(Target: self)` (3 self uses -> 2 provider nodes), admitted with `{compiled:true, status:UpToDate, errors:[], warnings:[], success:true}` despite the authored-position implicit-visible-helper reject contract. As in #2/#4/#5 they round-trip cleanly (decompile re-inlines them as `self`, no positioned line, no orphan/collision) — only their auto-assigned positions are lost. **Process angle (why the silent success harms callers):** it actively misled the attempt — its self-report rationalized the leak as intended ("2 transparent K2Node_Self provider nodes ... self-providers appear exempt, rendered inline by decompile"), and the CallAnalyzer's call-trace finding for the same Attempt proposed *documenting* an exemption (add `K2Node_Self` to the transparent-node list in `bpir.entry-points`). That docs proposal is **declined and merged here**: per this ticket the leak is a contract violation to fix (preferred fix (a): treat a `self`-context object pin as the editor's implicit self default so NO `K2Node_Self` is spawned), not a behavior to bless in docs — so the CallAnalyzer's "exemption" framing routes to this bug, not a new exemption note. (The attempt also spent one extra `get_node_details` call diagnosing the related `sequence(2)->sequence(3)` arity drift, tracked on `B-bpir-sequence-extra-exec-pin`.) Dedup: matched this OPEN ticket on rg `K2Node_Self`/`implicit helper`/`authored-position`; appended rather than re-filed. Severity unchanged High.
- `#7-reword-and-fix` `IN-REVIEW` developer — Reworded around the real root
  cause (the three validity lenses unanimously found the headline "guard not
  wired up" premise FALSE): the guard `RejectImplicitVisibleHelpersForAuthoredBlock`
  exists and runs at all three authored-position call sites
  (`BpirCompiler.cpp:3141`/`:3567`/`:3917`); the defect was a **blanket
  `IsNodePure()` exemption** (`BpirCompiler.cpp:2087-2093`) that skipped *every*
  pure node, including the visible `Break*` projection helpers the contract
  targets. Fix: narrowed the exemption to the decompiler's inline-operand set
  `{UK2Node_Self, UK2Node_VariableGet, UK2Node_Knot}` (the exact three classes
  `Decompiler/BpirDecompiler.cpp:817-821` re-inlines without a standalone line),
  so member-access `Break*`/`BreakStruct` helpers — pure but visible — are now
  rejected with the existing "forbids implicit visible helper" diagnostic, while
  `self`/`$Var`/knots stay accepted (no regression of the wiki §1b
  `IsValid(self)` example; consistent with the existing
  `AuthoredPositionAcceptsPureImplicitHelper` test, which uses `$Source` →
  `VariableGet`). Per the lenses the `K2Node_Self` position-loss (#2/#4/#5/#6) is
  benign/round-trips-clean and is correct-to-accept, so it is declared
  in-contract rather than a defect; the optional don't-spawn-`self` cleanup is
  noted out of scope. Files: `Source/PinWright/Private/Compiler/BpirCompiler.cpp`
  (narrowed exemption + `K2Node_Knot.h` include). Test: new
  `PinWright.bpir.compiler.integration.AuthoredPositionRejectsImplicitVisibleBreakHelper`
  in `Source/PinWright/Private/Tests/Bpir/TestCompilerIntegration.cpp` (compiles
  an all-positioned `set StoredX = $HitLocation.X @(300,0)` body forcing a
  visible `BreakVector` and asserts rejection); refreshed the stale comment on
  the adjacent `AuthoredPositionAcceptsPureImplicitHelper` test. Not compiled/run
  here (later phase verifies).
