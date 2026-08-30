---
id: E-configure-asc-echoes-invalid-replication-mode
title: "gas.configure_asc echoes an unrecognized replicationMode back as if applied (silently leaves the prior mode, no error)"
status: WONTFIX
severity: Low
category: ergonomic
tags: [gas, configure_asc, replication, echo, validation, readback, rpc-audit, removed-method]
---

# `gas.configure_asc` reports an invalid `replicationMode` as applied while silently keeping the old value

`gas.configure_asc` documents `replicationMode` as one of exactly `full, mixed,
minimal`. For a *valid* value it works and round-trips correctly (verified live:
`mixed`->`mixed`, `minimal`->`minimal` via `gas.get_gas_info`). But for an
*unrecognized* string it returns a clean success (`ok:true`, `isError:false`)
whose result field **echoes the rejected input verbatim** —
`"replicationMode":"bogusvalue"` — even though the component's replication mode
is silently left unchanged. There is no error, no warning, and no signal that
the value was not applied.

This is concretely misleading for the natural configure-then-trust-the-response
workflow: an agent that typos a mode (or tries an unsupported one) gets a green
success that *names the bad mode back to it*, so it reasonably concludes the bad
mode is now set. Only an independent `gas.get_gas_info` readback reveals the
truth (the component still holds its prior valid mode). The result misreports
the applied state.

## Verbatim repro (live, replayed against mcp__editor-automation__call)

Scaffold: `BP_ReplayHero` (parent `Character`) under `/Game/ReplayGAS`, ASC
added via `gas.add_ability_system_component`, then set to `mixed` via
`gas.configure_asc` (round-trip confirmed `mixed`).

1. `gas.configure_asc {"blueprintPath":"/Game/ReplayGAS/BP_ReplayHero","replicationMode":"bogusvalue"}`
   -> `{"componentName":"AbilitySystemComponent","replicationMode":"bogusvalue","assetPath":"/Game/ReplayGAS/BP_ReplayHero","assetName":"BP_ReplayHero","existsAfter":true,"assetClass":"Blueprint"}`
   (success; result field claims `replicationMode:"bogusvalue"` — looks applied)

2. `gas.get_gas_info {"assetPath":"/Game/ReplayGAS/BP_ReplayHero"}`
   -> `{...,"abilitySystemComponents":[{"name":"AbilitySystemComponent","source":"scs","replicationMode":"mixed"}]}`
   (the component is still `mixed` — the bogus value was never applied)

## Source confirmation

`Source/PinWright/Private/Handlers/Systems/GASHandler.cpp`,
`gas.configure_asc` handler (~L453-518):

- L504-508: `if (ReplicationModeFromString(ReplicationMode, RepMode)) { ASCTemplate->SetReplicationMode(RepMode); }` — `SetReplicationMode` is called **only** when the string parses; an unrecognized string deliberately leaves the prior mode untouched (the comment at L502-503 says so).
- L514: `Result->SetStringField(TEXT("replicationMode"), ReplicationMode);` — but the result always echoes the **raw input string**, not the mode that was actually applied. So for an unrecognized value the response advertises a mode that was rejected.

`ReplicationModeFromString` (L166-172) returns false for anything other than
`full`/`mixed`/`minimal`.

**Workaround:** never trust `configure_asc`'s echoed `replicationMode`; confirm
the real value with a `gas.get_gas_info` readback (whose ASC `replicationMode`
reflects the actual `ASC->ReplicationMode`).

**Fix (proposed):** when `ReplicationModeFromString` fails, either (a) return a
clean `INVALID_PARAM` error naming the allowed values (`full|mixed|minimal`),
or (b) set the result's `replicationMode` to the *actually-applied* mode
(`ReplicationModeToString(ASCTemplate->ReplicationMode)`) and add an
`applied:false`/note so the response stops misreporting an un-applied input.
Option (a) is preferable — an invalid documented-enum value should not be a
silent success.

