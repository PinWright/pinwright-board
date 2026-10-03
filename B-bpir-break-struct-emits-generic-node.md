---
id: B-bpir-break-struct-emits-generic-node
title: "BPIR `break<Rotator>` emits a generic BreakStruct node the BP compiler rejects — 'The structure cannot be broken using generic break node'; the compile still reports success"
status: DONE
severity: Medium
category: bug
tags: [bpir, compile_bpir, break, breakstruct, rotator, native-break, compile-warning]
encounters: 1
lastSeen: 2026-09-02T19:44:36Z
---

# `break<Rotator>` produces a node the Blueprint compiler will not accept

## Repro

```
blueprint.compile_bpir {assetPath:"/Game/FPS/UI/WBP_HUD", code:
"entry function UpdateCompass() {
    %pc = call GetOwningPlayer(Target: self)
    %rot = call GetControlRotation(Target: %pc)
    %br = break<Rotator>(%rot)
    %yaw = call ClampAxis(Angle: %br.Yaw)
    ...
}"}

-> {"nodeCount":66, "errors":[],
    "warnings":["The structure cannot be broken using generic 'break' node  Break Rotator .
                 Try use specialized 'break' function if available."],
    "compiled":true, "status":"UpToDateWithWarnings", "success":true}
```

Editor log, same moment:

```
[19.44.36:551] LogBlueprint: Warning: [AssetLog] ...\Content\FPS\UI\WBP_HUD.uasset: [Compiler]
  The structure cannot be broken using generic 'break' node  Break Rotator .
  Try use specialized 'break' function if available.
```

Replacing the line with `%br = call BreakRotator(InRot: %rot)` compiles clean, no warning, and
`%br.Yaw` resolves the same way.

## Diagnosis

`FRotator` is one of the engine structs that has a **native break** (`UKismetMathLibrary::
BreakRotator`) and is marked `HasNativeBreak`, so `UK2Node_BreakStruct` refuses it — the Kismet
compiler emits exactly this warning and the node does not lower. BPIR's `break<T>` opcode appears
to emit `UK2Node_BreakStruct` unconditionally rather than routing structs with a native break to
their break function. `FVector`, `FTransform`, `FLinearColor` and `FHitResult` are in the same
category and are likely affected identically; I only exercised `FRotator`.

`bpir.instructions` §2.7 documents `break<HitResult>(%hit.OutHit)` as the way to break a struct
and says nothing about native-break structs needing a different form, so the syntax the reference
teaches is the one that warns.

## What should happen

`break<T>` should emit the struct's native break function when it has one (`FRotator` →
`BreakRotator`) and `UK2Node_BreakStruct` only for structs without. Failing that, the parser should
reject `break<Rotator>` with a message naming `call BreakRotator(...)`, and `bpir.instructions`
§2.7 should list the native-break structs.

Secondary: the node is left in the graph in a state the compiler will not lower, but the RPC
answers `success:true` / `compiled:true`. `status:"UpToDateWithWarnings"` and the `warnings` array
do surface it, so this is not a silent failure — but "compiled true with a node that did not
compile" is a distinction the caller has to know to look for.

**Workaround:** `call BreakRotator(InRot: %rot)` and read `%br.Yaw` / `.Pitch` / `.Roll`.

severity rationale: impact=soft blocker — the documented syntax produces a broken node, and it is
recoverable only if you read `warnings` on a `success:true` response x reach=breaking a rotator or
vector is routine in any gameplay or HUD graph -> Medium.

