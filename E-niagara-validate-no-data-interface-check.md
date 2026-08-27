---
id: E-niagara-validate-no-data-interface-check
title: "niagara.validate does not check compiled-vs-resolved data-interface counts, so it reports clean on a system that will assert on its next tick"
status: OPEN
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
