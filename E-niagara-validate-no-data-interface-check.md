---
id: E-niagara-validate-no-data-interface-check
title: "niagara.validate does not check compiled-vs-resolved data-interface counts, so it reports clean on a system that will assert on its next tick"
status: IN-REVIEW
severity: Medium
category: ergonomic
tags: [niagara, validate, data-interface, vectorvm, latent-corruption, missing-check]
encounters: 1
lastSeen: 2026-08-27
---

# The check exists now; the read that should run it does not

`B-niagara-di-count-mismatch-vectorvm-assert-kills-editor` added
`PinWrightNiagara::CheckDataInterfaceCounts`, and `add_emitter` / `remove_emitter` gate their save on
it. But `niagara.validate` -- the verb whose entire job is to answer "is this asset sound" -- does not
call it.

So a system already carrying the mismatch (authored before the gate shipped, or reached by a verb that
does not gate) validates clean and then kills the editor on its next tick. Validate's verdict is
exactly where an author would look for this.

**Fix:** one call to `CheckDataInterfaceCounts` from validate, reported as an error rather than a
warning -- a system in that state cannot run under any reading, which is the same argument that made
`EMITTER_NOT_IN_SYSTEM_GRAPH` an error at every level.

Two related surfaces from the parent ticket's fix list, both also uncovered: a `sequencer.set_playhead`
pre-flight (that verb was the observed trigger, because opening the Level Sequence editor forces the
re-tick that detonates the corrupt system), and folding the check into
`NiagaraEdit::FinalizeNiagaraEdit`, the shared compile+save path for the rest of the family, which has
no post-write validity check at all.

## History
- `#1-uncovered-surfaces-from-the-di-fix` `OPEN` reporter -- Listed by the agent that wrote the
  checker, which scoped itself to the two verbs its ticket named.
- `#2-validate-runs-the-di-check` `IN-REVIEW` developer — "Wired `PinWrightNiagara::CheckDataInterfaceCounts` into `niagara.validate` in `Handlers/Niagara/NiagaraInspectHandler.cpp`. New file-local `AddDataInterfaceConsistencyIssues`, called from the `UNiagaraSystem` branch after `AddUninvokedEmitterIssues`, publishes the wire verdict as `dataInterfaceCheck` on every verdict (so a verified pass is distinguishable from one the check could not make) and raises one `NIAGARA_DATA_INTERFACE_MISMATCH` **error per offending script**, naming the script and both counts, at `basic` as well as `strict` — the same argument that made `EMITTER_NOT_IN_SYSTEM_GRAPH` an error at every level. The message names `niagara.list_orphan_data_interfaces` / `niagara.remove_orphan_data_interfaces` as the remedy. `Unverified` is a `NIAGARA_DATA_INTERFACE_UNVERIFIED` **warning**, not silence and not an error: an empty mismatch list on an unverified system is not evidence there are none, so silence would read as a pass, while erroring would fail validate on every system nothing has compiled this session. Raw `TEXT(...)` literals at the call site — the file does not cite `ErrorCodes::ERR_` and adopting the registry there would turn its pre-existing raw codes into hard failures of `core.error_codes.RegistryAdoptingFilesUseConstantsOnly`; `ERR_NIAGARA_DATA_INTERFACE_MISMATCH` already exists in `Handlers/ErrorCodes.h` and its comment was extended to record validate as a second emitter of the spelling. No new registry entry: validation *issue* codes (`NO_EMITTERS`, `DISABLED_EMITTER`, `EMITTER_NOT_IN_SYSTEM_GRAPH`) are a separate vocabulary from wire error codes and none of them is registered. **Deliberately skipped the two related surfaces this ticket names, neither silently.** (a) `NiagaraEdit::FinalizeNiagaraEdit`: it returns `bool` with `bOutCompiled`/`bOutSaved` and has no `FHandlerContext`, so it cannot report a refusal; folding the check in means a signature change plus a new `MakeMutationResult` field across ~25 call sites in 5 files, i.e. a contract change to the whole edit family rather than one call. A half-measure (refuse the save, report nothing) is worse than none — `saved:false` with no stated cause is the exact defect `quiescedInstances` was added to end. `NiagaraEditTypes.cpp` also took several concurrent edits this cycle. Wants its own ticket. (b) `sequencer.set_playhead` pre-flight: the verb is the hot per-frame driver for capture bursts, is not Niagara-scoped, and the systems that can detonate are not limited to the sequence's bindings (the editor re-ticks the level), so an honest pre-flight is a whole-level scan on every frame of every burst — and it would refuse a legitimate scrub because of an unrelated actor. The right shape is a level-wide audit verb or a check at editor-open time, not a gate on this verb. Regression test `PinWright.niagara.validate.DataInterfaceConsistencyIsChecked` appended to `Tests/Niagara/TestNiagaraDataInterfaceConsistency.cpp` (reuses that file's `MakeAuthorableSystem` / `NewTransientEmitter` fixtures, adds no new ones): asserts the response carries `dataInterfaceCheck` at all (the counterfactual — the field did not exist), that it **equals** `CheckDataInterfaceCounts` run against the same object (so it cannot be a constant), and, at `basic` and `strict` both, that each verdict has exactly one legal reporting shape read from the top-level `errors`/`warnings` arrays — mismatched ⇒ `error` + `valid:false`, unverified ⇒ `warning` and never a mismatch it could not measure, consistent ⇒ neither code — plus that an emitter asset gets no fabricated verdict. The mismatched branch is not manufactured, for the reason the file's other tests state: a genuinely corrupt system arms the appError that would kill the automation host. `Docs/wiki-src/niagara.md` `### niagara.validate` documents the field, both codes, the error-at-every-level rule and the remedy verbs. NOT compiled and NOT run — the orchestrator owns the build/test loop."
