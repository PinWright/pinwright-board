---
id: B-niagara-emitter-param-accepted-and-ignored
title: "The three system-wide parameter scopes accept an emitter argument and silently ignore it, across set_parameter, set_curve_keys and add_data_interface"
status: IN-REVIEW
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
ignore it -- which is a mitigation, not a fix. Superseded by the refusal in `ResolveParameterStore`
(see History `#2`); the descriptions on all seven parameter-store verbs now say `INVALID_ARGUMENT`.

## History
- `#1-found-while-declaring-the-param` `OPEN` reporter -- Found by the agent fixing
  `B-niagara-set-parameter-emitter-scope-unreachable`, which declared the parameter the resolver had
  always required and then checked what the other scopes do with it. Source-level claim.
- `#2-refused-in-the-shared-resolver` `IN-REVIEW` developer -- "Added a system-wide-scope guard to
  `ResolveParameterStore` in `NiagaraEditTypes.cpp` so `user` / `systemSpawnRapidIteration` /
  `systemUpdateRapidIteration` refuse a non-empty `emitter` with `INVALID_ARGUMENT` instead of
  resolving it and writing the system store; landed on all seven verbs that route through the
  resolver at once, refreshed their `emitter` descriptions in `NiagaraEditHandler.cpp`,
  `NiagaraCurveHandler.cpp` and `NiagaraAdvancedEditHandler.cpp`, corrected the now-false
  `docs/wiki-src/niagara.md` sentence, and added
  `PinWright.niagara.set_parameter.SystemScopeRejectsEmitter` in
  `Tests/Niagara/TestNiagaraSetParameterEmitterScope.cpp`, which fails before the guard."
- `#3-control-leg-fixture-corrected` `IN-REVIEW` developer -- "The new test's `[user]` CONTROL leg
  (the same write *without* `emitter`, which must still succeed) failed `PARAMETER_NOT_FOUND` with the
  store still at the seeded 1.0. Not the guard: it only fires when `EmitterName` is non-empty, and the
  two rapid-iteration scopes' control legs passed. Fixture defect. `System->GetExposedParameters()` is
  a `FNiagaraUserRedirectionParameterStore`, whose virtual `AddParameter` rewrites an un-namespaced
  entry to `User.<Name>` and files the bare name only as a redirect key; the base `SetParameterData(..,
  bAdd=true)` the fixture seeds through reaches that override. The verb matches on the STORED name
  (`ApplyParameterMutation` -> `FindParameterByName` -> `GetParameters()`), while the fixture's own
  read-back goes through the virtual `FindParameterOffset` and follows the redirect -- so the seed
  looked fine and was invisible to the verb. Gave each probe its own name in
  `Tests/Niagara/TestNiagaraSetParameterEmitterScope.cpp`: `User.PinWrightSystemScopeProbe` for the
  user store, the bare name for the two plain rapid-iteration stores. Guard untouched; both legs still
  mean something -- a guard that refused with an empty `emitter` still fails the control leg."
