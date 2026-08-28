---
id: B-niagara-add-emitter-reshapes-handles-before-kill
title: "add_emitter reshapes the emitter-handle array before quiescing live instances, while remove_emitter treats that same order as unsafe"
status: DONE
severity: High
category: bug
tags: [niagara, add_emitter, remove_emitter, live-instance, kill-system-instances, crash-adjacent, asymmetry]
encounters: 1
lastSeen: 2026-08-28T09:28:00+05:00
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
- `#3-ordering-verified-asymmetry-resolved` `DONE` verifier — 2026-08-28, rebuilt DLL at plugin HEAD `b79ba53e`, editor pid 14932. The asymmetry this ticket reports is gone, and the engine reading `#2` settled it against is correct.
  **Source at this HEAD.** Inside `add_emitter`'s transaction (opened `NiagaraHandler.cpp:236`): `PinWrightNiagara::KillSystemInstances(*System)` at **:252**, `System->Modify()` at **:253**, `System->AddEmitterHandle(...)` at **:254**, `RebuildSystemEmitterNodes` at **:259**. So the quiesce precedes the reshape rather than only bracketing the compile, which is what this ticket asked for, and it matches `remove_emitter` (kill `:452`, `Modify()` `:453`, `RemoveEmitterHandlesById` `:456`). A second, deliberate kill sits at **:293**, immediately before `RequestCompile` at **:294**, with a comment stating the double kill is intentional.
  **What could not be measured, said plainly rather than glossed.** The ordering is intra-call, and no engine tick interleaves an RPC handler, so no external oracle can observe "killed before reshaped". `UNiagaraComponent::is_active()` was tried and is not usable for it: it reads `false` after *any* structural edit, because the edit's own `FNiagaraSystemUpdateContext` reinitialises components — a `compile:false` `add_simulation_stage` deactivated the component just as an `add_emitter` did, so the oracle cannot separate the kill from the reinit. The ordering claim therefore rests on the source above plus `#2`'s engine citation (`FNiagaraEditorUtilities::AddEmitterToSystem` opening with `KillSystemInstances` under "Kill all system instances before modifying the emitter handle list"), both re-checked here.
  **What was measured.** `add_emitter` and `remove_emitter` with `compile:false` against a system with an activated `ANiagaraActor` component — `is_active()` confirmed `true` immediately before each call, so a live instance genuinely existed — produced no fault. Zero asserts and zero access violations across the whole session. This ticket was explicitly never reproduced, so the absence of a crash is consistent with both the fixed and unfixed states and is not offered as the proof; the resolved asymmetry is.
  **One asymmetry survives, in the other direction, and is worth a look without reopening this.** `add_emitter` now kills twice — before the reshape and again before its own compile — while `remove_emitter` kills only before the reshape and **not** before its own `RequestCompile(true)` at `NiagaraHandler.cpp:490`. By this ticket's own argument that is now the unjustified side. The shared finalizer covers every other edit verb (`NiagaraEditTypes.cpp:1581`), so `remove_emitter` is the one hand-rolled compile path without a pre-compile kill.
