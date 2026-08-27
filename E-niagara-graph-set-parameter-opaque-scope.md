---
id: E-niagara-graph-set-parameter-opaque-scope
title: "niagara.graph.set_parameter collapses three distinct failures into one opaque [PARAM_FAILED]; split into PARAMETER_NOT_FOUND vs UNSUPPORTED_PARAM_TYPE and point at niagara.set_parameter"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [niagara, niagara-graph, set-parameter, error-message, scope, docs]
encounters: 2
lastSeen: 2026-08-27T18:56:59+05:00
---

# `niagara.graph.set_parameter` is silently scoped to exposed User.* Float/Bool params

`niagara.graph.set_parameter` (`NiagaraGraphHandler.cpp` ~line 426-474) only
ever consults `System->GetExposedParameters()` (the user-redirection store),
and only probes the **Float** and **Bool** typedefs:

```cpp
FNiagaraUserRedirectionParameterStore& UserStore = System->GetExposedParameters();
// Try float
if (UserStore.FindParameterVariable(FNiagaraVariable(GetFloatDef(), Name))) { ... }
// Try bool
if (UserStore.FindParameterVariable(FNiagaraVariable(GetBoolDef(),  Name))) { ... }
Ctx.SendError(TEXT("PARAM_FAILED"),
    TEXT("Parameter not found or type not supported (Float/Bool only)."));
```

So the only thing the RPC can set is an **exposed `User.*` scalar of type Float
or Bool**. Every other parameter a caller might reasonably try collapses into
the **same single opaque error**:

- a `Constants.*` / module-namespace / per-emitter scalar (not in the exposed
  store) -> "not found" branch
- an exposed `User.*` param of a non-Float/Bool type (Vector, LinearColor, int)
  -> also falls through, because only the Float/Bool typedefs are probed

The caller cannot tell **which** case they hit from `[PARAM_FAILED] Parameter
not found or type not supported (Float/Bool only)`: is the name wrong, is the
parameter not exposed as `User.*`, or is it the wrong type? The error neither
names the scope restriction ("only exposed `User.*` params are settable here")
nor distinguishes "no such param" from "param exists but isn't a settable type."

This is the ergonomic-message twin of the silent-success defect tracked on
[`B-set-niagara-param-no-validation`](B-set-niagara-param-no-validation.md)
(the **runtime component-side** setters `effect.set_niagara_parameter` /
`niagara.modify_parameter`, which wrongly report success for bogus names). That
ticket fixes the *runtime* path's false positive; this ticket is the
**asset-side editor** path's false-but-opaque negative. They are different
methods and different failure directions — not a duplicate.

**The narrow scope of this legacy RPC is by design, but the error must say so.**
A richer sibling already exists: `niagara.set_parameter`
(`NiagaraEditHandler.cpp:1738`) takes an explicit `scope`
(user / rendererBindings / spawnRapidIteration / updateRapidIteration / …), an
arbitrary `type`, and `value:"any"`, so it CAN set the `Constants.*` /
rapid-iteration scalars and the non-Float/Bool types this RPC rejects. So the
reporter's `NS_Confetti` `Constants.*` case was settable all along via
`niagara.set_parameter` with a rapid-iteration scope — the re-do was partly a
wrong-method choice. The defect here is therefore narrowly the **opaque error**:
the legacy RPC must (a) tell the caller *which* of the three cases they hit and
(b) point them at `niagara.set_parameter` as the typed/multi-scope escape hatch,
rather than implying the only recourse is to add a `User.*` param or change
systems.

## Why it matters (process cost in the originating task)

In the `niagara.graph` op-splice task (38 calls) the final step was "expose/adjust
a user-facing scalar default." The chosen system (`NS_Confetti`) exposes **no
`User.*` scalars** — only `Constants.*` (e.g. `Constants...Velocity Speed Scale`,
`Constants...Spawn Probability`). Every `set_parameter` against those returned the
opaque `[PARAM_FAILED]`. Because the error gave no hint that the restriction was
"exposed `User.*` only," the agent could not recover in place: it
`niagara.inspect`'d multiple systems hunting for one with a user param
(`NS_SimpleRate_Lightweight` -> 0 user params; `NS_EQ_Reactive` -> 6, incl.
`User.ScaleMultiplier float`), then **abandoned `NS_Confetti` entirely and redid
the whole graph splice** (create op + 2 connect_pins + verify) in `NS_EQ_Reactive`
just to satisfy the single-system requirement. A scope-naming error message would
have let the agent skip straight to "pick a system with a `User.*` scalar" — or,
better, to `niagara.set_parameter` with a rapid-iteration scope, which would have
set the `Constants.*` scalar on `NS_Confetti` directly and avoided the re-do
entirely. The recovery cost was thus part method-choice (the legacy RPC is
deliberately narrow) and part opaque-error (it gave no pointer to the richer
sibling). This ticket fixes the half that is the handler's fault: the error.

