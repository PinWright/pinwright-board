---
id: B-blueprint-set-variable-settings-transient-internal-error
title: "`blueprint_set_variable_settings` first call returns transient internal error"
status: WONTFIX
severity: Low
category: bug
tags: [variable-settings, transient-error, race, retry, transport]
---

# `blueprint_set_variable_settings` first call returns transient internal error

The first call to `blueprint_set_variable_settings` after a related `blueprint_add_variable` call returned `[Tool result missing due to internal error]`. An identical retry — same arguments, same target BP — succeeded immediately on the next attempt. Could be a transport-layer hiccup, a serialization race against the just-added variable, or an editor refresh ordering problem.

## Repro (observed this session, `W_RenameReplay`)

Sequence (parallel call batch):

1. `mcp__editor_automation__.call path="blueprint.add_variable" args={"path":"/App/App/UI/LobbyAndMenu/Popups/W_RenameReplay.W_RenameReplay","variableName":"Replay","variableType":"EditorReplay"}` → success
2. `mcp__editor_automation__.call path="blueprint.add_variable" args={...}` with `variableName: "ParentScreen", variableType: "MyReplaySelectWidget"` → success

Then immediately (parallel call batch):

3. `mcp__editor_automation__.call path="blueprint.set_variable_settings" args={...}` with `variableName: "Replay", exposeOnSpawn: true, isInstanceEditable: true` → `[Tool result missing due to internal error]`
4. `mcp__editor_automation__.call path="blueprint.set_variable_settings" args={...}` with `variableName: "ParentScreen", exposeOnSpawn: true, isInstanceEditable: true` → `[Tool result missing due to internal error]`

Identical retry of both calls in the next message:

5. Same as #3 → success, applied `instanceEditable, exposeOnSpawn`
6. Same as #4 → success, applied `instanceEditable, exposeOnSpawn`

No editor restart, no other intervening calls, no parameter change. The two failures and two successes were 100% structurally identical except for the prior step.

## Impact

Low — the failure is recoverable by trivial retry. But:
- The error message "Tool result missing due to internal error" is opaque — doesn't say whether the operation was attempted, partially applied, or rejected outright.
- A worker dispatching parallel `set_variable_settings` calls will misinterpret these as hard failures and may abort or branch into a corruption-recovery path.
- If the root cause is a race against `add_variable`'s editor-side serialization, more complex BP-edit chains could hit it more frequently.

**Workaround:** retry on `[Tool result missing due to internal error]` once.

**Proposal:** investigate whether (a) the call is actually being dropped pre-handler in the transport, (b) the variable's `FBPVariableDescription` isn't yet reachable through the canonical lookup path on the next tick after `add_variable`, or (c) the error is a UE serialization-tick collision (the dispatcher's "defer during GC/serialization" path mentioned in `arch.md` could be misclassifying the variable-list mutation as a serialization context). If (b), add a brief retry loop inside the handler before failing. Either way, replace the opaque "Tool result missing" with an actual error code so callers can distinguish transient from permanent.

## History
- `#1-initial-repro` `OPEN` reporter — Hit during `W_RenameReplay` setup. Two parallel `set_variable_settings` calls returned `[Tool result missing due to internal error]`. Identical retry in the next message succeeded for both. No state damage observed.
- `#2-wontfix-reclassified` `WONTFIX` developer — Sprint investigation found the plugin-side code path is provably total on responses: every exit in `BlueprintPropertyHandler.cpp:587-838` calls `SendSuccess` or `SendError`; the dispatcher at `RpcDispatcher.cpp:99-217` preserves request IDs through deferred queueing; the transport at `RpcTransport.cpp:357-388` resolves completions by id and emits typed `TIMEOUT` errors (line 492) on expiry — not the observed opaque string. The literal text `[Tool result missing due to internal error]` is produced client-side by Claude Code's MCP layer when a batch request has fewer response array entries than requests, which strongly points at either the Node passthrough server in `mcp-server/src/index.js` or an uncatchable SEH crash inside a nested UE call (MSVC `catch(...)` without `/EHa` does not catch access violations). Since the reported symptom "retry succeeded immediately" rules out the long-tail timeout path, the most likely culprit is the Node bridge. Actionable improvements tracked in `E-set-variable-settings-error-code-taxonomy`: SEH-aware dispatcher wrapper, Node-bridge dropped-response detection, handler entry/exit verbose logging.
