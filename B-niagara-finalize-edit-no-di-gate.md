---
id: B-niagara-finalize-edit-no-di-gate
title: "The whole 30-verb Niagara edit family saves through FinalizeNiagaraEdit with no data-interface check, so a system in the VectorVM-assert state is persisted and reported saved:true"
status: IN-REVIEW
severity: High
category: bug
tags: [niagara, data-interface, vectorvm, finalize-edit, write-path, silent-false-success, missing-gate, latent-corruption]
encounters: 1
lastSeen: 2026-08-28
---

# Two verbs gate the write; the other thirty do not

`B-niagara-di-count-mismatch-vectorvm-assert-kills-editor` established that a Niagara system whose
compiled `FNiagaraVMExecutableData::DataInterfaceInfo` count disagrees with its
`FNiagaraScriptRuntimeCompiledData::ResolvedDataInterfaces` count asserts inside the VectorVM on a
concurrent worker the next time anything ticks it, which is an `appError` and takes the editor down.
That ticket's fix added `PinWrightNiagara::CheckDataInterfaceCounts`
(`Handlers/Niagara/NiagaraDataInterfaceConsistency.h`) and wired
`NiagaraHandler.cpp:RejectOnDataInterfaceMismatch` into exactly two verbs: `niagara.add_emitter` and
`niagara.remove_emitter`.

Every other mutating verb in the namespace saves through a different function —
`NiagaraEdit::FinalizeNiagaraEdit` (`Handlers/Niagara/NiagaraEditTypes.cpp`) — and that function has
no data-interface check at all. It compiles if asked, waits out the compile when `compile` and `save`
were both requested, and then calls `SaveAssetToDiskReportingPresence`. Nothing between the wait and
the write asks whether the thing about to be written is in the state the plugin's own checker
classifies as fatal-on-next-tick.

## What the caller gets

`NiagaraEdit::MakeMutationResult` builds the success envelope for that whole family:
`success`, `operation`, `assetPath`, `assetKind`, `compileRequested`, `compiled`, `saveRequested`,
`saved`, `quiescedInstances`. There is no `dataInterfaceCheck` field. So on a system already carrying
the mismatch — the exact `NS_FishSchool` population the parent ticket describes, a stock template
duplicated and rewritten in place — any of these verbs called with `save: true` writes the asset and
answers `saved: true` with nothing said. `add_emitter` on the same asset, one file over, refuses the
save and returns `NIAGARA_DATA_INTERFACE_MISMATCH` naming both counts per script.

The gap is not only the refusal. On `{compile:false, save:false}` the edit still dirties the package
and leaves the compiled scripts stale — the autosave primer documented in the parent ticket's
`#4-autosave-arms-it-with-no-plugin-save`, which wedged a shared editor for 14 minutes with no
PinWright save verb involved. The response carries no verdict there either, so a caller who has just
armed that clock has no way to learn it from the answer they got.

## Blast radius, measured

`FinalizeNiagaraEdit` has **24 production call sites across 4 files** (plus its own definition in
`NiagaraEditTypes.cpp` and declaration in `NiagaraEditTypes.h`, and 4 more sites in 3 test files):

- `Handlers/Niagara/NiagaraEditHandler.cpp` — 21 sites
- `Handlers/Niagara/NiagaraJsonHelpers.cpp` — 1 site, inside `EndEmitterMutationScope`
- `Handlers/Niagara/NiagaraGraphHandler.cpp` — 1 site
- `Handlers/Niagara/NiagaraRenameParameterHandler.cpp` — 1 site

Those reach **30 distinct RPC verbs**. Direct: `set_property`, `set_parameter`, `add_parameter`,
`remove_parameter`, `add_renderer`, `remove_renderer`, `move_renderer`, `add_module`, `remove_module`,
`move_module`, `set_stack_enabled`, `set_module_input`, `set_module_script`, `set_static_switch`,
`reset_module_input`, `clear_module_overrides`, `set_scalability_property`, `connect_pin`,
`disconnect_pin`, `set_pin_default`, `graph.create_node`, `rename_parameter`. Via
`EndEmitterMutationScope`: `add_event_handler`, `remove_event_handler`, `add_simulation_stage`,
`remove_simulation_stage`, `add_data_interface`, `remove_data_interface`, `set_curve_keys`,
`bind_curve_asset`.

`niagara.add_data_interface` and `niagara.remove_data_interface` are on that list. The two verbs whose
entire job is to mutate a system's data interfaces have no data-interface consistency check.

`MakeMutationResult` has 23 call sites across 3 files.

`niagara.validate` is the only surface that reports this today
(`E-niagara-validate-no-data-interface-check`, IN-REVIEW), and it is a separate call the caller has to
know to make.

## Impact

`saved: true` on an asset that the plugin's own checker would classify as fatal-on-tick is a silent
false-success on a normal path: the caller trusts the write landed cleanly, stops looking, and the
detonation arrives minutes later from an unrelated verb in an unrelated agent's session. The parent
ticket measured that cost — five editor crashes in one session, and a causal chain visible only in the
crash log.

