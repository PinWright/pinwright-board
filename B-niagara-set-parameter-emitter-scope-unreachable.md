---
id: B-niagara-set-parameter-emitter-scope-unreachable
title: "niagara.set_parameter cannot reach any of its three emitter-scoped stores: the scope resolver demands `emitter`, the verb never declares `emitter`, so the dispatcher rejects the payload with UNKNOWN_PARAMS before the handler runs"
status: DONE
severity: High
category: bug
tags: [niagara, set_parameter, rapid-iteration, emitter-scope, schema-gap, unreachable-code, emitter-required, unknown-params]
encounters: 1
lastSeen: 2026-08-28T09:15:00+05:00
---

# `niagara.set_parameter`'s emitter scopes are unreachable by any argument combination

`niagara.set_parameter` documents its `scope` parameter as
"user, rendererBindings, spawnRapidIteration, or updateRapidIteration". Three of
those four scopes are **emitter-scoped and cannot be reached at all**:

- Omit `emitter` → `EMITTER_REQUIRED`, "Parameter scope 'spawnRapidIteration'
  requires an emitter."
- Add `emitter` → `UNKNOWN_PARAMS`, because the verb does not declare that
  parameter and the dispatcher's declared-parameter gate refuses the payload
  before the handler body executes.

The two branches contradict each other. There is no third option: every emitter
rapid-iteration and renderer-binding value in every Niagara asset is unwritable
through this verb.

## Root cause (guilty source lines)

Two halves, both in this tree at PinWright `8e76cad5`.

**Half 1 — the resolver requires `emitter` for exactly these three scopes.**
`Plugins/PinWright/Source/PinWright/Private/Handlers/Niagara/NiagaraEditTypes.cpp`,
`ResolveParameterStore` (opens `:460`). After the three scopes that need no emitter
(`user`, `systemSpawnRapidIteration`, `systemUpdateRapidIteration`) are handled and
returned, `:489-492`:

```cpp
        if (!OutTarget.EmitterData)
        {
            return FNiagaraEditError::Make(TEXT("EMITTER_REQUIRED"), FString::Printf(TEXT("Parameter scope '%s' requires an emitter."), *Scope));
        }
```

Everything past that guard — `rendererBindings` (`:494`), `spawnRapidIteration`
(`:498`), `updateRapidIteration` (`:504`) — is unreachable without `EmitterData`.

**Half 2 — the verb never declares `emitter`, so it can never be supplied.**
`NiagaraEditHandler.cpp:2196-2205`:

```cpp
REGISTER_RPC_HANDLER("niagara.set_parameter", "niagara", "Set one Niagara parameter-store value.",
    RPC_PARAMS(
        RPC_PARAM_REQ("assetPath", "string", "Path to the Niagara System or Niagara Emitter asset"),
        RPC_PARAM_REQ("scope", "string", "Parameter scope, such as user, rendererBindings, spawnRapidIteration, or updateRapidIteration"),
        RPC_PARAM_REQ("name", "string", "Parameter name"),
        RPC_PARAM_REQ("type", "string", "Parameter type"),
        RPC_PARAM_REQ("value", "any", "Value to assign"),
        RPC_PARAM_OPT("compile", "boolean", "Compile after edit"),
        RPC_PARAM_OPT("save", "boolean", "Save after edit")
    ))
```

No `emitter`. `UNKNOWN_PARAMS` has a single emit site,
`Plugins/PinWright/Source/PinWright/Private/Dispatch/RpcDispatcher.cpp:159`, raised
by the declared-parameter gate at `:126-161` — so an added `emitter` key is refused
by the dispatcher and `ResolveParameterStore` never runs.

The verb's own `scope` doc string advertises two of the unreachable scopes by name.

**The sibling verbs prove this is an oversight, not a design.** Every other Niagara
verb that can target an emitter-scoped store declares `emitter`:

- `niagara.set_curve_keys` — `NiagaraCurveHandler.cpp:230` registration,
  `RPC_PARAM_OPT("emitter", ...)` at `:237` ("Emitter name for emitter-scoped stores").
- `niagara.add_data_interface` — `NiagaraAdvancedEditHandler.cpp:437` registration,
  `emitter` at `:443` (same doc string).

`niagara.add_parameter` (`:2232`) and `niagara.remove_parameter` (`:2268`) share the
same omission and are almost certainly broken the same way — **not separately
verified in this session**, but they are registered in the same block with the same
parameter list shape and route through the same `ResolveParameterStore`.

## Verbatim repro

```
niagara.set_parameter {assetPath: "<system path>", scope: "spawnRapidIteration",
                       name: "<param name>", type: "<type>", value: <value>}
  -> [EMITTER_REQUIRED] Parameter scope 'spawnRapidIteration' requires an emitter.

niagara.set_parameter {assetPath: "<system path>", scope: "spawnRapidIteration",
                       emitter: "<emitter handle name>",
                       name: "<param name>", type: "<type>", value: <value>}
  -> [UNKNOWN_PARAMS] ...
```

The same pincer applies to `updateRapidIteration` and `rendererBindings`, which take
the identical `:489` branch.

