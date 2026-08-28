---
id: E-niagara-mutation-result-no-quiesced-count
title: "A niagara.* edit that quiesces live system instances does not report how many it stopped"
status: IN-REVIEW
severity: Low
category: ergonomic
tags: [niagara, response-honesty, kill-system-instances, quiesce, MakeMutationResult]
encounters: 1
lastSeen: 2026-08-27
---

# The guard runs, and the response does not mention it

Every `niagara.*` edit carrying `compile: true` now destroys the target's running system instances
before requesting the recompile (`B-niagara-compile-while-live-component-vectorvm-assert`). Editing an
emitter asset quiesces every loaded system that uses it, which can be several.

Nothing in the response says so. A caller whose preview stops has no field explaining why, and a
caller wondering whether the guard applied to their case cannot tell.

**Fix:** a `quiescedInstances` count on `MakeMutationResult`. Held back because that changes the
response shape of every `niagara.*` edit verb at once and needs wiki updates across the namespace --
worth doing deliberately rather than as a rider on a crash fix.

Low severity: the behaviour is correct and documented on the namespace page; only the per-call
disclosure is missing.

## History
- `#1-deferred-from-the-quiesce-fix` `OPEN` reporter -- Proposed by the agent that added the quiesce.
- `#2-shipped-with-the-shared-editor-ticket` `IN-REVIEW` developer -- "Changed MakeMutationResult in
  NiagaraEditTypes.cpp to report `quiescedInstances`, fed by a new
  FNiagaraResolvedTarget::QuiescedInstances that the kill helpers in NiagaraInstanceUtils.cpp now
  count into via BeginEmitterMutationScope and RequestNiagaraCompile. Done together with
  `B-niagara-mutation-scope-blanks-open-preview`; wiki-src/niagara.md documents the field and names
  the verbs that build their own responses and do not carry it."
