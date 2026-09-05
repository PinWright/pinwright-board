---
id: B-bpir-second-return-branch-data-dropped
title: "A BPIR function with a return in each branch keeps only one branch's data: both execs are wired to the single FunctionResult, the other branch's value nodes are left orphaned, and the compile reports success with no warning"
status: IN-REVIEW
severity: High
category: bug
tags: [bpir, compile-bpir, return, function-result, branch, orphan-nodes, silent-wrong-result, multi-output]
encounters: 2
lastSeen: 2026-09-02T23:35:00+03:00
---

# Two `return`s, one branch's data survives, silently

A Blueprint function has exactly one `UK2Node_FunctionResult`, so BPIR has to reconcile a `return`
appearing in more than one branch. It reconciles it by wiring **both** exec paths to that node and
wiring **one** branch's data pins — the other branch's value nodes are created, left unconnected,
and the compile answers `compiled: true, errors: [], warnings: []`.

The function then returns the surviving branch's values **on both paths**. Nothing in the response,
the status, or the node count says so. The only signal is
`blueprint.graph.find_orphaned_nodes`, which nobody runs on a compile that reported success.

## Repro

```
entry function GetAimRay() -> (vector Origin, vector Direction) {
    %owner = call GetOwner(Target: self)
    %ok = call IsValid(Object: %owner)
    %b = branch(%ok) [true -> @eyes, false -> @muzzle]

@eyes:
    %vp = call GetActorEyesViewPoint(Target: %owner)
    %f = call Conv_RotatorToVector(InRot: %vp.OutRotation)
    return (Origin: %vp.OutLocation, Direction: %f)

@muzzle:
    %ml = call K2_GetComponentLocation(Target: $Muzzle)
    %mr = call K2_GetComponentRotation(Target: $Muzzle)
    %mf = call Conv_RotatorToVector(InRot: %mr)
    return (Origin: %ml, Direction: %mf)
}
```

`compile_bpir` -> `compiled: true, status: "UpToDate", errors: [], warnings: []`.

`blueprint.decompile_function` on the result:

```
    %n2 = branch(%n1) [false -> @merge, true -> @merge]

@merge:
    %n3 = call K2_GetComponentLocation(Target: $Muzzle)
    %n4 = call K2_GetComponentRotation(Target: $Muzzle)
    %n5 = call Conv_RotatorToVector(InRot: %n4)
    return %n3, %n5
    %n6 = call GetActorEyesViewPoint(Target: %n0)          <- after the return, unwired
    %n7 = call Conv_RotatorToVector(InRot: %n6.OutRotation) <- after the return, unwired
```

Note the shape: **both branch outputs point at the same label**, the second branch's body is what
survived, and the first branch's two nodes are emitted after the `return` line as dead text.
`find_orphaned_nodes` confirms it independently:

```
orphanedNodes: [
  { nodeId: 7ABF3616…, title: "GetActorEyesViewPoint",  graphName: "GetAimRay" },
  { nodeId: F9F53584…, title: "Get Rotation X Vector",  graphName: "GetAimRay" }
]
```

## Why it is worth a High

The decompiled text is *self-evidently* wrong once read — a `branch` whose two outputs go to the
same label is not a branch — but nothing routes a caller to read it. In this instance the function
was the aim-ray source of a hitscan weapon, so **every shot traced from the muzzle component instead
of the player's eyes**, on a weapon whose owner was always valid. That is a wrong world-space ray
with a plausible-looking result: bullets still fly, still hit, still spawn impacts, and land a few
centimetres off where the crosshair points. It survived a live PIE session, a 30-round burst and a
screenshot before an unrelated orphan sweep found it.

Same mechanism, second sighting in the same session: a two-output function whose success and failure
branches both returned merged into one `@merge` block, which pulled its *pure* `Map_Find` nodes onto
the failure path as well — i.e. the merge does not just drop data, it can also **execute nodes on a
path that never asked for them**, there reading a map off a null object on every miss.

## Workaround, which is also the shape a fix could adopt

Write to member variables under each branch's exec, and return those once:

```
@eyes:
    %vp = call GetActorEyesViewPoint(Target: %owner)
    set AimOrigin = %vp.OutLocation
    set AimDirection = call Conv_RotatorToVector(InRot: %vp.OutRotation)
    exec -> @out
@muzzle:
    …
    exec -> @out
@out:
    return (Origin: $AimOrigin, Direction: $AimDirection)
```

`set` is impure, so it only runs on its own path. Verified: the decompile now shows
`[false -> @else, true -> @then]` with distinct bodies, and `find_orphaned_nodes` reports
`orphanedCount: 0` across all 32 graphs. The cost is two member variables per multi-branch return,
which pollutes the class's variable list with things that are really function locals.

## The ask

1. Support it properly: emit local variables (or reroute/select nodes) so each branch's data reaches
   the single `FunctionResult`, which is what a human does in the editor.
2. If that is not on the table, **refuse the second `return`** with a diagnostic naming the branch,
   and suggest the member-variable shape. A `COMPILE_FAILED` here costs one edit; the current
   behaviour costs a wrong result that survives testing.
3. Cheapest partial fix, worth doing regardless: when a compile leaves nodes orphaned **in a graph
   it just authored**, say so in `warnings[]`. `find_orphaned_nodes` already computes exactly this;
   `compile_bpir` reporting it would have turned an hour into a minute, and would have caught the
   `key_released` defect filed separately today as well.
4. `bpir.instructions` §2.9 documents `return` variants without saying a function may only carry one
   data-bearing return. Say it there.

## Dedup

Board-wide search for `return`, `FunctionResult`, `multiple return`, `orphan`.
`B-bpir-function-return-class-type-lost` is about a return value's *type* surviving, not about which
branch's value is wired. `B-add-function-outputs-become-inputs` is `blueprint.add_function`'s pin
direction. `B-bpir-statement-cast-success-unwired-replace-shared-topology` is the closest relative —
also "compiled clean, left unwired" — but it is the cast node's exec pins under replace mode, not a
return. `B-bpir-orphan-warning-diagnostics-lossy` and `B-orphan-finder-vs-decompiler-disagree`
concern how orphans are reported once you go looking; ask (3) above is that they be reported without
going looking. No ticket covers multi-branch returns.

## Severity

**High** — the rubric's *"silent wrong … data on a normal path (the caller trusts a result that is a
lie and builds on it)"*. A branch that silently collapses to one arm is a wrong result rather than a
missing feature, and it is invisible to `compiled`, `status`, `errors`, `warnings` and node count
alike. Not Critical: nothing crashed and no asset was corrupted. Reach modifier declined — a
function returning different values from different branches is ordinary but not present in every
session.

## Fix

Ask (1) — proper support. Every `return` statement now gets **its own**
`UK2Node_FunctionResult`, which is the engine's own model (several result nodes per function graph
are legal; `FKCHandler_FunctionResult` merges them, `SyncWithPrimaryResultNode` keeps their pin sets
identical) and what a human authors when each branch ends in its own Return node. Rejected the
alternative of one result node fed through synthesized `select`/merge trees: it cannot express
N-way control flow, it changes runtime semantics (pure nodes get pulled onto paths that never asked
for them — exactly the second symptom in `#1`), and it does not round-trip, since the decompiler
would emit `select` where the author wrote two `return`s.

Root cause: exec pins are wired in Pass 3b, **after** every node in the block is emitted, so the
"reuse a result node whose exec-in has no links" test in `EmitInstruction` could never tell a free
result node from one an earlier `return` had already taken. Both returns therefore resolved to the
same node; its single-link data inputs kept whichever branch wired last, and
`TryCreateConnection` silently unwired the other.

Files changed (all under `Plugins/PinWright/`):

- `Source/PinWright/Private/Compiler/BpirCompiler.cpp`
  - `EmitInstruction`, `EBpirOpcode::Return` (function branch): a candidate result node is skipped
    when any earlier instruction in `EmitMap` already claims it, so the second `return` falls
    through to `CreateFunctionResult`. `EmitMap` is reset per block and only `return` stores a
    `UK2Node_FunctionResult` in it, so it is the authoritative claim record.
  - `WireDataPins`: tripwire — a `return` whose target pin is already fed by a node **this compile
    created** is now a hard `COMPILE_FAILED` naming the pin, instead of a silent re-link. Scoped to
    `UK2Node_FunctionResult` targets, so macro exit tunnels (one node, data pins shared by every
    exit, by engine design) and Extend-mode pre-existing links are untouched.
