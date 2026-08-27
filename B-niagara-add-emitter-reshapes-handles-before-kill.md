---
id: B-niagara-add-emitter-reshapes-handles-before-kill
title: "add_emitter reshapes the emitter-handle array before quiescing live instances, while remove_emitter treats that same order as unsafe"
status: IN-REVIEW
severity: High
category: bug
tags: [niagara, add_emitter, remove_emitter, live-instance, kill-system-instances, crash-adjacent, asymmetry]
encounters: 1
lastSeen: 2026-08-27
---

# The two emitter verbs disagree about whether reshaping under a live instance is safe

`niagara.remove_emitter` kills live system instances **before** it mutates the emitter-handle array.
`niagara.add_emitter` calls `System->AddEmitterHandle` **first** and quiesces only around the compile
-- so with `compile: false` the array is reshaped under a live instance with nothing having stopped it.

One of the two orderings is wrong. `remove_emitter`'s is the cautious one and was presumably chosen
for a reason; if that reason holds, `add_emitter` has the same exposure and does not guard against it.

**Not reproduced.** No crash has been attributed to this ordering. The claim is the asymmetry itself,
plus the fact that a live `FNiagaraSystemInstance` holds indices into the handle array. Establish
whether `remove_emitter`'s kill is load-bearing or merely defensive before copying it -- if it is
defensive, the cheaper resolution is to document why and drop it there rather than add it here.

## History
- `#1-asymmetry-noticed-during-the-compile-fix` `OPEN` reporter -- Raised by the agent that added
  `PinWrightNiagara::KillSystemInstances` to `FinalizeNiagaraEdit` and to `add_emitter`'s compile path
  for `B-niagara-compile-while-live-component-vectorvm-assert`. Source-level reading of the two
  handlers' ordering; deliberately not widened into, since that ticket was about the compile rather
  than the mutation order.
- `#2-engine-says-symmetric-so-add-emitter-now-quiesces-first` `IN-REVIEW` developer -- Settled the
  ordering against UE 5.8 engine source rather than by preference. `remove_emitter`'s kill is
  load-bearing, and `add_emitter` has the same exposure: a live `FNiagaraEmitterInstance` caches its
  POSITION in the handle array (`FNiagaraSystemInstance::InitEmitters` -> `EmitterInstance->Init(EmitterIdx)`)
  and reads it back unchecked as `Sys->GetEmitterHandles()[EmitterIndex]`
  (`NiagaraEmitterInstance.cpp:104-108`); `EmitterHandles` is a `TArray`, so `AddEmitterHandle`'s
  `Add` can reallocate the buffer that reference points into, and surviving indices are no reprieve
  because the system-side execution order is resized from `EmitterHandles.Num()` and then indexes the
  instance's shorter `Emitters` array unchecked in `Tick_Concurrent` (behind only a `checkSlow`).
  Decisive: the engine's own `FNiagaraEditorUtilities::AddEmitterToSystem` opens with
  `KillSystemInstances` under the comment "Kill all system instances before modifying the emitter
  handle list to prevent accessing deleted data" (`NiagaraEditorUtilities.cpp:2141`) -- byte-identical
  to `RemoveEmittersFromSystemByEmitterHandleId` at `:2199`. No asymmetry exists in the engine, so the
  asymmetry here was the defect. Fix: `NiagaraHandler.cpp` `add_emitter` now calls
  `PinWrightNiagara::KillSystemInstances` inside its transaction ahead of `System->Modify()` /
  `AddEmitterHandle` / `RebuildSystemEmitterNodes`, matching `remove_emitter` and the plugin's own
  `BeginEmitterMutationScope` convention; the compile path's existing kill was left in place and its
  rationale reworded (it no longer justifies itself by pointing at `remove_emitter`). Comments at both
  call sites now carry the mechanism and the engine citations. Test added:
  `PinWright.niagara.add_emitter.QuiescesBeforeReshapingHandles` in
  `Source/PinWright/Private/Tests/Niagara/TestNiagaraAddEmitterQuiesce.cpp`, which drives the verb with
  `compile:false` and records `System->GetEmitterHandles().Num()` from inside the
  `OnSystemInstanceChanged` broadcast -- so it asserts the ORDER (0 handles seen by the kill), not just
  that a kill happened.
