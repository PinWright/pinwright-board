---
id: F-inspect-class-members
title: "system.inspect.inspect_class returns only name/path/parent — no UFunctions or UProperties for discovery"
status: DONE
severity: Medium
category: feature
tags: [system-inspect, reflection, ufunction, fproperty, discovery, ergonomics]
---

# `system.inspect.inspect_class` should enumerate UFunctions and UProperties

`system.inspect.inspect_class` is the natural place to discover what
`system.call_subsystem`, `property.get`, `property.set`, and BPIR external
property access can hit on a UClass. Today it returns only `className`,
`classPath`, and `parentClass` — nothing about the class's reflected
members. Agents are forced to source-grep `Plugins/App/Source/...` (or
the engine) just to learn what UFUNCTION/UPROPERTY surface exists on a
class they already resolved via the same RPC.

**Repro of the sparse return shape:**

```
mcp__editor-automation__call path="system.inspect.inspect_class"
# wiki page documents only: className → { name, full path, parent class }
```

```
mcp__editor-automation__call path="system.inspect.inspect_class" args={"className": "UApiSubsystem"}
# {
#   "className":   "ApiSubsystem",
#   "classPath":   "/Script/App.ApiSubsystem",
#   "parentClass": "GameInstanceSubsystem",
#   "success":     true
# }
```

`UApiSubsystem` exposes dozens of `UFUNCTION(BlueprintCallable)` endpoints
and a stack of `BlueprintAssignable` delegates — none of which are visible
through this RPC. The agent has no way to learn from MCP that, e.g.,
`SignInAsync` is a callable UFunction with an `OnComplete` delegate
parameter, or that `bIsAuthenticated` exists as a `BlueprintReadOnly`
property. Symptom is amplified for every `*Subsystem` class an agent might
poke at runtime via `system.call_subsystem`.

The handler lives at
`Source/EditorAutomationRpcGateway/Private/Handlers/Environment/EnvironmentHandler.cpp:1478`
(`REGISTER_RPC_HANDLER("system.inspect.inspect_class", ...)`).

**Proposed extension** (in-place enhancement of the existing RPC — not a
new method; old fields preserved, new fields additive so existing callers
keep working):

```jsonc
{
  "className":   "ApiSubsystem",
  "classPath":   "/Script/App.ApiSubsystem",
  "parentClass": "GameInstanceSubsystem",

  // NEW:
  "functions": [
    {
      "name":       "SignInAsync",
      "returnType": "void",
      "params": [
        { "name": "Username",   "cppType": "FString",                          "direction": "in"  },
        { "name": "OnComplete", "cppType": "FApiAuthCompleteDelegate",         "direction": "in"  },
        { "name": "ReturnValue","cppType": "bool",                             "direction": "return" }
      ],
      "flags": ["BlueprintCallable", "Public"]
    }
    // ...
  ],
  "properties": [
    {
      "name":    "bIsAuthenticated",
      "cppType": "bool",
      "flags":   ["BlueprintReadOnly", "Transient"]
    }
    // ...
  ],
  "interfaces":       ["IFoo", "IBar"],   // optional
  "inheritanceChain": ["UApiSubsystem", "UGameInstanceSubsystem", "USubsystem", "UObject"]  // optional
}
```

Field meaning:
- `functions[].params[].direction` — `"in"` (regular input pin), `"out"`
  (UE `CPF_OutParm` non-return), `"return"` (the synthetic
  `CPF_ReturnParm` slot, if any).
- `functions[].flags` / `properties[].flags` — decoded string lists from
  `EFunctionFlags` / `EPropertyFlags` bitmasks. Common values worth
  surfacing on day one:
  - Function: `BlueprintCallable`, `BlueprintPure`, `BlueprintEvent`,
    `BlueprintImplementableEvent`, `BlueprintNativeEvent`, `Server`,
    `Client`, `NetMulticast`, `Reliable`, `Unreliable`, `Exec`,
    `Static`, `Const`, `Public`/`Protected`/`Private`.
  - Property: `EditAnywhere`, `EditDefaultsOnly`, `EditInstanceOnly`,
    `VisibleAnywhere`, `BlueprintReadWrite`, `BlueprintReadOnly`,
    `BlueprintAssignable`, `BlueprintCallable` (for multicast delegates),
    `Replicated`, `RepNotify`, `Transient`, `Config`, `SaveGame`,
    `Instanced`, `EditConst`.
