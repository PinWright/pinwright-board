---
id: B-bpir-const-ref-fname-literal-dropped
title: "BPIR silently drops a literal on a `const FName&` pin, so every Blackboard accessor fails the BP compile with \"by ref params expect a valid input\""
status: OPEN
severity: Medium
category: bug
tags: [bpir, compile_bpir, pin-default, fname, by-ref, blackboard, ai]
encounters: 1
lastSeen: 2026-09-02T19:55:00Z
---

# A quoted FName literal on a `const FName&` pin is dropped, not applied

## Repro

Any `UBlackboardComponent` accessor. Their key parameter is `const FName& KeyName`, so this is
every AI Blueprint that touches a Blackboard:

```
%bb = call GetBlackboard(Target: %c.AsAIController)
%t  = call GetValueAsObject(Target: %bb, KeyName: "TargetActor")
```

```
[BLUEPRINT_COMPILE_FAILED] BPIR compile failed Blueprint compile: The current value (None) of the
' Key Name ' pin is invalid: 'Key Name' in action 'GetValueAsObject' must have an input wired into
it ("by ref" params expect a valid input to operate on).
```

BPIR placement itself succeeds — the diagnosis comes from UE's Blueprint compiler afterwards, and
`(None)` is the tell: the `"TargetActor"` literal never reached the pin's `DefaultValue`. Nothing in
the BPIR layer reported a problem with the argument; it was accepted and then discarded.

In the Blueprint editor the same node accepts a typed key name in its `Key Name` field with no wire,
so the pin genuinely does hold literal defaults; only the BPIR path fails to write one.

## Workaround

Feed the pin from a `MakeLiteralName` node so it is genuinely *wired*:

```
%k  = call MakeLiteralName(Value: "TargetActor")
%bb = call GetBlackboard(Target: %c.AsAIController)
%t  = call GetValueAsObject(Target: %bb, KeyName: %k)
```

That compiles. The cost is one extra node and one extra BPIR line per Blackboard key touched — an AI
controller and its Behavior Tree tasks reference a dozen keys, so it is a dozen `MakeLiteralName`
nodes that a hand-authored graph would not have. `MakeLiteralInt` / `MakeLiteralFloat` /
`MakeLiteralString` presumably cover the same shape for other by-ref primitive params.

## Expected

Either write the literal into the pin's `DefaultValue` (what the editor does when a user types into
the field), or, if a `const T&` pin genuinely cannot carry a default in this code path, synthesize
the `MakeLiteral*` node automatically — the compiler knows the pin type and has the literal in hand.
Silently dropping the argument and letting UE's compiler fail two layers later, with an error naming
a pin (`' Key Name '`, with the display-name spacing) that does not match the BPIR argument name
(`KeyName`), is the worst of the three.

A one-line note on `bpir.types` § "Optional Empty Pins" or `bpir.errors` would also have saved the
detour: nothing in the wiki mentions that reference parameters need a wired source.

severity rationale: impact=soft blocker (doable, but only via an undocumented workaround discovered
by reading a UE compiler error) x reach=every AI session (all Blackboard get/set verbs, and any
other `const FName&` / `const T&` parameter) -> Medium

## History
- `#1-filed` `OPEN` reporter — Hit while authoring `/Game/FPS/AI/EQC_Target`, a
  `UEnvQueryContext_BlueprintBase` subclass whose `ProvideSingleActor` override reads `TargetActor`
  off the querier's Blackboard. First attempt used the plain quoted literal
  `call GetValueAsObject(Target: %bb, KeyName: "TargetActor")`; `compile_bpir` reported
  `BLUEPRINT_COMPILE_FAILED` with the UE compiler text quoted above. Adding
  `%k = call MakeLiteralName(Value: "TargetActor")` and passing `KeyName: %k` compiled clean
  (`nodeCount:8, compiled:true`) with no other change to the body, which isolates the by-ref pin as
  the cause. Distinct from `B-bpir-fname-pin-bare-identifier-not-resolved` (DONE): that one is about
  the *bare vs quoted* surface form on an ordinary `PC_Name` pin and fails at the BPIR layer with
  `Could not resolve value`; here the quoted form is accepted by BPIR and dropped, and the failure
  is UE's by-ref check on a `const FName&` parameter. Same UE 5.8 host, `EAContentExamples58`.