## This contradicts a shipped error message and a shipped wiki section

`E-niagara-graph-set-parameter-opaque-scope` (IN-REVIEW, Medium) asserts the exact
opposite capability, twice, and has already **shipped** both assertions:

> "A richer sibling already exists: `niagara.set_parameter`
> (`NiagaraEditHandler.cpp:1738`) takes an explicit `scope` (user /
> rendererBindings / spawnRapidIteration / updateRapidIteration / …) … so it **CAN**
> set the `Constants.*` / rapid-iteration scalars"

> "the reporter's `NS_Confetti` `Constants.*` case **was settable all along** via
> `niagara.set_parameter` with a rapid-iteration scope — the re-do was partly a
> wrong-method choice."

Its `#2-reword-and-fix` shipped that claim into (a) the new
`PARAMETER_NOT_FOUND` / `UNSUPPORTED_PARAM_TYPE` error messages on
`niagara.graph.set_parameter`, which now point callers at `niagara.set_parameter`
"(scope + type + value) to set a non-User scope", and (b) a new
`### niagara.graph.set_parameter` section in `Docs/wiki-src/niagara.graph.md` naming
the same recourse.

If this ticket is right, **that error string and that wiki section actively
misdirect**: they send a caller who hit an opaque failure on the legacy RPC toward a
sibling that cannot serve an emitter-scoped store either. The caller burns a
round-trip on `EMITTER_REQUIRED`, a second on `UNKNOWN_PARAMS`, and ends with no
route at all. Note the sibling ticket's `Constants.*` case is precisely a **per-emitter
rapid-iteration** scalar — the case this ticket says is unreachable.

That ticket's own line citation is also stale: it cites `niagara.set_parameter` at
`NiagaraEditHandler.cpp:1738`; in this tree the registration is at `:2196`.

Fixing the `emitter` declaration would make that ticket's claim true, which is the
cheapest way to resolve the contradiction. Until then the error message and the
wiki section are wrong. An encounter note recording this has been appended to
`E-niagara-graph-set-parameter-opaque-scope`.

## What it should do

Declare `RPC_PARAM_OPT("emitter", "string", "Emitter name for emitter-scoped stores")`
on `niagara.set_parameter` — and on `niagara.add_parameter` / `niagara.remove_parameter`,
which share the omission — matching `NiagaraCurveHandler.cpp:237` and
`NiagaraAdvancedEditHandler.cpp:443` verbatim. The payload parser and
`ResolveParameterStore` already consume `EmitterData`; nothing downstream needs to
change. If the parameter genuinely cannot be honoured for some scope, the
`EMITTER_REQUIRED` message must name a scope that is actually reachable rather than
demanding an argument the schema forbids.

## Workaround

None found for emitter-scoped stores. `user`, `systemSpawnRapidIteration` and
`systemUpdateRapidIteration` still work (they return before the `:489` guard).

## Distinct from related tickets

- `E-niagara-graph-set-parameter-opaque-scope` (IN-REVIEW, Medium) is about the
  **legacy** `niagara.graph.set_parameter`'s opaque `PARAM_FAILED`. Different RPC,
  different file (`NiagaraGraphHandler.cpp`). It is cited here because its remedy
  depends on a capability this ticket says does not exist.
- `B-set-niagara-param-no-validation` is the **runtime component-side** setter
  reporting success for bogus names — different store, opposite direction.
- `F-niagara-dynamic-input-nested-inputs` (OPEN) is blocked by this: a dynamic
  input's own sub-inputs DO land in the rapid-iteration store as
  `Constants.<Emitter>.<FunctionName>.<Input>`, so fixing this ticket would make
  that feature reachable today without the resolver machinery that ticket scopes.

severity rationale: impact=hard blocker with no workaround on three of four documented scopes — every emitter rapid-iteration and renderer-binding value is unwritable, and the two failure codes contradict each other so the caller cannot even tell the capability is missing x reach=emitter rapid-iteration is where per-emitter authored values live, and this also blocks `F-niagara-dynamic-input-nested-inputs`'s cheap path and falsifies a shipped error message plus a shipped wiki section -> High