Friction note, verbatim:

> "niagara.graph.set_parameter only resolves true User.* params — every
> rapid-iteration \"Constants.*\" float (the only scalars NS_Confetti exposes)
> failed with an opaque \"[PARAM_FAILED] Parameter not found or type not
> supported\", forcing me to abandon NS_Confetti and redo the whole graph splice
> in NS_EQ_Reactive (the one system with user-exposed scalars)."

Call-log corroboration (all `is_error:true` `[PARAM_FAILED]`):
- `set_parameter` `NS_Confetti Constants...Velocity Speed Scale=1` -> PARAM_FAILED
- `set_parameter` `NS_Confetti ConfettiBurst...Velocity Speed Scale=1` -> PARAM_FAILED
- `set_parameter` `NS_Confetti Constants...Spawn Probability=1` -> PARAM_FAILED
- (then 3 `niagara.inspect` calls to find a system with a `User.*` scalar)
- `set_parameter` `NS_EQ_Reactive User.ScaleMultiplier=1` -> ok (the redo)

## What it should do

**Split the single `PARAM_FAILED` fall-through into the two distinct cases,
reusing existing error codes — no new code, and no type-widening.** After the
Float and Bool probes both miss, re-probe the exposed store by name only
(`FNiagaraUserRedirectionParameterStore::FindParameterVariable(..., IgnoreType=true)`,
which returns the param's actual type) and branch:

1. **Name is NOT in the exposed store** → `PARAMETER_NOT_FOUND`: "'<name>' is not
   an exposed parameter; `niagara.graph.set_parameter` only sets exposed `User.*`
   scalars (Float/Bool). `Constants.*` / module-namespace / per-emitter params are
   not settable here — use `niagara.inspect` to list exposed params,
   `niagara.add_parameter` to expose a `User.*` knob, or `niagara.set_parameter`
   (scope + type + value) to set a non-User scope." (`PARAMETER_NOT_FOUND` already
   exists at `ErrorCodes.h:405`.)
2. **Name IS exposed but not Float/Bool** → `UNSUPPORTED_PARAM_TYPE`: name the
   actual type and that only Float/Bool are settable via this RPC, and point at
   `niagara.set_parameter` for other types. (`UNSUPPORTED_PARAM_TYPE` already
   exists at `ErrorCodes.h:563`; `UNSUPPORTED_TYPE` at `:568` is the generic
   alternative — reuse one of these, do **not** invent a new code.)

**Explicitly out of scope: widening this RPC's type coverage** (Vector /
LinearColor / int). `niagara.set_parameter` already covers those types AND the
rapid-iteration/renderer scopes, so adding them here would be redundant
gold-plating. The fix is purely the error-code split + message pointers.

## Docs (discoverability half)

Add a `### niagara.graph.set_parameter` section to overlay page
`docs/wiki-src/niagara.graph.md` (the file exists but currently documents only
`create_node`): state plainly that it sets **exposed `User.*` scalars only**
(Float/Bool), that `Constants.*` / module-namespace / per-emitter parameters and
non-Float/Bool types are **not** settable through it, and that the recourse is
either `niagara.add_parameter` a `User.*` param (or pick a system that already
exposes one) **or** use `niagara.set_parameter` (scope + type + value) for other
scopes/types. Cross-link `niagara.inspect` as the way to enumerate which params
are settable.

## Cross-ref

- `B-set-niagara-param-no-validation` (IN-REVIEW) — the runtime component-side
  twin; opposite failure direction (false success vs opaque false failure).
- `E-error-code-vocabulary-registry` (IN-REVIEW) — `PARAM_FAILED` is one of the
  ad-hoc single-use codes that registry would canonicalize; this ticket also
  argues it conflates two distinct conditions.
- `F-niagara-link-module-input-to-parameter` (OPEN) — same task family
  (exposing/driving a user-facing tunable in Niagara authoring).
- `niagara.set_parameter` (`NiagaraEditHandler.cpp:1738`) — NOT a ticket; the
  richer sibling RPC this one's new error points callers at. Takes scope + type +
  value:any with differentiated error codes, so it covers the `Constants.*` /
  rapid-iteration / non-Float-Bool cases the legacy `niagara.graph.set_parameter`
  rejects.

## Encounter 2026-08-27 — the escape hatch this ticket points callers at does not reach emitter scopes

**This section reports a contradiction with the fix shipped in `#2`. It does not
edit any text above; the body's reasoning stands as written by its author.**