- `interfaces` / `inheritanceChain` are optional but cheap and frequently
  useful (`inheritanceChain` answers "does this UObject derive from
  `UEditorSubsystem`?" without a second round-trip).

**Implementation hint:**

- Functions: `for (TFieldIterator<UFunction> It(Class); It; ++It)`; for
  each `UFunction*`, walk `Children` / `ChildProperties` to collect
  parameter `FProperty`s; tag each param by checking `CPF_OutParm` /
  `CPF_ReturnParm`; emit `cppType` via `FProperty::GetCPPType`. Decode
  `FunctionFlags` against the `FUNC_*` constants.
- Properties: `for (TFieldIterator<FProperty> It(Class); It; ++It)`; emit
  `name`, `cppType` (`GetCPPType`), and decode `PropertyFlags` against
  the `CPF_*` constants.
- For `inheritanceChain`, walk `Class->GetSuperClass()` until null.
- For `interfaces`, iterate `Class->Interfaces` and emit each
  `FImplementedInterface::Class->GetName()`.
- Set `Class->GetAuthoritativeClass()` first (the standard pattern in
  this plugin's `ClassUtils`) so Blueprint-generated classes (`_C`)
  return the same flag set as their authored UFUNCTION/UPROPERTY
  declarations.

Suggested defaults to keep payloads bounded:
- Include inherited members by default (agents almost always want them).
  If the function/property list grows unmanageable for deep hierarchies,
  add an optional `includeInherited` arg defaulting to `true` later —
  but don't ship the flag preemptively (YAGNI).
- Skip `UFunction`s with `FUNC_Delegate` set (those are signature stubs
  for delegate properties, not callable methods) — but DO surface the
  owning multicast delegate property itself under `properties` with the
  `BlueprintAssignable` flag set, since that's what an agent needs to
  see in order to know `bind_event_dispatcher` will work.

**Acceptance check:**

`system.inspect.inspect_class` with `{ "className": "UApiSubsystem" }`
returns:
- `properties` length ≥ 21 (UApiSubsystem has at least that many reflected
  UPROPERTYs across its auth/token/session state — exact count to be
  confirmed against current source, but the bar is "agent can see them
  all without grep").
- `properties` includes every `BlueprintAssignable` multicast delegate on
  the class (the agent uses these to know what `bind_event_dispatcher`
  targets exist).
- `functions` includes every `BlueprintCallable` UFUNCTION with its full
  parameter list and the correctly-decoded flag set.
- `property.get` driven by a name picked from the new `properties` array
  succeeds on the live `UApiSubsystem` instance (closes the loop:
  discovery → call).
- Old callers that only read `className` / `classPath` / `parentClass`
  see no breakage.

**Workaround until implemented:** Grep the source under
`Plugins/App/Source/.../<ClassName>.h` for `UFUNCTION` / `UPROPERTY`
declarations. Functional but defeats the purpose of having an
introspection RPC — and breaks down entirely for engine classes or any
class whose source isn't in the local checkout.

## History
- `#1-initial-repro` `OPEN` reporter — Confirmed `system.inspect.inspect_class` wiki page documents only `name` / `full path` / `parent class`, and a live call with `{ "className": "UApiSubsystem" }` returns exactly `{ className: "ApiSubsystem", classPath: "/Script/App.ApiSubsystem", parentClass: "GameInstanceSubsystem", success: true }` — no UFunctions, no UProperties, no interfaces, no inheritance chain. Handler is at `Source/EditorAutomationRpcGateway/Private/Handlers/Environment/EnvironmentHandler.cpp:1478` (`REGISTER_RPC_HANDLER("system.inspect.inspect_class", ...)`). Glob over `docs/board/*.md` shows no existing ticket for this gap — `B-inspect-class-short-name-fails.md` is the only adjacent file and addresses a different concern (short-name resolution failure, not the sparseness of the return shape). Proposed in-place extension adds `functions[]`, `properties[]`, and optional `interfaces` / `inheritanceChain`, decoded from `TFieldIterator<UFunction>` / `TFieldIterator<FProperty>` and the `FUNC_*` / `CPF_*` bitmasks; acceptance bar is ≥ 21 properties returned for `UApiSubsystem` including its `BlueprintAssignable` delegates.
- `#2-enumerate-class-members` `IN-REVIEW` developer — Extended `system.inspect.inspect_class` at `EnvironmentHandler.cpp:1478` to emit `functions[]`/`properties[]`/`interfaces[]`/`inheritanceChain[]` via new shared decoders in `Utils/PropertyUtils.{h,cpp}` (`DecodePropertyFlags`, `DecodeFunctionFlags`, `PropertyToInspectJson`, `FunctionToInspectJson`). Iteration walks the authoritative class via `GetAuthoritativeClass()`. Regression test `FInspectClassEnumeratesMembersTest` exercises the helpers against `UActorComponent`.
- `#3-verify-pass` `DONE` tester — Verified `system.inspect.inspect_class { className: "ApiSubsystem" }` now returns 27 UFunctions (with params, returnType, and flags like `BlueprintCallable`/`BlueprintPure`/`Const`/`Public`), 21 UProperties (matching the 21 `UPROPERTY()` decorations in `ApiSubsystem.h`, including the two `BlueprintAssignable` delegates with their `flags: ["BlueprintAssignable", "Instanced"]`), `interfaces: []`, and `inheritanceChain: ["ApiSubsystem", "GameInstanceSubsystem", "Subsystem", "Object"]`. Acceptance bar (≥ 21 properties + BlueprintAssignable delegates) met.