The population most exposed is the one trying to *repair* such a system: `list_orphan_data_interfaces`
tells you the system is broken, and every edit verb you then reach for will happily persist it again
without comment.

Not reachable through the `{compile:true, save:true}` race — `FinalizeNiagaraEdit` already closes that
one by refusing the save when the compile does not land. What remains is `save:true` on a system that
is *already* mismatched, an edit that introduces the mismatch itself, and the ungated
`{compile:false, save:false}` autosave primer.

**Workaround:** call `niagara.validate` (or `niagara.list_orphan_data_interfaces`) after every
Niagara mutation and before trusting `saved: true`. Post-hoc only — it diagnoses the write, it cannot
prevent it, and against the autosave route it can lose the race.

## Why the obvious objection does not hold

`E-niagara-validate-no-data-interface-check` `#2` declined this work on the grounds that
`FinalizeNiagaraEdit` returns `bool` with `bOutCompiled`/`bOutSaved` and has no `FHandlerContext`, so
folding the check in means a signature change plus a new `MakeMutationResult` field across ~25 sites in
5 files. The call-site count is right (24 in 4, +2 for the definition/declaration files). The
conclusion is not: **no signature change is needed, and no call site has to move.**

`FinalizeNiagaraEdit` already takes `FNiagaraResolvedTarget&` by non-const reference, and
`MakeMutationResult` already reads a mutation-accumulated field off that struct. The header comment on
`FNiagaraResolvedTarget::QuiescedInstances` states the pattern verbatim: *"Carrying it on the target is
what lets every mutation verb disclose the blanked preview without an extra out-parameter threaded
through 20+ call sites."* The data-interface verdict is the same shape of fact, produced at the same
point, needed by the same envelope.

The half-measure objection — that `saved:false` with no stated cause is the defect `quiescedInstances`
was added to end — is correct and is exactly why the verdict belongs on the target: carried there, the
cause *is* stated, in the same envelope, without touching a call site.

Separately: `FinalizeNiagaraEdit` returns `true` unconditionally and all 24 call sites discard the
result. That return value is dead today.

## Fix

**Stage 1 — the whole fix, zero call-site edits.**

1. Add to `FNiagaraResolvedTarget` (`NiagaraEditTypes.h`), beside `QuiescedInstances` and for the
   same stated reason: `PinWrightNiagara::EDataInterfaceConsistency DataInterfaceVerdict` (default
   `Unverified`) and `TArray<PinWrightNiagara::FDataInterfaceCountMismatch> DataInterfaceMismatches`.
2. In `FinalizeNiagaraEdit`, after the compile-wait block and before the `Options.bSave` write, run
   `PinWrightNiagara::CheckDataInterfaceCounts(*Target.System, ...)` when `Target.System` is set and
   record both onto `Target`. Ordering matters: a landed compile is what clears the mismatch, so the
   check must follow the wait.
3. On `Mismatched`, suppress the save the way the compile-wait refusal already does — one more
   conjunct on the existing `if (Options.bSave && Target.Asset && bCompileIsPersistable)` guard —
   with the same `UE_LOG(LogPinWrightSubsystem, Warning, ...)` naming the asset and the remedy verbs.
4. In `MakeMutationResult`, emit `dataInterfaceCheck` on **every** response (including `"unverified"`,
   so a pass the check could not make is not echoed as one it could), and `mismatchedScripts` when
   mismatched — same field spellings and same per-script shape
   (`scriptPath` / `emitter` / `compiledDataInterfaces` / `resolvedDataInterfaces`) that
   `NiagaraHandler.cpp:RejectOnDataInterfaceMismatch` and `NiagaraInspectHandler.cpp` already use, so
   the namespace has one vocabulary for one fact.

Cost: two struct fields, ~15 lines in one function, ~10 in another, one file. All 24 call sites and
all 30 verbs compile unchanged and gain both the gate and the field. Reuse
`Tests/Niagara/TestNiagaraDataInterfaceConsistency.cpp`'s existing fixtures; the mismatched branch
cannot be manufactured on every host (see the parent ticket `#5` / `#6`), so assert the field's
presence and its equality with `CheckDataInterfaceCounts` run against the same object, as the sibling
tests do.

**Constraint to document, not to fix:** `NiagaraEdit::ResolveTarget` sets `Target.System` only when the
edited asset *is* a `UNiagaraSystem`. Edits addressed at a standalone Niagara Emitter asset therefore
report `"unverified"`. That is honest and matches the `add_emitter` gate's own scope; it should be
stated in `Docs/wiki-src/niagara.md` rather than papered over.

**Stage 2 — optional, deferrable, and arguably wrong.** To match `add_emitter`'s hard
`NIAGARA_DATA_INTERFACE_MISMATCH` JSON-RPC error, each of the 24 sites would need an
`if (SendNiagaraEditError(Ctx, ...)) return true;` after the finalize call — that is the ~24-site
migration, and it is the only part that needs one. Against doing it: by the time finalize runs, the
mutation has already landed in memory, so failing the whole call misreports a partial success (which is
why `RejectOnDataInterfaceMismatch` has to attach `MutationFacts` to its error payload). Stage 1's
`success:true, saved:false, dataInterfaceCheck:"mismatched", mismatchedScripts:[...]` is a truer
contract for a verb whose in-memory edit did happen. If a hard signal is later wanted anyway, the dead
`bool` return is free.

