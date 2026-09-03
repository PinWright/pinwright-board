---
id: B-bpir-break-struct-emits-generic-node
title: "BPIR `break<Rotator>` emits a generic BreakStruct node the BP compiler rejects — 'The structure cannot be broken using generic break node'; the compile still reports success"
status: OPEN
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