Observed 2026-08-27 on UE 5.8 in the EAContentExamples58 checkout, PinWright at
`8e76cad5`, filed in full as `B-niagara-set-parameter-emitter-scope-unreachable`.

This ticket asserts twice, in "The narrow scope of this legacy RPC is by design"
and again in Cross-ref, that `niagara.set_parameter` "CAN set the `Constants.*` /
rapid-iteration scalars", and that the reporter's `NS_Confetti` case "was settable
all along via `niagara.set_parameter` with a rapid-iteration scope". In this tree it
cannot be:

- `niagara.set_parameter`'s three emitter-scoped scopes (`rendererBindings`,
  `spawnRapidIteration`, `updateRapidIteration`) all fall past
  `ResolveParameterStore`'s `if (!OutTarget.EmitterData) return EMITTER_REQUIRED`
  guard at `Handlers/Niagara/NiagaraEditTypes.cpp:489-492`.
- The verb's own registration at `Handlers/Niagara/NiagaraEditHandler.cpp:2196-2205`
  declares `assetPath`/`scope`/`name`/`type`/`value`/`compile`/`save` and **no
  `emitter`**, so supplying one is refused by the dispatcher's declared-parameter
  gate (`Dispatch/RpcDispatcher.cpp:126-161`, `UNKNOWN_PARAMS` at `:159`) before the
  handler body runs.

Live result: `scope:"spawnRapidIteration"` without `emitter` → `EMITTER_REQUIRED`;
with `emitter` → `UNKNOWN_PARAMS`. No argument combination reaches the store.

`NS_Confetti`'s `Constants...Velocity Speed Scale` / `Constants...Spawn Probability`
are **per-emitter** rapid-iteration scalars, so they are in exactly the unreachable
set. On the evidence above, the re-do in `#1` was not a wrong-method choice: there
was no right method.

**Two shipped artefacts inherit the claim and currently misdirect callers**, until
`B-niagara-set-parameter-emitter-scope-unreachable` is fixed:

1. The `PARAMETER_NOT_FOUND` and `UNSUPPORTED_PARAM_TYPE` messages `#2` added to
   `niagara.graph.set_parameter`, which point at "`niagara.set_parameter` (scope +
   type + value) to set a non-User scope".
2. The `### niagara.graph.set_parameter` section `#2` added to
   `Docs/wiki-src/niagara.graph.md`, which names the same recourse.

A caller who follows either now burns one round-trip on `EMITTER_REQUIRED` and a
second on `UNKNOWN_PARAMS` and ends with no route at all — worse than the opaque
`PARAM_FAILED` this ticket set out to fix, because it looks like a lead.

The cheapest resolution is to fix the sibling (declare `emitter` on
`niagara.set_parameter`, matching `NiagaraCurveHandler.cpp:237` and
`NiagaraAdvancedEditHandler.cpp:443`), which makes this ticket's claim and both
shipped artefacts true as written. Otherwise the message and the wiki section need
their pointer qualified to the `user` / `systemSpawnRapidIteration` /
`systemUpdateRapidIteration` scopes only.

Minor: this ticket cites `niagara.set_parameter` at `NiagaraEditHandler.cpp:1738`
in two places; in this tree the registration is at `:2196`.

