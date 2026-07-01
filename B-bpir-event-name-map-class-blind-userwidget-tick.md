---
id: B-bpir-event-name-map-class-blind-userwidget-tick
title: "BPIR compile maps `Tick` -> `ReceiveTick` unconditionally; fails on UserWidget where the UFunction is literally named `Tick`"
status: DONE
severity: High
category: bug
tags: [bpir, override, event-name-map, userwidget, round-trip]
---

# `EventNameMap` is class-blind; breaks Tick (and any AActor-vs-UserWidget overlap) on UMG widgets

`FBpirCompiler::FBpirCompiler` (`Compiler/BpirCompiler.cpp:1257-1268`) seeds an `EventNameMap` of clean -> `ReceiveXxx` UFunction names that assumes the parent is `AActor`. That map is consulted unconditionally before override resolution in `FBpirCompiler::SetupOverride` (`Compiler/BpirCompiler.cpp:4074-4077`): if `Name == "Tick"`, the resolver looks up `ReceiveTick` on the parent class.

`UUserWidget::Tick` (`Engine/Source/Runtime/UMG/Public/Blueprint/UserWidget.h:545-546`) is a `BlueprintImplementableEvent` with the literal UFunction name `Tick` (not `ReceiveTick`). So on any UMG widget BP, `TryResolveBlueprintOverride` (`Handlers/Blueprint/BlueprintHandlerUtils.cpp:543`) does `ParentClass->FindFunctionByName("ReceiveTick")` and returns false with `[COMPILE_FAILED] Line -1: No overridable parent function named 'ReceiveTick' was found` (`BlueprintHandlerUtils.cpp:551-557`).

The decompile side's symmetric `GetStandardOverrideEventNameMap` (`Decompiler/BpirTextEmitter.cpp:518-529`) only rewrites `ReceiveTick -> Tick`. On a UserWidget the entry node's `EventReference.MemberName` is already `Tick`, so decompile emits `entry override Tick(struct<Geometry> MyGeometry, float InDeltaTime)` -- a string the compile path then mishandles. Round-trip is broken for every UMG widget that ticks. Same trap latently exists for any other clean name on the map that happens to collide with a real non-`Receive` UFunction on a non-AActor parent.

## Repro

Asset `/App/App/UI/W_FoundGasLeaks` (parent `UUserWidget`). Call `mcp__editor-automation__call` with `path: "blueprint.compile_bpir"`, `args: { assetPath: "/App/App/UI/W_FoundGasLeaks", code: "entry override Tick(struct<Geometry> MyGeometry, float InDeltaTime) { %d = latent Delay(Duration: 0.2) [completed -> @after]\n@after:\n    ...\n}" }`. Returns `[COMPILE_FAILED] Line -1: No overridable parent function named 'ReceiveTick' was found`. The string passed in is the exact form `blueprint.decompile` emitted from a working pre-existing widget.

**Workaround:** rewrite `entry override Tick(...)` to `entry event Tick(...)` -- compiles, but drops the param signature (decompile shows `entry event Tick()` with no params) and may contribute to the corruption vector tracked separately in MCP-BP-edit guidance.

**Fix:** make `EventNameMap` lookup class-aware in `SetupOverride` (and the three other consumers at `BpirCompiler.cpp:2084/3459`, `BlueprintGraphHandler.cpp:1046`, `BlueprintEventHandler.cpp:211`). Try the literal name on `Blueprint->ParentClass->FindFunctionByName()` first; only fall back to the `Receive`-prefixed mapping when the literal lookup fails AND the parent is an `AActor` subclass. Equivalently: scope the map to AActor-only and let `TryResolveBlueprintOverride`'s existing parent-class fallback (`BlueprintHandlerUtils.cpp:541-549`) handle non-actor cases directly. The decompile-side `NormalizeEventName` (`BpirTextEmitter.cpp:533`) needs the same class gate so it doesn't strip `ReceiveTick -> Tick` on a non-AActor parent if one ever appears.

## History
- `#1-initial-repro` `OPEN` reporter — Found while round-tripping `/App/App/UI/W_FoundGasLeaks` through decompile -> recompile. Decompile emits `entry override Tick(struct<Geometry> MyGeometry, float InDeltaTime)`; recompile of that exact string fails with `No overridable parent function named 'ReceiveTick' was found`. Root cause: `EventNameMap` in `BpirCompiler.cpp:1259` rewrites `Tick -> ReceiveTick` before `TryResolveBlueprintOverride` runs, but `UUserWidget::Tick` (`UserWidget.h:545-546`) is a literal `Tick` UFunction with no `Receive` prefix. The map is AActor-shaped and applied unconditionally. Workaround that compiled (entry event Tick) drops the param signature and is not round-trip-safe.
- `#2-classaware-eventnamemap` `IN-REVIEW` developer — Introduced class-aware `ResolveOverrideEventName` helper in `BpirCompiler.cpp`; tries literal name on `ParentClass->FindFunctionByName` first, falls back to `Receive*` mapping only when the parent is an AActor subclass. Updated `NormalizeEventName` in `BpirTextEmitter.cpp` with the same class gate. UserWidget `Tick` round-trips.
- `#3-verified-userwidget-tick-roundtrips` `DONE` tester — Verified: `mcp__editor-automation__call` with `path: "blueprint.compile_bpir"`, `args: {assetPath: "/App/App/UI/W_GasLeaksDistance", code: "entry override Tick(struct<Geometry> MyGeometry, float InDeltaTime) { ... }"}` returns `compiled: true, success: true, errors: []`. Pre-fix this exact form errored `[COMPILE_FAILED] Line -1: No overridable parent function named 'ReceiveTick' was found`. Post-fix the override is found on the literal UserWidget `Tick` UFunction, body emits cleanly, decompile round-trips the entry signature including `struct<Geometry> MyGeometry, float InDeltaTime`. Same shape confirmed on `/App/App/UI/W_FoundGasLeaks` and `/App/App/UI/W_GasLeaksTargetFound` during the gas-leak null-guard fix in the same session.
