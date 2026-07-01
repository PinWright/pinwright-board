---
id: E-replace-node-noncallable-no-hint
title: "replace_node/create_node CallFunction 'not BlueprintCallable or is deprecated' error dead-ends — no callable-sibling list, no pointer to search_api"
status: OPEN
severity: Low
category: ergonomic
tags: [blueprint-graph, replace_node, create_node, callfunction, error-messages, error-hint, discovery, search_api]
encounters: 1
lastSeen: 2026-06-23T10:08:23Z
---

# `replace_node` CallFunction refusal offers no recovery path when the target exists but isn't BlueprintCallable

When `blueprint.graph.replace_node` (and the shared `create_node` `CallFunction`
path) is asked to target a UFUNCTION that exists on the resolved class but lacks
`FUNC_BlueprintCallable` (or carries `DeprecatedFunction`), it returns a flat
dead-end error:

> `[FUNCTION_NOT_FOUND] Function 'SetLightFColor' is not BlueprintCallable or is deprecated`

The message is accurate but offers **no recovery path**. Critically, by the time
this error fires the handler has *already resolved the function* — it knows the
exact owning `UClass` (e.g. `ULightComponent`) — yet it does not:

- name the callable sibling that the caller almost certainly wants (here
  `SetLightColor`, which *is* `BlueprintCallable` on the same component), nor
- list any of the class's `FUNC_BlueprintCallable` UFunctions, nor
- point at the existing discovery RPCs (`blueprint.search_api` after
  `blueprint.build_api_index`, or `system.inspect.inspect_class`, which now
  returns per-function `flags` including `BlueprintCallable`).

Faced with that wall, the caller's only path to "which setter *is* callable on
this class" is to grep the engine header (`C:/UE_5.7/.../LightComponent.h`) — a
fallback that defeats the typed-RPC surface entirely and breaks for any class
whose source isn't in the local checkout.

## Distinct from the underlying impossibility

The task that surfaced this was genuinely impossible (`SetLightFColor` is not a
graph-callable function), so the refusal itself is *correct* — this is not a tool
bug and there is nothing for the per-finding judge to file. The friction is the
**error ergonomics**: an accurate refusal that doesn't steer the caller to the
adjacent callable function or to the discovery RPC that enumerates them. The
caller had the connections/intent fully scouted and was one good hint away from
self-correcting to `SetLightColor` without leaving the MCP.

## Established board precedent (same friction shape)

This is the recurring "accurate error that doesn't suggest the next step"
pattern the board has fixed repeatedly for other RPCs:
- [`E-add-metasound-node-error-no-hint`](E-add-metasound-node-error-no-hint.md)
  (OPEN — `NODE_CLASS_NOT_FOUND` should point at `search_metasound_nodes`).
- [`E-make-struct-error-hint`](E-make-struct-error-hint.md) (DONE — `make<>`
  error now suggests the native `Make*` function).
- [`E-bpir-createwidget-pin-hint`](E-bpir-createwidget-pin-hint.md) (DONE — pin
  error now hints the correct `Class:` pin).

`blueprint.graph` is the next RPC family in this series.

## What it should do

On the `!FUNC_BlueprintCallable` / deprecated branch, enrich the error with a
recovery hint, e.g.:

> `[FUNCTION_NOT_FOUND] Function 'SetLightFColor' on ULightComponent is not
> BlueprintCallable (or is deprecated). Closest BlueprintCallable members:
> SetLightColor, SetIntensity, SetTemperature. Call blueprint.search_api
> { query: "SetLight", classFilter: ["LightComponent"] } (after
> build_api_index) to list valid CallFunction targets.`

Cheapest useful form: the function is already resolved at the refusal site
(`BlueprintGraphCrudHandler.cpp:1201-1204`, where `Func` is non-null but fails
the `FUNC_BlueprintCallable` check), so the owning `UClass` is in hand —
iterate `TFieldIterator<UFunction>` on that class, keep the
`FUNC_BlueprintCallable` non-deprecated ones, surface the top 1–3 by name
similarity to the requested member, and append the `search_api` / `inspect_class`
pointer. Even with zero near matches, the pointer alone removes the
grep-the-engine-header fallback.

## Wiki gap (docs angle)

`docs/wiki-src/blueprint.graph.md` documents `replace_node`'s refusal list and
`target` vocabulary but never tells the caller to **pre-flight the target's
BlueprintCallable-ness** before issuing the swap. The page's "See also"
already describes `blueprint.search_api` as the way to find "which function to
target inside a CallFunction" (line ~295), but the `replace_node`/`create_node`
sections do not cross-reference it. Add a one-line note in the `replace_node`
`CallFunction` `target` bullet: *"The target must be a BlueprintCallable,
non-deprecated UFUNCTION; vet it first with `blueprint.search_api` —
engine-native setters like `SetLightFColor` exist as C++ UFUNCTIONs but are not
graph-callable."*

## Evidence

From this task's friction note (`blueprint.graph.replace_node` focus): "engine
source (C:/UE_5.7/.../LightComponent.h) shows SetLightFColor is
UFUNCTION(Category=...) without BlueprintCallable, so it can't be a graph call
node, and replace_node correctly refused it with a clear [FUNCTION_NOT_FOUND]
message — no workaround exists short of changing the target to a
BlueprintCallable setter like SetLightColor." Call log: a single clean
`replace_node` attempt failed with the dead-end error after five correct
discovery/scouting calls; the caller resolved the *why* only by reading the
engine header out-of-band, not through any MCP RPC. The swap was abandoned
(`agent_fail`) because the caller could not, in-band, pivot to the callable
sibling the error itself knew about.

## History
- `#1-initial-audit` `OPEN` reporter — PROCESS friction from the `blueprint.graph.replace_node` fuzz task: a correct `[FUNCTION_NOT_FOUND] Function 'SetLightFColor' is not BlueprintCallable or is deprecated` refusal that dead-ends the caller. The error fires at `BlueprintGraphCrudHandler.cpp:1201-1204` *after* the function is resolved (owning `ULightComponent` in hand), yet lists no BlueprintCallable sibling and no pointer to `blueprint.search_api`/`system.inspect.inspect_class`; the caller had to grep `C:/UE_5.7/.../LightComponent.h` to learn the callable alternative (`SetLightColor`) and then abandoned the swap. Not a tool bug (the refusal is correct; judge filed nothing) — this is the error-ergonomics angle. Follows the established error-hint precedent series: `E-add-metasound-node-error-no-hint` (OPEN), `E-make-struct-error-hint` (DONE), `E-bpir-createwidget-pin-hint` (DONE). Also tagged `docs`: `docs/wiki-src/blueprint.graph.md` should add a "vet the target with search_api first" note to the `replace_node`/`create_node` CallFunction `target` bullet.
