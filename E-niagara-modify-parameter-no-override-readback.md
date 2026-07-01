---
id: E-niagara-modify-parameter-no-override-readback
title: "niagara.modify_parameter does not echo the value it wrote, so a set-then-verify loop must fall back to object.call_function GetVariable* (an exact UFUNCTION name found only by reading plugin source) — and niagara.inspect can't confirm it because it reads asset defaults, not the per-actor override store"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [niagara, modify-parameter, spawn-actor, readback, component-override, value-echo]
---

# `niagara.modify_parameter` does not echo the runtime override it wrote, leaving no first-class confirmation surface

The runtime preview workflow — `niagara.spawn_actor` → `niagara.modify_parameter`
(write a `User.*` override onto the spawned `ANiagaraActor`'s `UNiagaraComponent`)
— writes the per-actor `OverrideParameters` store but its **result does not echo
the value it just wrote**. The success response (`NiagaraHandler.cpp:583-589`)
carries `success`/`actorName`/`parameterName`/`parameterType` plus actor identity
(`AddActorVerification` → `actorPath`/`actorGuid`/`existsAfter`/…), but **never the
written value read back off the component**, so the natural set-then-verify loop
("did my override land?") has no first-class confirmation surface.

The only Niagara read RPC, `niagara.inspect`, cannot stand in: it takes `assetPath`
only and `LoadObject`s the asset (`NiagaraInspectHandler.cpp:207-228` —
`RPC_PARAM_REQ("assetPath", ...)`, then `LoadObject<UObject>(nullptr, *AssetPath)`);
it has no actor/component slot and returns the **asset's parameter defaults**, not
the per-actor `OverrideParameters` store that `modify_parameter` writes into.
Re-inspecting the system shows the asset defaults, unchanged.

So an agent must fall back to the generic `object.call_function` against the
component, calling `UNiagaraComponent::GetVariableFloat` / `GetVariableBool` /
`GetVariableVec3` / `GetVariableColor` (the engine UFUNCTION is `GetVariableColor`,
NOT the `GetVariableLinearColor` an agent might guess — getting the exact name wrong
silently fails). This path is not discoverable from the Niagara wiki: the
`niagara.md` overlay's "Cross-cluster overlap" section (line 83-85) names
`niagara.modify_parameter` as a runtime preview helper but says **nothing** about how
to read the override back, and there is no `GetVariable*` reference anywhere on the
page. In this task the agent only found the path by reading the plugin handler
source as an explicit **last resort** (friction note).

This is the runtime-component analogue of the now-shipped echo on
`niagara.set_module_input` (see `E-niagara-inspect-params-stale-after-override` `#4`:
that result now carries `value`, documented at `niagara.md:52`). The same cheap,
already-precedented remedy applies here: have `modify_parameter` read the override
back off the live component and echo it in its own result, so the round-trip
self-confirms in zero extra calls.

This is the read-back complement of `B-set-niagara-param-no-validation`
(IN-REVIEW): that ticket makes the *write* fail loudly for a bad name; even after
that fix lands, a *successful* write still has no first-class `niagara.*`
confirmation surface, so the "set then verify the runtime override" round-trip the
realistic preview task asks for stays a source-diving exercise. Distinct from
`E-niagara-graph-set-parameter-opaque-scope` (asset-side `set_parameter` error
message) and `E-effect-actor-name-slot-vs-actorname` (param-name flip on the
`actor.get_transform` readback). It is also the runtime/component-side analogue of
`E-get-material-info-no-param-defaults` (a read surface that omits the value a
caller just set), but on the Niagara component-override path rather than the
material parameter defaults.

## Why it matters (process cost in this task)

The seed task (`niagara.modify_parameter`) made the readback an explicit step:
step 5 was "read the spawned actor's Niagara component back ... confirm each value
you set is reflected as the runtime override," and step 6 wanted per-parameter
before/after values. The agent applied 4 overrides on `PreviewBurst`
(Float `User.Water Height`, Float `User.Collision Velocity Mult`, Bool
`User.Show Bounds`, Vector `User.World Grid Extents`) and then had to verify each
with a separate `object.call_function GetVariableFloat/Bool/Vec3`, plus a final
`niagara.inspect` that — as expected — showed the **asset defaults unchanged**,
confirming inspect is the wrong tool for component-override readback. Friction note,
verbatim: *"reading the plugin handler source (last-resort) ... revealed the
GetVariable* readback path since niagara.inspect only reads the asset, not
component overrides."* The before-state read at the top of the workflow had the
same problem — `actor.describe PreviewBurst NiagaraComponent` was used to get the
"before" values because no `niagara.*` verb reads the component.

**Workaround:** to read a runtime override off a spawned `ANiagaraActor`, call
`object.call_function` on its `UNiagaraComponent` with
`GetVariableFloat` / `GetVariableBool` / `GetVariableVec3` /
`GetVariableLinearColor` and the `User.<name>` parameter name; `niagara.inspect`
will only ever show asset defaults.

## What it should do

The set-then-verify loop should confirm a `niagara.modify_parameter` from the
**write result itself**, without a source-dived `object.call_function GetVariable*`
fallback and without an `inspect` that only reads asset defaults.

**Fix (the cheap, already-precedented remedy):**

- Have `modify_parameter`'s own result **echo the value it wrote, read back off the
  live component**. After the successful `SetFloat/Vector/Color/BoolParameter`, call
  the matching `UNiagaraComponent::GetVariableFloat` / `GetVariableVec3` /
  `GetVariableColor` / `GetVariableBool` (each reads the same `OverrideParameters`
  store, so it returns exactly what was just written) and put it on the result as a
  `value` field (with a `valueReadBack:true`/`overrideStored` confirmation flag). The
  round-trip then self-confirms from the write result — no second call. This mirrors
  the echo `niagara.set_module_input` already ships (see
  `E-niagara-inspect-params-stale-after-override` `#4`).
- Docs (`docs/wiki-src/niagara.md`): under "Cross-cluster overlap", state that
  `modify_parameter`'s result echoes the read-back override `value`, that
  `niagara.inspect` reads **asset defaults only** (never the per-actor override store),
  and — for callers who still want a manual readback — that the component readback verb
  is `object.call_function … GetVariableColor` (NOT `GetVariableLinearColor`).

Optional follow-up *feature* angle (out of scope here): a first-class
`niagara.get_component_parameters` (or a `readOverrides`/actor-label mode on
`niagara.inspect`) that resolves the `ANiagaraActor` by label and dumps its whole
`OverrideParameters` store, for reading back overrides an agent did **not** just set.

## History
- `#1-initial-audit` `OPEN` reporter — Struggle audit of the `niagara.modify_parameter` runtime-preview task (outcome agent_fail on an unrelated Color-availability gap; this is a distinct PROCESS angle). After spawning `PreviewBurst` and applying 4 runtime overrides, the story's step-5 readback ("confirm each value is reflected as the runtime override") had no `niagara.*` surface: `niagara.inspect` is `assetPath`-only (`NiagaraInspectHandler.cpp:208-236`, `LoadObject` on the asset) and re-inspecting showed asset defaults unchanged. The agent verified each override via `object.call_function GetVariableFloat/Bool/Vec3` on the component — a path it found only by reading the plugin handler source as a last resort (friction note, verbatim quoted above) — and used `actor.describe` for the "before" state for the same reason. Read-back complement of `B-set-niagara-param-no-validation` (IN-REVIEW, fixes the write); distinct from `E-niagara-graph-set-parameter-opaque-scope` (asset-side set_parameter error) and `E-effect-actor-name-slot-vs-actorname` (param-name flip). Ripgrep across OPEN/closed found no existing ticket on reading the spawned-actor component override store back. Page to improve: `docs/wiki-src/niagara.md` (Cross-cluster overlap section names `niagara.modify_parameter` but has zero readback/`GetVariable*` mention).
- `#2-retriage` `OPEN` triage — Low→Medium: no niagara.* readback of the runtime component override forces an object.call_function fallback found only by source-diving, niche preview path.
- `#3-reword-to-value-echo` `OPEN` developer — Reworded toward the narrower, higher-leverage scope (per the adversarial validity lens, and matching the now-shipped echo on `niagara.set_module_input`, `E-niagara-inspect-params-stale-after-override` `#4`). The original docs-only scope (point agents at a 4-verb source-dived `object.call_function GetVariable*` workaround) is strictly weaker than the already-precedented value-echo: re-scoped the Fix so `niagara.modify_parameter` reads the override back off the live component and echoes it on its own result, with the docs note as a complement. Severity stays Medium. Also corrected a factual error the docs-only fix would have shipped — the engine color readback verb is `GetVariableColor`, not the `GetVariableLinearColor` the original ticket named. Status OPEN→IN-REVIEW; see `#4`.
- `#4-echo-written-value` `IN-REVIEW` developer — `niagara.modify_parameter` now reads the override straight back off the spawned actor's `UNiagaraComponent` `OverrideParameters` store (via `GetVariableFloat`/`GetVariableVec3`/`GetVariableColor`/`GetVariableBool` — the same store `SetFloat/Vector/Color/BoolParameter` write to, so it returns exactly what landed) and echoes it on the success result as `value` plus an `overrideStored:true` confirmation flag, so a set-then-verify loop self-confirms from the write result without the source-dived `object.call_function` fallback (`niagara.inspect` still can't help — it reads asset defaults, not this per-actor store). Files: `Source/PinWright/Private/Handlers/Niagara/NiagaraHandler.cpp` (the `niagara.modify_parameter` handler, readback block + response fields), `docs/wiki-src/niagara.md` (Cross-cluster overlap note: result echoes `value`, inspect reads asset defaults only, manual readback uses `GetVariableColor` not `GetVariableLinearColor`). Test: `Source/PinWright/Private/Tests/Niagara/TestNiagaraRuntimeParamValidation.cpp::FNiagaraModifyParameterEchoesWrittenValueTest` spawns a real `ANiagaraActor` with seeded `User.SpawnRate` (Float) + `User.TintColor` (Color) user params, drives `niagara.modify_parameter` through the production dispatcher (`InvokeHandlerWithCapture`), and asserts the result echoes `value`=137.5 (Float) and `value`={r:0.25,g:0.5,b:0.75} (Color) with `overrideStored:true` — both would be absent/false if the echo were reverted.