- `Source/PinWright/Private/Compiler/CodeNodeEmitter.cpp` — `CreateFunctionResult` now calls
  `AllocateDefaultPins()` only when the node still has no pins. With a primary result node present,
  `PostPlacedNewNode` → `SyncWithPrimaryResultNode` already reconstructs it; calling
  `AllocateDefaultPins` again adds a **second** `execute` exec pin (the engine's user-pin loop is
  `FindPin`-guarded, its exec `CreatePin` is not).
- `Source/PinWright/Private/Decompiler/BpirTextEmitter.cpp` — `EmitReturn` picks its form from the
  result node's data-pin **count**: one pin emits `return %v`, more than one emits the named
  `return (Name: %v, ...)` form. The old `return %a, %b` output was not re-parseable at all (the
  parser reads the whole comma list as one value reference), so the decompile in this ticket's repro
  could not have round-tripped even with the data fixed.
- `Source/PinWright/Private/Tests/Bpir/TestBpirMultiBranchReturn.cpp` (new, 2 tests).
- Docs: `docs/wiki-src/bpir.instructions.md` §2.9 (ask (4) — multiple returns documented, with the
  macro exit-tunnel exception called out), `docs/bpir-compiler-internals.md` §7a (new),
  `docs/bpir-test-matrix.md` (file-index row + `return` rows).

Ask (3) — orphan warnings from `compile_bpir` — is deliberately **not** in this change; it is a
cross-cutting reporting feature tracked by `B-bpir-orphan-warning-diagnostics-lossy` /
`B-orphan-finder-vs-decompiler-disagree`, and the tripwire above closes the specific silent path
this ticket is about.

**Not compiled and not run** — a separate compile pass follows.

### Reviewer verification

1. `PinWright.bpir.compiler.MultiBranchReturnKeepsBothBranches` — compiles
   `entry function PickValue(bool bUseHigh, int Base) -> (int Value, int Doubled)` with a `return`
   in each branch and asserts: exactly **2** `UK2Node_FunctionResult` nodes (pre-fix: 1); both
   outputs wired on **both** nodes; the two nodes' `Value`/`Doubled` sources are **different** nodes;
   the branch's `then`/`else` reach **different** nodes (pre-fix: the same one — the `@merge`
   signature); all 4 producers present and **none** orphaned.
2. `PinWright.bpir.round_trip.MultiBranchReturn` — decompiles the same function and asserts two
   `return (` lines, both `Add_IntInt` producers present, named `Value:` / `Doubled:` arguments, and
   two **distinct** branch labels; then recompiles the decompiled text onto a fresh Blueprint and
   asserts it succeeds and rebuilds 2 result nodes.
3. Live re-check of the ticket's own repro (`GetAimRay`), plus `blueprint.graph.find_orphaned_nodes`
   on the authored graph → `orphanedCount: 0`, and `blueprint.decompile_function` showing two
   distinct labels with each branch's own values.
4. Regression surface to watch: multi-exit macro returns (`return [ExitA] (…)` /
   `return [ExitB] (…)`) must still compile — they are excluded from the tripwire — and Extend-mode
   compiles that splice into an existing function with an already-wired result node.