## History
- `#1-initial-repro` `OPEN` reporter — Found building the Atlantis level (map as forcing function; host `CLAUDE.md` § "What this project is for"), 2026-08-27, UE 5.8, PinWright at `8e76cad5` in this checkout. `niagara.set_parameter {scope:"spawnRapidIteration"}` without `emitter` returns `EMITTER_REQUIRED`; adding `emitter` returns `UNKNOWN_PARAMS`, so the store is unreachable by any argument combination. Source-confirmed in this tree, both halves: `ResolveParameterStore` (`NiagaraEditTypes.cpp:460`) guards `:489-492` with `if (!OutTarget.EmitterData) return EMITTER_REQUIRED`, gating `rendererBindings` (`:494`), `spawnRapidIteration` (`:498`) and `updateRapidIteration` (`:504`); the `niagara.set_parameter` registration at `NiagaraEditHandler.cpp:2196-2205` declares assetPath/scope/name/type/value/compile/save and **no `emitter`**, and `UNKNOWN_PARAMS`'s single emit site `Dispatch/RpcDispatcher.cpp:159` (declared-param gate `:126-161`) rejects the key before the handler body runs. Sibling verbs that DO declare it: `niagara.set_curve_keys` (`NiagaraCurveHandler.cpp:237`) and `niagara.add_data_interface` (`NiagaraAdvancedEditHandler.cpp:443`), so the one-line schema addition is the fix. `niagara.add_parameter` (`:2232`) / `niagara.remove_parameter` (`:2268`) share the omission — flagged, not separately verified. Contradiction recorded: `E-niagara-graph-set-parameter-opaque-scope` (IN-REVIEW) asserts twice that `niagara.set_parameter` CAN set `Constants.*` rapid-iteration scalars and shipped that claim into both a new error message on `niagara.graph.set_parameter` and a new `Docs/wiki-src/niagara.graph.md` section; if this ticket holds, both misdirect. That ticket's `NiagaraEditHandler.cpp:1738` citation for `set_parameter` is also stale (`:2196` in this tree). A dated encounter section and History entry recording the contradiction were appended to that ticket. Board-wide, `EMITTER_REQUIRED` appears on no other ticket.
- `#2-declare-emitter-param` `IN-REVIEW` developer — Declared `RPC_PARAM_OPT("emitter", "string", ...)` on `niagara.set_parameter` (`Source/PinWright/Private/Handlers/Niagara/NiagaraEditHandler.cpp`, registration now at `:2254`), which is the only half that was missing: `ParseParameterPayload` already reads the `emitter` key into `FNiagaraParameterEditPayload::EmitterName`, `ValidateParameterPayload` already copies it into the target spec, and `ResolveTarget` already resolves it to an `FNiagaraEmitterHandle` (name match, case-insensitive, via `FindEmitterHandle`) plus `EmitterData`, so all three emitter scopes work unchanged once the dispatcher stops rejecting the payload. Also corrected the `scope` description to enumerate all six accepted scopes instead of "such as" four. The new description states which scopes consume `emitter` and that the system-wide scopes ignore it. No resolver or handler-body change; nothing else in the file touched. Regression tests added in `Source/PinWright/Private/Tests/Niagara/TestNiagaraSetParameterEmitterScope.cpp`: `PinWright.niagara.set_parameter.EmitterParamReachesTheHandler` (declared spec is an optional string; a payload carrying `emitter` dispatched through a real `FRpcDispatcher` reaches `ResolveTarget` and fails `ASSET_NOT_FOUND` rather than `UNKNOWN_PARAMS`, with an undeclared control key still rejected so the gate is proven live) and `PinWright.niagara.set_parameter.EmitterScopedStoreIsWritable` (seeds a float into a real emitter's spawn rapid-iteration store, asserts the emitter-omitted call still fails `EMITTER_REQUIRED` and writes nothing, then asserts the `emitter`-carrying call lands 7.5 in that store — read back from the store, not the response echo). Not fixed, same omission confirmed by a namespace-wide sweep of every `REGISTER_RPC_HANDLER` param list: `niagara.add_parameter` and `niagara.remove_parameter` are the only other Niagara verbs that route through `ResolveParameterStore` without declaring `emitter`; every other parameter-store verb (`set_curve_keys`, `bind_curve_asset`, `add_data_interface`, `remove_data_interface`, `rename_parameter`) already declares it.

- `#3-verified-against-built-binary` `DONE` verifier - Behavioural repro against the running editor (PinWright HEAD `b79ba53e`), 2026-08-28, UE 5.8, on a scratch `asset.duplicate` of `SimpleExplosion`. Baseline read off the store: `Constants.UpwardMeshBurst.InitializeParticle.Lifetime Max` = **2** in `spawnRapidIteration`. Omitting `emitter` still returns `[EMITTER_REQUIRED] Parameter scope 'spawnRapidIteration' requires an emitter.` - unchanged and correct. Adding `emitter:"UpwardMeshBurst"` now returns `success:true` where `#1` measured `UNKNOWN_PARAMS`, and re-reading the **store** (`niagara.inspect {parametersOnly:true, parameterName:"InitializeParticle.Lifetime Max"}`, not the response echo) gives **9.5**, while the sibling `Constants.OmnidirectionalBurst.InitializeParticle.Lifetime Max` is untouched at 2.25 - so `emitter` resolves to the right store rather than any store. The two branches no longer contradict each other and the three emitter-scoped stores are reachable. This also closes the contradiction recorded here and in `E-niagara-graph-set-parameter-opaque-scope` `#3`: per-emitter `Constants.*` scalars really are settable through `niagara.set_parameter`, so the shipped error text and wiki section that point there are true as written. **`#2`'s "Not fixed" note on the two siblings is stale.** The running registry declares `emitter` on `niagara.add_parameter` and `niagara.remove_parameter` too (both generated wiki pages carry the identical param doc), and `add_parameter {scope:"spawnRapidIteration", emitter:"SimpleSpriteBurst", name:"Constants.SimpleSpriteBurst.VerifyProbe", type:"float", defaultValue:3.75}` returned `success:true` against the same fixture.
