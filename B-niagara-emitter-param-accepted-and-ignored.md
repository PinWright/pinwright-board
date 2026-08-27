---
id: B-niagara-emitter-param-accepted-and-ignored
title: "The three system-wide parameter scopes accept an emitter argument and silently ignore it, across set_parameter, set_curve_keys and add_data_interface"
status: OPEN
severity: Medium
category: bug
tags: [niagara, set_parameter, set_curve_keys, add_data_interface, parameter-store, accepted-and-ignored]
encounters: 1
lastSeen: 2026-08-27
---

# An argument that is resolved, then discarded

`ResolveTarget` resolves `emitter` into an `EmitterHandle` for every scope, but `ResolveParameterStore`
returns the *system* store for the three system-wide scopes (`user`, `systemSpawnRapidIteration`,
`systemUpdateRapidIteration`) and the resolved handle has no effect on the write.

So a caller who passes `emitter` with a system scope gets success, and the emitter they named had
nothing to do with what was written. That is `rpc-design.md` section 21's exact failure mode: a
parameter accepted and ignored.

Scope is wider than one verb. `set_parameter`, `set_curve_keys` and `add_data_interface` all route
through `ResolveParameterStore` and all ignore it the same way, so a guard added to one of them alone
would create a *new* divergence rather than remove one. The fix belongs in `ResolveParameterStore`, or
in a shared validation step above it, and should land on all three at once.

Confirmed not to redirect anything else: `FinalizeNiagaraEdit` branches on `Target.System` first, so a
stray `emitter` does not send the compile somewhere unexpected. The effect really is confined to being
ignored.

Documented in the meantime -- `set_parameter`'s `emitter` description now says the system-wide scopes
ignore it -- which is a mitigation, not a fix.

## History
- `#1-found-while-declaring-the-param` `OPEN` reporter -- Found by the agent fixing
  `B-niagara-set-parameter-emitter-scope-unreachable`, which declared the parameter the resolver had
  always required and then checked what the other scopes do with it. Source-level claim.
