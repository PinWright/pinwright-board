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
- `#2-validate-surveys-placed-components` `IN-REVIEW` developer — "Suggested fix 1 only; 2/3/4 are
  untouched and this ticket should not be closed on them. Pre-existing hole, not a gap in today's
  data-interface work: `dataInterfaceCheck` compares compiled vs resolved DI counts and never
  looked at a component. New `Handlers/Niagara/NiagaraComponentActivation.{h,cpp}` —
  `SurveyPlacedComponents(System, OutComponents)` walks `TActorIterator` over
  `GEditor->GetEditorWorldContext().World()` (editor world only; never PIE, never the asset
  editor's preview world) collecting every `UNiagaraComponent` whose `GetAsset()` is the system,
  with its measured `IsActive()` and `bAutoActivate`, and returns `active` / `none_active` /
  `no_components` / `unverified`. `niagara.validate` calls it on Niagara Systems after the DI check
  and publishes `componentActivation` on every system verdict, plus `componentCount` and a
  `components` array of `{actor, component, componentPath, isActive, autoActivate}` capped at 16
  entries for the three measured verdicts. `none_active` raises `NIAGARA_NO_ACTIVE_COMPONENT` —
  warning at `basic`, error at `strict` via the existing `NormalizeValidationSeverity` escalation
  set, the layering this ticket proposed; not an every-level error like
  `EMITTER_NOT_IN_SYSTEM_GRAPH`, because a system gameplay spawns or activates legitimately has no
  active placed component and erroring at `basic` would fail validate on sound content.
  `unverified` (no editor world) raises `NIAGARA_COMPONENT_ACTIVATION_UNVERIFIED` as a warning at
  both levels and deliberately publishes **no** count or array — a zero there would read as 'the
  level places none', which is the fabricated pass this check exists to stop. `no_components`
  raises nothing: the survey reaches only the loaded actors of the open level, so an unloaded World
  Partition cell and a level that is simply not open are indistinguishable from there genuinely
  being none, and warning would fire on nearly every call. Codes are raw literals at the call site,
  matching the other validate issue codes (`NO_EMITTERS`, `EMITTER_NOT_IN_SYSTEM_GRAPH`,
  `NIAGARA_DATA_INTERFACE_UNVERIFIED`) — none is registered in `ErrorCodes.h` because none reaches
  `SendError`, and the registry scan is anchored on emission, not on issue payloads. Regression
  test `PinWright.niagara.validate.InactivePlacedComponentIsNotAPass` in
  `Tests/Niagara/TestNiagaraValidateComponentActivation.cpp`: validates the unplaced fixture first
  (must report `no_components` and raise nothing — the false-positive direction pinned before the
  failure), then spawns an actor rooted on a registered `bAutoActivate:false` NiagaraComponent
  under `FScopedEditorWorldActorGuard`, asserts `IsActive()` is false so the premise holds, and
  requires `none_active`, `componentCount:1`, the components entry reporting both flags off, a
  `warning` at `basic` and an `error` at `strict`. Every part-2 assertion fails before the fix —
  no field, no block, no issue. NOT COMPILED and NOT RUN (both were out of scope for this pass);
  the test needs a live-editor suite run. Docs: `Docs/wiki-src/niagara.md` gains an *Is anything in
  the level running it?* subsection under `### niagara.validate`, and the 'no other codes change
  between levels' sentence is corrected."
- `#3-returned-three-of-four-defects-stand` `OPEN` tester — "Returned to OPEN: the previous entry set IN-REVIEW after implementing only defect 1 of the four this ticket lists, and its own report says the ticket should not be closed on it. What landed is real and stays: niagara.validate now surveys placed components and publishes componentActivation, with no_components raising nothing (the survey reaches only loaded actors, so 'found none' is not 'there are none') and unverified omitting componentCount so a zero cannot read as an empty level. Still unfixed: defect 2 (niagara.inspect does not carry the block), defect 3 (effect.activate_niagara sets active:true immediately after Activate() with no readback, and deactivate_niagara hardcodes false the same way - verified in EffectHandler.cpp), and defect 4 (verbs writing bAutoActivate:false do not say so). Defect 3 is a silent false-success in its own right and is being split into its own ticket."
