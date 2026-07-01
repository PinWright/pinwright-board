---
id: B-ensure-exists-create-missing-name
title: "blueprint.ensure_exists can never auto-create: create dispatch omits required 'name' param"
status: IN-REVIEW
severity: Medium
category: bug
tags: [blueprint, ensure_exists, create, dispatch]
encounters: 1
lastSeen: 2026-06-27T14:53:56Z
---

# `blueprint.ensure_exists` auto-create branch is dead — fails with a foreign-param error

`blueprint.ensure_exists` is documented as an "Idempotent BP existence guard:
probes the asset and creates it (via blueprint.create) when missing. Useful at
the top of provisioning scripts so subsequent operations always have a target
asset." `createIfMissing` defaults to `true`.

But the auto-create branch is completely non-functional: whenever the target
asset does **not** exist — the exact case the guard exists to handle — the call
fails with:

```
[MISSING_REQUIRED_PARAM] Missing required parameter 'name' (type: string)
```

The error references `name`, a parameter of a *different* method
(`blueprint.create`) that the caller of `ensure_exists` never passes and the
`ensure_exists` wiki never mentions (its documented required param is `path`,
with `name` only as an alias). So a caller who provides exactly the documented
input gets a clean-but-confusing rejection and the asset is never created. The
probe-only path (asset already exists, or `createIfMissing:false`) works fine;
only the create path — the method's primary purpose — is broken.

## Root cause (source)

`Source/PinWright/Private/Handlers/Blueprint/BlueprintInfoHandler.cpp`
(`blueprint.ensure_exists`) builds the dispatch payload for `blueprint.create`
using the wrong key:

```cpp
CreatePayload->SetStringField(TEXT("blueprintPath"), Path);   // line ~311
// (no "name", no "savePath")
bool bCreateResult = Subsystem->DispatchMethod(TEXT("blueprint.create"), ...);
```

But `blueprint.create` (BlueprintCreationHandler.cpp) declares
`RPC_PARAM_REQ("name", ...)` and reads only `name` + `savePath`; it never reads
`blueprintPath`. The framework's required-param validation rejects the dispatched
payload for missing `name` *before* the create handler body runs (hence the
`MISSING_REQUIRED_PARAM` framing rather than the handler's own
`INVALID_ARGUMENT "blueprint_create requires a name."`). The `blueprintPath`
field is silently ignored.

## Repro (verbatim, replay-confirmed live)

1. `blueprint.ensure_exists` `{path: "/Game/BP_EnsureExistsReplayProbe_7f3a", parentClass: "Actor"}`
   (path does not exist; `createIfMissing` defaults true)
   -> `[MISSING_REQUIRED_PARAM] Missing required parameter 'name' (type: string)`

The asset is not created. The documented `name` alias for `path` resolves the
path fine inside `ensure_exists`, then hits the same dead create dispatch, so
`{name: "/Game/..."}` fails identically.

**Workaround:** call `blueprint.create` directly with `{name: "<AssetName>",
savePath: "<Folder>", parentClass: ...}` (note: `create` wants the bare asset
name + folder split out, not a combined path) and skip the existence guard, or
probe with `blueprint.exists` first.

**Fix:** in `ensure_exists`, split `Path` into folder + asset name and dispatch
`blueprint.create` with `{name, savePath, parentClass}` instead of
`blueprintPath`. (Alternatively, teach `blueprint.create` to accept a combined
`blueprintPath`/`path` alias and derive `name`/`savePath` from it — which would
also fix the foreign-param error surface.)

## History
- `#1-initial-repro` `OPEN` reporter — Filed from a compile_bpir round-trip probe whose setup step tried `ensure_exists` first. Replay-confirmed live against a fresh non-existent path `/Game/BP_EnsureExistsReplayProbe_7f3a` with `parentClass:Actor`: returns `[MISSING_REQUIRED_PARAM] Missing required parameter 'name'` and creates nothing. Root cause traced to BlueprintInfoHandler.cpp dispatching `blueprint.create` with a `blueprintPath` field while `create` requires `name`/`savePath`.
- `#2-fix` `IN-REVIEW` developer — Fixed the dead auto-create dispatch in `Source/PinWright/Private/Handlers/Blueprint/BlueprintInfoHandler.cpp` (`blueprint.ensure_exists`): the create branch now splits the normalized `CheckPath` into a folder + bare asset name and dispatches `blueprint.create` with `{name, savePath, parentClass}` — the params `create` actually consumes — instead of the ignored `blueprintPath`, and guards against a null subsystem (SUBSYSTEM_NOT_FOUND). Added regression test `PinWright.blueprint.ensure_exists.AutoCreatesMissingAsset` in `Source/PinWright/Private/Tests/Blueprint/TestBlueprintDispatcherHandlers.cpp`: drives `ensure_exists` through the live `UPinWrightSubsystem` against a fresh non-existent `/Game/__EA_GatewayTests/BP_EnsureExists_<guid>` and asserts the asset now exists; reverting the payload split makes `create` fail its required-`name` validation so the asset is never created and the test fails.