## History
- `#1-filed` `OPEN` reporter — Hit on EAContentExamples58 (UE 5.8) authoring `UpdateCompass` on `/Game/FPS/UI/WBP_HUD`, which breaks the player's control rotation to read yaw. `break<Rotator>(%rot)` returned `success:true, compiled:true, status:"UpToDateWithWarnings"` with the generic-break warning; the same body with `call BreakRotator(InRot: %rot)` returned `status:"UpToDate"` and no warnings, and `blueprint.decompile_function` confirms the wired graph. Same pattern hit again in `ShowDamage`, which breaks two rotators. Related but separate log noise observed on the same compiles: `LogUObjectGlobals: Warning: Failed to find object 'Class Vector2D'` and `'Class false'` from `make<Vector2D>` and a `select` bool literal — those are benign (the decompiled graph shows a correct `MakeVector2D` call node), so they are not part of this ticket, but they do not appear in the RPC's `warnings` array at all, which is worth noting if anyone audits warning propagation.
- `#2-native-break-routing` `IN-REVIEW` developer — Reproduced by reading source, no editor run: the `BreakStruct` opcode in `FBpirCompiler::EmitInstruction` always called `CreateBreakStructNode`, while `make<T>` and the implicit `%v.Member` break (`BpirValueResolver::ResolveStructMemberThroughPin`) already route `HasNativeBreak` structs to the native function. Correction to the filed diagnosis: the generic node does lower (`FKCHandler_BreakStruct` registers terms after the warning, `K2Node_BreakStruct.cpp:149-163`); the user-visible defects are the warning and, for `FHitResult`, zero member pins (see `B-bpir-break-struct-pin-not-named` #3). Fixed with the MakeStruct routing policy: the bare-name form of a `HasNativeBreak` struct (`break<Rotator>`, `break<HitResult>`) emits the native break `UK2Node_CallFunction`; the F-prefixed form (`break<FVector>`) keeps the literal BreakStruct node, so `PinWright.bpir.compiler.type_prefix.BreakStructFPrefix` is unaffected. Behaviour change callers will see: member names on a native-broken struct are the function's output pins (same names for Rotator/Vector; `BreakTransform` gives `Location`/`Rotation`/`Scale`), and decompile prints `call Break...`. Docs: `bpir.instructions.md` §2.7 and `bpir.examples.struct-make-break.md` (whose concurrent text said `break<HitResult>` cannot work) updated to the new behaviour; `CHANGELOG.md`. The secondary point (`success:true` with warnings) is unchanged: `status:"UpToDateWithWarnings"` + `warnings[]` is the contract. Files: `Source/PinWright/Private/Compiler/BpirCompiler.cpp`, docs above, `Source/PinWright/Private/Tests/Bpir/TestBpirWildcardResolution.cpp`. Test: `PinWright.bpir.compiler.break_native.RoutesToNativeBreakFunction` (`break<Rotator>` -> `BreakRotator` with `Yaw` wired, `break<HitResult>` -> `BreakHitResult` with `HitBoneName` wired, no `UK2Node_BreakStruct`, Kismet status `BS_UpToDate`; fails before the fix). Compile-checked with -SingleFile; not run.
- `#3-review-roundtrip-and-lesson` `IN-REVIEW` developer — Review follow-up. (a) `docs/lessons.md` carried the opposite rule ("always emit UK2Node_BreakStruct ... it expands to the native break internally") on a false premise — `FKCHandler_BreakStruct` (`K2Node_BreakStruct.cpp:149-163`) reads members directly and logs the generic-break warning; rewritten to the new rule with that reason so a later agent does not revert this fix. (b) Round-trip regression closed: `FBpirTextEmitter::EmitBreakStruct` now prints an existing generic Break Struct node on a `HasNativeBreak` struct in the F form (`break<FTransform>(...)`), which recompiles to the same generic node; the bare form would have become `BreakTransform` (`Location`/`Rotation`(Rotator)/`Scale` instead of `Translation`/`Rotation`(Quat)/`Scale3D`). CHANGELOG states that rename + Quat->Rotator as breaking for hand-written `break<Transform>`; §2.7 notes `break<FVector>` etc. still draw the generic-break warning. (c) The native make/break UFunction lookup is one helper (`FindStructNativeFunction`) shared by MakeStruct and BreakStruct. Tests: `PinWright.bpir.compiler.break_native.RoutesToNativeBreakFunction` now also decompiles (asserts `call BreakRotator(` / `call BreakHitResult(`) and recompiles into a fresh BP; new `PinWright.bpir.compiler.break_native.GenericNodeRoundTripsAsFForm` (`break<FTransform>` + `.Translation` decompiles as `break<FTransform>(` and recompiles to the generic node with `Translation` wired; fails without the emitter change). Files: `Source/PinWright/Private/Decompiler/BpirTextEmitter.cpp`, `Source/PinWright/Private/Compiler/BpirCompiler.cpp`, `docs/lessons.md`, `docs/wiki-src/bpir.instructions.md`, `CHANGELOG.md`, `Source/PinWright/Private/Tests/Bpir/TestBpirWildcardResolution.cpp`.
- `#4-review-fixes` `IN-REVIEW` developer — The `#3` emitter change (`break<FTransform>` F-prefixed form for a generic node on a HasNativeBreak struct) changes `bpir.txt` bytes; its mention is folded into the shared `bpir.txt` 12 comment in `Handlers/Asset/AssetDumpCache.cpp` (G04 owns the 11 -> 12 bump hunk). No code change.
- `#5-verified-linux` `DONE` tester — Run3 on PinWright 7230b41d, UE 5.8 Linux, all non-skipped in run3/full. `PinWright.bpir.compiler.break_native.RoutesToNativeBreakFunction`: `break<Rotator>` emits `BreakRotator` with Yaw wired, and `break<HitResult>` emits `BreakHitResult` with HitBoneName wired. No UK2Node_BreakStruct is created, Kismet status is BS_UpToDate (no generic-break warning), and the decompile and recompile round trip holds. That meets the primary ask: emit the native break when the struct has one. `PinWright.bpir.compiler.break_native.GenericNodeRoundTripsAsFForm` shows an existing generic node decompiles as `break<FTransform>` and recompiles unchanged. `PinWright.bpir.compiler.type_prefix.BreakStructFPrefix` (F-prefixed form keeps the literal node) passed too. Doc: `bpir.instructions.md` §2.7 explains HasNativeBreak routing. The secondary point (success:true with warnings) is unchanged by design, and the ticket itself called it not silent. Limit: `break<FVector>` etc. in F form still draw the warning, which is documented.