## History
- `#1-initial-repro` `OPEN` reporter — Replayed live: `gas.configure_asc` with `replicationMode:"bogusvalue"` returned success echoing `replicationMode:"bogusvalue"`, but the immediate `gas.get_gas_info` readback showed the ASC still at `mixed` (the prior valid value) — the bogus mode was never applied and no error was raised. Valid values (`mixed`, `minimal`) do round-trip correctly, so this is purely the invalid-input path: a misleading echo of an un-applied mode. Source (`GASHandler.cpp` L504-514) confirms `SetReplicationMode` is gated on a successful parse while the result always echoes the raw input string.
- `#2-fix` `IN-REVIEW` developer — Applied Fix option (a): `gas.configure_asc` now validates `replicationMode` before mutating and rejects an unrecognized value with `SendError("INVALID_PARAMS", "Unrecognized replicationMode '<x>'; must be one of full|mixed|minimal.")` instead of silently keeping the prior mode and echoing the bad input back as success. The check is placed BEFORE `MarkBlueprintAsModified` so a rejected call does not dirty the asset; valid values still round-trip and the result echo now only runs on an applied mode. This matches the established house validate-before-mutate / fail-loud convention already used in the same file (e.g. the `INVALID_PARAMS` + droppedTags rejection at GASHandler.cpp:1047, B-set-ability-cooldown-cost-class-path-silent-drop). Source: `Plugins/EditorAutomationRpcGateway/Source/EditorAutomationRpcGateway/Private/Handlers/Systems/GASHandler.cpp` (gas.configure_asc handler, ~L626-643). Regression test: `FGasConfigureAscRejectsInvalidReplicationModeTest` (`EditorAutomationRpcGateway.gas.configure_asc.RejectsInvalidReplicationMode`) in `Private/Tests/Gameplay/TestGASHandlers.cpp` — builds a Character BP, adds an ASC, asserts a valid mode (`mixed`) still succeeds and round-trips while `"bogusvalue"` is rejected with `INVALID_PARAMS` (not a silent success). Reverting the fix makes the bogus-value call succeed with an empty error code, failing the rejection assertions.
- `#3-method-removed-in-audit` `WONTFIX` developer - Closed by removal, not by refusal. `gas.configure_asc` was removed entirely in the batch-2 RPC audit recorded in [`E-rpc-audit-43-record`](E-rpc-audit-43-record.md) (one of the 5 broken-or-destructive removals): the audit found the handler wrote the replication mode to a **non-serialized component template**, so the write never persisted to the asset and there is no property-level fix. That makes this ticket's whole subject moot. The `#2` `INVALID_PARAMS` validation fix and its regression test (`FGasConfigureAscRejectsInvalidReplicationModeTest`, `TestGASHandlers.cpp`) were deleted with the handler, so there is nothing left for a tester to verify and the ticket cannot honestly reach `DONE`. Correcting the record on `#1`'s live evidence: the "valid values round-trip correctly (`mixed`->`mixed` via `gas.get_gas_info`)" observation did not prove persistence. The readback hit the same in-memory template the write had just touched, inside one editor session. Follow-up: the capability is re-filed as [`F-gas-configure-asc-bpir`](F-gas-configure-asc-bpir.md) (emit a `SetReplicationMode` call into the Blueprint graph via the BPIR compiler), which carries this ticket's `INVALID_PARAMS`-on-unrecognized-mode requirement forward so the validation is not re-lost.
- `#4-repoint-citations-after-module-rename` `DONE` reporter — Citation maintenance only; **no claim in this ticket changes and the status is untouched**. The plugin module directory was renamed `Source/EditorAutomationRpcGateway/` → `Source/PinWright/` (plugin commit `8962f163`), and `Source/EditorAutomationRpcGatewayTests/` was folded into `Source/PinWright/Private/Tests/`, so every citation under the old root was an **unresolvable path** a fixer could not open — not a stale line number. 1 body citation repointed in place; every rewritten path was confirmed to exist at plugin HEAD `ef8a1f1b`. 1 citation sits in history rows and is left verbatim per the append-only rule, mapping by the same rule; the mapped path was confirmed present at HEAD too. No citation in this ticket carries a line number, so nothing here required line re-verification. Sweep-wide record, including the cases that could not be repointed: `E-module-rename-citation-sweep`.