## History
- `#1-initial-audit` `OPEN` reporter — Process-audit finding from the `niagara.graph` op-splice task (38 calls). `niagara.graph.set_parameter` (`NiagaraGraphHandler.cpp` ~426-474) only consults `System->GetExposedParameters()` and only probes Float/Bool typedefs, so it can set exactly one thing: an exposed `User.*` Float/Bool scalar. Three distinct failure cases (name not exposed, exposed-but-wrong-type, unsupported type) all collapse into one opaque `[PARAM_FAILED] Parameter not found or type not supported (Float/Bool only).` In the task this hid the `User.*`-only scope: `NS_Confetti` exposes only `Constants.*` scalars, so all three `set_parameter` attempts failed identically, and with no scope hint the agent abandoned `NS_Confetti` and redid the entire op-splice in `NS_EQ_Reactive` (the one system with a `User.*` scalar) — discovered by inspecting three systems by exhaustion. Distinct from `B-set-niagara-param-no-validation` (different methods: runtime `effect.set_niagara_parameter`/`niagara.modify_parameter`; opposite direction: false success). Proposes splitting the error into `PARAMETER_NOT_FOUND` vs a type-unsupported code that names the actual type + scope, plus a `docs/wiki-src/niagara.graph.md` note that the RPC is exposed-`User.*`-scalar-only and `niagara.add_parameter` is the way to create a settable knob. Ripgrep across OPEN/closed found no existing ticket on `niagara.graph.set_parameter`'s scope or error message.
- `#2-reword-and-fix` `IN-REVIEW` developer — REWORD then implement. Reword corrections (per three validity lenses): (a) fixed wrong citation — `PARAMETER_NOT_FOUND` is at `ErrorCodes.h:405`, not :399; (b) reuse the existing `UNSUPPORTED_PARAM_TYPE` (`:563`) / `UNSUPPORTED_TYPE` (`:568`) code for the wrong-type branch instead of inventing a new `PARAM_TYPE_UNSUPPORTED`; (c) dropped the type-widening as out-of-scope gold-plating (the richer sibling `niagara.set_parameter`, `NiagaraEditHandler.cpp:1738`, already covers other scopes/types); (d) acknowledged that sibling as the escape hatch the reporter could have used on `NS_Confetti`'s `Constants.*` scalar, so the re-do was partly a wrong-method choice. FIX: in `niagara.graph.set_parameter` (`Source/EditorAutomationRpcGateway/Private/Handlers/Niagara/NiagaraGraphHandler.cpp`), after the Float and Bool typedef probes both miss, re-probe the exposed (User.*) store by name only via `FNiagaraUserRedirectionParameterStore::FindParameterVariable(..., IgnoreType=true)` (returns `FNiagaraVariableWithOffset*` carrying the actual type) and branch: name absent → `PARAMETER_NOT_FOUND` (names the User.*-only scope, points at `niagara.inspect`/`niagara.add_parameter`/`niagara.set_parameter`); name present but not Float/Bool → `UNSUPPORTED_PARAM_TYPE` (names the actual type via `GetType().GetName()`, points at `niagara.set_parameter`). Added `#include "NiagaraParameterStore.h"` + `"NiagaraUserRedirectionParameterStore.h"`. DOCS: added a `### niagara.graph.set_parameter` section to `Docs/wiki-src/niagara.graph.md` stating the exposed-User.*-scalar-only (Float/Bool) restriction and the `niagara.add_parameter` / `niagara.set_parameter` recourses. TESTS (`Source/EditorAutomationRpcGateway/Private/Tests/Assets/TestNiagaraHandlers.cpp`): `niagara.graph.set_parameter.NotFoundCode` (non-exposed name → expects `PARAMETER_NOT_FOUND`) and `niagara.graph.set_parameter.UnsupportedTypeCode` (exposed `User.ColorTint` Vec3 → expects `UNSUPPORTED_PARAM_TYPE` and message naming the actual `Vector3f` type); both would fail under the reverted single-`PARAM_FAILED` fall-through. Not compiled/tested here (later phase).
- `#3-encounter-sibling-escape-hatch-unreachable` `IN-REVIEW` reporter — Additional evidence, no status change and no edit to existing text. Contradiction recorded: this ticket asserts twice (body § "The narrow scope of this legacy RPC is by design", and Cross-ref) that `niagara.set_parameter` CAN set `Constants.*` / rapid-iteration scalars, and `#2` shipped that claim into the new `PARAMETER_NOT_FOUND` / `UNSUPPORTED_PARAM_TYPE` messages on `niagara.graph.set_parameter` AND into the new `### niagara.graph.set_parameter` section of `Docs/wiki-src/niagara.graph.md`. Observed 2026-08-27 on UE 5.8 in the EAContentExamples58 checkout (PinWright `8e76cad5`): `niagara.set_parameter {scope:"spawnRapidIteration"}` returns `EMITTER_REQUIRED` without `emitter` and `UNKNOWN_PARAMS` with it, so the emitter-scoped stores are unreachable by any argument combination. Source-confirmed in that tree: `ResolveParameterStore`'s `if (!OutTarget.EmitterData) return EMITTER_REQUIRED` at `Handlers/Niagara/NiagaraEditTypes.cpp:489-492` gates `rendererBindings`/`spawnRapidIteration`/`updateRapidIteration`, while the `niagara.set_parameter` registration at `Handlers/Niagara/NiagaraEditHandler.cpp:2196-2205` declares no `emitter` and the dispatcher's declared-param gate (`Dispatch/RpcDispatcher.cpp:126-161`, emit at `:159`) refuses the key first. `NS_Confetti`'s `Constants...` scalars are per-emitter, i.e. in exactly the unreachable set, so `#1`'s re-do was not a wrong-method choice. Consequence: the shipped error message and wiki section now point callers at a dead end. Filed in full as `B-niagara-set-parameter-emitter-scope-unreachable`; fixing that (declare `emitter`, per `NiagaraCurveHandler.cpp:237` / `NiagaraAdvancedEditHandler.cpp:443`) makes this ticket's claim and both artefacts true as written. Minor: the `NiagaraEditHandler.cpp:1738` citation for `set_parameter` is stale (`:2196` in that tree). `encounters` 1→2, `lastSeen` refreshed.
