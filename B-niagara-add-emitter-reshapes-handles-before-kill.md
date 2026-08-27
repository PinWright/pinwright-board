---
id: B-niagara-add-emitter-reshapes-handles-before-kill
title: "add_emitter reshapes the emitter-handle array before quiescing live instances, while remove_emitter treats that same order as unsafe"
status: OPEN
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