## History
- `#1-filed` `OPEN` reporter — A BPIR function with a `return` in each branch keeps only one branch's data and reports success. Repro: `entry function GetAimRay() -> (vector Origin, vector Direction)` with `branch(%ok) [true -> @eyes, false -> @muzzle]`, each branch ending in its own `return (Origin: …, Direction: …)`. `compile_bpir` answered `compiled: true, status: "UpToDate", errors: [], warnings: []`. `decompile_function` shows `branch(%n1) [false -> @merge, true -> @merge]` — both outputs at the same label — with only the **muzzle** branch's `K2_GetComponentLocation` / `K2_GetComponentRotation` / `Conv_RotatorToVector` wired into the single return, and the eyes branch's `GetActorEyesViewPoint` and `Conv_RotatorToVector` emitted after the `return` line as dead text. `blueprint.graph.find_orphaned_nodes` confirms independently: `orphanedNodes: [GetActorEyesViewPoint, "Get Rotation X Vector"], graphName: "GetAimRay"`. Consequence in situ: this was the aim-ray source of a hitscan weapon whose owner is always valid, so **every shot traced from the muzzle component instead of the player's eyes** — bullets still flew, still hit, still spawned impacts, just from the wrong origin. It survived a live PIE session, a 30-round burst and a screenshot; an unrelated orphan sweep found it. Second sighting of the same mechanism in the same session: a two-output function whose success and failure branches both returned merged into one block, which then pulled its **pure** `Map_Find` nodes onto the failure path too, reading a map off a null object on every miss — so the merge can execute nodes on a path that never asked for them, not merely drop data. Workaround verified and shipped: write each branch's values into member variables under that branch's exec (impure `set`, so it runs only on its own path) and `return` those once from a shared `@out` — decompile then shows `[false -> @else, true -> @then]` with distinct bodies and `find_orphaned_nodes` reports `orphanedCount: 0` across all 32 graphs; the cost is member variables standing in for function locals. Asked for: emit local variables or reroute/select nodes so each branch reaches the single `FunctionResult`; failing that **refuse** the second return with a diagnostic naming the branch; and — worth doing regardless and cheapest of the three — have `compile_bpir` report in `warnings[]` when it leaves nodes orphaned in a graph it just authored, which `find_orphaned_nodes` already computes and which would also have caught `B-bpir-key-released-entry-silently-dropped`. Also: `bpir.instructions` §2.9 lists the `return` variants without saying a function may carry only one data-bearing return. Severity High per the silent-wrong-data band.
- `#2-three-functions-literal-returns-and-a-runtime-repro` `OPEN` reporter — Independent hit, same host and session, on three functions in one Blueprint, plus the runtime consequence and a rewrite pattern that works. Two additions to the picture in `#1`. **(a) With literal returns the function becomes a constant.** `GetAmmoStateInt() -> int` was written with three terminal blocks — `@lowr: return 1`, `@okr: return 0`, `@none: return 2` — and decompiled back as `%n6 = branch(%n5) [false -> @merge, true -> @merge]` with a single `@merge: return 2`. Both outcomes of the final branch point at one return whose value is `2`; the `1` and the `0` do not exist anywhere in the graph. So the function returns 2 unconditionally: at runtime, with a full 30-round magazine, `GetAmmoStateInt()` answered `2` ("empty"), which sent every enemy down the Behavior Tree's Reload branch forever and made the whole engage path unreachable. **(b) It also silently inverts a boolean.** `HasLOSToActor(Actor) -> bool` had `return %same` (hit actor is the target), `return true` (trace hit nothing) and `return false` (invalid target); the collapse kept only `return %n9` where `%n9 = EqualEqual_ObjectObject(HitActor, TargetActor)`, and wired the trace branch's *both* pins into it. A completely clear line of sight — the trace returning false, HitActor null — therefore reported **no** line of sight. That is the exact opposite of the authored meaning, and it disabled sight-based engagement for the entire AI stream. `GetAimPoint() -> (vector, bool)` on a second Blueprint had the same shape. Every one of these compiled with `compiled:true, errors:[], warnings:[]`, and re-reading my own BPIR source could never reveal it — the only thing that exposed it was probing the live function in PIE and finding a 30-round weapon reporting empty, then decompiling. **Working workaround, in case it helps the fix's test matrix:** rewrite to exactly one `return` and move the branching into data. Booleans compose with `Not_PreBool` / `BooleanOR` / `BooleanAND` over unconditional calls; integers need `select(cond:..., true:%a, false:%b)` where **both options come from `MakeLiteralInt`** — an inline integer literal in `select` leaves the wildcard pins untyped and the Blueprint compile fails with *"The type of Option 0 is undetermined"*; and where a `cast` is unavoidable, wire **both** its exec exits to one label holding the sole `return` (`%a = cast<Actor>(%t) [success -> @done, fail -> @done]`), which keeps one FunctionResult and yields null on failure. All three rewritten functions decompile correctly and behave correctly. Suggest the fix also cover the literal case explicitly: it is strictly worse than `#1`'s, because there is no surviving branch whose data is merely "the wrong one" — the function has no branch-dependent behaviour left at all. `encounters` 1 -> 2.
- `#3-one-result-node-per-return` `IN-REVIEW` developer — Verified TRUE against current source, then fixed with ask (1). Root cause: `FBpirCompiler::EmitInstruction`'s `EBpirOpcode::Return` function branch reused any `UK2Node_FunctionResult` whose exec-in had no links, but exec wiring is Pass 3b and runs AFTER every node in the block is emitted — at emit time every result node's exec-in is empty, so the test could not distinguish a free node from one an earlier `return` had taken. Both returns landed on one node; its data inputs are single-link, so `TryCreateConnection` silently replaced the first branch's wires with the second's and left the first branch's producers orphaned, while both branch execs stacked onto the one multi-link exec-in — the `[false -> @merge, true -> @merge]` signature. Fix: each `return` now claims its own result node (candidates already present in `EmitMap` are skipped; `CreateFunctionResult` makes a fresh one otherwise), which is the engine's model and what a human authors. Rejected a single result node fed by synthesized select/merge trees: it cannot express N-way flow, it drags pure nodes onto paths that never asked for them (the second symptom in #1), and it does not round-trip. Added a hard-error tripwire in `WireDataPins` for any `return` pin already fed by a node this compile created, scoped to `UK2Node_FunctionResult` so macro exit tunnels (one shared node by engine design) and Extend-mode pre-existing links are unaffected. Fixed two latent hazards the fix exposes: `CodeNodeEmitter::CreateFunctionResult` was calling `AllocateDefaultPins()` after `PostPlacedNewNode`'s `SyncWithPrimaryResultNode` had already reconstructed the node, which adds a second `execute` exec pin; and `BpirTextEmitter::EmitReturn` emitted `return %a, %b` for a multi-output result node, which the parser reads as ONE value reference and cannot re-parse — it now emits the named `return (Name: %v, ...)` form whenever the node has more than one data pin, chosen on the pin COUNT so a partially wired multi-output node still names its pin. Two new tests in `Tests/Bpir/TestBpirMultiBranchReturn.cpp`. Ask (4) documented in `bpir.instructions` §2.9 (including the macro exit-tunnel exception); ask (3) left to the orphan-reporting tickets. Ask (2) is moot — the second return is supported rather than refused. NOT compiled and NOT run; a separate compile pass follows.
- `#4-verified-in-fps-build` `IN-REVIEW` AI-stream - Direct route retried after the 2026-09-05 plugin pull; **works**. Re-authored the exact function from `#2(a)` in its natural multi-branch form: `compile_bpir {blueprintPath:'/Game/FPS/AI/BP_EnemyCharacter', mode:'replace'}` with `entry function GetAmmoStateInt() -> int` branching on `IsValid(Weapon)`, then `LessEqual(mag,0)`, then `LessEqual(mag,8)` into three terminal blocks `@dry: return 2`, `@lowammo: return 1`, `@ok: return 0`. `decompile_function` now returns three separate returns on their own labels (`@merge: return 2`, `@then_2: return 1`, `@else_2: return 0`) instead of `#2`'s single `return 2` reached from both pins of the final branch. The literal case that made the function a constant is gone. On disk: `Content/FPS/AI/BP_EnemyCharacter.uasset` 562640 bytes, mtime 2026-09-05 20:45:18, saved with `asset.save {force:true}` reporting `saveState:'written'`. The `select` + `MakeLiteralInt` workaround from `#2` has been removed from this function and is no longer needed. Left IN-REVIEW for the tester.
- `#5-verified-in-fps-build` `IN-REVIEW` FPS-tester — Independent re-verification of `#3`'s fix on a purpose-built probe, at **graph level** rather than from decompiled text (`#4` above rests on `decompile_function`, which the decompiler's adjacent-terminal-block artefact can make lie about control flow). Probe: `blueprint.create {name:"BP_PWProbe_Return", savePath:"/Game/FPS/Weapons/Test", parentClass:"Actor", waitForCompletion:true}`, then `blueprint.compile_bpir {assetPath:"/Game/FPS/Weapons/Test/BP_PWProbe_Return", mode:"append", code:"entry function PickValue(bool bUseHigh, int Base) -> (int Value, int Doubled) { %b = branch($bUseHigh) [true -> @high, false -> @low] @high: call PrintString(InString:\"high\"); %h = call Add_IntInt(A:$Base, B:100); %h2 = call Add_IntInt(A:%h, B:%h); return (Value:%h, Doubled:%h2) @low: call PrintString(InString:\"low\"); %l = call Add_IntInt(A:$Base, B:1); %l2 = call Add_IntInt(A:%l, B:%l); return (Value:%l, Doubled:%l2) }"}` — the reviewer-verification shape from the Fix section, one impure marker per branch so the exec arms are separable. Response `nodeCount:9, errors:[], warnings:[], compiled:true, status:"UpToDate"`. **`blueprint.graph.get_nodes {graphName:"PickValue", namesOnly:true}` → 10 nodes including TWO `K2Node_FunctionResult`**: `2CED7D004659C357FD0D56BCE2FA8F63` @(960,0) and `08829B4D4DA7D3F5E5220CB1C06CD58F` @(960,272). Pre-fix this ticket's whole mechanism was that there is exactly one. **`blueprint.graph.get_execution_flow {graphName:"PickValue", includeAllEntryPoints:true, includeDataInputs:true, includeExecOutputs:true}` is the deciding evidence**: Branch `C679D96D4327186211EA30A8F33F79D5` reports `execOutputs:[{pin:"then",target:D4FADB584F899379C2B930A736BCDCFE(PrintString)},{pin:"else",target:B58EB4E14B821DCE3048309A3D4B8C32(PrintString)}]` — two DIFFERENT targets, not the `[false -> @merge, true -> @merge]` signature; `then`-arm PrintString → Return `2CED7D00…` whose `dataInputs` are `Value ← 70B0840D…ReturnValue` and `Doubled ← D31E5951…ReturnValue`; `else`-arm PrintString → Return `08829B4D…` whose `dataInputs` are `Value ← D998DF99…ReturnValue` and `Doubled ← E8D54151…ReturnValue`. Four distinct `Add_IntInt` producers, no pin shared between the two results, so neither branch's data was dropped or re-linked. `blueprint.graph.find_orphaned_nodes {assetPath:"…/BP_PWProbe_Return"}` → `orphanedCount:0` over `["EventGraph","UserConstructionScript","PickValue"]`, `totalNodes:14`. Corroborating only, `blueprint.decompile_function {functionName:"PickValue"}` returns `%n0 = branch($bUseHigh) [false -> @else, true -> @then]` with two distinct labels and two named `return (Value: %n1, Doubled: %n2)` / `return (Value: %n3, Doubled: %n4)` lines carrying each branch's own producers — the multi-output named `return (...)` form from the `BpirTextEmitter::EmitReturn` change, which the old `return %a, %b` could not re-parse. **Regression surface from the Fix section checked and clean:** a multi-exit macro on the same Blueprint — `entry macro Validate(int X) -> (bool Result) [Pass -> @pass, Fail -> @fail]` with `return [Pass] (Result: true)` and `return [Fail] (Result: false)` — compiled `nodeCount:4, errors:[], warnings:[]`, so the new `WireDataPins` tripwire is correctly scoped to `UK2Node_FunctionResult` and does not fire on a macro exit tunnel's shared data pins. **On disk:** `Content/FPS/Weapons/Test/BP_PWProbe_Return.uasset` went 22777 bytes / mtime 2026-09-05 20:46:14 → 54302 bytes / 20:46:58 across `asset.save {assetPath:"…/BP_PWProbe_Return", force:true}` reporting `saved:true, saveState:"written"`; `grep -a` occurrence counts on the file are 0 for every authored token before the save and after it `PickValue` 4, `bUseHigh` 1, `Doubled` 1, `Add_IntInt` 2, `PrintString` 1, plus both branch literals `high` and `low`. Caveat for anyone repeating this: the package FName table dedupes, so `K2Node_FunctionResult` appears exactly **once** in the bytes no matter how many result exports reference it (the export's number suffix is stored separately) — a byte grep can prove the function reached disk but can never count the result nodes; that count is only available from `get_nodes` / `get_execution_flow`. Probe asset is throwaway and was created solely for this check. Status left `IN-REVIEW`; no encounter recorded, this is a pass not a failure.
