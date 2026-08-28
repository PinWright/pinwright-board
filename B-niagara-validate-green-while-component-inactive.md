---
id: B-niagara-validate-green-while-component-inactive
title: "niagara.validate returns valid:true for a system no placed component is running, and no verb reports component activation"
status: OPEN
severity: High
category: bug
tags: [niagara, validate, inspect, false-success, silent-noop, bAutoActivate, component-activation, level-actor, placed-effect, survives-reload, no-readback]
encounters: 1
lastSeen: 2026-08-28
---

# A system can be perfectly healthy and still render nothing, and every verb says it is fine

`niagara.validate` and `niagara.inspect` describe the **asset**. Whether anything in the level is
actually *running* that asset is a property of the placed `UNiagaraComponent`, and no `niagara.*`
verb reads it. So a system whose components are all inactive validates `valid: true`, with zero
errors and zero warnings, while the level renders nothing.

Observed on `/Game/Atlantis/VFX/NS_FishSchool`, `_Orange` and `_Silver`. All three drew no
particles at all for a full day. Every available health signal was green:

| check | verdict | reality |
|---|---|---|
| `niagara.validate level:basic` | `valid: true`, 0 errors | renders nothing |
| `EMITTER_NOT_IN_SYSTEM_GRAPH` | not raised | correct — emitter *is* invoked |
| `grep -c NiagaraNodeEmitter NS_*.uasset` | `2` on all three | correct |
| `grep -c "Data interface count mismatch"` | `0` in the session log | correct |
| engine compile log | `took 2.058021 sec` / `0.183419` / `0.403470` | compiled fine |
| `LogSavePackage` | `.tmp` moved to `.uasset`, `AssetCheck` clean | saved fine |
| `object.call_function IsActive` | **`false`** | the only signal that was not green |

The cause was `bAutoActivate = false` persisted onto the three `UNiagaraComponent`s and saved into
`Atlantis.umap`. The 18 sibling `ANiagaraActor`s in the same level (plankton, ambient bubbles,
vents) sat at the CDO default `true` and all rendered — so the level was a controlled A/B with one
property between the working and the broken set, and no verb could see it.

## Why this is a plugin defect and not just an authoring mistake

1. **The one working oracle is not a verb.** `Docs/map/atlantis-spec.md` already documents
   `object.call_function {objectPath: "...NiagaraComponent0", function: "IsActive"}` as "the only
   reliable *is it alive* signal". Reaching for a raw reflected-UFUNCTION call is the tell that a
   first-class readback is missing. `B-niagara-authored-emitter-forces-inert` `#2` independently
   arrived at the same `IsActive()` discriminator for a *different* root cause — two unrelated
   inert-system bugs, one missing readback.
2. **`validate` already grew a level-aware check and stopped one step short.**
   `B-niagara-authored-emitter-forces-inert` `#6` added `EMITTER_NOT_IN_SYSTEM_GRAPH` precisely
   because "an emitter handle the system graph never invokes cannot run under any reading of the
   asset". A placed component with `bAutoActivate:false` and no activator cannot run either, by the
   same argument, one level further out.
3. **`effect.activate_niagara` echoes `{"active": true}` unconditionally**, so the obvious
   confirmation after an activate is also blind. Already recorded in
   `B-niagara-authored-emitter-forces-inert` `#2` as "a second, separable reporting defect"; this
   ticket is where it costs something.

## Reproduction

    actor.set_component_properties {actorName: "<NiagaraActor>", componentName: "NiagaraComponent0",
                                    properties: {bAutoActivate: false}}
    # save the level, reload it
    niagara.validate {assetPath: "<the system that actor points at>"}
    # -> valid: true, errors: [], warnings: []      ... and the level renders nothing

Cheap disk-level discriminator, no editor needed — `bAutoActivate` is a CDO default, so it is only
serialised into the `.umap` when something overrode it:

    grep -ac bAutoActivate Content/Maps/<Level>.umap    # 0 = every component at default; 1 = at least one override

That single grep separated the last-known-good revision from the broken one across seven commits of
an LFS-tracked map. (`git show <rev>:<path>` returns the LFS *pointer* for these, not the map — read
the blob from the LFS object store instead, or the grep silently reports 0 for every revision.)

## Suggested fixes

1. **`niagara.validate` should report placed-component state.** For a system with placed components
   in the open level, emit a `components` block — count, and per component `isActive` /
   `bAutoActivate` — and raise an issue (`NIAGARA_NO_ACTIVE_COMPONENT` / `NIAGARA_COMPONENT_AUTOACTIVATE_OFF`)
   when a system has placed components and none of them can run. Warning at `basic`, error at
   `strict`, matching how the three structural codes are already layered.
2. **`niagara.inspect` should carry the same block**, so "is this thing running" is answerable
   without `object.call_function`.
3. **`effect.activate_niagara` / `deactivate_niagara` should return the measured
   `IsActive()` after the call**, not a hardcoded verdict.
4. **Any verb that writes `bAutoActivate:false` onto a level component should say so in its
   response** — it is a change that outlives the session and silently disables shipped content.

Adjacent but distinct: `B-niagara-mutation-scope-blanks-open-preview` observes that a killed
instance "does not come back on its own for a component that is not auto-activating". That is the
transient, preview-viewport face of the same asymmetry; this ticket is the persisted, shipped-level
face. No encounter was added there because that defect (`BeginEmitterMutationScope` blanking a
preview) was not the one observed here.

## History
- `#1-three-shipped-systems-dark-for-a-day-every-verb-green` `OPEN` reporter — 2026-08-28. Found
  while diagnosing why `VFX_FishSchool_Blue/Orange/Silver` rendered nothing on `/Game/Maps/Atlantis`
  after an editor restart, having been measured at 398 live particles the previous evening.
  **Measured, not inferred.** `niagara.validate {level:"basic"}` returned `valid:true` with an empty
  `errors` array on all three; `grep -c NiagaraNodeEmitter` returned `2` per system on disk (so
  `B-niagara-authored-emitter-forces-inert` was excluded); the engine log showed all three compiles
  completing (`took 2.058021 / 0.183419 / 0.403470 sec`) and all three `.tmp`→`.uasset` moves
  landing with clean `AssetCheck` (so `B-niagara-compile-wait-does-not-wait` and
  `B-niagara-di-count-mismatch-vectorvm-assert-kills-editor` were excluded — that session logged
  **0** `Data interface count mismatch` lines on those saves). `actor.describe` showed
  `bAutoActivate: false, is_overridden_locally: true` on exactly the three dark components and on no
  other Niagara actor in the level; `object.call_function IsActive` returned `false` on those three
  and `true` on plankton and bubble-vent controls. `LogActorComponent: Warning: SetAutoActivate
  called on component ... NiagaraActor_16/17/18 ... after construction!` at `2026.08.27-16.56.32`
  UTC dates the write to a quiesce step taken to get still frames for an unrelated comparison, ~49 s
  after the capture that needed the scene frozen; the map save that followed baked it in and it was
  committed. Restoring `bAutoActivate` to the CDO default returned 123 + 119 + 154 = 396 live
  particles (sim-cache `read_position_attribute` length), median per-particle displacement
  507-552 uu over 30 ticks with a zero-displacement fraction of 0.000, and the schools render in
  captures from the poses that were previously empty. **Not reproduced from a clean start** — this
  is a post-hoc reconstruction from logs, LFS blobs and live reads, not a deliberate repro.