**Rejected alternative — leave it to `niagara.validate`.** Insufficient on four counts: validate is a
separate call the caller must know to make, and the callers who need it are the ones who do not know;
validate runs after the write, so it diagnoses rather than prevents; the autosave route arms 0–10
minutes after any uncompiled edit, so a post-hoc validate can lose the race; and the plugin has already
ruled that this check belongs on the write path, for `add_emitter` and `remove_emitter`. Thirty sibling
verbs without it is an inconsistency inside one namespace, not a design.

## History
- `#1-finalize-family-ungated` `OPEN` reporter — Raised while `E-niagara-validate-no-data-interface-check`
  was implemented; that ticket's `#2` names this surface and defers it. Verified against current source
  rather than taken on trust: `FinalizeNiagaraEdit` does have the `bool` + two out-params signature and
  no `FHandlerContext`, and the call-site count (24 production sites in 4 files, 30 verbs) matches the
  "~25 in 5" estimate. The cost conclusion drawn from that count does not hold — the
  `Target.QuiescedInstances` precedent, whose header comment gives this exact rationale, carries the
  verdict to `MakeMutationResult` with no signature change and no call-site edits. Not fixed, not
  compiled, not run.
- `#2-stage-1-landed-stage-2-deferred` `IN-REVIEW` developer — **Stage 1 landed; Stage 2 deliberately
  not done.** `FNiagaraResolvedTarget` gained `DataInterfaceVerdict` (default `Unverified`) and
  `DataInterfaceMismatches`; `FinalizeNiagaraEdit` runs `CheckDataInterfaceCounts` after the
  compile-wait block and before the write when `Target.System` is set, and the save guard gained a
  fourth conjunct through a new `NiagaraEdit::MayPersistAfterDataInterfaceCheck(Verdict)` predicate
  (only `Mismatched` refuses) plus a `LogPinWrightSubsystem` Warning naming the asset, both counts
  and the remedy verbs. `MakeMutationResult` emits `dataInterfaceCheck` on **every** response and
  `mismatchedScripts` when non-empty, in the same per-script shape
  `NiagaraHandler.cpp:RejectOnDataInterfaceMismatch` uses. Zero call-site edits, as predicted; the
  cost was two struct fields plus ~55 lines across `NiagaraEditTypes.h`/`.cpp`. The predicate is one
  addition the Fix did not list — it exists so the refusal is testable the way
  `PinWrightNiagara::MayPersistAfterCompileWait` already is, and it pins the direction that matters
  most: `Unverified` must NOT block a write. **Correction to the Fix's claim that "all 30 verbs then
  get both the gate and a stated cause":** `niagara.rename_parameter` and `niagara.graph.create_node`
  do not call `MakeMutationResult` — they build their own envelopes (already documented in
  `Docs/wiki-src/niagara.md` as verbs that do not carry `quiescedInstances` either). Both get the
  gate; neither gets the field, so a refusal there reads as `saved: false` with the cause only in the
  log. Left as-is rather than editing two more files under heavy concurrent editing; noted in the
  wiki. **Stage 2 remains deferred** for the reason the ticket gives — the in-memory mutation has
  already landed by the time finalize runs, so a hard `NIAGARA_DATA_INTERFACE_MISMATCH` from these 24
  sites would misreport a partial success; `success:true, saved:false, dataInterfaceCheck:"mismatched"`
  is the truer contract. Documented constraint preserved: `ResolveTarget` sets `Target.System` only
  for `UNiagaraSystem` assets, so standalone-emitter edits report `"unverified"` and no verdict is
  fabricated — stated in `Docs/wiki-src/niagara.md` under a new `## Every edit verb gates the save on
  a data-interface check, and says so`, and the stale "`add_emitter` / `remove_emitter` refuse"
  sentence in the `niagara.validate` section corrected. Regression test:
  `Source/PinWright/Private/Tests/Niagara/TestNiagaraFinalizeEditDataInterfaceGate.cpp`,
  `PinWright.niagara.finalize_edit.DataInterfaceGateOnEveryMutationVerb` — asserts the predicate's
  three verdicts, that `niagara.add_data_interface`'s envelope carries `dataInterfaceCheck` equal to
  `CheckDataInterfaceCounts` run against the same object (the assertion that fails on a revert, since
  the field did not exist), that finalize is what records the verdict on the target, and that a
  seeded `Mismatched` target refuses a real write while the same asset with a non-mismatched verdict
  writes (positive control, so the refusal is not just an unsavable fixture). The refusal is driven
  through a seeded verdict on a target with no `System` because a genuinely mismatched system is not
  portable across hosts and building one would arm the editor-killing assert. **Not compiled, not
  run** — the working tree is under concurrent edit by many agents and building was out of scope.
